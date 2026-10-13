# Guía: arquitectura de informes y despliegue en AWS

Cómo replicar a mano todo lo que se construyó: la arquitectura en AWS, los cambios de código en cada repositorio y los secrets y el orden de despliegue.

> **Paso 1 de todo: el entorno local.** Antes de tocar AWS, construye y prueba el sistema completo en tu máquina con **[GUIA-LOCAL-PASO-A-PASO.md](GUIA-LOCAL-PASO-A-PASO.md)** (LocalStack + Postgres con Docker Compose, el servicio de avisos, las Lambdas, SNS/SQS y los triggers, comando por comando, en Mac, Linux y Windows). Esta guía empieza donde esa termina: la arquitectura, el código de cada repo y el paso a AWS real.

Los identificadores van como marcadores (`<ACCOUNT_ID>`, `<EC2_IP>`, `TU-APP`…): sustitúyelos por los tuyos. Región: `us-east-2`.

## Los 4 repositorios

**Todo este proyecto vive en cuatro repositorios de GitHub, todos de la cuenta `docentedev`.** Cada uno se versiona y se despliega por separado:

| # | Repositorio | URL | Qué contiene | Carpeta local | Dónde se despliega |
|---|---|---|---|---|---|
| 1 | `cloud-enviroment-chati` | https://github.com/docentedev/cloud-enviroment-chati | Documentación (esta guía), scripts multiplataforma (`npm run …`) y el stack local | `AWS/` (la raíz) | No se despliega: es documentación y herramientas |
| 2 | `cloud-lambda-pg` | https://github.com/docentedev/cloud-lambda-pg | Las dos Lambdas: `report-request-recorder` y `report-generator` | `AWS/lambdas/pg/` | AWS Lambda (GitHub Actions) |
| 3 | `cloud-s3-chati` | https://github.com/docentedev/cloud-s3-chati | El front React: login con Cognito y el botón "Solicitar informe" | `AWS/front/` | S3 + CloudFront (GitHub Actions) |
| 4 | `cloud-chati-bff` | https://github.com/docentedev/cloud-chati-bff | El BFF Spring Boot: CRUD de productos y `POST /api/reports` | `AWS/bff/` | EC2 en un contenedor Docker, vía GHCR (GitHub Actions) |

El repo **#1 es el punto de entrada**: contiene las instrucciones y las herramientas para trabajar con los otros tres. Los repos #2, #3 y #4 están dentro de la carpeta del #1, pero cada uno tiene su propio `.git`; el #1 los ignora. Comandos para clonarlos en la [parte 2.2](#22-clonar-los-repositorios).

## Índice
1. [Visión general](#1-visión-general)
2. [Antes de empezar: requisitos y clonar los repos](#2-antes-de-empezar-requisitos-y-clonar-los-repos)
3. [Infraestructura en AWS (a mano)](#3-infraestructura-en-aws-a-mano)
4. [Cambios de código por repositorio](#4-cambios-de-código-por-repositorio)
5. [Secrets y orden de despliegue](#5-secrets-y-orden-de-despliegue)
6. [Entorno local](#6-entorno-local)
7. [Probar la API de avisos](#7-probar-la-api-de-avisos)
8. [Problemas frecuentes](#8-problemas-frecuentes)
9. [Lista de verificación final](#9-lista-de-verificación-final)
10. [Pendientes y seguridad](#10-pendientes-y-seguridad)
11. [Glosario](#11-glosario)

**Qué leer según lo que quieras hacer**

| Objetivo | Ve a |
|---|---|
| Entender qué hace el sistema | 1 |
| **Paso 1: construir y probar todo en local, comando por comando** | [GUIA-LOCAL-PASO-A-PASO.md](GUIA-LOCAL-PASO-A-PASO.md) |
| Montar el proyecto en una máquina nueva | 2 |
| Programar el BFF y el front contra el entorno local | 6 |
| Recrear la infraestructura en una cuenta de AWS | 3, 4 y 5 |
| Algo no funciona | 8 |

---

## 1. Visión general

```
Front (React, login Cognito, rol Admin)
   │  POST /api/reports  (JWT)
   ▼
API Gateway (HTTP API) ──► BFF (Spring Boot en EC2)
                             ├─ CRUD de productos ──► Supabase Postgres (tabla products)
                             └─ publica {type:"report.requested", body:{…}}
                                          ▼
                                   SNS report-events   (filtro por type, sobre el cuerpo)
                          ┌───────────────┴────────────────┐
                          ▼                                ▼
                SQS report-requests (+DLQ)       SQS report-generation (+DLQ)
                          ▼                                ▼
            Lambda report-request-recorder       Lambda report-generator
            guarda en report_requests            lee los 10 productos más caros
                          ▼                      y publica un aviso en la API de avisos
                  Supabase Postgres  ◄───────────────────┘ (lee products)
```

**Recorrido de una solicitud:** un usuario del grupo `Admin` pulsa "Solicitar informe" → el BFF valida el JWT y el rol, y publica el evento en SNS → SNS lo copia a las dos colas → una Lambda guarda la solicitud en `report_requests` y la otra arma un aviso con los 10 productos más caros y lo publica en la API de avisos.

### Decisiones de diseño
- **Dos colas independientes** desde un SNS maestro: guardar la solicitud y generar el informe no dependen entre sí.
- **Filtro sobre el cuerpo del mensaje** (`FilterPolicyScope=MessageBody`), no sobre atributos: el `type` va dentro del JSON.
- **DLQ con 3 reintentos** en cada cola (ver [glosario](#11-glosario)).
- **Un rol por Lambda**, con permisos mínimos (logs + su cola).
- **El BFF no usa claves de AWS**: usa el rol de la instancia EC2.
- **El informe es un aviso:** `report-generator` consulta los 10 productos más caros en Supabase y los publica con `POST /api/saas/notices` (`Authorization: ApiKey <token>`, cuerpo `{"text": "…"}`, de 1 a 2000 caracteres).
- **La API cuelga de `/api`:** `API_NOTIFICATION` es la URL del sitio (`https://TU-APP/`) y el código agrega `/api`; si ya termina en `/api`, no lo duplica. Sin ese prefijo, el nginx del sitio responde `405 Not Allowed` con una página HTML (así falló la primera vez).

### Contrato del mensaje (BFF → SNS)
```json
{
  "type": "report.requested",
  "body": {
    "requestId": "uuid generado por el BFF",
    "requester": "email del JWT (o cognito:username, o sub)",
    "requesterId": "sub del JWT",
    "reportType": "INVENTORY",
    "params": {},
    "requestedAt": "2026-10-13T06:00:00Z"
  }
}
```
SNS entrega a SQS sin *raw delivery*, así que SQS recibe además un sobre: `{"Type":"Notification","Message":"<el JSON de arriba como string>",…}`. Las Lambdas lo desenvuelven.

### Tabla de destino
```sql
CREATE TABLE IF NOT EXISTS report_requests (
  id           varchar(255) PRIMARY KEY,
  requester    varchar(255),
  requester_id varchar(255),
  type         varchar(255),
  params       jsonb,
  requested_at timestamptz
);
```
En AWS la crea el BFF al arrancar (`ddl-auto=update`, entidad `ReportRequest`). Ese SQL es para crearla a mano.

---

## 2. Antes de empezar: requisitos y clonar los repos

### 2.1 Requisitos
| Herramienta | Para qué | Comprobar |
|---|---|---|
| Node ≥ 20.12 | scripts, front y Lambdas | `node --version` |
| Java 21 | BFF | `java --version` |
| Docker (Docker Desktop en Mac/Windows) | Entorno local (paso 1) | `docker --version` |
| Un Postgres | Supabase en la nube; uno local para pruebas | `psql --version` (opcional) |
| AWS CLI v2 con credenciales | solo para la parte 3 (crear en AWS) | `aws sts get-caller-identity` |
| `gh` (GitHub CLI) | opcional | `gh --version` |

Recomendado: un usuario IAM con permisos acotados, no el usuario root.

### 2.2 Clonar los repositorios
Son los [4 repositorios](#los-4-repositorios) de arriba, que conviven en una misma carpeta. Los scripts esperan estos nombres de carpeta exactos (`bff`, `front`, `lambdas/pg`), así que clónalos así: el raíz (#1) primero, y los otros tres (#2, #3 y #4) dentro.

```bash
git clone https://github.com/docentedev/cloud-enviroment-chati.git AWS
cd AWS
git clone https://github.com/docentedev/cloud-chati-bff.git bff
git clone https://github.com/docentedev/cloud-s3-chati.git front
git clone https://github.com/docentedev/cloud-lambda-pg.git lambdas/pg
```

El formato es `git clone <url> <carpeta>`. Resultado:

| Repo en GitHub | Carpeta | Qué contiene |
|---|---|---|
| #1 `cloud-enviroment-chati` | `AWS/` (raíz) | Scripts (`scripts/`), stack local (`local/`) y esta documentación |
| #4 `cloud-chati-bff` | `AWS/bff/` | Backend Spring Boot |
| #3 `cloud-s3-chati` | `AWS/front/` | Frontend React + Vite |
| #2 `cloud-lambda-pg` | `AWS/lambdas/pg/` | `report-request-recorder/` y `report-generator/` |

`git clone` crea solo la carpeta intermedia `lambdas/`. Los tres repos de dentro están en el `.gitignore` del raíz: cada uno es independiente, así que haz `commit` y `push` desde su propia carpeta, no desde la raíz.

- **Con SSH** (si tienes la clave registrada en GitHub): `git clone git@github.com:docentedev/cloud-chati-bff.git bff`. Si usas un alias de `~/.ssh/config` (por ejemplo `github.com-personal`), reemplaza `github.com` por tu alias.
- **Repos privados:** `git clone` pedirá autenticación con la cuenta dueña (`docentedev`), no con una cuenta corporativa.

### 2.3 Después de clonar
```bash
cd AWS
npm install                          # dependencias de los scripts de la raíz
```
Los archivos `.env` (el de la raíz y `local/.env`) están ignorados por git: ahí van las credenciales, nunca en un commit. El entorno local se arma con la [guía del paso 1](GUIA-LOCAL-PASO-A-PASO.md).

---

## 3. Infraestructura en AWS (a mano)
Los comandos son bash. En Windows, usa WSL o Git Bash, o consulta la **versión PowerShell de los mismos comandos** en [GUIA-LOCAL-PASO-A-PASO.md](GUIA-LOCAL-PASO-A-PASO.md#1-requisitos-y-cómo-leer-la-guía) (secciones 6, 7 y 9), que usan los mismos recursos.

### 3.1 Variables
```bash
export AWS_REGION=us-east-2
export ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)
```
> En zsh, escribe siempre `${ACCOUNT_ID}` con llaves: `$ACCOUNT_ID:role` se interpreta como un modificador de variable y rompe el ARN.

### 3.2 SNS
```bash
TOPIC_ARN=$(aws sns create-topic --name report-events --query TopicArn --output text)
```

### 3.3 Colas SQS con DLQ, política para SNS y suscripción
Se repite para las dos colas: `report-requests` y `report-generation`.
```bash
for Q in report-requests report-generation; do
  DLQ_URL=$(aws sqs create-queue --queue-name "$Q-dlq" --attributes MessageRetentionPeriod=1209600 --query QueueUrl --output text)
  DLQ_ARN=$(aws sqs get-queue-attributes --queue-url "$DLQ_URL" --attribute-names QueueArn --query Attributes.QueueArn --output text)

  cat > "$Q-attrs.json" <<EOF
{"VisibilityTimeout":"90","MessageRetentionPeriod":"345600","RedrivePolicy":"{\"deadLetterTargetArn\":\"$DLQ_ARN\",\"maxReceiveCount\":\"3\"}"}
EOF
  Q_URL=$(aws sqs create-queue --queue-name "$Q" --attributes "file://$Q-attrs.json" --query QueueUrl --output text)
  Q_ARN=$(aws sqs get-queue-attributes --queue-url "$Q_URL" --attribute-names QueueArn --query Attributes.QueueArn --output text)

  # Permite que SNS escriba en la cola
  cat > "$Q-policy.json" <<EOF
{"Policy":"{\"Version\":\"2012-10-17\",\"Statement\":[{\"Effect\":\"Allow\",\"Principal\":{\"Service\":\"sns.amazonaws.com\"},\"Action\":\"sqs:SendMessage\",\"Resource\":\"$Q_ARN\",\"Condition\":{\"ArnEquals\":{\"aws:SourceArn\":\"$TOPIC_ARN\"}}}]}"}
EOF
  aws sqs set-queue-attributes --queue-url "$Q_URL" --attributes "file://$Q-policy.json"

  # Suscripción con filtro sobre el cuerpo
  aws sns subscribe --topic-arn "$TOPIC_ARN" --protocol sqs --notification-endpoint "$Q_ARN" \
    --attributes '{"FilterPolicy":"{\"type\":[\"report.requested\"]}","FilterPolicyScope":"MessageBody","RawMessageDelivery":"false"}'
done
```
El *visibility timeout* (90 s) debe ser mayor que el timeout de la Lambda (15 s).

### 3.4 Roles IAM de las Lambdas (mínimos)
```bash
TRUST='{"Version":"2012-10-17","Statement":[{"Effect":"Allow","Principal":{"Service":"lambda.amazonaws.com"},"Action":"sts:AssumeRole"}]}'

for PAIR in report-request-recorder:report-requests report-generator:report-generation; do
  FN=${PAIR%%:*}; Q=${PAIR##*:}
  aws iam create-role --role-name "$FN-role" --assume-role-policy-document "$TRUST"
  aws iam attach-role-policy --role-name "$FN-role" \
    --policy-arn arn:aws:iam::aws:policy/service-role/AWSLambdaBasicExecutionRole
  aws iam put-role-policy --role-name "$FN-role" --policy-name "consume-$Q" --policy-document \
    "{\"Version\":\"2012-10-17\",\"Statement\":[{\"Effect\":\"Allow\",\"Action\":[\"sqs:ReceiveMessage\",\"sqs:DeleteMessage\",\"sqs:GetQueueAttributes\"],\"Resource\":\"arn:aws:sqs:${AWS_REGION}:${ACCOUNT_ID}:$Q\"}]}"
done
sleep 12   # propagación de IAM
```

### 3.5 Lambdas
```bash
# Ambas necesitan la dependencia pg
(cd lambdas/pg/report-request-recorder && npm ci --omit=dev && zip -r ../recorder.zip .)
(cd lambdas/pg/report-generator        && npm ci --omit=dev && zip -r ../generator.zip .)

# Las variables van en archivos (las connection strings tienen símbolos que rompen la sintaxis corta)
echo '{"Variables":{"DATABASE_URL":"postgresql://USUARIO:CLAVE@HOST:5432/postgres"}}' > env-recorder.json
echo '{"Variables":{"API_NOTIFICATION":"https://TU-APP/","API_TOKEN_INTEGRATION":"crr_...","DATABASE_URL":"postgresql://USUARIO:CLAVE@HOST:5432/postgres"}}' > env-generator.json

aws lambda create-function --function-name report-request-recorder --runtime nodejs22.x \
  --handler index.handler --timeout 15 --memory-size 128 \
  --role "arn:aws:iam::${ACCOUNT_ID}:role/report-request-recorder-role" \
  --zip-file fileb://lambdas/pg/recorder.zip --environment file://env-recorder.json

aws lambda create-function --function-name report-generator --runtime nodejs22.x \
  --handler index.handler --timeout 15 --memory-size 128 \
  --role "arn:aws:iam::${ACCOUNT_ID}:role/report-generator-role" \
  --zip-file fileb://lambdas/pg/generator.zip --environment file://env-generator.json

# Triggers SQS → Lambda
aws lambda create-event-source-mapping --function-name report-request-recorder --batch-size 5 \
  --event-source-arn "arn:aws:sqs:${AWS_REGION}:${ACCOUNT_ID}:report-requests"
aws lambda create-event-source-mapping --function-name report-generator --batch-size 5 \
  --event-source-arn "arn:aws:sqs:${AWS_REGION}:${ACCOUNT_ID}:report-generation"

rm env-recorder.json env-generator.json   # no dejes secretos en disco
```

### 3.6 Permiso del BFF para publicar
El BFF corre en la EC2 y usa el rol de su instance profile (`<ROL_EC2>`):
```bash
aws iam put-role-policy --role-name <ROL_EC2> --policy-name publish-report-events --policy-document \
  "{\"Version\":\"2012-10-17\",\"Statement\":[{\"Effect\":\"Allow\",\"Action\":\"sns:Publish\",\"Resource\":\"$TOPIC_ARN\"}]}"
```
Si el BFF corre en un contenedor Docker, el metadata de la instancia debe aceptar 2 saltos de red, o el contenedor no obtendrá credenciales:
```bash
aws ec2 describe-instances --instance-ids <INSTANCE_ID> --query 'Reservations[].Instances[].MetadataOptions.HttpPutResponseHopLimit'
aws ec2 modify-instance-metadata-options --instance-id <INSTANCE_ID> --http-put-response-hop-limit 2   # solo si no es ≥ 2
```

### 3.7 Prueba de humo en la nube
```bash
aws sns publish --topic-arn "$TOPIC_ARN" --message '{"type":"report.requested","body":{"requestId":"smoke-1","requester":"test@example.com","requesterId":"sub-test","reportType":"SMOKE_TEST","params":{},"requestedAt":"2026-01-01T00:00:00Z"}}'
aws logs tail /aws/lambda/report-generator --since 5m
aws logs tail /aws/lambda/report-request-recorder --since 5m
```
Si el recorder falla con `relation "report_requests" does not exist`, la tabla aún no existe: la crea el BFF al arrancar con el código nuevo, o ejecuta el SQL de la parte 1. Los mensajes fallidos terminan en `report-requests-dlq` tras 3 intentos; se limpia con `aws sqs purge-queue`. Esa prueba publica un aviso real en la API: usa mejor la [parte 7](#7-probar-la-api-de-avisos) para probar solo la API.

### 3.8 Si migras desde la arquitectura anterior (limpieza)
La versión previa usaba el SNS `transporte`, la SQS `miCola` y las Lambdas `miReceptor`, `productsCatalog` y `eventProcExample`. Para retirarla, **después** de comprobar que lo nuevo funciona:
1. Quita del API Gateway las rutas `/productos` y `/event/proc/example`, y sus integraciones (`aws apigatewayv2 delete-route`, `delete-integration`).
2. Borra los *event source mappings* **antes** que las funciones; luego las funciones y sus log groups.
3. Borra la cola y el topic antiguos, y los roles de servicio de esas Lambdas (con sus políticas).
4. Si ya no usas DynamoDB, elimina la tabla y limpia el rol de la EC2.

---

## 4. Cambios de código por repositorio

### 4.1 BFF (`cloud-chati-bff`, Spring Boot)
1. **`build.gradle`:** agrega `software.amazon.awssdk:sns` (el BOM del SDK ya está importado) y `com.fasterxml.jackson.core:jackson-databind`.
2. **`application.properties`:**
   ```properties
   aws.sns.topic-arn=${SNS_TOPIC_ARN:arn:aws:sns:us-east-2:<ACCOUNT_ID>:report-events}
   aws.region=${AWS_REGION:us-east-2}
   spring.datasource.password=${DB_PASSWORD:}
   ```
   El valor por defecto vacío de `DB_PASSWORD` permite una base local sin clave y evita variables vacías en Windows.
3. **Archivos nuevos:** `config/SnsConfig` (bean `SnsClient`, sin claves) y el paquete `reports/`: `ReportController`, `ReportService`, `ReportEventPublisher`, `dto/ReportRequestDto`, `dto/ReportResponseDto`, `entity/ReportRequest`.
4. **Comportamiento clave:**
   - `POST /api/reports` con `@PreAuthorize("hasRole('Admin')")` → responde **202** con el `requestId`.
   - El solicitante sale del JWT: `email` → `cognito:username` → `sub`.
   - El publisher serializa `{type:"report.requested", body:{…}}` con su propio `ObjectMapper` y publica en SNS **sin atributos de mensaje**.
   - El rol sale del claim `cognito:groups`, que el conversor existente ya convierte en `ROLE_<grupo>`.
5. Compila: `./gradlew compileJava` (Windows: `gradlew.bat compileJava`).

### 4.2 Lambdas (`cloud-lambda-pg`)
Estructura:
```
lambdas/pg/
├── report-request-recorder/   index.js, package.json (pg)
├── report-generator/          index.js, src/{handler, report.service, product.repository, notifier}.js
└── .github/workflows/         deploy-report-request-recorder.yml, deploy-report-generator.yml
```
**`report-request-recorder`:** por cada registro SQS, desenvuelve el sobre SNS, exige `type === "report.requested"` y ejecuta `INSERT … ON CONFLICT (id) DO NOTHING` con `$5::jsonb`. Es idempotente: SQS entrega "al menos una vez". Se conecta con SSL (`ssl: {rejectUnauthorized:false}`) salvo que `DATABASE_SSL=false`.

**`report-generator`:** capas `handler → report.service → product.repository / notifier`.
- `product.repository.js`: `SELECT id, name, price, stock FROM products WHERE price IS NOT NULL ORDER BY price DESC LIMIT 10`.
- `report.service.js`: valida la solicitud y arma el texto: un encabezado con el tipo y el solicitante, y una línea por producto (`1. Nombre - $precio (stock N)`), recortado a 2000 caracteres.
- `notifier.js`: `POST {API_NOTIFICATION}/api/saas/notices` con `Authorization: ApiKey {API_TOKEN_INTEGRATION}` y `redirect: "manual"` (un redirect convertiría el POST en GET). Solo un **2xx con JSON** cuenta como entregado. Una redirección, una página HTML, un código distinto de 2xx o variables faltantes **lanzan error**: el mensaje se reintenta y, tras 3 intentos, queda en la DLQ en vez de perderse. Con `NOTIFY_MODE=log` (solo desarrollo local) imprime el aviso sin publicarlo. El token nunca se escribe en los logs.
- Variables de entorno de la Lambda: `API_NOTIFICATION`, `API_TOKEN_INTEGRATION`, `DATABASE_URL`.

Las dos Lambdas lanzan error ante cualquier fallo, y SQS se encarga de reintentar.

### 4.3 Front (`cloud-s3-chati`, React + Vite + `react-oidc-context`)
1. **`src/config.ts`:** `API_BASE = import.meta.env.VITE_API_BASE ?? '<URL del API Gateway>'`.
2. **`src/main.tsx`:** `authority`, `client_id` y `redirect_uri` desde `VITE_OIDC_*`, con los valores de Cognito como respaldo (producción no cambia).
3. **`src/components/ReportRequestPanel.tsx`:** selector de tipo + botón "Solicitar informe" → `POST {API_BASE}/api/reports` con `Authorization: Bearer <id_token>`; muestra el `requestId` o el error (403 si no es Admin).
4. **`src/pages/DashboardPage.tsx`:** muestra el panel solo si `cognito:groups` incluye `Admin`.
5. **`vite.config.ts`:** proxy de `/api` → `http://localhost:8080` agregando el header `X-Secret-Gateway` (solo en `npm run dev`).
6. **Solo local:** `front/.env.development.local` con `VITE_OIDC_AUTHORITY=http://localhost:9000`, `VITE_OIDC_CLIENT_ID=local-client`, `VITE_OIDC_REDIRECT_URI=http://localhost:5173/` y `VITE_API_BASE=`. Está ignorado por git y nunca entra al build de producción (`npm run local:dev` lo crea si falta).
7. Verifica: `npm run build`.

---

## 5. Secrets y orden de despliegue

### 5.1 Secrets de GitHub por repo
| Repo | Secrets / variables |
|---|---|
| BFF | `GH_CR_PAT`, `EC2_SSH_KEY`, `API_GATEWAY_SECRET`, `SUPABASE_DB_USER`, `SUPABASE_DB_PASSWORD`; variable `TARGET_HOST_IP` (la IP **vigente** de la EC2) |
| Lambdas | `AWS_ACCESS_KEY_ID`, `AWS_SECRET_ACCESS_KEY`, `AWS_REGION=us-east-2`, `LAMBDA_RECORDER_FUNCTION_NAME=report-request-recorder`, `LAMBDA_GENERATOR_FUNCTION_NAME=report-generator`, **`API_NOTIFICATION`** (URL del sitio de la app, `https://TU-APP/`), **`API_TOKEN_INTEGRATION`** (API key `crr_…`), **`DATABASE_URL`** (la de Supabase) |
| Front | `AWS_ACCESS_KEY_ID`, `AWS_SECRET_ACCESS_KEY`, `AWS_REGION`, `S3_BUCKET_NAME`, `CLOUDFRONT_DISTRIBUTION_ID` |

Los secrets de GitHub **solo existen dentro del workflow**. Para que lleguen a la Lambda:
- **`report-generator`:** su workflow copia `API_NOTIFICATION`, `API_TOKEN_INTEGRATION` y `DATABASE_URL` a las variables de entorno de la función con `update-function-configuration`. Ese comando **reemplaza todas** las variables, por eso las envía juntas. Si falta alguna, el paso se salta con un aviso (`::warning::`) y la Lambda conserva su configuración anterior.
- **`report-request-recorder`:** su workflow solo toca las variables si existe el secret `DATABASE_URL`; en ese caso deja únicamente `DATABASE_URL` (para Supabase basta: el SSL va activo por defecto).

Para crear un secret desde la terminal: `gh secret set NOMBRE --repo docentedev/<repo> --body "valor"` (requiere `gh` autenticado con la cuenta dueña del repo). También se puede desde la web: *Settings → Secrets and variables → Actions*.

### 5.2 Qué dispara cada push a `main`
| Repo | Workflow | Qué hace |
|---|---|---|
| BFF | `Deploy Spring Boot to EC2 via GHCR` | Compila con Gradle, sube la imagen a GHCR y reinicia el contenedor en la EC2 por SSH |
| Lambdas | un workflow por carpeta | Empaqueta la Lambda y actualiza su código (y, en el generator, sus variables) |
| Front | `Deploy React to S3` | Compila, sube a S3 e invalida CloudFront |

El del BFF tarda unos segundos en reiniciar: el API Gateway responde 503 mientras tanto.

### 5.3 Orden
1. Infraestructura en AWS (parte 3).
2. **BFF:** crea además `report_requests`. Comprueba que `GET /health` responda 200.
3. **Lambdas**, con los secrets ya creados.
4. **Front.**
5. Prueba real: entra con un usuario del grupo `Admin` y pulsa "Solicitar informe".

Para ver el estado de un despliegue: `gh run list --repo docentedev/<repo> --limit 1 | cat` (el `| cat` evita el paginador).

---

## 6. Entorno local

El entorno local (LocalStack, Postgres, el servicio de avisos, las Lambdas, SNS/SQS y los triggers) se construye a mano en **[GUIA-LOCAL-PASO-A-PASO.md](GUIA-LOCAL-PASO-A-PASO.md)**: es el paso 1 y no se repite aquí.

Para **programar el BFF y el front contra un entorno local** (con un OIDC falso en lugar de Cognito, un solo comando `npm run local:dev`), el repositorio trae un atajo con scripts de Node, documentado en [local/README.md](local/README.md).

---

## 7. Probar la API de avisos
Sirve para validar la URL y la API key **sin pasar por AWS**, y para diagnosticar fallos. Pon `API_NOTIFICATION` y `API_TOKEN_INTEGRATION` en el `.env` de la raíz (ignorado por git) y ejecuta:
```bash
npm run notify:test                                      # envía un aviso de prueba
npm run notify:test -- "Texto del aviso"
npm run notify:test -- --base https://otra-url.example   # prueba otra URL base sin editar el .env
npm run notify:test -- --dry-run                         # muestra qué enviaría, sin enviarlo
```
Hace `POST {API_NOTIFICATION}/api/saas/notices` e imprime el estado HTTP, un extracto de la respuesta y un diagnóstico. Nunca muestra el token (solo su largo).

| Respuesta | Qué significa |
|---|---|
| `201` con JSON | Aviso aceptado: aparece en la sección de avisos de la app |
| Página HTML (p. ej. `405 Not Allowed` de nginx) | La URL no responde como la API: revisa la base |
| `401` / `403` | La API key no es válida o no tiene permiso |
| `404` | La ruta `/api/saas/notices` no existe en esa base |
| `3xx` | Redirección: usa el destino como base (un redirect convierte el POST en GET) |
| Error de red | El dominio no existe, o no responde |

Código de salida: `0` aceptado, `1` rechazado o sin respuesta, `2` falta configuración.

---

## 8. Problemas frecuentes

| Síntoma | Causa | Solución |
|---|---|---|
| El recorder falla con `relation "report_requests" does not exist` | El BFF nuevo aún no arrancó contra esa base | Desplegar el BFF, o crear la tabla con el SQL de la parte 1 |
| 503 del API Gateway tras desplegar el BFF | El contenedor tarda en arrancar | Esperar y repetir `GET /health` |
| Los mensajes de informe no llegan a las colas | El filtro de la suscripción no incluye el `type` publicado | Revisar `FilterPolicy` y que el scope sea `MessageBody` |
| El aviso no llega y el log del generador dice `HTTP 405 … nginx` con una página HTML | La petición iba a `/saas/notices` en vez de `/api/saas/notices`: el nginx del sitio no acepta el POST | Ya corregido en el código. `API_NOTIFICATION` basta con la URL del sitio; se prueba con `npm run notify:test` |
| El log del generador dice `Faltan API_NOTIFICATION y/o API_TOKEN_INTEGRATION` | El workflow no copió los secrets a la Lambda (falta alguno de los tres) | Crear los tres secrets y relanzar el workflow del generator |
| Los mensajes se acumulan en una DLQ | La Lambda falló 3 veces con ese mensaje | Ver el log de la Lambda (`aws logs tail`), corregir y reenviar o purgar la DLQ |
| `Unknown options: --region …` en zsh | zsh no divide una variable con varios flags | Usar una función o repetir los flags |
| ARN con `ole/…` roto | `$ACCOUNT_ID:role` en zsh es un modificador | Usar `${ACCOUNT_ID}` con llaves |
| `git commit` falla al firmar | Firma GPG activa y no accesible desde un entorno restringido | Hacer el commit desde tu propia terminal |
| `gh secret` da 403 | `gh` está autenticado con otra cuenta | `gh auth login --hostname github.com --git-protocol https --web` con la cuenta dueña del repo, o crear los secrets desde la web |
| `gh run list` abre un paginador y se queda en `(END)` | `gh` usa `less` | Pulsa `q`, o agrega `\| cat` |

---

## 9. Lista de verificación final
- [ ] `aws sns list-subscriptions-by-topic` muestra las dos colas con filtro `report.requested` sobre `MessageBody`.
- [ ] Las dos Lambdas existen y sus mapeos están `Enabled`.
- [ ] `report-generator` tiene las tres variables (`aws lambda get-function-configuration --query 'keys(Environment.Variables)'`).
- [ ] El rol de la EC2 tiene `sns:Publish` y el hop limit del metadata es ≥ 2.
- [ ] `GET /health` del API responde 200 y `report_requests` existe en la base.
- [ ] `npm run notify:test` devuelve `OK aviso aceptado`.
- [ ] Un usuario `Admin` ve el panel "Informes"; uno que no es Admin no lo ve, y un `POST` directo da 403.
- [ ] Tras pulsar el botón: respuesta 202 con `requestId`, fila nueva en `report_requests` y un aviso con los 10 productos más caros en la app.
- [ ] Las DLQ están vacías.

---

## 10. Pendientes y seguridad
- **Rotar la contraseña de Supabase**, que se expuso en texto plano en variables de entorno durante el trabajo.
- **Dejar de usar root con access keys:** crear un usuario IAM o SSO acotado, y uno de despliegue solo con permisos de Lambda, S3 y CloudFront.
- **Quitar del BFF el log `DEBUG SECRETO`** de `SecretGatewayFilter`, que imprime el secreto de la pasarela, y bajar el log de Spring Security a `INFO` en producción.
- **API Gateway → EC2 va por HTTP plano** a una IP sin Elastic IP: usar una IP fija o un ALB con HTTPS.
- **SSH (22) abierto a Internet:** restringir a tu IP o usar SSM Session Manager.
- **Verificar `TARGET_HOST_IP`** del BFF: debe ser la IP actual de la EC2.
- **Avisos duplicados:** SQS entrega "al menos una vez", así que un reintento puede publicar el mismo aviso dos veces. Si la API de avisos lo permite, usar una clave de idempotencia con el `requestId`.
- **No versionar identificadores si el repo es público:** esta guía usa marcadores, pero `README.md`, `ENV.md` y `comandos.md` contienen identificadores reales de la infraestructura.

---

## 11. Glosario
- **SNS (Simple Notification Service):** servicio que recibe un mensaje y lo reparte a varios destinos (aquí, dos colas). Es el "altavoz".
- **SQS (Simple Queue Service):** cola donde los mensajes esperan hasta que alguien los procesa. Desacopla quien envía de quien consume.
- **DLQ (Dead Letter Queue):** cola de "mensajes muertos". Cuando un mensaje falla 3 veces (`maxReceiveCount`), SQS lo mueve ahí en vez de reintentarlo para siempre, y lo conserva 14 días para inspeccionarlo. Una DLQ con mensajes es la alarma de que algo falla.
- **Visibility timeout:** tiempo durante el cual un mensaje en proceso queda oculto para otros consumidores. Debe superar el timeout de la Lambda.
- **Event source mapping:** el "trigger" que conecta una cola SQS con una Lambda.
- **Raw message delivery:** opción de SNS que entrega el mensaje tal cual. Desactivada, SQS recibe además un sobre `{Type, Message, …}`.
- **OIDC / JWT:** estándar de inicio de sesión y token firmado que prueba quién eres y a qué grupos perteneces (aquí, `cognito:groups`).
- **BFF (Backend for Frontend):** servidor intermedio entre el front y los datos o servicios; aquí es el Spring Boot.
- **LocalStack:** emulador local de servicios de AWS (SNS, SQS, Lambda…); ver el [paso 1](GUIA-LOCAL-PASO-A-PASO.md).
- **Idempotente:** una operación que se puede repetir sin cambiar el resultado (un reintento no duplica la fila).
