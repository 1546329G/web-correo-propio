# Estado y pendientes al 13 de noviembre de 2026

## Confirmado directamente por pruebas compartidas

- `dig @1.1.1.1` y `@8.8.8.8` muestran MX único `10 mail.gandywilliam.dev.`.
- SPF `v=spf1 mx -all`, DMARC `p=reject` y dos DKIM publicados.
- PTR IPv4 apunta a `mail.gandywilliam.dev`; TLS de SMTP/IMAP válido.
- Autenticación IMAP del buzón `contacto@gandywilliam.dev`.
- Correo entrante Gmail → Stalwart localizado con `* SEARCH 2`.
- Correo saliente Stalwart → Gmail recibido **en Spam**.
- Cloudflare añadió correctamente dos rutas webmail/JMAP.
- `correo.gandywilliam.dev` respondió HTTP 200.
- `jmap.gandywilliam.dev/.well-known/jmap` GET redirige a `/jmap/session`, HTTP 200.
- Desde Gandy Mail se envió «hola querido» con archivo adjunto; Gmail lo recibió **en Spam**.

## Reportado por Claude; no auditado otra vez por SSH

- Bulwark 1.21.2 compilado desde código fuente en `/home/gandy/apps/bulwark-webmail`.
- Dos contenedores `bulwark-webmail` y `jmap-gateway`, puertos loopback 3100/3101; volumen `bulwark-data`, proyecto Compose `gandy-mail`.
- Servicios del gimnasio y Stalwart permanecieron funcionando; backups iniciales de Bulwark.
- Problema grave de contraseña SSH/sudo reutilizada en contenido público. Rotación aún no acreditada.

## Pendientes priorizados

| Prioridad | Trabajo | Criterio de cierre |
|---|---|---|
| Crítica | Rotar contraseña SSH, retirar exposición del gimnasio | Login nuevo comprobado; exposición retirada y contraseñas relacionadas rotadas |
| Alta | Diagnosticar Gmail Spam | Obtener «Mostrar original» con SPF, DKIM, DMARC + alineación, investigar IP/reputación |
| Alta | Auditar gateway JMAP y cabeceras proxy | Rutas, login, CORS, rate limit, IP cliente verificados sin exponer Admin |
| Alta | Comprobar alias `postmaster` | Entrega real de reportes DMARC/TLSRPT |
| Media | Backups consistentes cifrados y automatizados | Restauración ensayada y monitoreo |
| Media | Renovación Let's Encrypt antes de febrero 2027 | Nuevo certificado servido correctamente en 465/993 |
| Media | Pruebas webmail integrales | Leer/responder, adjuntos, carpetas, búsqueda, múltiples usuarios |
| Media | Auditar y versionar ajustes del deployment Claude | Examinar y documentar configs y scripts locales saneados |
| Media | Confirmar MTA-STS/TLSRPT | Política HTTPS y recepción de informes verificada |
| Baja | Configurar claves SSH | Login por clave probado antes de cambiar sshd |

## Consideraciones

No confundir correo aceptado por Stalwart con entrega al destinatario, ni HTTP 200 con todas las funciones JMAP validadas. Los registros TXT por sí mismos no prueban resultados SPF/DKIM/DMARC de un mensaje. Este repositorio público no debe usarse para volúmenes, archivos privados, backups ni secretos.


## Cierre del hito: envío y recepción externos — 13/11/2026

**Estado: CUMPLIDO (funcionalidad de envío y recepción).** Además de Gmail, una captura de Outlook Web demuestra que Microsoft recibió el mensaje «hola» desde `contacto@gandywilliam.dev`, a las 13:52 hora Perú. Apareció en **Correo no deseado**, igual que varias pruebas en Gmail aparecieron en Spam. Esto valida **entrega técnica a Gmail y Outlook**, no buena clasificación de bandeja principal.

La investigación antispam queda como **mejora pendiente**, separada del hito de funcionalidad. Ver [diagnóstico](12-entregabilidad-gmail.md).
