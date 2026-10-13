# Documentación AWS — cuenta 258344940817

Última actualización: **2026-10-13**. Credenciales CLI: perfil `default`, identidad **root** (ver recomendaciones).
Variables de entorno y secrets: [ENV.md](ENV.md). Comandos: [comandos.md](comandos.md). Guía paso a paso para replicarlo todo: [GUIA.md](GUIA.md). **Guía recomendada, en orden y probada: [GUIA-LOCAL-PASO-A-PASO.md](GUIA-LOCAL-PASO-A-PASO.md)** (LocalStack + Postgres con Docker Compose, servicio de avisos Spring Boot, Lambdas, SNS/SQS/triggers, paso a AWS real). Referencia más extensa: guía para construir SNS, SQS y las Lambdas a mano en LocalStack y llevarlos a AWS real, sin scripts: [GUIA-LOCALSTACK-MANUAL.md](GUIA-LOCALSTACK-MANUAL.md). Stack local y comandos `npm run …`: [local/README.md](local/README.md).

## Estructura de este directorio

| Ruta | Qué es |
|---|---|
| `package.json` | Todos los comandos (`npm run local:dev`, `db:check`…), iguales en Mac, Linux y Windows. |
| `scripts/` | Código de esos comandos: `lib/` (utilidades), `local/` (stack local), `db-check.js`. |
| `local/` | `docker-compose.yml` de LocalStack, `schema.sql`, `.env` (no versionado) y su README. |
| `bff/`, `lambdas/pg/`, `front/` | Repos de GitHub clonados (ver abajo). |
| `ENV.md`, `comandos.md` | Variables de entorno y bitácora de comandos AWS. |

## Los 4 repositorios

Todo el proyecto vive en cuatro repositorios de GitHub, de la cuenta `docentedev`. Cada uno se versiona y se despliega por separado. Cómo clonarlos, en la [GUIA.md](GUIA.md#22-clonar-los-repositorios).

| # | Repositorio | Carpeta local | Qué es |
|---|---|---|---|
| 1 | [cloud-enviroment-chati](https://github.com/docentedev/cloud-enviroment-chati) | `AWS/` (raíz) | Este repo: documentación, scripts multiplataforma (`npm run …`) y stack local. **Punto de entrada.** |
| 2 | [cloud-lambda-pg](https://github.com/docentedev/cloud-lambda-pg) | `lambdas/pg/` | Las dos Lambdas: `report-request-recorder/` y `report-generator/` (Node). |
| 3 | [cloud-s3-chati](https://github.com/docentedev/cloud-s3-chati) | `front/` | React + Vite, servido por S3 + CloudFront, login con Cognito. |
| 4 | [cloud-chati-bff](https://github.com/docentedev/cloud-chati-bff) | `bff/` | Spring Boot (Java 21): CRUD de productos directo a Supabase y `POST /api/reports`, que publica en SNS. |

Los repos #2, #3 y #4 están dentro de la carpeta del #1, pero cada uno tiene su propio `.git`; el #1 los ignora.

## Arquitectura

```
Front (Cognito, rol Admin) ──► API Gateway (HTTP API) ──► BFF en EC2
                                                           ├─ products CRUD ──► Supabase Postgres (products)
                                                           └─ POST /api/reports (solo Admin)
                                                                   │  {type:"report.requested", body:{…}}
                                                                   ▼
                                                        SNS report-events
                                                  (filtro body.type = report.requested)
                                              ┌─────────────────┴─────────────────┐
                                              ▼                                   ▼
                                   SQS report-requests                  SQS report-generation
                                   (DLQ, 3 reintentos)                  (DLQ, 3 reintentos)
                                              ▼                                   ▼
                                   Lambda report-request-recorder       Lambda report-generator
                                   guarda en report_requests            arma el informe → console.log
                                              ▼                         (email con SES: pendiente)
                                   Supabase Postgres
```

## Recursos

### us-east-1 — Frontend
| Recurso | Detalle |
|---|---|
| S3 `cloud-s3-chati` | Build Vite (5 objetos, ~327 KB). Public Access Block total; solo CloudFront (OAC) puede leer. Sin versionado. |
| CloudFront `E14JJ18WEECQCL` | `d3nchyae80v81y.cloudfront.net`, root object `index.html`, sin dominio propio. |

### us-east-2 — Backend
| Recurso | Detalle |
|---|---|
| EC2 `mi-instancia-de-ejemplo` | `i-054b42a6afc086e9e`, t3.micro, Amazon Linux, IP pública `3.138.140.224` (no es Elastic IP). Key pair `mi-llave-privada` (el `.pem` vive en `~/.ssh/`, nunca en un repo). Corre el BFF en Docker. SG `launch-wizard-2`: 22, 80 y 443 abiertos a `0.0.0.0/0`. |
| API Gateway HTTP `mi-pasarela-api` | `yxpnxitdnh`. Rutas: `GET /health` y `GET/POST/PUT/PATCH/DELETE /api/{proxy+}` → BFF por HTTP a la EC2. Log group `/aws/apigateway/mi-api-academica`. |
| SNS `report-events` | Dos suscripciones SQS, filtro `{"type":["report.requested"]}`, scope `MessageBody`, sin raw delivery. |
| SQS `report-requests` / `report-generation` | Visibility 90 s, retención 4 días, redrive a su DLQ tras 3 intentos. |
| SQS `report-requests-dlq` / `report-generation-dlq` | Retención 14 días. |
| Lambda `report-request-recorder` | nodejs22.x, 128 MB, 15 s. Env: `DATABASE_URL`. Trigger: `report-requests` (batch 5). |
| Lambda `report-generator` | nodejs22.x, 128 MB, 15 s. Sin env. Trigger: `report-generation` (batch 5). |

### IAM
| Entidad | Detalle |
|---|---|
| Roles de Lambda | `report-request-recorder-role`, `report-generator-role`: logs + recibir/borrar solo de su cola. |
| `EC2-DynamoDB-Access-Role` | Rol de la EC2. Conserva DynamoDB FullAccess/ReadOnly (ya sin uso) y suma la inline `publish-report-events` (`sns:Publish` sobre `report-events`). |
| `APIGatewayCloudWatchLogsRole` | Logs del API Gateway. |
| Usuarios | `app-dynamodb-user` (sin keys), `clientedos` y `consumer` (keys activas, sin permisos). |
| Root | MFA activo; **tiene access keys** (las que usa este CLI). |

## Historial de cambios
- **2026-10-13:** rediseño. Se eliminaron `transporte`, `miCola`, las Lambdas `miReceptor`, `productsCatalog` y `eventProcExample`, sus roles, sus log groups, las rutas `/productos` y `/event/proc/example`, y la tabla DynamoDB `Products`. Se crearon `report-events`, 4 colas (2 con DLQ), 2 Lambdas y 2 roles con permisos mínimos. El CRUD de productos quedó solo contra Supabase.
- Prueba de humo: un mensaje `SMOKE_TEST` publicado en `report-events` llegó a ambas Lambdas; `report-generator` generó el informe y `report-request-recorder` conectó a Supabase (ver pendientes).

## Pendientes
1. **Desplegar el BFF** con `POST /api/reports`: crea la tabla `report_requests` al arrancar (`ddl-auto=update`). Hasta entonces el recorder falla con `relation "report_requests" does not exist` y los mensajes terminan en `report-requests-dlq`.
2. Botón de informes en el front (solo rol Admin).
3. Email con SES en `report-generator` (`src/notifier.js`).
4. Rotar la contraseña de Supabase (expuesta en texto plano durante esta sesión).
5. Limpiar `EC2-DynamoDB-Access-Role` (quitar DynamoDB) y revisar el filtro de seguridad.

## Recomendaciones de seguridad
1. **Root con access keys:** crear un usuario IAM/SSO acotado y eliminar las keys de root.
2. **API Gateway → EC2 por HTTP plano** a una IP no fija: usar Elastic IP o ALB con HTTPS.
3. **SSH (22) abierto a Internet:** restringir a tu IP o usar SSM Session Manager.
4. Usuarios `clientedos` y `consumer` con keys activas sin permisos: eliminar si no se usan.
5. Docs antiguas en `lambdas/pg/docs/` y `DEPLOY.md` mencionan los recursos viejos (`miCola`, `transporte`, `productsCatalog`): están desactualizadas.
