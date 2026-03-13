# Proceso de Lanzamiento

## Versionado

AstraDNS sigue [Semantic Versioning](https://semver.org/): `vX.Y.Z`

- **Major (X):** Cambios incompatibles en la API
- **Minor (Y):** Nuevas funcionalidades, compatibles hacia atrás
- **Patch (Z):** Correcciones de errores

## Lista de Verificación Pre-Lanzamiento

```bash
# 1. Run all tests
make test

# 2. Run linter
make lint

# 3. Run e2e tests
make test-e2e

# 4. Run integration tests (multi-K8s version)
make test-integration-matrix

# 5. Validate SLOs
make test-slo

# 6. Validate release artifacts
make release-check
```

## Pasos del Lanzamiento

### 1. Actualizar la Versión

Actualice la versión y appVersion en `Chart.yaml`:

```yaml
version: 0.2.5
appVersion: "0.2.5"
```

### 2. Etiquetar el Lanzamiento

```bash
git tag -a v0.2.5 -m "Release v0.2.5"
git push origin v0.2.5
```

### 3. Pipeline Automatizado

El push de la etiqueta activa el workflow `release-package.yml` que:

1. Ejecuta `make release-check`
2. Ejecuta `make test` y `make lint`
3. Empaqueta el chart de Helm (`make package-chart`)
4. Sube el artefacto del chart

### 4. Crear el Release en GitHub

Cree un release en GitHub a partir de la etiqueta con:

- Changelog resumiendo los cambios desde el último lanzamiento
- Enlace al artefacto del chart de Helm
- Instrucciones de actualización (si las hay)
- Problemas conocidos (si los hay)

## Empaquetado del Chart de Helm

```bash
# Package the chart
make package-chart

# Output: dist/astradns-0.2.5.tgz
```

## Reversión

Si un lanzamiento necesita ser revertido:

```bash
# Helm rollback
helm rollback astradns <previous-revision> -n astradns-system

# Verify
kubectl get pods -n astradns-system
kubectl run dns-test --rm -it --restart=Never \
  --image=busybox:1.37 -- nslookup example.com
```
