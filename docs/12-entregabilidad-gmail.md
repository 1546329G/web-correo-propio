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
