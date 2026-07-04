# Backend Despachos — API REST Spring Boot

**Innovatech Chile | ISY1101 — EP3 (Orquestación y automatización en la nube)**

API REST de gestión de **despachos** (Spring Boot 3.4.4 + Java 17). En EP3 corre
como contenedor en **Amazon ECS (Fargate)**, con imagen en **Amazon ECR**,
conectada a **RDS MySQL 8.0**, detrás de un **Application Load Balancer** y con
**autoscaling** + **logs en CloudWatch**. Entrega continua con **GitHub Actions**.

---

## Stack

| Componente | Tecnología |
|---|---|
| Lenguaje / Framework | Java 17 · Spring Boot 3.4.4 |
| ORM | Spring Data JPA + Hibernate |
| Base de datos | **AWS RDS MySQL 8.0** (gestionada) |
| Contenedor | Docker multi-stage (JDK build → JRE runtime, no-root) |
| Registro de imágenes | **Amazon ECR** |
| Orquestación | **Amazon ECS (Fargate)** |
| Balanceo | **Application Load Balancer** (path `/api/v1/despachos*`) |
| Escalado | **ECS Target Tracking** (CPU 50%, 1–4 tareas) |
| Logs | **Amazon CloudWatch** (`/ecs/innovatech-backend-despachos`) |
| Secrets | **SSM Parameter Store** (password de BD como SecureString) |
| CI/CD | GitHub Actions |
| Docs API | Springdoc OpenAPI (Swagger) |

---

## Arquitectura (EP3)

```
Navegador → ALB (:80) ──/api/v1/despachos*──→ ECS Service (Fargate :8081)
                                                   │  secrets vía SSM
                                                   ▼
                                              RDS MySQL  (schema despachos_db)
```
El servicio no expone IP pública directa: el único punto de entrada es el ALB.
La BD solo acepta conexiones desde el Security Group de las tareas ECS.

---

## Configuración (variables de entorno)

La app lee la conexión a BD desde variables; en ECS llegan así:

| Variable | Origen en ECS | Valor |
|---|---|---|
| `DB_ENDPOINT` | **SSM** `/innovatech/db_endpoint` | endpoint del RDS |
| `DB_PASSWORD` | **SSM** `/innovatech/db_password` (SecureString) | contraseña maestra |
| `DB_PORT` | task def (environment) | `3306` |
| `DB_NAME` | task def (environment) | `despachos_db` |
| `DB_USERNAME` | task def (environment) | `admin` |

El JDBC trae `createDatabaseIfNotExist=true`, así que el schema `despachos_db`
se crea solo la primera vez. Ver `.env.example` para ejecución local.

---

## Ejecución local con Docker Compose

```bash
cp .env.example .env          # editar credenciales locales
docker compose up -d          # levanta API + MySQL local
docker compose logs -f backend-despachos
```
- Swagger UI: http://localhost:8081/swagger-ui.html
- Health (ALB usa este): http://localhost:8081/v3/api-docs

---

## Pipeline CI/CD (GitHub Actions → ECR → ECS)

Se activa con `push` a la rama **`deploy`** (`.github/workflows/deploy.yml`):

```
checkout
  → credenciales AWS (Learner Lab, con AWS_SESSION_TOKEN)
  → resolver ACCOUNT_ID/REGION en ecs-task-def.json
  → login Amazon ECR
  → docker build (contexto ./Springboot-API-REST-DESPACHO) + push (SHA + latest)
  → render task definition con la nueva imagen
  → deploy a ECS (rolling update, espera estabilidad)
```

### Secrets requeridos

| Secret | Para qué |
|---|---|
| `AWS_ACCESS_KEY_ID` / `AWS_SECRET_ACCESS_KEY` / `AWS_SESSION_TOKEN` | credenciales temporales del Learner Lab |

Se refrescan cada sesión con `innovatech-ep3-infra/update_secrets.py`.

### task definition (`ecs-task-def.json`)

Fargate 512 CPU / 1024 MB · `LabRole` como execution/task role · logs a
CloudWatch · `DB_ENDPOINT` y `DB_PASSWORD` inyectados desde SSM (bloque
`secrets`). `ACCOUNT_ID`/`REGION` son placeholders que el pipeline resuelve.

---

## Dockerfile — decisiones técnicas

1. **Multi-stage**: builder con JDK (~400 MB) → runtime con JRE (~170 MB), sin
   Maven ni código fuente en la imagen final.
2. **Usuario no-root** (`appuser`): principio de mínimo privilegio.
3. **Caché de capas**: dependencias Maven se cachean aparte del código.
4. **Health check** en `/v3/api-docs` (responde 200 sin redirección, apto para
   el health check del Target Group del ALB).

---

## Endpoints principales

| Método | Endpoint | Descripción |
|---|---|---|
| GET | `/api/v1/despachos` | Listar despachos |
| GET | `/api/v1/despachos/{id}` | Obtener por ID |
| POST | `/api/v1/despachos` | Crear |
| PUT | `/api/v1/despachos/{id}` | Actualizar / cerrar |
| DELETE | `/api/v1/despachos/{id}` | Eliminar |

A través del ALB: `http://<ALB-DNS>/api/v1/despachos`.

---

## Despliegue completo en AWS

Montaje paso a paso (ECR, RDS, ALB, cluster, servicios, autoscaling) en
[`innovatech-ep3-infra/GUIA-AWS-EP3.md`](../innovatech-ep3-infra/GUIA-AWS-EP3.md).

---

## Autores

- Tomás Escobar — ISY1101 | DuocUC 2026
- Matías Ampuero — ISY1101 | DuocUC 2026
