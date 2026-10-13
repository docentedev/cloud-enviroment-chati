# Atajo local con `npm run …` (BFF + front)

> **El entorno local de referencia se construye a mano en [GUIA-LOCAL-PASO-A-PASO.md](../GUIA-LOCAL-PASO-A-PASO.md)** (paso 1). Este documento es solo un atajo opcional con scripts de Node para **programar el BFF y el front** contra LocalStack, con un OIDC falso en lugar de Cognito.

Funciona igual en **Mac, Linux y Windows**: todos los comandos son Node (`npm run …` desde la raíz del proyecto).
Emula SNS, SQS y Lambda con LocalStack; la base de datos es la que indiques en `local/.env`.

```
BFF local ──► SNS report-events ─┬─► SQS report-requests ──► report-request-recorder ──► tu Postgres
                                 └─► SQS report-generation ─► report-generator (trae el top 10 de products y arma el aviso)
```

## Requisitos
- Node 20.12 o superior, y Docker (Docker Desktop en Mac/Windows).
- Java 21 para el BFF (el repo trae `gradlew` y `gradlew.bat`).
- Un Postgres accesible. Para uno local sin contraseña: `createdb reports_local` (viene con Postgres).

## Primera vez
```bash
npm install
cp local/.env.example local/.env      # Windows (PowerShell): Copy-Item local/.env.example local/.env
```
Edita `local/.env` y pon tu `DATABASE_URL`. Es la única variable obligatoria; el BFF y la Lambda derivan de ella usuario, clave y host.

```bash
npm run db:check      # valida la conexión y muestra qué tablas existen
npm run db:schema     # crea report_requests (no hace falta psql; el BFF también la crea al arrancar)
```

## Cada vez que programes
```bash
npm run local:up      # levanta LocalStack
npm run local:init    # crea topic, colas, DLQs, suscripciones y Lambdas (repetible)
npm run local:dev     # OIDC falso + BFF + front; Ctrl+C detiene todo
```
`local:up` y `local:init` solo se repiten si reiniciaste Docker (LocalStack pierde los recursos al reiniciar).

Abre http://localhost:5173 y pulsa "Ingresar con Cognito": entra de inmediato como `admin@local.test` con el grupo `Admin`.

## Comandos
| Comando | Qué hace |
|---|---|
| `npm run local:up` / `local:down` | Enciende / apaga LocalStack. |
| `npm run local:init` | Crea los recursos y las dos Lambdas en LocalStack. Repetible; recrea las Lambdas con el código actual. |
| `npm run local:dev` | Levanta OIDC falso, BFF y front con las variables ya resueltas. |
| `npm run local:oidc` | Solo el OIDC falso (si prefieres arrancar BFF y front a mano). |
| `npm run local:send-test` | Publica una solicitud en el SNS local y muestra los logs de ambas Lambdas. |
| `npm run db:check` | Valida una credencial de Postgres/Supabase: `npm run db:check -- "postgresql://user:clave@host:5432/postgres"`. |
| `npm run db:schema` | Crea `report_requests` en la base de `DATABASE_URL`. |

La prueba `local:send-test` escribe una fila con `type = 'LOCAL_TEST'`. Bórrala con:
```sql
DELETE FROM report_requests WHERE type = 'LOCAL_TEST';
```

## Probar el 403 de un no-admin
Agrega `MOCK_GROUPS=Colaborador` a `local/.env`, reinicia `local:dev`, cierra sesión en el navegador y entra de nuevo: el panel de informes no aparece y un `POST /api/reports` directo responde 403. Otros valores: `MOCK_EMAIL`, `MOCK_SUB`, `MOCK_PORT`.

## Usar Supabase en vez de Postgres local
En `local/.env`: la URL de Supabase y quita `DATABASE_SSL=false` (Supabase exige SSL). Las pruebas escribirán filas reales.

## Cómo funciona por dentro
- **Autenticación:** el OIDC falso emite tokens RS256 con `email` y `cognito:groups`. El BFF los valida de verdad (firma, issuer, rol); solo cambia a quién le pregunta, con `SPRING_SECURITY_OAUTH2_RESOURCESERVER_JWT_ISSUER_URI`. Escucha en `127.0.0.1` porque un BFF con `-Djava.net.preferIPv4Stack=true` no alcanza un servidor enlazado solo a `::1`.
- **Front:** `npm run dev` usa `front/.env.development.local` (lo crea `local:dev` si falta; solo aplica a `vite` en desarrollo, nunca al build de producción). Un proxy de Vite reenvía `/api` al BFF con el header `X-Secret-Gateway`, porque el BFF no tiene CORS propio.
- **Lambda → base local:** `localhost` se reescribe a `host.docker.internal` para que la Lambda, que corre en un contenedor, alcance tu base.

## El aviso del informe en local
`report-generator` lee los 10 productos más caros de la tabla `products` de tu base y arma un aviso. En local, por defecto **solo lo imprime** en su log (`NOTIFY_MODE=log`), sin llamar a ninguna API. Para verlo, crea antes algún producto en tu base local (el BFF crea la tabla `products` al arrancar) y ejecuta `npm run local:send-test`.

Para publicarlo de verdad en la API de avisos, define en `local/.env` estas dos variables y repite `npm run local:init`:
```
API_NOTIFICATION=https://tu-api.example.com/
API_TOKEN_INTEGRATION=crr_...
```
El token es un secreto: `local/.env` está ignorado por git, y nunca debe subirse.

## Notas
- La primera invocación de una Lambda descarga la imagen de runtime; puede tardar ~1 minuto.
- Versión de LocalStack fijada (`3.8.1`): las recientes pueden exigir `LOCALSTACK_AUTH_TOKEN`.
- Las Lambdas locales usan `nodejs20.x` (en AWS, `nodejs22.x`); el código es el mismo.
- No probado en Windows ni Linux: el diseño evita sintaxis de shell y usa `gradlew.bat`, `taskkill` y `shell: true` donde corresponde, pero conviene validarlo en cada sistema.
