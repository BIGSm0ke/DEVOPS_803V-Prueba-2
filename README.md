# DEVOPS_803V-Prueba-2 — Innovatech Platform

## Arquitectura

```
                              ┌─────────────────┐
                              │   Internet      │
                              └────────┬────────┘
                                       │ :80
                              ┌────────▼────────┐
                              │  ALB Público    │
                              │ Innovatech-alb  │
                              │ Path-based      │
                              │ routing         │
                              └──┬────┬────┬────┘
                    ┌────────────┘    │    └────────────┐
                    │Default          │/api/v1/ventas*  │/api/v1/despachos*
               ┌────▼────┐      ┌────▼────┐      ┌────▼────┐
               │ front-tg│      │ventas-tg│      │despachos│
               │ :80     │      │ :8080   │      │-tg:8081 │
               └────┬────┘      └────┬────┘      └────┬────┘
                    │                │                │
               ┌────▼────┐      ┌────▼────┐      ┌────▼────┐
               │ECS Front│      │ECS Ventas│      │ECS Desp │
               │1-3 tasks│      │1 task    │      │1 task   │
               │Fargate  │      │Fargate   │      │Fargate  │
               └─────────┘      └────┬─────┘      └────┬─────┘
                                     │                 │
                                     └──────┬──────────┘
                                            │ :3306
                                     ┌──────▼──────┐
                                     │  RDS MySQL  │
                                     │ innovatech-db│
                                     │ privado     │
                                     └─────────────┘
```

## Componentes

| Componente | Tecnología | Puerto | Auto Scaling |
|---|---|---|---|
| **Frontend** | nginx (React/Angular) | 80 | Sí (CPU 50%, min 1, max 3) |
| **Backend Ventas** | Spring Boot (Java 17) | 8080 | No |
| **Backend Despachos** | Spring Boot (Java 17) | 8081 | No |
| **Base de Datos** | MySQL 8.4 (RDS db.t3.micro) | 3306 | No |
| **ALB** | Application Load Balancer | 80 | N/A |

## Recursos AWS

| Recurso | Nombre | Detalle |
|---|---|---|
| **VPC** | `Innovatech Chile` (10.0.0.0/20) | 6 subredes (3 públicas, 3 privadas) |
| **Cluster ECS** | `cluster-Innovatech` | Fargate + Fargate Spot |
| **ALB** | `Innovatech-alb` | Público, listener :80, path-based routing |
| **RDS** | `innovatech-db` | MySQL 8.4, 20GB gp2, privado |
| **ECR** | 3 repositorios | `innovatech-frontend`, `innovatech-back-ventas`, `innovatech-back-despachos` |
| **SSM** | `/innovatech/db-password` | SecureString con contraseña BD |

## ALB Routing

| Priority | Path Pattern | Target Group | Puerto |
|---|---|---|---|
| 10 | `/api/v1/ventas*` | `ventas-tg` | 8080 |
| 20 | `/api/v1/despachos*` | `despachos-tg` | 8081 |
| Default | `/*` | `front-tg` | 80 |

## Endpoints

- **Frontend:** `http://Innovatech-alb-1870642275.us-east-1.elb.amazonaws.com/`
- **Ventas API:** `http://Innovatech-alb-1870642275.us-east-1.elb.amazonaws.com/api/v1/ventas`
- **Despachos API:** `http://Innovatech-alb-1870642275.us-east-1.elb.amazonaws.com/api/v1/despachos`

## CI/CD (GitHub Actions)

3 pipelines independientes en `.github/workflows/`:

| Workflow | Trigger | Build | Push | Deploy |
|---|---|---|---|---|
| `frontend.yml` | Push a `deploy` con cambios en `front_despacho/**` | Node.js 20, lint, build | ECR `innovatech-frontend` | ECS force-new-deployment |
| `backend-ventas.yml` | Push a `deploy` con cambios en `back-Ventas_SpringBoot/**` | Maven (JDK 17) | ECR `innovatech-back-ventas` | ECS force-new-deployment |
| `backend-despachos.yml` | Push a `deploy` con cambios en `back-Despachos_SpringBoot/**` | Maven (JDK 17) | ECR `innovatech-back-despachos` | ECS force-new-deployment |

### Secrets requeridos en GitHub

| Secret | Descripción |
|---|---|
| `AWS_ACCESS_KEY_ID` | AWS Access Key (IAM con permisos ECR+ECS) |
| `AWS_SECRET_ACCESS_KEY` | AWS Secret Key |
| `AWS_SESSION_TOKEN` | AWS Session Token (si aplica, ej. Vocareum) |

## Base de Datos (RDS)

- **Endpoint:** `innovatech-db.cysmgjjlzbzq.us-east-1.rds.amazonaws.com`
- **Puerto:** 3306
- **Usuario:** `admin`
- **Contraseña:** Almacenada en SSM Parameter Store (`/innovatech/db-password`)
- **Esquema:** Se crea automáticamente via `createDatabaseIfNotExist=true` y `ddl-auto=update`

## Variables de Entorno (ECS Task Definitions)

| Variable | Fuente | Uso |
|---|---|---|
| `DB_ENDPOINT` | Environment | Endpoint RDS |
| `DB_PORT` | Environment | 3306 |
| `DB_NAME` | Environment | `innovatech_db` |
| `DB_USERNAME` | Environment | `admin` |
| `DB_PASSWORD` | SSM Parameter Store (`/innovatech/db-password`) | Contraseña RDS |

## Health Checks

| Target Group | Path | Códigos aceptados | Intervalo | Threshold |
|---|---|---|---|---|
| `front-tg` | `/` | 200 | 30s | 3 healthy |
| `ventas-tg` | `/swagger-ui.html` | 200, 302 | 30s | 3 healthy |
| `despachos-tg` | `/swagger-ui.html` | 200, 302 | 30s | 3 healthy |
