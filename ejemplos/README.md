# Plantillas de laboratorio

**Estas plantillas son genéricas y deben usarse en un VPS nuevo**, no sobre el VPS de producción de Gandy. No son un volcado del Docker Compose real instalado por Claude y no recrean automáticamente el gateway nginx específico de producción.

- [Stalwart/compose.yaml](stalwart/compose.yaml): Docker con volúmenes y puertos 25/465/993; panel solo en loopback, HTTPS interno en 8443. Diseñado para el escenario con 80/443 ocupados.
- [Bulwark/compose.yaml](bulwark/compose.yaml): Docker de webmail completo con asistente inicial y volumen persistente, solo loopback 3100. Necesita un JMAP HTTPS funcional accesible desde el navegador.

**Cómo usar en una máquina de pruebas:**

```bash
mkdir -p ~/infra-correo/stalwart
cp ejemplos/stalwart/compose.yaml ~/infra-correo/stalwart/compose.yaml
cd ~/infra-correo/stalwart
docker compose config
docker compose up -d
```

Después del wizard Stalwart y el alta DNS, Bulwark:

```bash
mkdir -p ~/infra-correo/bulwark
cp ejemplos/bulwark/compose.yaml ~/infra-correo/bulwark/compose.yaml
cd ~/infra-correo/bulwark
docker compose config
docker compose up -d
docker compose logs --tail 80
```

La instalación de Bulwark presenta asistente de primera ejecución porque no se fijó `JMAP_SERVER_URL`. Protege el token del wizard y no publiques el puerto directamente. Configura JMAP y el dominio propio mediante los pasos en [la guía](../docs/10-recrear-desde-cero.md).

Para publicar con Cloudflare Tunnel sin ocupar 80/443, crea una ruta `correo.TU_DOMINIO` a `http://localhost:3100` y una API JMAP segura en HTTPS. **No apuntes a `127.0.0.1:8443` de Stalwart desde un túnel HTTP sin estudiar su TLS y la exposición del administrador.** El gateway filtrado específico del proyecto histórico requiere exportar su nginx real.

**Por seguridad**: comprobar la versión y el digest de cada imagen, no usar `latest` a ciegas, backup previo a cambios, sin usar usuario `admin` como cuenta personal. No copiar tokens, claves, contraseñas ni datos del VPS histórico.

Referencias: https://www.stalw.art/docs/install/platform/docker/ y https://bulwarkmail.org/docs/deployment/docker/compose
