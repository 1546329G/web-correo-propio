# Seguridad e incidentes

## Incidentes registrados

| Incidente | Síntoma | Resolución / situación |
|---|---|---|
| Cloudflare Error 525 | Panel no abría | Corregir DNS/proxy para apuntar a Tunnel, sin tocar 443 del gimnasio |
| Stalwart «Failed to fetch» | Login web fallaba | Detectadas redirecciones Cloudflare Access 302 y sesión con duración 0; cambiar a 24 horas; login verificado |
| ACME sin contacto | «At least one contact email is required» | Añadir correo válido al proveedor Let's Encrypt |
| Desafío ACME equivocado | TLS-ALPN-01, 443 ocupado | Cambiar a DNS-01 a través de Cloudflare |
| Puertos SMTP/IMAP inaccesibles | Docker expone en imagen pero no publica host | Recrear Stalwart preservando volúmenes, publicar 25,465,993 |
| Registro PTR incorrecto | Hostname de Contabo | Modificar PTR IPv4 a `mail.gandywilliam.dev` |
| MX bloqueado | Cloudflare Email Routing administra DNS | Desactivar Email Routing y crear MX Stalwart |
| Timeout 25 desde Arch | `exit 124` | Comprobador SMTP independiente alcanzó banner; restricción posible de red cliente |
| Comando hostname | `bash: mail.gandywilliam.dev: command not found` | Un DNS hostname no es comando; consultar con `dig` |
| curl IMAP | `curl: (21) Quote command returned error` | Usar `--request 'SEARCH SUBJECT "..."'` |
| Bulwark build | Node out-of-memory | Claude informó ajuste de memoria en Dockerfile |
| JMAP HEAD 404 | HEAD `/.well-known/jmap` | GET siguió 307 hasta session 200; no confundir métodos |
| Mensajes enviados a Spam | Gmail recibió, no clasificó en Inbox | **Pendiente:** examinar autenticación real, reputación y formato |

## URGENTE: contraseña SSH/sudo

El informe de Claude indicó que la contraseña SSH coincidía con una cadena publicada en contenidos del sitio del gimnasio. También informó eliminación del script temporal de acceso, pero **la eliminación no sustituye la rotación**. Por seguridad, esta documentación pública **no reproduce la contraseña, su cadena, ni rutas concretas del sitio que revelan el secreto**.

Pasos:
1. Ejecutar `passwd` desde sesión SSH confiable de `gandy`; escribir la nueva contraseña solo en terminal segura.
2. **Sin cerrar sesión actual**, comprobar login en otra ventana.
3. Retirar la cadena de archivos públicos del gimnasio, sus bundles y despliegues; revisar si se reutilizó en otros servicios, y rotar allí también.
4. Preparar claves SSH ed25519, instalar **solo clave pública** en `authorized_keys`, probar el acceso; después considerar apagar login por contraseña.
5. Inspeccionar archivos temporales/historial y realizar auditoría adicional en registro privado, nunca exponer evidencias confidenciales en GitHub.

No afirmar que la contraseña ya fue cambiada: la conversación no contiene confirmación.

## JMAP y Cloudflare Access

El gateway `jmap.gandywilliam.dev` debe exponer solo las rutas API necesarias con autenticación Stalwart, CORS acotado y límites de intentos. Claude indicó que Stalwart puede ver la IP del proxy en lugar de la del cliente, afectando bloqueos por intentos fallidos. Revisar trusted proxies y `X-Forwarded-For` con cuidado, no confiar en headers de clientes no autenticados.

La administración `admin.gandywilliam.dev` debe continuar protegida por Cloudflare Access. No abrir el panel 8080 a Internet.

## Riesgos de correo

DMARC está publicado con `p=reject`. Aún debe comprobarse si mensajes concretos tienen SPF/DKIM alineados, para evitar rechazos futuros. Nunca abrir relay anónimo. Verificar existencia de alias `postmaster` para reportes. Probar renovación Let's Encrypt antes del 10/02/2027.

## Exclusiones deliberadas de este repositorio público

No publicar: claves privadas, passwords, claves DKIM privadas, token Cloudflare Tunnel/API, bearer tokens, OAuth secrets, archivos `.env`, backups, contenido de buzones, dumps RocksDB, cookies `CF_Authorization`, historial SSH, identificadores internos confidenciales. No se adjuntaron logs crudos con credenciales.
