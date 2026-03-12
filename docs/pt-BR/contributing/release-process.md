# Processo de Release

## Versionamento

O AstraDNS segue o [Versionamento Semantico](https://semver.org/): `vX.Y.Z`

- **Major (X):** Alteracoes incompativeis na API
- **Minor (Y):** Novos recursos, retrocompativeis
- **Patch (Z):** Correcoes de bugs

## Checklist Pre-Release

```bash
# 1. Execute todos os testes
make test

# 2. Execute o linter
make lint

# 3. Execute os testes e2e
make test-e2e

# 4. Execute os testes de integracao (multiplas versoes do K8s)
make test-integration-matrix

# 5. Valide os SLOs
make test-slo

# 6. Valide os artefatos de release
make release-check
```

## Passos do Release

### 1. Atualizar a Versao

Atualize a versao e o appVersion no `Chart.yaml`:

```yaml
version: 0.2.0
appVersion: "0.2.0"
```

### 2. Criar a Tag do Release

```bash
git tag -a v0.2.0 -m "Release v0.2.0"
git push origin v0.2.0
```

### 3. Pipeline Automatizado

O push da tag aciona o workflow `release-package.yml` que:

1. Executa `make release-check`
2. Executa `make test` e `make lint`
3. Empacota o chart Helm (`make package-chart`)
4. Faz upload do artefato do chart

### 4. Criar Release no GitHub

Crie um release no GitHub a partir da tag com:

- Changelog resumindo as alteracoes desde o ultimo release
- Link para o artefato do chart Helm
- Instrucoes de atualizacao (se houver)
- Problemas conhecidos (se houver)

## Empacotamento do Chart Helm

```bash
# Empacotar o chart
make package-chart

# Saida: dist/astradns-0.2.0.tgz
```

## Rollback

Se um release precisar ser revertido:

```bash
# Rollback via Helm
helm rollback astradns <previous-revision> -n astradns-system

# Verificar
kubectl get pods -n astradns-system
kubectl run dns-test --rm -it --restart=Never \
  --image=busybox:1.37 -- nslookup example.com
```
