# TiendaPerrito

Aplicación web de gestión de productos para una tienda de alimentos para mascotas. Desarrollada como práctica DevOps con contenedorización Docker, despliegue en AWS EC2 y pipeline CI/CD con GitHub Actions.

---

## Arquitectura general

```
Internet
   │
   ▼
[EC2 Frontend - subred pública]
  Nginx (puerto 80)
  Proxy /api/ → EC2 Backend
   │
   ▼
[EC2 Backend - subred privada]
  Node.js / Express (puerto 3001)
   │
   ▼
[EC2 Base de datos - subred privada]
  MySQL 8 (puerto 3306)
  Volumen Docker: dbdata
```

- El **Frontend** es el único componente accesible desde Internet.
- El **Backend** y la **Base de datos** están en subredes privadas, solo accesibles internamente.
- La comunicación Frontend → Backend se realiza a través del proxy inverso de Nginx.

---

## Estructura del proyecto

```
TiendaPerrito/
├── frontend/
│   ├── Dockerfile          # Multi-stage: builder (node) + producción (nginx)
│   ├── index.html          # Interfaz HTML del CRUD
│   ├── app.js              # Lógica JavaScript del frontend
│   └── default.conf        # Configuración Nginx con proxy inverso al backend
├── backend/
│   ├── Dockerfile          # Multi-stage: builder + producción con usuario no root
│   ├── server.js           # API REST Express (CRUD productos)
│   └── package.json        # Dependencias Node.js
├── db/
│   ├── Dockerfile          # Imagen MySQL 8 con script de inicialización
│   └── init.sql            # Creación de tabla productos e inserción de datos
├── .github/
│   └── workflows/
│       ├── cicd-tienda-frontend.yml   # Pipeline CI/CD del Frontend
│       ├── cicd-tienda-backend.yml    # Pipeline CI/CD del Backend
│       └── cicd-tienda-db.yml        # Pipeline CI/CD de la Base de datos
└── README.md
```

---

## Tecnologías utilizadas

| Capa | Tecnología |
|---|---|
| Frontend | HTML, JavaScript, Nginx Alpine |
| Backend | Node.js 18, Express, mysql2 |
| Base de datos | MySQL 8 |
| Contenedorización | Docker (multi-stage builds) |
| Registro de imágenes | Amazon ECR |
| Infraestructura | AWS EC2 |
| CI/CD | GitHub Actions + AWS SSM |

---

## Requisitos previos

- Docker Desktop instalado
- AWS CLI configurado
- Cuenta AWS Academy (Learner Lab)
- Repositorios ECR creados en AWS (uno por capa)
- Instancias EC2 activas con Docker y SSM Agent instalados
- Secrets configurados en GitHub (ver sección siguiente)

---

## Secrets de GitHub Actions requeridos

Configurar en: `Settings → Secrets and variables → Actions`

| Secret | Descripción |
|---|---|
| `AWS_ACCESS_KEY_ID` | Credencial AWS |
| `AWS_SECRET_ACCESS_KEY` | Credencial AWS |
| `AWS_SESSION_TOKEN` | Token de sesión AWS Academy |
| `AWS_REGION` | Región AWS (ej: `us-east-1`) |
| `ECR_REGISTRY` | URL base del registro ECR |
| `ECR_REPO_URL_FRONTEND` | URL completa repositorio ECR Frontend |
| `ECR_REPO_URL_BACKEND` | URL completa repositorio ECR Backend |
| `ECR_REPO_URL_DB` | URL completa repositorio ECR Base de datos |
| `EC2_FRONTEND_INSTANCE_ID` | ID instancia EC2 del Frontend (ej: `i-0abc123`) |
| `EC2_BACKEND_INSTANCE_ID` | ID instancia EC2 del Backend |
| `EC2_DB_INSTANCE_ID` | ID instancia EC2 de la Base de datos |
| `DB_HOST` | IP privada de la EC2 donde corre la base de datos |
| `DB_USER` | Usuario MySQL (ej: `root`) |
| `DB_PASSWORD` | Contraseña MySQL (ej: `admin123`) |

---

## Pipeline CI/CD

Cada capa tiene su propio workflow independiente. El pipeline se activa automáticamente cuando se hace `push` a `main` con cambios en la carpeta correspondiente.

### Flujo de cada pipeline

```
git push → GitHub Actions
              │
              ├─ 1. Checkout del código
              ├─ 2. Configurar credenciales AWS
              ├─ 3. Login en Amazon ECR
              ├─ 4. docker build (imagen de la capa)
              ├─ 5. docker tag + docker push → ECR
              └─ 6. aws ssm send-command → EC2
                        │
                        ├─ Login ECR dentro de la EC2
                        ├─ docker pull (nueva imagen)
                        ├─ docker stop + docker rm (contenedor anterior)
                        └─ docker run (nuevo contenedor)
```

### Triggers por carpeta

| Workflow | Se activa con cambios en |
|---|---|
| `cicd-tienda-frontend.yml` | `frontend/**` |
| `cicd-tienda-backend.yml` | `backend/**` |
| `cicd-tienda-db.yml` | `db/**` |

---

## Ejecutar localmente con Docker

### Solo el backend y la base de datos

```bash
# Construir y levantar la base de datos
cd db
docker build -t tienda-db .
docker run -d --name tienda-db -p 3306:3306 -v dbdata:/var/lib/mysql tienda-db

# Construir y levantar el backend
cd ../backend
docker build -t tienda-backend .
docker run -d --name tienda-backend -p 3001:3001 \
  -e DB_HOST=<IP_CONTENEDOR_DB> \
  -e DB_USER=root \
  -e DB_PASSWORD=admin123 \
  -e DB_NAME=tienda_perritos \
  tienda-backend

# Construir y levantar el frontend
cd ../frontend
docker build -t tienda-frontend .
docker run -d --name tienda-frontend -p 80:80 tienda-frontend
```

### Verificar contenedores activos

```bash
docker ps
```

### Ver logs de un contenedor

```bash
docker logs tienda-backend
docker logs tienda-frontend
docker logs tienda-db
```

---

## Validar la base de datos

```bash
docker exec -it tienda-db mysql -u root -padmin123 \
  -e "USE tienda_perritos; SELECT * FROM productos;"
```

---

## Endpoints del Backend

| Método | Ruta | Descripción |
|---|---|---|
| GET | `/api/productos` | Listar todos los productos |
| GET | `/api/productos/:id` | Obtener un producto por ID |
| POST | `/api/productos` | Crear un nuevo producto |
| PUT | `/api/productos/:id` | Actualizar un producto |
| DELETE | `/api/productos/:id` | Eliminar un producto |
| GET | `/api/health` | Estado del backend |

---

## Pasos pendientes para completar el despliegue

- [ ] Crear los 3 repositorios ECR en AWS
- [ ] Configurar los 14 secrets en GitHub Actions
- [ ] Actualizar `frontend/default.conf` con la IP privada real del backend
- [ ] Crear el archivo `docker-compose.yml` para levantar el stack completo
- [ ] Hacer push para activar los pipelines y publicar imágenes en ECR
- [ ] Verificar contenedores corriendo en cada EC2 con `docker ps`
- [ ] Probar la app desde el navegador con la IP pública del frontend

---

## Autores

Proyecto desarrollado para la asignatura **ISY1101 - Introducción a Herramientas DevOps**  
Duoc UC — 2025
