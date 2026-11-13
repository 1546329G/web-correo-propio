# Checklist reproducible para compañeros

**Estudiante:** __________ · **Dominio:** __________ · **VPS:** __________ · **Fecha:** __________  
**Ubuntu:** __________ · **Docker/Compose:** __________ · **Imagen Stalwart (tag+digest):** __________ · **Bulwark (tag+digest):** __________

Este checklist se usa al terminar la [guía de recreación](10-recrear-desde-cero.md) y se entrega con evidencia **sin secretos**. Estado «OK» exige prueba real; «PENDIENTE» no se puede reportar como terminado.

## Etapa A — VPS y dominio

- [ ] SSH seguro con usuario no-root y sudo; existe acceso de recuperación.
- [ ] Docker y Docker Compose instalados y versiones registradas.
- [ ] El dominio usa nameservers Cloudflare.
- [ ] `mail.MI_DOMINIO` tiene A DNS-only a IPv4 de MI VPS.
- [ ] PTR inverso IPv4 apunta al hostname `mail.MI_DOMINIO` y vuelve al mismo A.
- [ ] El proveedor permite entrada TCP/25 y salida a MX externos.
- [ ] Puertos 80/443 ocupados (o libres) documentados; no se interrumpe otro servicio.

## Etapa B — Stalwart

- [ ] Contenedor funcionando; `docker ps` y `docker port` revisados.
- [ ] Volúmenes de configuración y buzones persistentes.
- [ ] Wizard completado con dominio/hostname/administrador propios.
- [ ] Administración no expuesta en 8080 público.
- [ ] ACME Let's Encrypt: correo contacto, desafío DNS-01 si 443 ocupado, certificado válido en 465/993.
- [ ] Cuenta de correo ordinaria creada separada de superadministración.
- [ ] No relay abierto a terceros: prueba de rechazo de relay no autenticado.
- [ ] No se publicaron contraseñas ni tokens.

## Etapa C — Cloudflare DNS y Tunnel

- [ ] MX único `10 mail.MI_DOMINIO.`; Email Routing deshabilitado si correspondía.
- [ ] SPF acorde con emisores autorizados.
- [ ] DKIM de Stalwart, con selectores generados para MI dominio.
- [ ] DMARC publicado con política apropiada para fase de pruebas.
- [ ] Alias `postmaster@` probado si se configuró como receptor de reportes.
- [ ] Tunnel online con aplicación `admin.MI_DOMINIO`.
- [ ] Access protege admin; probar visitante no autorizado.
- [ ] Si se usa webmail, Tunnel para `correo` y API HTTPS JMAP operativa.
- [ ] JMAP no expone panel administrativo ni usa CORS `*` con credenciales.

## Etapa D — Correos reales

- [ ] TLS verificado con openssl para 465 y 993.
- [ ] Cliente IMAP autenticado y LIST/SEARCH funcional.
- [ ] SMTP autenticado aceptó mensaje local.
- [ ] Gmail envió a cuenta del dominio; asunto encontrado en INBOX.
- [ ] Cuenta del dominio envió a Gmail y fue **recibido realmente** (indicar Inbox/Spam).
- [ ] «Mostrar original» en Gmail: SPF __, DKIM __, DMARC __ (PASS/FAIL).
- [ ] SPF/DKIM/DMARC están alineados en ese correo, no solo publicados en DNS.

## Etapa E — Bulwark

- [ ] Contenedor webmail saludable y volumen persistente.
- [ ] Servidor JMAP accesible por HTTPS desde navegador, sin login HTML de Access.
- [ ] `/.well-known/jmap` lleva a sesión válida; autenticación efectuada.
- [ ] Webmail permite iniciar sesión con usuario ordinario.
- [ ] Bandeja muestra mensaje Gmail; permite responder, enviar y adjuntar.
- [ ] Borrador, enviados, carpetas y archivos probados (si aplican).
- [ ] Se reinicia solo Bulwark y persisten configuración y datos.
- [ ] El resto de los servicios VPS continúa operativo.

## Etapa F — Operación, seguridad y entrega

- [ ] Script/configuración de backup documentada y restauración probada en ambiente aislado.
- [ ] Actualización a nuevo tag probada sin pérdida y rollback documentado.
- [ ] Alertas/logs acotados; no contienen contraseñas.
- [ ] Versiones y cambios respecto de esta guía documentados en un archivo `INFORME.md`.
- [ ] Capturas saneadas de DNS, Docker y Gmail para evaluación.
- [ ] No se han publicado `.env`, backups, volúmenes, claves TLS/DKIM privadas, tokens ni correos personales.

## Evaluación rápida

| Resultado | ¿Prueba necesaria? | Estado |
|---|---|---|
| Correo entrante | Gmail → cuenta Stalwart + IMAP SEARCH | __ |
| Correo saliente | Stalwart/Bulwark → Gmail recibido | __ |
| Cifrado | certificado TLS válido 465/993 y HTTPS web | __ |
| Autenticación | SPF/DKIM/DMARC del mensaje real | __ |
| Webmail | Login, bandeja, envío, adjunto | __ |
| Seguridad | Panel privado, secretos fuera de Git y relay cerrado | __ |
| Persistencia | Volúmenes y backup/restauración | __ |

### Entrega sugerida

Cada compañero crea su fork o branch de práctica con un archivo `INFORME-LAB.md` que registre fechas, dominio de prueba, versiones, evidencias resumidas, errores y soluciones, **sin secretos**. Comparar después su historial con el del proyecto de Gandy: [01](01-cronologia.md), [07](07-seguridad-incidentes.md).
