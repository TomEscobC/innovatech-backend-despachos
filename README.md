# Backend Despachos — API REST Spring Boot

**Innovatech Chile | ISY1101 EP2**

API REST para la gestión de despachos, desarrollada con Spring Boot 3.4.4 y Java 17. Containerizada con Docker y desplegada automáticamente en AWS EC2 mediante GitHub Actions.

---

## Stack Tecnológico

| Componente | Tecnología |
|---|---|
| Lenguaje | Java 17 |
| Framework | Spring Boot 3.4.4 |
| ORM | Spring Data JPA + Hibernate |
| Base de datos | MySQL 8.0 |
| Contenedorización | Docker (multi-stage build) |
| CI/CD | GitHub Actions |
| Registro de imágenes | Docker Hub |
| Despliegue | AWS EC2 |
| Documentación API | Springdoc OpenAPI (Swagger UI) |

---

## Estructura del Repositorio

```
back-Despachos_SpringBoot/
├── Springboot-API-REST-DESPACHO/     # Código fuente Spring Boot
│   ├── src/
│   │   └── main/
│   │       ├── java/com/citt/
│   │       │   ├── config/           # CORS, OpenAPI
│   │       │   ├── controller/       # DespachoController
│   │       │   ├── exceptions/       # Manejo de errores
│   │       │   └── persistence/      # Entidades, repositorios, servicios
│   │       └── resources/
│   │           └── application.properties
│   ├── Dockerfile                    # Multi-stage build
│   └── pom.xml
├── docker-compose.yml                # Stack completo (API + MySQL)
├── .env.example                      # Variables de entorno de referencia
├── .github/
│   └── workflows/
│       └── deploy.yml               # Pipeline CI/CD
└── README.md
```

---

## Variables de Entorno

Copiar `.env.example` a `.env` y completar con los valores reales. **Nunca commitear el archivo `.env`.**

| Variable | Descripción | Ejemplo |
|---|---|---|
| `DB_ENDPOINT` | Host del servidor MySQL | `db-despachos` (Docker) / IP RDS (producción) |
| `DB_PORT` | Puerto MySQL | `3306` |
| `DB_NAME` | Nombre de la base de datos | `despachos_db` |
| `DB_USERNAME` | Usuario de la base de datos | `app_user` |
| `DB_PASSWORD` | Contraseña del usuario | `password_seguro` |
| `MYSQL_ROOT_PASSWORD` | Contraseña root MySQL (solo Docker) | `root_password` |

---

## Ejecución Local con Docker Compose

### Requisitos
- Docker Desktop instalado y corriendo
- Git

### Pasos

```bash
# 1. Clonar el repositorio
git clone https://github.com/TU_USUARIO/back-despachos.git
cd back-despachos

# 2. Crear archivo de variables de entorno
cp .env.example .env
# Editar .env con tus valores

# 3. Levantar el stack completo (MySQL + API)
docker compose up -d

# 4. Verificar que los contenedores están corriendo
docker compose ps

# 5. Ver logs
docker compose logs -f backend-despachos
```

### Acceder a la API

- **Swagger UI:** http://localhost:8081/swagger-ui.html
- **API Base:** http://localhost:8081/api/despachos

### Detener el stack

```bash
docker compose down          # Detiene contenedores (datos persisten en el volumen)
docker compose down -v       # Detiene contenedores Y elimina volúmenes (borra la BD)
```

---

## Pipeline CI/CD — GitHub Actions

El pipeline se activa automáticamente con cada `push` a la rama `deploy`.

### Flujo

```
push → rama deploy
    └── JOB 1: Build & Push
        ├── Checkout del código
        ├── Login en Docker Hub
        ├── Build imagen Docker (multi-stage)
        └── Push a Docker Hub (tags: SHA + latest)
            └── JOB 2: Deploy en EC2
                ├── SSH a instancia EC2 Backend
                ├── Pull de la nueva imagen
                ├── Stop/rm del contenedor anterior
                ├── docker run con variables de entorno
                └── Verificación de estado
```

### Secrets requeridos en GitHub

Configurar en: `Settings → Secrets and variables → Actions`

| Secret | Descripción |
|---|---|
| `DOCKERHUB_USERNAME` | Usuario de Docker Hub |
| `DOCKERHUB_TOKEN` | Access token de Docker Hub |
| `EC2_BACKEND_HOST` | IP pública de la EC2 Backend |
| `EC2_USERNAME` | Usuario SSH de EC2 (`ec2-user` o `ubuntu`) |
| `EC2_SSH_PRIVATE_KEY` | Contenido completo del archivo `.pem` |
| `DB_ENDPOINT` | IP/hostname de la base de datos en EC2 |
| `DB_PORT` | Puerto MySQL (`3306`) |
| `DB_NAME` | Nombre de la base de datos |
| `DB_USERNAME` | Usuario de la base de datos |
| `DB_PASSWORD` | Contraseña de la base de datos |

### Hacer un deploy

```bash
# Desde la rama principal, hacer merge a deploy
git checkout deploy
git merge main
git push origin deploy
# El pipeline se activa automáticamente
```

---

## Dockerfile — Decisiones Técnicas

Se usa **multi-stage build** por las siguientes razones:

1. **Imagen de producción más pequeña:** el JDK (builder) pesa ~400MB, el JRE (runtime) ~170MB. La imagen final no incluye herramientas de compilación.
2. **Seguridad:** sin Maven ni código fuente en la imagen de producción.
3. **Usuario no-root:** el proceso corre como `appuser` en lugar de `root`, siguiendo el principio de mínimo privilegio.
4. **Limpieza de capas:** las dependencias se cachean en una capa separada al código fuente.

---

## Persistencia de Datos

Se usa **named volume** (`despachos-mysql-data`) en lugar de bind mount porque:

- Docker gestiona la ubicación del volumen → portable entre el equipo local y EC2
- No expone rutas del sistema de archivos del host
- Los datos de MySQL persisten aunque se destruya y recree el contenedor
- Compatible con AWS EBS si se necesita migrar a almacenamiento gestionado

---

## Endpoints Principales

| Método | Endpoint | Descripción |
|---|---|---|
| GET | `/api/despachos` | Listar todos los despachos |
| GET | `/api/despachos/{id}` | Obtener despacho por ID |
| POST | `/api/despachos` | Crear nuevo despacho |
| PUT | `/api/despachos/{id}` | Actualizar despacho |
| DELETE | `/api/despachos/{id}` | Eliminar despacho |

Documentación completa disponible en Swagger UI al ejecutar la aplicación.

---

## Autores

- Tomás [Apellido] — ISY1101 | DuocUC 2025
- Matías Ampuero — ISY1101 | DuocUC 2025
