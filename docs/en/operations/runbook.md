# Runbook

Operational procedures for diagnosing and resolving AstraDNS issues.

## Primary Health Checks

Run these checks first when investigating any DNS issue:

```bash
# 1. Are all pods running?
kubectl get pods -n astradns-system

# 2. Are CRDs healthy?
kubectl get dnsupstreampools -n astradns-system
kubectl get dnscacheprofiles -n astradns-system

# 3. Is the ConfigMap populated?
kubectl get configmap astradns-agent-config -n astradns-system \
  -o jsonpath='{.data.config\.json}' | jq .

# 4. Are metrics flowing?
kubectl port-forward -n astradns-system ds/astradns-agent 9153:9153 &
curl -s http://localhost:9153/metrics | grep astradns_queries_total

# 5. Is the ServiceMonitor scraping?
kubectl get servicemonitor -n astradns-system
```

## Common Incidents

### DNS queries fail after deploy

**Symptoms:** Pods get SERVFAIL or timeout resolving external domains.

**Diagnosis:**

```bash
# Check agent health
kubectl logs -n astradns-system -l app.kubernetes.io/component=agent --tail=50

# Check if engine is running
kubectl exec -n astradns-system ds/astradns-agent -- ps aux | grep -E 'unbound|coredns|pdns'

# Check readiness
kubectl get pods -n astradns-system -l app.kubernetes.io/component=agent \
  -o jsonpath='{range .items[*]}{.metadata.name}: {.status.conditions[?(@.type=="Ready")].status}{"\n"}{end}'

# Direct DNS test to agent
kubectl run dns-debug --rm -it --restart=Never \
  --image=busybox:1.37 -- nslookup -port=5353 example.com <agent-pod-ip>
```

**Resolution:**

1. If engine process is not running: check agent logs for startup errors, verify ConfigMap has valid JSON
2. If engine returns SERVFAIL: verify upstream resolvers are reachable from node
3. If agent pod is CrashLooping: check resource limits, verify engine binary exists in image

### CoreDNS was not patched

**Symptoms:** DNS works but queries bypass AstraDNS (no metrics increment).

**Diagnosis:**

```bash
# Check if CoreDNS Corefile has the forward directive
kubectl get configmap coredns -n kube-system -o yaml | grep -A2 forward

# Check the patch job status
kubectl get jobs -n kube-system | grep astradns
kubectl logs -n kube-system job/astradns-coredns-patch
```

**Resolution:**

1. Re-run the patch job: `kubectl delete job astradns-coredns-patch -n kube-system` then `helm upgrade ...`
2. Manual patch: edit the CoreDNS ConfigMap to add `forward . 169.254.20.11`
3. Restart CoreDNS: `kubectl rollout restart deployment coredns -n kube-system`

### Webhook blocks legitimate pool creation

**Symptoms:** `kubectl apply` returns admission error about existing pool.

**Diagnosis:**

```bash
# List existing pools in the namespace
kubectl get dnsupstreampools -n <namespace>

# Check webhook configuration
kubectl get validatingwebhookconfigurations | grep astradns
```

**Resolution:**

The webhook enforces one `DNSUpstreamPool` per namespace. To change configuration:

1. **Update** the existing pool instead of creating a new one
2. **Delete** the old pool first, then create the new one
3. If the webhook is blocking all operations, temporarily disable: `helm upgrade ... --set webhook.enabled=false`

### Config reload fails

**Symptoms:** `astradns_agent_config_reload_errors_total` is increasing.

**Diagnosis:**

```bash
# Check agent logs for reload errors
kubectl logs -n astradns-system -l app.kubernetes.io/component=agent | grep -i reload

# Validate ConfigMap content
kubectl get configmap astradns-agent-config -n astradns-system \
  -o jsonpath='{.data.config\.json}' | python3 -m json.tool
```

**Resolution:**

1. Fix the upstream pool CRD spec (check for invalid addresses/ports)
2. The operator will re-render and update the ConfigMap
3. The agent will detect the change and reload automatically

## Rollback Procedure

### Helm Rollback

```bash
# List revisions
helm history astradns -n astradns-system

# Rollback to previous revision
helm rollback astradns <revision> -n astradns-system
```

### Post-Rollback Verification

```bash
# Verify pods are running
kubectl get pods -n astradns-system

# Verify DNS resolution
kubectl run dns-test --rm -it --restart=Never \
  --image=busybox:1.37 -- nslookup example.com

# Verify metrics are flowing
curl -s http://localhost:9153/metrics | grep astradns_queries_total
```

### Emergency: Disable AstraDNS Without Uninstall

If AstraDNS is causing cluster-wide DNS issues and you need to restore DNS immediately:

```bash
# Revert CoreDNS to original config
kubectl get configmap coredns-backup-astradns -n kube-system \
  -o jsonpath='{.data.Corefile}' > /tmp/original-corefile

kubectl create configmap coredns -n kube-system \
  --from-file=Corefile=/tmp/original-corefile \
  --dry-run=client -o yaml | kubectl apply -f -

kubectl rollout restart deployment coredns -n kube-system
```

This restores CoreDNS to its pre-AstraDNS configuration. AstraDNS pods will continue running but will no longer receive queries.
