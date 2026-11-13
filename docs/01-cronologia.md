# Cronología del correo propio — 12 y 13 de noviembre de 2026

> Documento reconstruido a partir de la conversación, las salidas de terminal y el informe de Claude. «Verificado» es evidencia compartida; «informado» es trabajo descrito por Claude y no inspeccionado de nuevo por SSH. Horario: America/Lima.

## 12 de noviembre: Stalwart y Contabo

1. **Arquitectura:** se eligió Stalwart 1.1.0 en Docker, sobre el VPS Contabo `vmi3724729`, Ubuntu 24.04, 4 vCPU/8 GB aprox., IP pública `13.140.207.155`. Se mantuvo el frontend del gimnasio en 80/443, sin tocar esos puertos.
2. **Cloudflare:** se conservó el DNS autoritativo en Cloudflare. Se instaló `cloudflared` desde repositorios oficiales en el VPS, se registró como servicio systemd y se configuró el túnel `stalwart-admin-g`.
3. **Panel:** `admin.gandywilliam.dev` apunta al túnel → `http://127.0.0.1:8080`. Inicialmente se detectó error 525 por ruta/proxy DNS incorrecto y después «Failed to fetch» durante login, debido a redirecciones de Access y sesión con duración cero. Se configuró Access con política individual y duración de 24 h; el panel terminó funcionando.
4. **Stalwart:** instalación con volúmenes Docker `stalwart-etc` → `/etc/stalwart` y `stalwart-data` → `/var/lib/stalwart`; wizard de cinco pasos: hostname `mail.gandywilliam.dev`, dominio `gandywilliam.dev`, RocksDB, directorio interno, logs en consola y Cloudflare como proveedor DNS. Administración `admin@gandywilliam.dev`. Se resolvió una etapa de recovery y se volvió a modo normal, con `STALWART_PUBLIC_URL=https://admin.gandywilliam.dev`.
5. **ACME:** primer intento falló por falta de correo de contacto. Se corrigió el proveedor Let's Encrypt, seleccionando **DNS-01** en lugar de TLS-ALPN-01 porque 443 estaba ocupado. Se emitió certificado wildcard/apex válido 12/11/2026–10/02/2027, observado en TLS SMTP 465 e IMAP 993.
6. **Puertos:** Stalwart tenía listeners, pero la imagen no había publicado externamente 25/465/993. Se inspeccionaron montajes, se detuvo y renombró el contenedor a `stalwart-anterior` (detenido), y se recreó `stalwart` conservando volúmenes y versión, con SMTP 25/465 e IMAP 993 públicos; administración 8080 y HTTPS 8443 solo en loopback. Se confirmó `healthy`.
7. **PTR:** en Contabo → Network → DNS Management → Reverse DNS se cambió PTR IPv4 de `vmi3724729.contaboserver.net` a `mail.gandywilliam.dev`. Verificado después en 1.1.1.1 y 9.9.9.9. IPv6 PTR no se modificó.
8. **Arch Linux:** se instalaron herramientas `dig` (paquete `bind`) y se diagnosticó teclado/sudo local. El MX todavía apuntaba a Cloudflare Email Routing.

## 13 de noviembre: MX, correo real y Gandy Mail

9. Se creó/probó la cuenta ordinaria `contacto@gandywilliam.dev`, distinta del administrador. Login IMAP correcto; SMTP 465 aceptó mensaje; IMAP encontró prueba local `* SEARCH 1`.
10. Una prueba desde Arch a TCP 25 agotó tiempo (exit 124), pero MxToolbox desde otra red confirmó banner `220 mail.gandywilliam.dev Stalwart ESMTP`, STARTTLS y rechazo correcto de relay anónimo. El VPS mostró conectividad SMTP saliente a MX Gmail.
11. **MX:** los tres registros Cloudflare Email Routing `route1/2/3.mx.cloudflare.net` eran bloqueados por el servicio. Se desactivó Email Routing mediante Settings → Disable → confirmar dominio. Luego se creó MX `@ → mail.gandywilliam.dev`, prioridad 10, TTL Auto. `dig @1.1.1.1` y `@8.8.8.8` devolvieron solo `10 mail.gandywilliam.dev.`. SPF y dos DKIM quedaron publicados.
12. **Gmail→Stalwart:** mensaje «Prueba real desde Gmail» enviado a contacto; IMAP SEARCH devolvió `* SEARCH 2`.
13. **Stalwart→Gmail:** envío autenticado con `curl` por SMTPS 465, terminal mostró «SMTP: MENSAJE ACEPTADO» y Gmail recibió el correo en Spam.
14. Se decidió instalar un webmail de código abierto editable, inspirado en la experiencia Gmail/Outlook; seleccionado **Bulwark**, repo https://github.com/bulwarkmail/webmail.
15. **[Informado por Claude]** desplegó Bulwark 1.21.2 (upstream `933bc18f5`), código editable en `/home/gandy/apps/bulwark-webmail`, imagen `gandy/bulwark-webmail:1.21.2-933bc18f5-b1`, contenedor `bulwark-webmail` (127.0.0.1:3100), gateway nginx JMAP (127.0.0.1:3101), volumen `bulwark-data` y Compose independiente `deploy/gandy`. Informó ajuste de memoria Node durante build, y que no tocó el gimnasio ni Stalwart.
16. **Cloudflare Tunnel:** se agregaron rutas `correo.gandywilliam.dev → http://localhost:3100` y `jmap.gandywilliam.dev → http://localhost:3101` en `stalwart-admin-g`. Cloudflare confirmó rutas y DNS CNAME. Inicialmente se introdujo URL inválida `http/localhost:3100`; se corrigió a `http://localhost:3100`.
17. **Webmail/JMAP:** `curl` dio HTTP 200 en `correo.gandywilliam.dev`; JMAP GET `/.well-known/jmap` redirigió 307 y terminó HTTP 200 en `/jmap/session`. HEAD devolvió 404 (método distinto). En la interfaz Gandy Mail se envió «hola querido» con adjunto; Gmail lo recibió a las 12:02 p. m. **en Spam**.
18. **Pendiente:** diagnosticar resultados SPF/DKIM/DMARC del mensaje real y la clasificación en Spam. Claude reportó un problema urgente: contraseña SSH/sudo coincidía con texto público del sitio del gimnasio; rotación todavía sin confirmar.

**Nota:** los datos del 12/11 son históricos; no asumir que sus MX antiguos son el estado final del 13/11.
