# TakeOver

## Información
- Plataforma: TryHackMe
- IP:10.49.180.72
- SO:Linux
- Dificultad:Easy
- Fecha:29/10/26

## Reconocimiento 
└─$ nmap -Pn -sV -oA 02_scans/initial_scan 10.49.180.72
Starting Nmap 7.991 ( https://nmap.org ) at 2026-10-28 19:54 -0400
Nmap scan report for 10.49.180.72
Host is up (0.13s latency).
Not shown: 997 closed tcp ports (reset)
PORT    STATE SERVICE  VERSION
22/tcp  open  ssh      OpenSSH 8.2p1 Ubuntu 4ubuntu0.13 (Ubuntu Linux; protocol 2.0)
80/tcp  open  http     Apache httpd 2.4.41 ((Ubuntu))
443/tcp open  ssl/http Apache httpd 2.4.41 ((Ubuntu))
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel


┌──(n0wak㉿kali)-[~/Desktop/TakeOver]
└─$ curl -I http://10.49.180.72
HTTP/1.1 302 Found
Date: Thu, 29 Oct 2026 00:01:39 GMT
Server: Apache/2.4.41 (Ubuntu)
Location: https://futurevera.thm/
Content-Type: text/html; charset=UTF-8


┌──(n0wak㉿kali)-[~/Desktop/TakeOver]
└─$ curl -k -I https://10.49.180.72
HTTP/1.1 200 OK
Date: Thu, 29 Oct 2026 00:02:40 GMT
Server: Apache/2.4.41 (Ubuntu)
Last-Modified: Sun, 13 Mar 2022 08:48:57 GMT
ETag: "11fd-5da15a2613040"
Accept-Ranges: bytes
Content-Length: 4605
Vary: Accept-Encoding
Content-Type: text/html


┌──(n0wak㉿kali)-[~/Desktop/TakeOver]
└─$ getent hosts futurevera.thm
10.49.180.72   futurevera.thm

└─$ curl -I https://futurevera.thm
curl: (60) SSL certificate OpenSSL verify result: self-signed certificate (18)
More details here: https://curl.se/docs/sslcerts.html

curl failed to verify the legitimacy of the server and therefore could not
establish a secure connection to it. To learn more about this situation and
how to fix it, please visit the webpage mentioned above.

└─$ curl -k -I https://futurevera.thm
HTTP/1.1 200 OK
Date: Thu, 29 Oct 2026 00:10:23 GMT
Server: Apache/2.4.41 (Ubuntu)
Last-Modified: Sun, 13 Mar 2022 08:48:57 GMT
ETag: "11fd-5da15a2613040"
Accept-Ranges: bytes
Content-Length: 4605
Vary: Accept-Encoding
Content-Type: text/html


## Enumeración
ffuf -w /usr/share/seclists/Discovery/DNS/subdomains-top1million-5000.txt -u https://10.49.180.72/ -H "Host: FUZZ.futurevera.thm" -fs 4605 -k -of json -o 01_recon/vhosts.json
 
Qué hace cada opción:

    -w: usa una lista de posibles nombres.

    FUZZ: sustituye esa palabra por cada candidato.

    -H "Host: FUZZ.futurevera.thm": prueba distintos virtual hosts en el servidor.

    -fs 4605: filtra las respuestas de 4605 bytes, que es el tamaño que observaste en la página principal. Si aparecen resultados inesperados, revisaremos este filtro.

    -k: permite continuar aunque el certificado HTTPS no sea de confianza.

    -of json -o ...: guarda los resultados en JSON.

Enumeración de virtual hosts con ffuf
Objetivo: 10.49.180.72
Técnica: fuzzing del encabezado HTTP Host
Wordlist: subdomains-top1million-5000.txt
Filtro: -fs 4605
Resultado: dos respuestas distintas de la página por defecto.
Host descubierto	Estado HTTP	Tamaño
blog.futurevera.thm	200	3838 bytes
support.futurevera.thm	200	1522 bytes

Conclusión: ambos nombres parecen corresponder a virtual hosts distintos. Se requiere inspección adicional para determinar su contenido y relevancia.

┌──(n0wak㉿kali)-[~/Desktop/TakeOver]
└─$ curl -k -sS --max-time 15   --resolve blog.futurevera.thm:443:10.49.190.211   https://support.futurevera.thm/   -o 05_evidence/support.html
curl: (6) Could not resolve host: support.futurevera.thm

┌──(n0wak㉿kali)-[~/Desktop/TakeOver]
└─$ curl -k -sS --max-time 15   --resolve support.futurevera.thm:443:10.49.190.211   https://support.futurevera.thm/   -o 05_evidence/support.html

┌──(n0wak㉿kali)-[~/Desktop/TakeOver]
└─$ wc -c 05_evidence/blog.html 05_evidence/support.html
3838 05_evidence/blog.html
1522 05_evidence/support.html
5360 total

┌──(n0wak㉿kali)-[~/Desktop/TakeOver]
└─$ grep -Ein 'futurevera|href=|src=|http[s]?://' \
  05_evidence/blog.html 05_evidence/support.html
05_evidence/blog.html:8:        <title>FutureVera - Blogs</title>
05_evidence/blog.html:9:        <link rel="icon" type="image/x-icon" href="assets/favicon.ico" />
05_evidence/blog.html:10:        <link href="css/styles.css" rel="stylesheet" />
05_evidence/blog.html:16:                <a class="navbar-brand" href="index.html">FutureVera</a>
05_evidence/blog.html:23:                        <li class="nav-item"><a class="nav-link px-lg-3 py-3 py-lg-4" href="index.html">Home</a></li>
05_evidence/blog.html:24:                        <li class="nav-item"><a class="nav-link px-lg-3 py-3 py-lg-4" href="about.html">About</a></li>
05_evidence/blog.html:25:                        <li class="nav-item"><a class="nav-link px-lg-3 py-3 py-lg-4" href="post.html">Post</a></li>
05_evidence/blog.html:36:                            <h1>FutureVera Blog</h1>
05_evidence/blog.html:37:                            <span class="subheading">Blogging website for futurevera.thm</span>
05_evidence/blog.html:49:                        <a href="post.html">
05_evidence/blog.html:55:                            <a href="#!">FutureVera</a>
05_evidence/blog.html:69:                        <div class="small text-center text-muted fst-italic">Copyright &copy;futurevera.thm</div>
05_evidence/blog.html:74:       <!-- Theme is taken from https://startbootstrap.com -->
05_evidence/blog.html:76:        <script src="js/bootstrap.bundle.min.js"></script>
05_evidence/blog.html:78:        <script src="js/scripts.js"></script>
05_evidence/support.html:8:        <title>FutureVera - Support</title>
05_evidence/support.html:9:        <link rel="icon" type="image/x-icon" href="assets/favicon.ico" />
05_evidence/support.html:10:        <link href="css/styles.css" rel="stylesheet" />
05_evidence/support.html:14:        <video class="bg-video" playsinline="playsinline" autoplay="autoplay" muted="muted" loop="loop"><source src="assets/mp4/bg.mp4" type="video/mp4" /></video>
05_evidence/support.html:23:                    <div class="small text-center text-muted fst-italic">Copyright &copy;futurevera.thm</div>
05_evidence/support.html:27:    <!-- Theme is taken from https://startbootstrap.com -->
05_evidence/support.html:29:        <script src="js/bootstrap.bundle.min.js"></script>
05_evidence/support.html:31:        <script src="js/scripts.js"></script>

NADA RARO, entonces 

┌──(n0wak㉿kali)-[~/Desktop/TakeOver]
└─$ timeout 10 openssl s_client \
  -connect 10.49.190.211:443 \
  -servername blog.futurevera.thm </dev/null 2>/dev/null |
  openssl x509 -noout -subject -issuer -dates -ext subjectAltName |
  tee 05_evidence/blog_certificate.txt                                                                                            
No extensions in certificate                                                                                                      
subject=C=US, ST=Oregon, L=Portland, O=Futurevera, OU=Thm, CN=blog.futurevera.thm                                                 
issuer=C=US, ST=Oregon, L=Portland, O=Futurevera, OU=Thm, CN=blog.futurevera.thm                                                  
notBefore=Mar 13 10:22:57 2022 GMT                                                                                                
notAfter=Mar 13 10:22:57 2023 GMT                                                                                                 
                                                                                                                                  
┌──(n0wak㉿kali)-[~/Desktop/TakeOver]
└─$ timeout 10 openssl s_client \                                                                                                 
  -connect 10.49.190.211:443 \
  -servername support.futurevera.thm </dev/null 2>/dev/null |
  openssl x509 -noout -subject -issuer -dates -ext subjectAltName |
  tee 05_evidence/support_certificate.txt
subject=C=US, ST=Oregon, L=Portland, O=Futurevera, OU=Thm, CN=support.futurevera.thm
issuer=C=US, ST=Oregon, L=Portland, O=Futurevera, OU=Thm, CN=support.futurevera.thm
notBefore=Mar 13 14:26:24 2022 GMT
notAfter=Mar 12 14:26:24 2024 GMT
X509v3 Subject Alternative Name: 
    DNS:secrethelpdesk934752.support.futurevera.thm

Hallazgo: nombre DNS alternativo en certificado TLS
Host analizado: support.futurevera.thm
IP del laboratorio: 10.49.190.211
Herramienta: openssl s_client y openssl x509
Campo: X509v3 Subject Alternative Name (SAN)
Nombre descubierto: secrethelpdesk934752.support.futurevera.thm
Validez del certificado: 13/03/2022 – 12/03/2024
Observación: el certificado incluye un nombre DNS adicional que no habíamos identificado mediante el HTML de las páginas.

Evidencia: 05_evidence/support_certificate.txt

Conclusión provisional: se ha identificado un nuevo hostname a partir del certificado TLS. Se debe comprobar si corresponde a un servicio accesible.

┌──(n0wak㉿kali)-[~/Desktop/TakeOver/05_evidence]
└─$ getent hosts secrethelpdesk934752.support.futurevera.thm

┌──(n0wak㉿kali)-[~/Desktop/TakeOver/05_evidence]
└─$ curl -k -I --connect-timeout 5 --max-time 15 \
  --resolve secrethelpdesk934752.support.futurevera.thm:443:10.49.190.211 \
  https://secrethelpdesk934752.support.futurevera.thm/
HTTP/1.1 200 OK
Date: Thu, 29 Oct 2026 00:53:50 GMT
Server: Apache/2.4.41 (Ubuntu)
Last-Modified: Sun, 13 Mar 2022 08:48:57 GMT
ETag: "11fd-5da15a2613040"
Accept-Ranges: bytes
Content-Length: 4605
Vary: Accept-Encoding
Content-Type: text/html


ANALIZAR CERT
└─$ timeout 10 openssl s_client \
  -connect 10.49.190.211:443 \
  -servername secrethelpdesk934752.support.futurevera.thm \
  </dev/null 2>/dev/null |
  openssl x509 -noout -subject -issuer -dates -ext subjectAltName
subject=C=US, ST=Oregon, L=Portland, O=Futurevera, OU=Thm, CN=futurevera.thm
issuer=C=US, ST=Oregon, L=Portland, O=Futurevera, OU=Thm, CN=futurevera.thm
notBefore=Mar 13 10:05:19 2022 GMT
notAfter=Mar 13 10:05:19 2023 GMT
No extensions in certificate

NO HUBO NADA RELEVANTE POR HTTPS, entonces ahora por puerto 80

┌──(n0wak㉿kali)-[~/Desktop/TakeOver/05_evidence]
└─$ curl -i --connect-timeout 5 --max-time 15 \
  --resolve secrethelpdesk934752.support.futurevera.thm:80:10.49.190.211 \
  http://secrethelpdesk934752.support.futurevera.thm/
HTTP/1.1 302 Found
Date: Thu, 29 Oct 2026 01:00:28 GMT
Server: Apache/2.4.41 (Ubuntu)
Location: http://flag{beea0d6edfcee06a59b83fb50ae81b2f}.s3-website-us-west-3.amazonaws.com/
Content-Length: 0
Content-Type: text/html; charset=UTF-8
