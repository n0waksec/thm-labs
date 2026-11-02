# TakeOver

## 1. Información del objetivo

- **Plataforma:** TryHackMe
- **Sistema operativo:** Linux (identificación preliminar)
- **Dificultad:** Easy
- **IP inicial:** `10.49.180.72`
- **Estado:** Finalizado

## 2. Resumen ejecutivo

El reconocimiento inicial identificó los puertos SSH, HTTP y HTTPS. La enumeración web permitió descubrir dos virtual hosts: `blog.futurevera.thm` y `support.futurevera.thm`.

La inspección del certificado TLS de `support.futurevera.thm` reveló un hostname adicional mediante el campo Subject Alternative Name (SAN). La comprobación posterior por HTTP devolvió una redirección hacia un dominio de Amazon S3 que contiene una cadena con formato de flag.

**Resultado confirmado:** descubrimiento de virtual hosts, hostname adicional y redirección HTTP relevante.

## 3. Reconocimiento

### 3.1 Escaneo de puertos

**Objetivo:** identificar puertos TCP abiertos y versiones de servicios.

```bash
nmap -Pn -sV -oA 02_scans/initial_scan 10.49.180.72
```

**Resultados:**

| Puerto | Servicio | Versión |
|---|---|---|
| 22/tcp | SSH | OpenSSH 8.2p1 |
| 80/tcp | HTTP | Apache 2.4.41 |
| 443/tcp | HTTPS | Apache 2.4.41 |

**Interpretación:** los servicios web constituyen la principal superficie de enumeración inicial. SSH queda identificado para una posible investigación posterior.

**Evidencia:** salida de Nmap en formato normal, XML y grepable, generada mediante `-oA`.

### 3.2 Comportamiento HTTP/HTTPS

La petición HTTP a la IP devolvió un código `302 Found` y redirigió a `https://futurevera.thm/`.

La resolución local confirmó que `futurevera.thm` apuntaba a la IP inicial.

La conexión HTTPS funcionó con `curl -k`. Sin esa opción, curl rechazó el certificado por no ser de confianza.

**Interpretación:** el servidor utiliza un nombre de host específico y un certificado que no supera la validación de confianza del cliente. Esto justifica continuar con la enumeración de virtual hosts y certificados.

## 4. Enumeración web

### 4.1 Descubrimiento de virtual hosts

```bash
ffuf -w /usr/share/seclists/Discovery/DNS/subdomains-top1million-5000.txt \
  -u https://10.49.180.72/ \
  -H "Host: FUZZ.futurevera.thm" \
  -fs 4605 -k -of json -o 01_recon/vhosts.json
```

**Resultados:**

| Hostname | Código HTTP | Tamaño |
|---|---:|---:|
| `blog.futurevera.thm` | 200 | 3838 bytes |
| `support.futurevera.thm` | 200 | 1522 bytes |

El filtro `-fs 4605` excluyó las respuestas del tamaño observado en la página predeterminada.

**Interpretación:** ambos nombres devolvieron respuestas diferentes de la página por defecto. Se descargaron y examinaron sus contenidos HTML.

### 4.2 Inspección del contenido

Se revisaron títulos, enlaces, recursos y referencias a dominios mediante `grep`.

- `blog.futurevera.thm`: página identificada como FutureVera Blog.
- `support.futurevera.thm`: página identificada como FutureVera Support.
- En el HTML revisado no se identificó el hostname que posteriormente apareció en el certificado.

**Conclusión:** la inspección del HTML no proporcionó el siguiente hallazgo; se amplió la enumeración a los certificados TLS.

## 5. Enumeración TLS

### 5.1 Inspección del certificado de Support

Se consultaron el emisor, el sujeto, las fechas de validez y las extensiones del certificado.

**Hallazgo:** el certificado de `support.futurevera.thm` contiene el siguiente nombre en el campo SAN:

`secrethelpdesk934752.support.futurevera.thm`

**Evidencia:** `05_evidence/support_certificate.txt`

**Interpretación:** el certificado revela un hostname adicional que merece validación. Su presencia en el certificado no demuestra por sí sola que exista un registro DNS resoluble ni que el servicio sea vulnerable.

### 5.2 Validación del hostname

La resolución mediante `getent hosts` no devolvió una dirección para el hostname descubierto.

Se utilizó `curl --resolve` para enviar la petición HTTPS directamente a la IP del laboratorio, conservando el hostname solicitado.

**Resultado:** respuesta `200 OK`.

La respuesta tuvo un tamaño de 4605 bytes, igual al de la página predeterminada observada previamente.

**Interpretación:** el servicio respondió, pero el contenido obtenido no permitió confirmar un hallazgo adicional por HTTPS.

## 6. Hallazgo por HTTP

Se probó el mismo hostname por el puerto 80.

**Resultado:** respuesta `302 Found`.

El encabezado `Location` redirigió a un dominio de Amazon S3 cuyo subdominio contenía una cadena con formato de flag.

**Evidencia:** salida de la petición HTTP.

**Conclusión:** esta redirección constituye el hallazgo relevante de la enumeración. La cadena identificada debe registrarse como evidencia del laboratorio y distinguirse de una flag obtenida mediante un mecanismo de validación independiente, si este fuera necesario.

## 7. Vulnerabilidades y hallazgos

### H-01 — Exposición de hostname en certificado TLS

- **Activo:** `support.futurevera.thm`
- **Técnica:** inspección de la extensión SAN del certificado.
- **Evidencia:** `support_certificate.txt`
- **Resultado:** hostname adicional descubierto.
- **Impacto:** aporta información útil para ampliar la superficie de enumeración.
- **Estado:** hallazgo de reconocimiento; no demuestra por sí solo una vulnerabilidad explotable.

### H-02 — Redirección HTTP hacia dominio externo con cadena de flag

- **Activo:** `secrethelpdesk934752.support.futurevera.thm`
- **Técnica:** petición HTTP y análisis del encabezado `Location`.
- **Resultado:** redirección `302 Found` a un dominio de Amazon S3.
- **Evidencia:** salida de curl.
- **Estado:** hallazgo observado; registrar la validación final según los criterios del laboratorio.

## 8. Explotación

N/A

## 9. Escalada de privilegios

N/A

## 10. Evidencias y flags

- Escaneo inicial de Nmap.
- Resultados JSON de ffuf.
- HTML de los virtual hosts.
- Certificados TLS extraídos.
- Respuesta HTTP y encabezado `Location`.
- Flag observada, registrada sin modificar.

## 11. Resolución paso a paso

1. Escanear los puertos TCP.
2. Identificar la redirección HTTP y el hostname principal.
3. Enumerar virtual hosts mediante el encabezado `Host`.
4. Descargar e inspeccionar las páginas descubiertas.
5. Examinar los certificados TLS.
6. Extraer el hostname adicional del campo SAN.
7. Validar el hostname mediante `curl --resolve`.
8. Probar HTTP y examinar la redirección.
9. Registrar la evidencia y verificar la finalización del laboratorio.

## 12. Aprendizajes

- Los virtual hosts pueden descubrirse modificando el encabezado HTTP `Host`.
- Los certificados TLS pueden revelar nombres adicionales en el campo SAN.
- `curl --resolve` permite probar un hostname contra una IP concreta.
- Una respuesta HTTP `200 OK` no implica que el contenido sea relevante.
- Las redirecciones deben analizarse mediante sus encabezados, especialmente `Location`.
- Un hallazgo de reconocimiento no debe clasificarse automáticamente como una vulnerabilidad explotable.

