# Guía reproducible desde cero — Stalwart + Cloudflare + Bulwark

**Objetivo:** que un compañero implemente su propio correo en un VPS y, opcionalmente, su propio webmail; **no copiar literalmente las credenciales, la IP o el dominio de la instalación histórica**. La sección [histórica](01-cronologia.md) explica qué hicimos realmente y por qué. 

**Importante:** la instalación histórica Gandy Mail tiene una pasarela nginx JMAP particular programada por Claude. Hasta que se exporten y auditen sus archivos de configuración sin secretos, este repositorio da **un procedimiento reproducible estándar** y documenta la diferencia con aquella variante. NO afirmamos que los comandos de ejemplo sean un dump probado del VPS original.

## Índice

1. [Escenario y requisitos](#1-escenario-y-requisitos)
2. [Preparar dominio y VPS](#2-preparar-dominio-y-vps)
3. [Instalar Docker](#3-instalar-docker)
4. [Instalar Stalwart](#4-instalar-stalwart)
5. [Configurar Stalwart y certificado](#5-configurar-stalwart-y-certificado)
6. [Publicar SMTP e IMAP y DNS](#6-publicar-smtp-e-imap-y-dns)
7. [Cloudflare Tunnel y protección del panel](#7-cloudflare-tunnel-y-protección-del-panel)
8. [Crear buzones y probar correo](#8-crear-buzones-y-probar-correo)
9. [Instalar Bulwark Webmail](#9-instalar-bulwark-webmail)
10. [Configurar JMAP por HTTPS](#10-configurar-jmap-por-https)
11. [Pruebas finales](#11-pruebas-finales)
12. [Qué falta para recrear exactamente el VPS original](#12-qué-falta-para-recrear-exactamente-el-vps-original)

## 1. Escenario y requisitos

Cada integrante elige:

| Dato | Ejemplo didáctico | Qué sustituir |
|---|---|---|
| Dominio propio | example.com | Dominio registrado y controlado en Cloudflare |
| Mail host | mail.example.com | Subdominio que resuelve al VPS |
| Admin | admin.example.com | Panel Stalwart por Cloudflare Access |
| Webmail | correo.example.com | Cliente Bulwark |
| API JMAP | jmap.example.com | API HTTPS si se utiliza host separado |
| Usuario de correo | contacto@example.com | Buzón ordinario |
| IP del VPS | IPv4 pública propia | No copiar la IP del proyecto original |
| VPS | Ubuntu 24.04 LTS | Al menos 2 vCPU y 4 GB RAM recomendables para laboratorio, dimensionar producción |

También: acceso SSH seguro con sudo, registros DNS controlados, tráfico entrante SMTP TCP/25 admitido por proveedor, conectividad saliente TCP/25 a MX externos, disco persistente, y una cuenta Cloudflare. Comprobar antes con proveedor: varios bloquean 25 y algunos exigen solicitar desbloqueo. HTTPS para webmail no sustituye SMTP/IMAP.

**Dos escenarios diferentes:**
- **Laboratorio/VPS nuevo:** 443 libre; puede publicar HTTPS de Stalwart y usar directamente \`https://mail.example.com\` para JMAP, simplificando la arquitectura.
- **VPS compartido como el histórico:** 80/443 ocupados por otro proyecto; se configura acceso a admin y webmail mediante Cloudflare Tunnel, y JMAP por una ruta proxy privada/segura. El gateway nginx original está documentado en [Bulwark](06-bulwark-jmap.md) pero sus archivos exactos todavía deben exportarse.

No sustituir ni mezclar ambos escenarios sin adaptar rutas y certificados.

## 2. Preparar dominio y VPS

En Cloudflare: agregar el dominio a la cuenta, confirmar que usa nameservers de Cloudflare y crear **A DNS only** para \`mail.example.com → IPv4_VPS\`. La nube **gris** es obligatoria para SMTP/IMAP normal: el proxy naranja estándar no es un proxy de correo. No tocar registros de otros sitios.

En Contabo o proveedor: configurar PTR **IPv4 inverso** de la IP a \`mail.example.com\`; debe coincidir con el A. Verificar:

\`\`\`bash
dig +short A mail.example.com
dig -x IPV4_DEL_VPS +short
\`\`\`

En Ubuntu:

\`\`\`bash
sudo apt update
sudo apt install -y ca-certificates curl git dnsutils openssl
hostnamectl
ss -lnt
\`\`\`

No copiar una contraseña SSH de otro compañero. Crear usuario con sudo, claves SSH ed25519 y probar la sesión antes de deshabilitar acceso por contraseña. No almacenar contraseñas de VPS en scripts permanentes.

## 3. Instalar Docker

Seguir el procedimiento **oficial** actualizado para Ubuntu (repositorio apt de Docker), no scripts de procedencia incierta:
https://docs.docker.com/engine/install/ubuntu/

Verificar:

\`\`\`bash
docker --version
docker compose version
sudo docker info >/dev/null && echo 'Docker accesible'
\`\`\`

Anotar en el informe la versión real de Docker y Ubuntu. Abrir solo los puertos estrictamente necesarios; para correo 25/465/993 y para administración SSH restringido. No abrir 8080 de Stalwart públicamente.

## 4. Instalar Stalwart

La guía Docker oficial actual es https://www.stalw.art/docs/install/platform/docker/ . Contiene un ejemplo con volúmenes \`/etc/stalwart\` y \`/var/lib/stalwart\`. Deben persistir.

Para reproducir un VPS con **443 ocupado**, crear una carpeta exclusiva:
\`\`\`bash
mkdir -p ~/infra-correo/stalwart
cd ~/infra-correo/stalwart
\`\`\`

Copiar el ejemplo de [plantilla Stalwart para VPS compartido](../ejemplos/stalwart/compose.yaml) y verificarlo:
\`\`\`bash
docker compose -f compose.yaml config
docker compose -f compose.yaml up -d
docker compose -f compose.yaml ps
docker logs stalwart --tail 80
\`\`\`

**No ejecutar este ejemplo en el VPS histórico**: el nombre de contenedor y puertos colisionarían con el existente.

La plantilla usa un tag de línea mayor \`v1\` del repositorio oficial; **para repetir de forma bit-a-bit una versión se debe fijar un tag/digest disponible y escribirlo en el informe**. La instalación histórica fue Stalwart 1.1.0; una línea \`v1\` puede actualizar de versión. No se recomienda \`latest\`.

Para primer arranque en VPS compartido, acceder al wizard mediante SSH:

\`\`\`bash
ssh -L 8080:127.0.0.1:8080 USUARIO@IP_DEL_VPS
\`\`\`

Abrir \`http://localhost:8080\` desde el navegador local. No publicar 8080 en Internet antes de instalar Access o de asegurar la autenticación.

## 5. Configurar Stalwart y certificado

Seguir wizard de primera ejecución de la versión instalada: hostname **\`mail.example.com\`**, dominio **\`example.com\`**, almacenamiento persistente, directorio de usuarios y credenciales fuertes de admin. No copiar la contraseña del ejemplo ni pegarla en Git.

Para **TLS con 443 ocupado**: en Stalwart configurar proveedor ACME Let's Encrypt con correo de contacto real y desafío **DNS-01** vía Cloudflare; generar token API Cloudflare con privilegios mínimos para el DNS de ese dominio y guardarlo exclusivamente en servidor/secret manager, nunca en docs. No usar TLS-ALPN-01 sobre 443 ocupado. Configurar los nombres del certificado para mail host y servicios usados; esperar emisión y comprobar TLS en los puertos.

Si el wizard exige datos particulares de la versión actual, registrar diferencias, no forzar instrucciones antiguas. El desafío DNS-01 no prueba por sí solo que los servicios sirven el certificado correcto.

## 6. Publicar SMTP e IMAP y DNS

Registrar en Cloudflare:

| Tipo | Nombre | Contenido / prioridad |
|---|---|---|
| A, DNS only | \`mail\` | IPv4 del VPS |
| MX | \`@\` | \`10 mail.example.com\` |
| TXT SPF | \`@\` | \`v=spf1 mx -all\` **si ese MX es el único emisor autorizado** |
| TXT DKIM | Selector provisto por Stalwart | **Copiar exactamente cada valor público del panel** |
| TXT DMARC | \`_dmarc\` | En fase de pruebas es razonable empezar por \`v=DMARC1; p=none; rua=mailto:postmaster@example.com\`, endurecer tras validar |
| PTR | En proveedor VPS | \`mail.example.com\` |

No copiar claves DKIM públicas del dominio histórico: cada dominio debe generar sus propias claves. DKIM **privada nunca en DNS/GitHub**. Crear alias/buzón \`postmaster@example.com\` si se utiliza para informes; verificar que recibe reportes.

**Si Cloudflare Email Routing ya administra MX**, primero comprobar sus reglas, luego deshabilitarlo desde Email Routing → Settings → Disable. Solo después crear el MX a Stalwart. Nunca dejar MX de Cloudflare y Stalwart mezclados accidentalmente.

Validar:
\`\`\`bash
dig @1.1.1.1 +short MX example.com
dig @8.8.8.8 +short MX example.com
dig +short TXT example.com
dig +short TXT _dmarc.example.com
dig -x IPV4_DEL_VPS +short
\`\`\`

El resultado MX debe mostrar solo el destino propio y la prioridad definida. TTL y caches pueden atrasar cambios. Validar la IP destino A; no confundir resolver Cloudflare con Cloudflare Email Routing.

En \`mail.example.com\` no habilitar el proxy naranja para SMTP normal.

## 7. Cloudflare Tunnel y protección del panel

Docs oficiales: https://developers.cloudflare.com/cloudflare-one/connections/connect-networks/

Crear **Tunnel Cloudflared** desde Cloudflare Zero Trust → Networks → Tunnels y seguir instrucciones de instalación del servicio cloudflared en Ubuntu que ofrece la consola. El token de túnel es un **secreto**: no publicarlo ni pegarlo a un repositorio. Verificar \`systemctl status cloudflared\`.

Crear ruta publicada:
\`\`\`text
Hostname: admin.example.com
Service: http://127.0.0.1:8080
\`\`\`

Cloudflare asignará DNS CNAME al túnel. Crear aplicación Cloudflare Access self-hosted para **\`admin.example.com\`** y permitir únicamente correos/identidades concretas; sesión limitada y MFA. Verificar que un visitante no autorizado no puede obtener el panel. El login del panel debe permitir las llamadas API AJAX que necesita: si aparece «Failed to fetch» revisar Access, redirecciones y expiración, no abrir el puerto 8080 al mundo como arreglo.

Recomendación: **antes de publicar un panel**, asegurar la política Access y los usuarios autorizados. Evitar reglas Access wildcard que interrumpan JMAP; no colocar una página HTML de Cloudflare Access en una API que espera respuesta JSON salvo integración compatible.

## 8. Crear buzones y probar correo

Crear usuario normal \`contacto@example.com\` en Stalwart; no usar cuenta de superadmin para correo cotidiano. Dar contraseña fuerte. Conectar clientes:

- IMAP: \`mail.example.com:993\`, SSL/TLS, usuario dirección completa.
- SMTP envío: \`mail.example.com:465\`, SSL/TLS, usuario completo.
- SMTP entre servidores: TCP/25 sin autenticación de destinatarios externos, relay anónimo **rechazado**.

Verificar login:
\`\`\`bash
curl --silent --show-error --max-time 15 \
  --user 'contacto@example.com' \
  --output /dev/null \
  'imaps://mail.example.com/INBOX'
\`\`\`

Este comando solicita la contraseña en terminal. **No** incluirla en argumentos o logs. Enviar desde Gmail a ese buzón, y comprobar IMAP:
\`\`\`bash
curl --silent --show-error --max-time 20 \
  --user 'contacto@example.com' \
  --request 'SEARCH SUBJECT "Prueba desde Gmail"' \
  'imaps://mail.example.com/INBOX'
\`\`\`

Comprobar envío exterior desde un cliente SMTP/IMAP (Thunderbird o Curl). Confirmar **entrega real** en Gmail, incluyendo Spam y «Mostrar original» con SPF/DKIM/DMARC. Un «mensaje aceptado» por SMTP solo demuestra aceptación del servidor de salida.

Si desde tu conexión doméstica TCP/25 falla pero desde un verificador externo funciona, investigar **red del cliente**; no tocar Docker a ciegas.

## 9. Instalar Bulwark Webmail

Docs oficiales: https://bulwarkmail.org/docs/deployment/docker/compose

Para **implementación estándar de laboratorio, con JMAP HTTPS ya disponible**, usar el compose de ejemplo [Bulwark](../ejemplos/bulwark/compose.yaml). Elegir un tag estable existente, registrarlo e inspeccionar sus cambios antes de actualizar.

\`\`\`bash
mkdir -p ~/infra-correo/bulwark
cd ~/infra-correo/bulwark
# Copiar compose.yaml desde ejemplos/bulwark
docker compose config
docker compose up -d
docker compose ps
docker compose logs --tail 80
\`\`\`

La plantilla usa volumen \`bulwark-data\` en \`/app/data\`; un host port de loopback, no de Internet. En instalación con wizard, omitir \`JMAP_SERVER_URL\` para habilitarlo y acceder al token de primer uso de forma privada. Si se establece \`JMAP_SERVER_URL\`, el wizard queda omitido y deben configurarse de forma coherente administración/sesiones según documentación. No copiar credenciales del proyecto histórico.

Para **personalizar el código**, clonar https://github.com/bulwarkmail/webmail.git, usar una rama propia y \`upstream\`; ver guía de desarrollo del README oficial. Conservar fuente editable y respetar obligaciones **AGPL-3.0** de software modificado usado a través de red.

## 10. Configurar JMAP por HTTPS

Bulwark usa JMAP del servidor desde el navegador; el endpoint **debe ser accesible desde el navegador**, no solo desde la red Docker. Verificar \`https://mail.example.com/.well-known/jmap\` si Stalwart posee HTTPS público en 443 y certificado válido.

**Si 443 está ocupado** (como en el VPS histórico), no poner un nuevo proxy 443 que afecte otros servicios. El diseño documentado fue:

\`\`\`text
correo.example.com → Cloudflare Tunnel → Bulwark localhost:3100
jmap.example.com   → Cloudflare Tunnel → gateway JMAP localhost:3101 → Stalwart local
\`\`\`

Para reconstruir **fielmente** este gateway, se requiere exportar una **copia sanitizada** de \`deploy/gandy\` creado por Claude (nginx config, Compose, reglas de reescritura, env.example y scripts), revisando que no contenga secretos. Es incorrecto simular el gateway con un proxy genérico que expondría administración de Stalwart, o afirmar que la configuración original está aquí si no lo está. Consultar [tareas pendientes](09-estado-pendientes.md).

La ruta pública JMAP exige autenticación, CORS limitado al origen webmail, endpoints filtrados, protección contra ataques de fuerza bruta y configuración segura de cabeceras de proxy. No forzar Cloudflare Access HTML en endpoints API JMAP. Evitar \`Access-Control-Allow-Origin: *\` combinado con credenciales.

## 11. Pruebas finales

Usar [checklist completo](11-checklist-recreacion.md). Deben aprobarse DNS, PTR, TLS, conexión SMTP, envío/recepción real Gmail, login webmail, JMAP, adjuntos, persistencia tras reinicio y que otras apps sigan respondiendo.

El resultado esperado no es solo página HTTP 200: debe poder enviarse y leerse correo con una cuenta nueva. Documentar fecha, versiones exactas, capturas saneadas y salida de comandos sin secretos.

## 12. Qué falta para recrear exactamente el VPS original

Este repositorio contiene el historial y un laboratorio reproducible **genérico**, pero reproducir *idéntica* instancia de Claude/Gandy requiere aún:
- Copia revisada de \`/home/gandy/apps/bulwark-webmail/deploy/gandy/\` de Claude: Compose, Dockerfile modificado, nginx gateway, scripts de deployment/backup/rollback/check, README. **Excluir** \`.env.bulwark\`, backups, cuentas y tokens.
- Versiones/digests exactos de imágenes de Stalwart y nginx; versión del cloudflared; configuración no secreta de ACME y Tunnel.
- Procedimientos de restauración probados en VPS diferente.
- Ejecutar procedimiento completo en una **VM de laboratorio aislada**, con dominio propio y credenciales propias; guardar evidencias de que funciona.

En ningún caso copiar entre compañeros el volumen de buzones, su administrador ni claves privadas.

## Referencias

- [Stalwart — instalación Docker](https://www.stalw.art/docs/install/platform/docker/)
- [Bulwark — Compose](https://bulwarkmail.org/docs/deployment/docker/compose)
- [Bulwark — primer uso](https://bulwarkmail.org/docs/getting-started/installation)
- [Docker — instalar Engine Ubuntu](https://docs.docker.com/engine/install/ubuntu/)
- [Cloudflare — Tunnel](https://developers.cloudflare.com/cloudflare-one/connections/connect-networks/)
