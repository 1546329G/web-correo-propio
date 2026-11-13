# Mantenimiento, copias, actualización y recuperación

## Regla crítica: VPS multiservicio

No detener ni recrear `gym-frontend`, `gym-backend` ni el conversor. **No ejecutar** `docker system prune --volumes`, `docker compose down -v` ni limpieza global. No ejecutar una recreación de Stalwart simplemente para comprobar que funciona. El volumen de buzón es producción.

## Comandos diagnósticos de solo lectura

En VPS:

```bash
hostnamectl
docker ps --format 'table {{.Names}}\t{{.Status}}\t{{.Ports}}'
docker port stalwart
docker inspect stalwart --format '{{json .Mounts}}'
docker volume ls
sudo ss -lntp
sudo systemctl status cloudflared --no-pager
docker logs --tail 100 stalwart
```

Desde Arch:

```bash
dig @1.1.1.1 +short MX gandywilliam.dev
dig @8.8.8.8 +short MX gandywilliam.dev
dig @1.1.1.1 +short A mail.gandywilliam.dev
dig @1.1.1.1 +short -x 13.140.207.155
curl -sS -o /dev/null -w 'Webmail HTTP %{http_code}\n' https://correo.gandywilliam.dev
curl -sSL -o /dev/null -w 'JMAP HTTP %{http_code} URL %{url_effective}\n' https://jmap.gandywilliam.dev/.well-known/jmap
```

Los resultados históricos fueron correctos el 13/11; repetirlos después de cambios.

## Backup Stalwart

Volúmenes `stalwart-etc` y `stalwart-data` contienen información de usuarios, certificados, credenciales y buzones. Preparar respaldo cifrado fuera del VPS y coherente con el motor de base de datos; no copiar en caliente sin evaluar consistencia. Probar restauración aislada. `stalwart-anterior` es solo un contenedor detenido, **no** backup de datos.

## Backup Bulwark

Según Claude: volumen `bulwark-data`, copia inicial en `deploy/gandy/backups` y script `deploy/gandy/scripts/backup.sh`. Su informe mencionó cron para copia diaria como propuesta: **no se ha confirmado que ese cron esté instalado**. Inspeccionar y probar el script antes de utilizarlo.

## Actualizar Bulwark editable

```bash
cd /home/gandy/apps/bulwark-webmail
git status
git remote -v
git fetch upstream --tags
```

Procedimiento: guardar cambios locales → backup verificable → seleccionar tag estable y changelog → crear rama de prueba → integrar upstream respetando fork/AGPL → ejecutar test, lint y typecheck → crear imagen con tag fijo → desplegar **solo** `gandy-mail` → validar login, JMAP, IMAP/SMTP, adjuntos y estado del gimnasio → rollback si falla.

Claude señaló `deploy/gandy/scripts/deploy.sh`, `rollback.sh` y `check.sh`. No ejecutar sin inspeccionar su código. Existe documentación adicional creada por Claude en el VPS; **esta documentación no se ha sincronizado automáticamente con sus archivos internos**.

## Si algo falla

- Webmail 502/1033: revisar Cloudflare Tunnel, servicio `cloudflared`, rutas a localhost 3100/3101 y contenedores.
- JMAP GET 200 pero login falla: revisar reescritura a `jmap.gandywilliam.dev`, cookies, CORS, autenticación, Access wildcard, códigos 401/403 y logs.
- SMTP no entra: revisar MX, A DNS-only, conectividad TCP 25 desde otra red y logs Stalwart.
- SMTP sale pero Gmail Spam: abrir «Mostrar original», comprobar SPF/DKIM/DMARC y reputación.
- TLS no valida: comprobar certificado realmente servido por Stalwart en 465/993, fecha y ACME DNS-01.
- Panel no abre: revisar Cloudflare Access sesión, DNS/túnel; **no** abrir el 8080 al público como arreglo.
- Falla contenedor: comprobar montajes/versiones y backups antes de considerar rollback; no arrancar dos Stalwart compartiendo volumen.
