# Diagnóstico de entregabilidad Gmail — 13 noviembre 2026

## Evidencias hasta ahora

| Prueba | Resultado | Alcance |
|---|---|---|
| MX, PTR, TLS y SMTP | Correctos en pruebas anteriores | No determina clasificación de Gmail |
| Gmail: correo terminal «Prueba Stalwart a Gmail» | SPF PASS; DKIM RSA PASS; DMARC PASS; TLS 1.3 | **Solo** ese correo de terminal |
| Ese correo de terminal: `Message-ID` | Google creó `...SMTPIN_ADDED_MISSING@mx.google.com` | El mensaje manual carecía de un ID original válido |
| Comprobador Spamhaus para IPv4 `13.140.207.155` | «has no issues» (captura del 13/11) | No equivale a reputación alta en Gmail |
| Mensaje webmail «hola querido» | Llegó a Gmail, pero en Spam | Entrega confirmada, no Inbox |
| MIME del mensaje «hola querido» (archivo exportado) | `Message-ID` y `Date` presentes; `text/plain` vacío; HTML solo imagen inline con `cid:` | Es posible factor de puntuación antispam; sin prueba causal |
| Encabezados de recepción Gmail del mensaje «hola querido» | **No incluidos** en el archivo exportado | No se conocen aún SPF/DKIM/DMARC ni `Authentication-Results` de **este** mensaje |

### Hallazgo técnico del archivo adjunto

El mensaje de Bulwark «hola querido» presenta:
- `From: "Contacto" <contacto@gandywilliam.dev>`.
- `Message-ID: <mwf85umt.61851898-663b-4c32-98b7-a5f8c7dcabf5@gandywilliam.dev>`.
- `Date: Fri, 13 Nov 2026 17:02:27 +0000` (12:02 Perú).
- `MIME-Version: 1.0`.
- `multipart/alternative` con sección `text/plain; charset=utf-8` **vacía**, y sección `multipart/related` que contiene HTML con un `<img src="cid:...">` y el PNG inline `image.png`.
- No contiene `Authentication-Results:`, `Received:`, `Return-Path:` ni `DKIM-Signature:`. Esto indica que **el archivo compartido no incluye la trazabilidad SMTP del mensaje ya recibido por Gmail**; no se puede usar para dar PASS/FAIL en Gmail de ese correo.

No hay que identificar «imagen sin texto» como única causa confirmada. Gmail evalúa reputación propia e historial, contenido, autenticación y señales de abuso. Una IP sin listas Spamhaus puede ir a Spam en Gmail.

## Pasos de diagnóstico recomendados

1. En **Gmail**, abrir el mensaje «hola querido» de la carpeta Spam → tres puntos junto a Responder → **Mostrar original**, en la página de Gmail. Guardar solo el resumen **SPF, DKIM, DMARC** y, si es necesario, encabezados `Authentication-Results` **saneados**.
2. No cambiar MX/SPF/DKIM/DMARC hasta ver los resultados **del mensaje de Bulwark**, no asumir los del mensaje de prueba terminal.
3. Enviar un único correo de prueba desde Gandy Mail a una cuenta Gmail propia y consentida, con **asunto descriptivo** y **varias frases reales en texto plano**, sin imágenes, archivos ni enlaces. Comprobar recepción Inbox vs Spam, y de nuevo el resumen SPF/DKIM/DMARC. No enviar múltiples pruebas repetitivas.
4. En correo legítimo propio clasificado en Spam, marcar «No es spam» en Gmail (afecta principalmente la experiencia de esa cuenta, no garantiza entrega global).
5. Si autenticación/alineación pasa y aún marca Spam, investigar reputación propia de Gmail con [Postmaster Tools](https://postmaster.google.com/); el panel puede no mostrar datos con tráfico bajo. Revisar que el hostname HELO, IP/PTR/A, TLS y formato RFC 5322 sean adecuados.
6. Anotar resultados reales en este archivo, fecha y pruebas. Solo después considerar cambios específicos.

## Fuentes de criterios

- [Google — lineamientos para remitentes](https://support.google.com/mail/answer/81126?hl=es-419).
- [Google — paneles de Postmaster Tools](https://support.google.com/mail/answer/14668346).
- [Spamhaus — preguntas frecuentes sobre listas](https://www.spamhaus.org/faqs/general-questions/).

**No publicar el MIME completo con imágenes ni direcciones personales del receptor en GitHub; este documento registra solo aspectos técnicos.**


## Actualización: segunda prueba enviada desde Gandy Mail — 13/11, 13:19 (America/Lima)

El usuario compartió los encabezados originales **recibidos por Gmail** del correo con asunto `saludar`, remitente `contacto@gandywilliam.dev`.

| Comprobación | Resultado |
|---|---|
| SPF | **PASS**, IP emisora 13.140.207.155, `smtp.mailfrom=gandywilliam.dev` (cuenta contacto) |
| DKIM RSA | **PASS**, `d=gandywilliam.dev`, selector `v1-rsa-20261112` |
| DKIM Ed25519 | `neutral (no key)`, selector `v1-ed25519-20261112` |
| DMARC | **PASS**, `header.from=gandywilliam.dev`; dominio alineado |
| SMTP | ESMTPS, **TLS 1.3**, `TLS_AES_256_GCM_SHA384` |
| `Message-ID` | Presente, generado por Gandy Mail: dominio `@gandywilliam.dev` |
| `Date` | Presente; entrega a Gmail ~7 s después |
| MIME | `multipart/alternative` con **texto plano y HTML** no vacíos |
| Spam/Inbox para esta segunda prueba | **NO INFORMADO** en los encabezados compartidos |

**Conclusión de la evidencia:** la autenticación del correo webmail funciona. La anotación `neutral (no key)` de Ed25519 no invalida el `PASS` de RSA y DMARC. No modificar SPF, DKIM RSA, DMARC o MX para intentar solucionar la carpeta Spam. La primera prueba en texto solo y la de imagen no permitían aislar totalmente contenido/reputación; esta segunda prueba confirma formato correcto de mensajes y autenticación.

**Diagnóstico siguiente:** verificar si el correo `saludar` se depositó en bandeja de entrada o Spam. Si entró a Spam, el siguiente foco es reputación/historial del dominio y la IP en Gmail, relevancia del mensaje y clasificación por destinatario. Configurar [Google Postmaster Tools](https://postmaster.google.com/) para el dominio `gandywilliam.dev` usando la verificación DNS que Google proporcione, sin alterar MX/SPF/DKIM/DMARC. La herramienta puede no mostrar métricas con tráfico demasiado bajo. Para mensajes legítimos de prueba clasificados en Spam, el destinatario puede usar «No es spam»; esto no garantiza mejora global. No realizar envíos masivos ni pruebas repetitivas.

Fuentes de criterios: [Directrices Google remitentes](https://support.google.com/mail/answer/81126?hl=es) y [Postmaster Tools](https://support.google.com/mail/answer/9981691?hl=es-ES).
