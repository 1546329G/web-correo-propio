# Cloudflare, DNS, ACME, PTR y Tunnel

## Registros públicos observados el 13/11

| Tipo | Nombre | Valor |
|---|---|---|
| A DNS only | `mail.gandywilliam.dev` | `13.140.207.155` |
| MX | `@` / `gandywilliam.dev` | `10 mail.gandywilliam.dev.` |
| TXT SPF | `@` | `v=spf1 mx -all` |
| TXT SPF | `mail.gandywilliam.dev` | `v=spf1 a -all` |
| TXT DMARC | `_dmarc` | `v=DMARC1; p=reject; rua=mailto:postmaster@gandywilliam.dev` |
| TXT DKIM | `v1-rsa-20261112._domainkey` | `v=DKIM1; k=rsa; ...` (clave pública publicada) |
| TXT DKIM | `v1-ed25519-20261112._domainkey` | `v=DKIM1; k=ed25519; ...` (clave pública publicada) |
| PTR | `13.140.207.155` en Contabo | `mail.gandywilliam.dev` |

En Cloudflare también aparecían CNAME `autoconfig`, `autodiscover`, `mta-sts`, `ua-auto-config`; SRV `_imaps._tcp`, `_submissions._tcp`, `_jmap._tcp` y otros; TXT `_mta-sts`, `_smtp._tls`. Publicar registros no demuestra configuración funcional de MTA-STS, TLS-RPT ni recepción de reportes por postmaster.

### Migración del MX

Cloudflare Email Routing controlaba inicialmente tres MX `route1/2/3.mx.cloudflare.net` y no permitía añadir otros: «This zone is managed by Email Routing. Disable Email Routing to add/modify MX records.»

Se procedió a Email Routing → Settings → **Disable** → confirmar `gandywilliam.dev`. Tras ver «Enable Email Routing» / «Onboard Domain», se creó DNS Records → MX, Name `@`, mail server `mail.gandywilliam.dev`, prioridad `10`, TTL Auto. Ambos resolutores públicos devolvieron solo el nuevo MX. No volver a activar Cloudflare Email Routing para este mismo dominio.

### PTR en Contabo

Contabo Network → DNS Management → Reverse DNS → editar **IPv4**; cambiar `vmi3724729.contaboserver.net` por `mail.gandywilliam.dev`. Después `dig @1.1.1.1 -x 13.140.207.155` y Quad9 mostraron nombre correcto. IPv6 PTR no se cambió.

### Certificados TLS

En Stalwart Settings → TLS → ACME Providers se configuró Let's Encrypt. Primer intento: «Failed to create ACME account: Invalid request: At least one contact email is required». Se configuró contacto y se seleccionó desafío **DNS-01** (no TLS-ALPN-01: puerto público 443 ocupado por gimnasio). Se asoció al dominio y Cloudflare DNS management publicó los challenges. Certificado `*.gandywilliam.dev` y `gandywilliam.dev`, issuer Let's Encrypt YE2, válido desde **12/11/2026 hasta 10/02/2027**; SMTP 465 e IMAP 993 lo sirvieron correctamente en las pruebas. Revisar renovación con antelación.

### Cloudflare Access y túneles

Túnel definitivo `stalwart-admin-g`, con servicio systemd `cloudflared`. Inicialmente el panel dio Cloudflare 525 por ruta DNS incorrecta; se corrigió a CNAME proxied del túnel. «Failed to fetch» durante login de Stalwart coincidió con API redirigida a Access y política de sesión de duración cero; se fijó 24 h y se confirmó acceso.

Rutas finales:

```text
admin.gandywilliam.dev  -> http://127.0.0.1:8080
correo.gandywilliam.dev -> http://localhost:3100
jmap.gandywilliam.dev   -> http://localhost:3101
```

Cloudflare confirmó «Route added successfully» y «DNS record created». Al agregar `correo` se corrigió la URL mal escrita `http/localhost:3100` a `http://localhost:3100`.

### Auditoría DNS rápida

```bash
dig @1.1.1.1 +short MX gandywilliam.dev
dig @8.8.8.8 +short MX gandywilliam.dev
dig @1.1.1.1 +short A mail.gandywilliam.dev
dig @1.1.1.1 +short TXT gandywilliam.dev
dig @1.1.1.1 +short TXT _dmarc.gandywilliam.dev
dig @1.1.1.1 +short TXT v1-rsa-20261112._domainkey.gandywilliam.dev
dig @1.1.1.1 +short TXT v1-ed25519-20261112._domainkey.gandywilliam.dev
dig @1.1.1.1 +short -x 13.140.207.155
```

**Advertencia:** `p=reject` de DMARC es estricto. Debe comprobarse SPF/DKIM alineados en encabezados reales, no solo registros publicados. Los secretos del token API Cloudflare y claves privadas nunca van en este repositorio.
