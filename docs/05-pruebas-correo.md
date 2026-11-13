# Pruebas de correo: DNS, SMTP, IMAP, JMAP y Gmail

## Configuración de clientes

| Protocolo | Servidor | Puerto | Seguridad |
|---|---|---|---|
| IMAP | `mail.gandywilliam.dev` | 993 | SSL/TLS |
| SMTP submission | `mail.gandywilliam.dev` | 465 | SSL/TLS, autenticación |
| SMTP entre MTAs | `mail.gandywilliam.dev` | 25 | STARTTLS si disponible |
| Webmail | `https://correo.gandywilliam.dev` | 443 | HTTPS |

Buzón normal `contacto@gandywilliam.dev`; cuenta administrativa separada `admin@gandywilliam.dev`. El alias `postmaster@` no se verificó, aunque DMARC/TLSRPT lo referencian.

## 1. DNS MX y propagación — VERIFICADO

```bash
dig @1.1.1.1 +short MX gandywilliam.dev
dig @8.8.8.8 +short MX gandywilliam.dev
```

Ambas consultas devolvieron:

```text
10 mail.gandywilliam.dev.
```

## 2. Login IMAP — VERIFICADO

```bash
curl --silent --show-error --max-time 15 \
  --user 'contacto@gandywilliam.dev' \
  --output /dev/null \
  'imaps://mail.gandywilliam.dev/INBOX'
```

`curl` solicitó contraseña sin mostrarla en el comando. Login correcto. Evitar pegar la contraseña.

## 3. Entrega local — VERIFICADO

Se creó un mensaje de prueba y se envió mediante SMTPS `465` desde el buzón a sí mismo; Stalwart aceptó el mensaje. Verificación de su almacenamiento:

```bash
curl --silent --show-error --max-time 15 \
  --user 'contacto@gandywilliam.dev' \
  --request 'SEARCH SUBJECT "Primera prueba Stalwart"' \
  'imaps://mail.gandywilliam.dev/INBOX'
# * SEARCH 1
```

Un intento previo usando consulta IMAP como parámetro `?SUBJECT` falló con `curl: (21) Quote command returned error`; se corrigió con `--request 'SEARCH ...'`.

## 4. Gmail → Stalwart — VERIFICADO

Se mandó un correo real desde Gmail a `contacto@gandywilliam.dev`, asunto «Prueba real desde Gmail». Después:

```bash
curl --silent --show-error --max-time 20 \
  --user 'contacto@gandywilliam.dev' \
  --request 'SEARCH SUBJECT "Prueba real desde Gmail"' \
  'imaps://mail.gandywilliam.dev/INBOX'
# * SEARCH 2
```

## 5. Stalwart → Gmail — VERIFICADO

El cliente `curl` hizo `--url 'smtps://mail.gandywilliam.dev:465'`, `--user 'contacto@gandywilliam.dev'`, `--mail-from`, `--mail-rcpt` y `--upload-file /tmp/stalwart-gmail.eml`. Respuesta: `SMTP: MENSAJE ACEPTADO`. Gmail recibió el mensaje «Prueba Stalwart a Gmail» **en Spam**.

**Importante:** aceptación SMTP por el servidor de salida no prueba entrega en el destinatario, pero la captura de Gmail sí confirmó entrega. El correo generado por terminal era muy básico y carecía de algunos encabezados habituales como Date y Message-ID.

## 6. Webmail Gandy Mail → Gmail — VERIFICADO

Desde `correo.gandywilliam.dev` se envió el correo «hola querido», con un archivo adjunto, desde `contacto@gandywilliam.dev`. Gmail lo recibió en Spam el 13/11 hacia 12:02 hora de Perú. Esto prueba una operación end-to-end desde Bulwark; **no** valida todas las funciones de su interfaz.

## 7. Puerto 25: observaciones aparentemente contradictorias

- Desde una conexión Arch Linux TCP/25 a `13.140.207.155` la prueba terminó por timeout (exit 124); ocurrió incluso sin WARP.
- Desde un comprobador externo MxToolbox: `220 mail.gandywilliam.dev Stalwart ESMTP`, STARTTLS, PTR correcto, relay anónimo rechazado (correcto por seguridad).
- El VPS pudo establecer conexión saliente TCP/25 a MX Gmail, según la comprobación compartida.

Conclusión: la red de la estación Arch no pudo conectar a 25, pero ello no significaba un fallo global de acceso al servidor.

## 8. JMAP / HTTPS — VERIFICADO parcialmente

```bash
curl -sS -o /dev/null -w 'Gandy Mail: HTTP %{http_code}\n' https://correo.gandywilliam.dev
# Gandy Mail: HTTP 200

curl -sSL -o /dev/null \
  -w 'JMAP final: HTTP %{http_code}\nURL final: %{url_effective}\n' \
  https://jmap.gandywilliam.dev/.well-known/jmap
# JMAP final: HTTP 200
# URL final: https://jmap.gandywilliam.dev/jmap/session
```

GET inicial devolvió 307; HEAD dio 404, que no contradice la ruta GET 307→200. HTTP 200 no valida por sí solo autenticación de todas las acciones JMAP, pero el posterior envío de correo desde webmail corroboró funcionamiento práctico de esa ruta.

## 9. TLS

```bash
openssl s_client -connect mail.gandywilliam.dev:465 -servername mail.gandywilliam.dev -verify_return_error
openssl s_client -connect mail.gandywilliam.dev:993 -servername mail.gandywilliam.dev -verify_return_error
openssl s_client -starttls smtp -connect mail.gandywilliam.dev:25 -servername mail.gandywilliam.dev -verify_return_error
```

Certificado Let's Encrypt wildcard validado en SMTP/IMAP en pruebas realizadas.

## 10. Pendiente: Gmail Spam

Gmail mostró «Este mensaje es similar a mensajes que se identificaron como spam en el pasado». No significa automáticamente que SPF/DKIM/DMARC fallen. Falta abrir el mensaje → tres puntos → **Mostrar original** y registrar resultados de autenticación SPF, DKIM y DMARC; también revisar reputación de IP, contenido y frecuencia. Que los TXT estén publicados **no** prueba PASS en un mensaje.
