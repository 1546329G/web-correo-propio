# Stalwart en Contabo — instalación, configuración y Docker

**Estado:** reconstruido a partir de pruebas y un informe histórico (12–13/11/2026). No representa inspección SSH actual.

## VPS y restricciones

- Contabo VPS `vmi3724729`, Ubuntu Server 24.04.x; IP IPv4 de correo `13.140.207.155`.
- Aproximadamente 4 vCPU, 8 GiB RAM, 96–100 GB de almacenamiento. Claude informó el 13/11 uso de 27/96 GB en disco y 1.3/7.8 GiB RAM; son medidas puntuales.
- Contenedores preexistentes: `gym-frontend` (puertos públicos 80/443), `gym-backend` y `xeflix-media-converter-1`. **No deben detenerse ni recrearse durante mantenimiento del correo.**
- Hostname del correo `mail.gandywilliam.dev`; administración `admin.gandywilliam.dev`.

## Stalwart 1.1.0

Se instaló Stalwart en Docker usando volúmenes:
- `stalwart-etc` montado en `/etc/stalwart`
- `stalwart-data` montado en `/var/lib/stalwart`

Wizard inicial (5 pasos): hostname `mail.gandywilliam.dev`, dominio `gandywilliam.dev`, almacenamiento **RocksDB**, directorio **Internal directory**, logs en consola, proveedor DNS Cloudflare y gestión DNS/DKIM automática. Administrador `admin@gandywilliam.dev`.

Inicialmente hubo un período de `recovery mode` para reparar administración; tras recuperar acceso se regresó a modo normal, conservando configuración y fijando `STALWART_PUBLIC_URL=https://admin.gandywilliam.dev`. Se informaron backups de configuración en el VPS, **no se copiaron a GitHub**.

## Problema y corrección de los puertos

Los listeners en Stalwart incluían SMTP 25, Submission TLS 465, IMAPS 993, HTTP 8080, HTTPS 443 y otros (POP3S 995 / Sieve 4190). Sin embargo, `docker port stalwart` mostró inicialmente que solo estaban publicados los puertos locales de panel. La declaración EXPOSE de imagen **no publica puertos del host**.

Operación histórica ejecutada: identificar imagen exacta y montajes; detener `stalwart`; renombrarlo como `stalwart-anterior` (detenido para referencia/rollback); crear un nuevo `stalwart` con mismos volúmenes e imagen, exponiendo 25/465/993 públicamente, 8080 y 8443 únicamente en loopback. El nuevo contenedor quedó `healthy`; el gimnasio siguió funcionando.

**No repetir este procedimiento automáticamente sobre el servidor en uso.** El contenedor anterior no es un respaldo de los datos y no debe iniciarse simultáneamente usando el mismo almacenamiento.

## Puertos confirmados

| Publicación del VPS | Destino / finalidad |
|---|---|
| `0.0.0.0:25` | Stalwart SMTP entre servidores |
| `0.0.0.0:465` | Stalwart Submission con TLS implícito |
| `0.0.0.0:993` | Stalwart IMAPS |
| `127.0.0.1:8080` | Panel HTTP protegido por Tunnel |
| `127.0.0.1:8443` | HTTPS interno del contenedor Stalwart |
| `80/443` | **Gimnasio (no modificar)** |
| `587` | No publicado en esta instalación |

UFW estaba inactivo; ello no excluye filtros del proveedor ni políticas de Docker.

## Comandos solo de lectura

```bash
hostnamectl
docker ps --format 'table {{.Names}}\t{{.Status}}\t{{.Ports}}'
docker inspect stalwart --format '{{json .Mounts}}'
docker port stalwart
docker volume ls
sudo ss -lntp
sudo ufw status verbose
docker logs --tail 100 stalwart
sudo systemctl status cloudflared --no-pager
```

Antes de compartir salidas de logs, ocultar mensajes, direcciones personales y secretos. Nunca publicar archivos `.env`, copias de RocksDB, volúmenes ni credenciales.
