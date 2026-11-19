# Lab local de DNS con CoreDNS, equivalente al codelab de Cloud DNS ResourceRecordSets

Reproducción en local del flujo del codelab de Google Cloud "Cloud DNS ResourceRecordSets". Usé CoreDNS en Docker en lugar de la API de Cloud DNS porque no pude activar la facturación de GCP. El flujo es el mismo: crear una zona, crear registros A y CNAME, consultarlos, corregirlos y limpiarlos.

## Qué hace

- Sirve la zona `my-domain.com` desde un archivo de zona con CoreDNS.
- Resuelve `my-domain.com` a `1.2.3.4` con un registro A.
- Resuelve `www.my-domain.com` como alias (CNAME) de `my-domain.com`.
- Se consulta con `dig` desde un contenedor que comparte la red de CoreDNS.

## Equivalencia con el codelab

| Codelab (Cloud DNS) | Este lab |
|---|---|
| `managed-zones create` | `Corefile` + `my-domain.com.zone` |
| `record-sets create` | Línea nueva en el archivo de zona |
| `record-sets update` | Editar la línea y subir el serial |
| `record-sets delete` | Borrar la línea |
| `dig` desde la VM por SSH | `dig` contra CoreDNS en Docker |

## Cómo reproducirlo (Windows, PowerShell, Docker Desktop)

```powershell
docker run -d --name coredns -p 1053:53/udp -v "${PWD}:/root" coredns/coredns -conf /root/Corefile

function digq($n, $t="A") { docker run --rm --network container:coredns alpine sh -c "apk add -q bind-tools && dig @127.0.0.1 $n $t" }

digq my-domain.com
digq www.my-domain.com
```

Tras cada cambio en el archivo de zona, sube el serial del SOA y ejecuta `docker restart coredns`.

Para limpiar: `docker rm -f coredns`.

## Qué significa cada línea de la zona

| Línea | Significado |
|---|---|
| `$ORIGIN my-domain.com.` | Dominio base. Los nombres sin punto final se completan con él |
| `@ ... SOA ...` | Ficha de la zona: servidor principal, contacto, serial y tiempos de caché |
| `@ ... NS ns.my-domain.com.` | Servidor de nombres de la zona |
| `ns ... A 127.0.0.1` | IP del servidor de nombres |
| `@ 60 IN A 1.2.3.4` | `my-domain.com` apunta a esa IPv4, con TTL de 60 segundos |
| `www 60 IN CNAME my-domain.com.` | `www` es un alias de `my-domain.com` |

## Experimentos propios

- [Qué pasó con dos registros A para el mismo nombre, y qué devuelve `dig`.]
- [Qué observé del TTL en un resolvedor público: cómo baja y qué pasa al llegar a 0.]

Las salidas de `dig` están en `evidencia/`.

## Qué aprendí

- [Diferencia entre un registro A y un CNAME, y por qué se usa un alias para `www`.]
- [Qué es el TTL y qué implica tenerlo alto o bajo durante un cambio.]
- [Diferencia entre NXDOMAIN y una respuesta NOERROR sin registros.]
- [Por qué se sube el serial del SOA en cada cambio.]
