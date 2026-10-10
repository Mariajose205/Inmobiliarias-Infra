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
EC2 Backend 34.204.64.107            EC2 Frontend 54.156.18.1
   db (postgres)                        caddy :443 -> nginx :80 (estaticos)
   inmobiliaria-service :8081
        ^
        │ VPC Link
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
| `SSH_HOST_BACKEND`        | `34.204.64.107`                               |
| `SSH_USER_BACKEND`        | `ec2-user`                                    |
| `SSH_PRIVATE_KEY_FRONTEND`| Contenido de `ClavesFrontend.pem`             |
| `SSH_HOST_FRONTEND`       | `54.156.18.1`                                 |
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
- **Backend** (34.204.64.107): puerto `8081`, pero **solo desde el VPC Link / NLB
  dentro de la VPC**. Nunca `0.0.0.0/0` en 8081, y el `8080` ya no se usa.
- **Frontend** (54.156.18.1): puerto `80` (HTTP) y más adelante `443` (HTTPS).

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

> El SG del EC2 backend debe permitir TCP 8081 desde el VPC Link. Sin eso, la API
> responde `504 Integration timeout`.