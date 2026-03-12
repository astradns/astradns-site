# Selección de Motor

AstraDNS soporta tres motores DNS. El motor se selecciona en el momento del despliegue mediante la variable de entorno `ASTRADNS_ENGINE_TYPE` (o el valor de Helm `agent.engineType`).

## Comparación

| Característica | Unbound | CoreDNS | PowerDNS Recursor |
|----------------|---------|---------|-------------------|
| **Tipo** | Resolver recursivo | Autoritativo + forwarding | Resolver recursivo |
| **Lenguaje** | C | Go | C++ |
| **Huella de memoria** | ~20-50 MB | ~30-60 MB | ~40-80 MB |
| **Rendimiento de caché** | Excelente | Bueno | Bueno |
| **Validación DNSSEC** | Incorporada | Basada en plugins | Incorporada |
| **Recarga de configuración** | `unbound-control reload` | Plugin de auto-reload | `rec_control reload-zones` |
| **Madurez** | 20+ años | 10+ años | 15+ años |
| **Por defecto** | Sí | No | No |

## Cuándo Usar Cada Uno

### Unbound (por defecto)

La mejor opción para la mayoría de los despliegues. Unbound ofrece:

- Caché de primera clase con optimización agresiva de NSEC
- La menor huella de memoria
- La configuración más simple (un solo archivo)
- Probado en batalla en producción durante más de 20 años

```yaml
# values.yaml
agent:
  engineType: unbound
```

### CoreDNS

Elija CoreDNS cuando:

- Su equipo ya opera CoreDNS y desea una sola tecnología DNS
- Necesita plugins personalizados (por ejemplo, descubrimiento de servicios, middleware personalizado)
- Desea integración de observabilidad nativa de Go

```yaml
# values.yaml
agent:
  engineType: coredns
  engineImages:
    coredns: coredns/coredns:1.12.1
```

### PowerDNS Recursor

Elija PowerDNS cuando:

- Necesita scripting Lua para zonas de política de respuesta (RPZ)
- Su organización estandariza en PowerDNS
- Necesita soporte avanzado de EDNS Client Subnet

```yaml
# values.yaml
agent:
  engineType: powerdns
  engineImages:
    powerdns: powerdns/pdns-recursor:5.2
```

## Imágenes de Motor

Por defecto, el contenedor del agent viene con Unbound preinstalado. Para otros motores, debe proporcionar una imagen personalizada o especificar la imagen del motor:

```yaml
agent:
  engineImages:
    coredns: "my-registry/coredns:1.12.1"
    powerdns: "my-registry/pdns-recursor:5.2"
```

!!! warning
    La imagen por defecto del agent solo incluye los binarios de Unbound. Usar `engineType: coredns` o `engineType: powerdns` sin proporcionar la imagen del motor o una imagen personalizada del agent fallará al iniciar.
