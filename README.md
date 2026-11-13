# Inmobiliarias-Infra

Repositorio de **infraestructura**: docker-compose de producción y workflows de
despliegue (GitHub Actions) para la plataforma inmobiliaria.

El código fuente vive en el repo de **código** (`InmobiliariasCloud`, con las
carpetas `backend/` y `frontend-react/`).

## Arquitectura

```
GitHub Actions (repo de codigo)
   │  push a main (cambios en backend/** o frontend-react/**)
   ▼
Build y push de imagenes a Docker Hub  (mariacarrero/*:latest)
   │  repository_dispatch (deploy-backend | deploy-frontend)
   ▼
Este repo (Inmobiliarias-Infra) -> .github/workflows/deploy-*.yml
   │  SSH + scp
   ▼
EC2 Backend 54.88.89.184            EC2 Frontend 98.83.252.223
   db (postgres)                        caddy :443 -> nginx :80 (estaticos)
   rabbitmq + rabbitmq2 (cluster 2 nodos, informes)
   kafka + kafka2 + kafka3 (cluster KRaft 3 brokers, wishlist)
   inmobiliaria-service :8081
   wishlist-service :8082
   kafka-ui :8080 (administracion de topics y particiones)
        ^
        │ VPC Link (un NLB interno por microservicio)
   AWS API Gateway (HTTP API)
```

> La API **ya no es una aplicacion propia**: es un API Gateway de AWS. Se
> eliminaron el `api-gateway` de Spring Boot (puerto 8080) y el proxy `/api` de
> nginx. El navegador llama directo a `execute-api...amazonaws.com` y el CORS lo
> resuelve la propia API. Ver `backend/aws/README.md` en el repo de codigo.

## Flujo de trabajo

1. Haces `git push` a `main` del repo de código.
2. El workflow correspondiente (`backend.yml` o `frontend.yml`) construye las
   imágenes Docker y las sube a Docker Hub.
3. El mismo workflow dispara este repo con un `repository_dispatch`.
4. Los workflows `deploy-backend.yml` / `deploy-frontend.yml`:
   - generan el `.env` desde los secretos (`envsubst` sobre los `.template`),
   - copian por SCP el docker-compose + `.env` al EC2,
   - ejecutan `docker compose pull && up -d` por SSH.

Las imágenes son **públicas** en Docker Hub, así que los EC2 no necesitan
credenciales para descargarlas.

## Requisitos antes del primer deploy

### 1. Crear los repositorios en GitHub
- Repo de código: `Mariajose205/InmobiliariasCloud` (ya existe; el código fuente
  va aquí con los workflows `.github/workflows/backend.yml` y `frontend.yml`).
- Este repo de infra: créealo como **`Inmobiliarias-Infra`** y súbale todo el
  contenido de esta carpeta.

### 2. Imagen del frontend en Docker Hub
La primera vez hay que publicar `mariacarrero/inmobiliarias-frontend:latest`
(este repo de infra no lo construye; el workflow del código lo hará en el
primer push a `frontend-react/**`).

### 3. Secretos en GitHub

En el repo de **código** (`Settings -> Secrets and variables -> Actions`):

| Nombre                | Valor                                              |
|-----------------------|----------------------------------------------------|
| `DOCKER_HUB_USERNAME` | `mariacarrero`                                     |
| `DOCKER_HUB_TOKEN`    | Access token de Docker Hub (no la contraseña)      |
| `GH_PAT`              | Personal Access Token de GitHub con scope `repo`   |
| `DEPLOY_REPO`         | `Mariajose205/Inmobiliarias-Infra`                 |

Variables en el repo de **código** (una por fila, `Settings -> Variables`):

| Nombre                   | Valor                                                                 |
|--------------------------|-----------------------------------------------------------------------|
| `VITE_AZURE_CLIENT_ID`   | ID de la app de Entra ID                                              |
| `VITE_AZURE_AUTHORITY`   | `https://login.microsoftonline.com/<tenant>`                          |
| `VITE_AZURE_REDIRECT_URI`| `https://TU-DOMINIO` (debe ser HTTPS; ver paso 5)                    |
| `VITE_AZURE_API_SCOPE`   | `api://<client-id>/access_as_user`                                   |
| `VITE_API_BASE_URL`      | URL del API Gateway: `https://<api-id>.execute-api.<region>.amazonaws.com` |

> `VITE_API_BASE_URL` debe ser la salida `ApiInvokeUrl` del stack de
> CloudFormation (`backend/aws/api-gateway.yaml`). Sin barra final.

En el repo de **infra** (`Inmobiliarias-Infra`) secretos:

| Nombre                    | Valor                                         |
|---------------------------|-----------------------------------------------|
| `SSH_PRIVATE_KEY_BACKEND` | Contenido de `ClavesBackend.pem`              |
| `SSH_HOST_BACKEND`        | `54.88.89.184`                                |
| `SSH_USER_BACKEND`        | `ec2-user`                                    |
| `SSH_PRIVATE_KEY_FRONTEND`| Contenido de `ClavesFrontend.pem`             |
| `SSH_HOST_FRONTEND`       | `98.83.252.223`                               |
| `SSH_USER_FRONTEND`       | `ec2-user`                                    |
| `POSTGRES_DB`             | `inmobiliarias`                               |
| `POSTGRES_USER`           | `inmob_admin`                                 |
| `POSTGRES_PASSWORD`       | La misma del `.env` actual del backend         |
| `ADMIN_EMAIL`             | Correo del administrador (rol ADMIN)          |
| `ADMIN_OID`               | Object ID del administrador (opcional)         |
| `CORREDOR_EMAIL`          | Correo del corredor (rol CORREDOR)            |
| `CORREDOR_OID`            | Object ID del corredor                        |
| `RABBITMQ_USER`           | Usuario del broker RabbitMQ (no usar `guest`) |
| `RABBITMQ_PASS`           | Password del broker RabbitMQ                  |
| `SITENAME`                | `inmobiliariasduoc.duckdns.org`               |

> **Ya no se usan** `JWT_SECRET` y `ADMIN_PASSWORD`: pertenecían al antiguo
> `api-gateway` de Spring Boot, que emitía su propio JWT. Ahora el token lo
> emite Entra ID y lo valida API Gateway.
> **Tampoco** `API_GATEWAY_URL`: el frontend llama directo al API Gateway de AWS.

> Los valores de `POSTGRES_*` y los correos/oids de rol puedes copiarlos del
> `.env` que ya está en el EC2 backend (`/home/ec2-user/backend/.env`).

### 4. Abrir puertos en los Security Groups (AWS)
- **Backend** (54.88.89.184): puertos `8081` y `8082`, pero **solo desde el
  CIDR de la VPC / VPC Link / NLB**. Nunca `0.0.0.0/0` en 8081 ni 8082.
  Además, **solo desde My IP** (consolas de administración sin login propio o
  con credenciales del lab): `15672` y `15673` (RabbitMQ), `8025` (MailHog) y
  `8080` (kafka-ui). Nunca abrir esos a `0.0.0.0/0`.
- **Frontend** (98.83.252.223): puertos `80` (HTTP) y `443` (HTTPS).

### 5. Configurar Azure Entra ID (una sola vez)
El `redirectUri` que registres en la app de Entra ID debe coincidir **exactamente**
con `VITE_AZURE_REDIRECT_URI` y debe ser **HTTPS** (Azure lo exige para
dominios desplegados). Hoy el frontend corre en HTTP (puerto 80), por lo que:
- las páginas públicas cargan igual,
- el login (MSAL) no funcionará hasta subir HTTPS (CloudFront, ALB o
  Let's Encrypt) y registrar el dominado HTTPS en Azure.

Para producción con HTTPS sobre este mismo EC2 lo más simple es un proxy
`caddy` o `nginx` con Let's Encrypt apuntando al puerto 80/443 del frontend
(añadir después; queda documentado aquí para no olvidarlo).

## Deploy manual (sin tocar código)

Puedes disparar el deploy a mano desde GitHub:
`Actions -> Deploy Backend / Deploy Frontend -> Run workflow`.

> Si el paso "Copiar docker-compose y .env al EC2 backend" falla con
> `Permission denied`, revisa que los archivos del compose sean del usuario SSH
> del deploy (no de `root`). Por ejemplo:
> `sudo chown -R ec2-user:ec2-user /opt/inmobiliarias/backend/rabbitmq`.

## Clústeres del backend

### RabbitMQ (2 nodos)

`rabbitmq` (nodo `rabbit@rabbit1`) y `rabbitmq2` (nodo `rabbit@rabbit2`) forman
un clúster al arrancar, sin pasos manuales:

- comparten la **cookie Erlang** (`backend/rabbitmq/erlang.cookie`, copiada al
  volumen antes de iniciar el server porque la imagen oficial ya no acepta
  `RABBITMQ_ERLANG_COOKIE`),
- `rabbitmq2` se une por **peer discovery classic config**
  (`backend/rabbitmq/rabbitmq-cluster.conf`, montado como `20-cluster.conf`),
- `rabbitmq` NO lleva esa configuración: si ambos nodos esperaran mutuamente,
  el clúster nunca se formaría.

Consolas: nodo 1 en `:15672` y nodo 2 en `:15673` (ambas con regla de SG desde
My IP). En Overview se ven los 2 nodos del clúster.

### Kafka (3 brokers, KRaft)

`kafka`, `kafka2` y `kafka3` comparten `CLUSTER_ID` y el quórum de controllers
`1@kafka:9093,2@kafka2:9093,3@kafka3:9093` (mayoría 2 de 3). Los topics
declarados por `wishlist-service` tienen replication factor 3 y
`min.insync.replicas=2` (los productores usan `acks=all`).

Los puertos 9092/9093 **no se publican** al host: todo el tráfico es interno de
la red del compose. La administración se hace con kafka-ui (`:8080`, regla de
SG desde My IP) o `docker exec inmobiliarias-kafka /opt/kafka/bin/…`.

Si se agregan topics nuevos y quedan con RF=1, se sube a 3 con
`kafka-reassign-partitions.sh` (ver historial de este repo) y luego
`kafka-leader-election.sh --election-type preferred --all-topic-partitions`
para balancear los líderes.

## Resumen de novedades (digest)

Para no saturar al usuario con un correo por cada propiedad, `wishlist-service`
acumula las novedades de las comunas que sigue y las envía agrupadas:

```
Propiedad creada → propiedades-nuevas
   → PropiedadNuevaNotificador guarda 1 Novedad PENDIENTE por usuario
      (resumen_id = null, sin correo)
   → al juntar NOTIF_UMBRAL pendientes (6 por defecto):
        ResumenNovedades + 1 evento en notificaciones-sistema
        → NotificacionDispatchConsumer → 1 correo con TODAS las propiedades
   → barrido diario (NOTIF_CRON, 20:00 America/Santiago por defecto):
        agrupa lo que quedó pendiente (1..5) para que nadie quede sin aviso
```

- La app también agrupa: `GET /api/wishlist/novedades` devuelve una entrada por
  resumen (`wishlist_novedades_resumen`) con el detalle de sus propiedades, y
  `DELETE /api/wishlist/novedades/{id}` descarta el resumen completo.
- La clave Kafka del resumen es el **correo del destinatario**, para que los
  resúmenes de un mismo usuario se consuman en orden.
- Variables en `backend/docker-compose.yml`:

  | Variable | Default | Para qué |
  |---|---|---|
  | `NOTIF_UMBRAL` | `6` | pendientes que disparan el resumen inmediato |
  | `NOTIF_CRON` | `0 0 20 * * *` | barrido diario (formato Spring, con segundos) |
  | `NOTIF_ZONA` | `America/Santiago` | zona horaria del CRON |

- Para forzar el barrido sin esperar al CRON (solo ADMIN):

  ```bash
  curl -X POST -H "X-User: administrador@inmobiliariaduoc.onmicrosoft.com" \
    http://localhost:8082/api/admin/notificaciones/resumen
  ```

## Scripts útiles (en el EC2)

```bash
# Ver estado
sudo docker compose -p backend ps

# Ver logs
sudo docker logs -f inmobiliaria-service
sudo docker logs -f inmobiliarias-db

# Reiniciar solo el microservicio
sudo docker compose -p backend restart inmobiliaria-service

# Backup de la base de datos
docker exec inmobiliarias-db pg_dump -U inmob_admin inmobiliarias > backup_$(date +%F).sql
```

## API Gateway (AWS)

La API se despliega por CloudFormation con `backend/aws/api-gateway.yaml` del repo
de codigo. Necesita `VpcId`, `VpcLinkSubnetIds` y `BackendEndpointDnsName`. El
detalle paso a paso esta en `backend/aws/README.md`.

> El SG del EC2 backend debe permitir TCP 8081 **y 8082** desde el VPC Link. Sin
> el 8081, la API responde `504 Integration timeout`; sin el 8082, el target group
> de wishlist queda *unhealthy* y `/api/wishlist/*` responde `503 Service
> Unavailable` (incluso rutas públicas como `/api/wishlist/comunas`).