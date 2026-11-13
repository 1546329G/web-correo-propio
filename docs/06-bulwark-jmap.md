# Bulwark Webmail — Gandy Mail

## Propósito

Se eligió [Bulwark Webmail](https://github.com/bulwarkmail/webmail), cliente de código abierto para Stalwart/JMAP, para crear un correo web propio editable en VS Code con experiencia similar a Gmail y actualizaciones del upstream. Alternativas evaluadas: Roundcube y Thunderbird. Licencia Bulwark: **AGPL-3.0**, cuyas obligaciones de entrega de código fuente deben revisarse al distribuir o servir una versión modificada.

## Infraestructura creada — INFORMADA POR CLAUDE

El informe de Claude Code del 13/11 informó:

| Campo | Valor |
|---|---|
| Versión | Bulwark 1.21.2 |
| Commit upstream | `933bc18f5` |
| Clon del código | `/home/gandy/apps/bulwark-webmail` |
| Rama editable | `gandy/main` (`HEAD 000530adc` informado) |
| Imagen compilada | `gandy/bulwark-webmail:1.21.2-933bc18f5-b1` |
| Contenedores | `bulwark-webmail` y `jmap-gateway` |
| Puertos host | `127.0.0.1:3100` y `127.0.0.1:3101` |
| Volumen | `bulwark-data` → `/app/data` |
| Docker Compose | `/home/gandy/apps/bulwark-webmail/deploy/gandy/docker-compose.yml`, proyecto `gandy-mail` |
| Nombre/idioma | «Gandy Mail» en español, zona `America/Lima` |
| Telemetría | Desactivada según informe |
| Secretos | Archivo local del VPS `.env.bulwark` (no copiar ni subir a Git) |

La compilación inicialmente falló por memoria/heap de Node; Claude informó ajuste `BUILD_NODE_OPTIONS` en Dockerfile. El directorio upstream contiene scripts para despliegue, respaldo y rollback, según Claude, pero aquí **no se auditó su código**. Claude indicó que el puerto 443 del gimnasio no se modificó.

## Integración JMAP

- Stalwart anuncia originalmente su URL pública como `https://admin.gandywilliam.dev`, que está protegida por Cloudflare Access.
- Para que el navegador no reciba un login HTML de Access en llamadas API, Claude creó un nginx `jmap-gateway`, que reportó permitir únicamente rutas JMAP/auth necesarias, reescribir URL públicas a `https://jmap.gandywilliam.dev` y permitir CORS desde `https://correo.gandywilliam.dev` únicamente.
- En Cloudflare Tunnel se publicaron `correo.gandywilliam.dev → localhost:3100` y `jmap.gandywilliam.dev → localhost:3101`, sin cambiar la ruta administrativa `admin.gandywilliam.dev → 127.0.0.1:8080`.

Se comprobó `correo` HTTP 200 y JMAP `/.well-known/jmap` GET 307→`/jmap/session` HTTP 200. Posteriormente la web envió un mensaje a Gmail con adjunto. **No se ha hecho en esta documentación un análisis de seguridad de la configuración completa del nginx.**

## Desarrollo y actualizaciones

Repositorio upstream: `https://github.com/bulwarkmail/webmail.git`.

```bash
cd /home/gandy/apps/bulwark-webmail
git status
git remote -v
git branch --show-current
git fetch upstream --tags
```

No mezclar automáticamente un tag upstream con personalizaciones sin analizar changelog, conflictos, compatibilidad de Stalwart, nuevas env vars y licencia. Flujo sugerido: backup verificable → rama de prueba → integrar versión upstream → tests/lint/typecheck → crear imagen con tag inmutable → desplegar **solo el proyecto Bulwark** → pruebas JMAP, login, correo → rollback si falla.

Según Claude, en VPS existen `deploy/gandy/scripts/backup.sh`, `deploy.sh`, `rollback.sh`, `check.sh`, junto con `deploy/gandy/README.md` y docs SSH/Cloudflare/auditoría. Se deben inspeccionar antes de ejecutar; **estos archivos no se copiaron aquí**.

## Diagnóstico local sin modificar servicios

```bash
cd /home/gandy/apps/bulwark-webmail/deploy/gandy
docker compose ps
docker compose config --services
docker compose logs --tail 100
```

Para chequeos JMAP, Claude recomendó `JMAP_USER=contacto@gandywilliam.dev scripts/check.sh`: requiere contraseña privada que se debe introducir localmente sin pegarla en chats ni logs.

## Seguridad pendiente

Autenticación y CORS correctos no equivalen a protección exhaustiva; inspeccionar límites de intentos, cabeceras `X-Forwarded-For`, rutas expuestas por gateway, sesiones y cookies. Según Claude, trusted proxies de Stalwart no se cambió y posiblemente todos los clientes aparezcan desde IP del proxy. OAuth/SSO no quedó configurado. Se debe rotar la contraseña inicial de administrador Bulwark y no subir `.env.bulwark`.
