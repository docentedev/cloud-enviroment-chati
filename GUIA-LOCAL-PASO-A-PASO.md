# Guía paso a paso: el sistema completo en local (LocalStack + Postgres) y cómo llevarlo a AWS real

Esta guía construye **todo a mano**, en orden, con cada comando explicado. Sirve para Mac, Linux y Windows (cada bloque trae su versión **bash** y su versión **PowerShell** cuando difieren).

> **LocalStack, no OpenStack.** LocalStack es el emulador de AWS que corre en Docker (SNS, SQS, Lambda…). OpenStack es otra cosa (una nube privada), y no se usa aquí.

## Qué vamos a construir

```
 curl / BFF ──► SNS topic "report-events"
                  │  (filtro por el campo "type" del mensaje)
                  ├─► SQS report-requests   ──► Lambda 1 "report-request-recorder" ──► Postgres (tabla report_requests)
                  │        └─ DLQ                  guarda la solicitud
                  └─► SQS report-generation ──► Lambda 2 "report-generator"        ──► API /api/notice (Spring Boot)
                           └─ DLQ                  lee los 10 productos más caros de Postgres y publica el aviso
```

| Pieza | En local | En AWS real |
|---|---|---|
| SNS, SQS, Lambda | LocalStack (Docker) | AWS |
| Postgres | contenedor Postgres (Docker) | Supabase |
| Servicio de avisos `/api/notice` | proyecto Spring Boot en tu máquina | `https://api.claudioviajando.com/api/notice` |

**Índice**
1. [Requisitos y cómo leer la guía](#1-requisitos-y-cómo-leer-la-guía)
2. [Docker Compose: LocalStack y Postgres](#2-docker-compose-localstack-y-postgres)
3. [Configurar el CLI hacia LocalStack](#3-configurar-el-cli-hacia-localstack)
4. [El servicio de avisos: proyecto Spring Boot con `/api/notice`](#4-el-servicio-de-avisos-proyecto-spring-boot-con-apinotice)
5. [Escribir las dos Lambdas](#5-escribir-las-dos-lambdas)
6. [Desplegar las Lambdas](#6-desplegar-las-lambdas)
7. [SNS, SQS, filtros y triggers](#7-sns-sqs-filtros-y-triggers)
8. [Probar todo de punta a punta](#8-probar-todo-de-punta-a-punta)
9. [Pasar a AWS real](#9-pasar-a-aws-real)
10. [Apagar, limpiar y errores frecuentes](#10-apagar-limpiar-y-errores-frecuentes)

---

## 1. Requisitos y cómo leer la guía

**Instala:** Docker (en Windows, Docker Desktop con WSL2), AWS CLI v2 (`aws --version`, 2.13 o superior), Node 20 o superior (`node -v`) y Java 21 (`java --version`).

**Qué terminal usar**

| Sistema | Terminal | Versión de los comandos |
|---|---|---|
| Mac / Linux | Terminal | **bash** (también vale zsh) |
| Windows | WSL2 (recomendada) | **bash**, sin cambios |
| Windows | Git Bash | **bash**, tras ejecutar `export MSYS_NO_PATHCONV=1` (evita que Git Bash deforme argumentos que empiezan con `/`) |
| Windows | PowerShell | **PowerShell** |

**Diferencias que ya están aplicadas en los bloques PowerShell**

| bash | PowerShell |
|---|---|
| `export VAR=valor` | `$env:VAR = "valor"` (variable de entorno) o `$VAR = "valor"` (solo de la sesión) |
| `X=$(comando)` | `$X = comando` |
| `\` al final de línea | `` ` `` (acento grave) |
| `cat > f <<'EOF' … EOF` | `@' … '@ \| Set-Content -Encoding ascii f` (con `@" … "@` se sustituyen las variables; con `@' … '@`, no) |
| `zip` | `tar -a -c -f archivo.zip …` |
| `curl` | `curl.exe` (en PowerShell `curl` es otro comando) |
| `a && b` | `a; b` |
| JSON dentro del comando | JSON en un **archivo** y `file://archivo` (PowerShell 5.1 deforma las comillas) |

**Regla en Windows:** todo JSON que se pasa al CLI se guarda en un archivo con `Set-Content -Encoding ascii` (así no se agrega la marca BOM) y se referencia con `file://`.

> **Estado de la guía:** los bloques **bash** se ejecutaron tal cual, de principio a fin, contra LocalStack 3.8.1 (resultado en la sección 8). Los bloques **PowerShell** están validados solo en su estructura: no se han ejecutado en una máquina Windows. Si alguno falla, anota el comando y el error.

**Carpeta de trabajo.** Todo se crea dentro de una carpeta nueva, **fuera de cualquier repositorio** (la guía genera archivos con claves de ejemplo y un `git add -A` descuidado los subiría):

**bash**
```bash
mkdir -p ~/aws-local && cd ~/aws-local
mkdir db lambdas
```
**PowerShell**
```powershell
mkdir $HOME\aws-local; cd $HOME\aws-local
mkdir db, lambdas
```
- `mkdir -p`: crea la carpeta aunque ya exista, sin error.
- `db` guardará el SQL inicial de la base; `lambdas`, el código de las dos funciones.

**Puertos que usa la guía**

| Puerto | Qué |
|---|---|
| 4566 | LocalStack (SNS, SQS, Lambda, logs… todo en un solo puerto) |
| 5433 | Postgres del contenedor (no es el 5432, para no chocar con un Postgres que ya tengas en tu máquina) |
| 8082 | El servicio de avisos (Spring Boot) |

---

## 2. Docker Compose: LocalStack y Postgres

Un solo archivo levanta los dos servicios. Primero el SQL que crea las tablas y carga productos de ejemplo; Postgres lo ejecuta solo la primera vez que arranca.

**bash**
```bash
cat > db/init.sql <<'EOF'
-- Solicitudes de informe: las guarda la Lambda 1.
CREATE TABLE IF NOT EXISTS report_requests (
  id           varchar(255) PRIMARY KEY,
  requester    varchar(255),
  requester_id varchar(255),
  type         varchar(255),
  params       jsonb,
  requested_at timestamptz
);

-- Productos: de aquí lee la Lambda 2 los 10 más caros.
CREATE TABLE IF NOT EXISTS products (
  id    bigserial PRIMARY KEY,
  name  varchar(255) NOT NULL,
  price numeric(12,2),
  stock integer
);

INSERT INTO products (name, price, stock) VALUES
  ('Notebook Pro 16',      1899990, 5),  ('Monitor 4K 32',        459990, 12),
  ('Teclado mecánico',       89990, 40),  ('Mouse inalámbrico',     29990, 80),
  ('Silla ergonómica',      249990, 9),   ('Escritorio regulable', 389990, 7),
  ('Auriculares ANC',       179990, 25),  ('Webcam Full HD',        59990, 33),
  ('Disco SSD 2TB',         129990, 18),  ('Impresora láser',      219990, 6),
  ('Router WiFi 6',          99990, 21),  ('Tablet 11',            329990, 14),
  ('Parlantes 2.1',          69990, 27),  ('Cable HDMI 2m',          5990, 150),
  ('Hub USB-C',              39990, 60);
EOF
```
**PowerShell**
```powershell
@'
-- Solicitudes de informe: las guarda la Lambda 1.
CREATE TABLE IF NOT EXISTS report_requests (
  id           varchar(255) PRIMARY KEY,
  requester    varchar(255),
  requester_id varchar(255),
  type         varchar(255),
  params       jsonb,
  requested_at timestamptz
);

-- Productos: de aquí lee la Lambda 2 los 10 más caros.
CREATE TABLE IF NOT EXISTS products (
  id    bigserial PRIMARY KEY,
  name  varchar(255) NOT NULL,
  price numeric(12,2),
  stock integer
);

INSERT INTO products (name, price, stock) VALUES
  ('Notebook Pro 16',      1899990, 5),  ('Monitor 4K 32',        459990, 12),
  ('Teclado mecanico',       89990, 40),  ('Mouse inalambrico',     29990, 80),
  ('Silla ergonomica',      249990, 9),   ('Escritorio regulable', 389990, 7),
  ('Auriculares ANC',       179990, 25),  ('Webcam Full HD',        59990, 33),
  ('Disco SSD 2TB',         129990, 18),  ('Impresora laser',      219990, 6),
  ('Router WiFi 6',          99990, 21),  ('Tablet 11',            329990, 14),
  ('Parlantes 2.1',          69990, 27),  ('Cable HDMI 2m',          5990, 150),
  ('Hub USB-C',              39990, 60);
'@ | Set-Content -Encoding ascii db/init.sql
```
- `report_requests`: una fila por solicitud. `id` es la clave: sirve para que guardar dos veces el mismo mensaje no duplique la fila.
- `products`: la tabla que ya existe en producción (Supabase); aquí se crea con las columnas que la Lambda 2 usa (`name`, `price`, `stock`).
- (La versión PowerShell no lleva tildes, para que el archivo ASCII sea válido en cualquier consola.)

Ahora el `docker-compose.yml`:

**bash**
```bash
cat > docker-compose.yml <<'EOF'
name: aws-local

services:
  localstack:
    image: localstack/localstack:3.8.1
    ports:
      - "127.0.0.1:4566:4566"
    environment:
      SERVICES: sns,sqs,lambda,logs,iam,sts
      # Las Lambdas se ejecutan en contenedores aparte: se conectan a la misma red que Postgres,
      # y así alcanzan la base con el nombre "postgres".
      LAMBDA_DOCKER_NETWORK: aws-local-net
      # Solo Linux: para que una Lambda alcance servicios de TU máquina con host.docker.internal (ver más abajo).
      # En Mac y Windows se deja vacío: Docker Desktop ya define ese nombre, y este flag lo rompería.
      LAMBDA_DOCKER_FLAGS: ${LAMBDA_FLAGS:-}
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock
    networks: [aws-local-net]

  postgres:
    image: postgres:16-alpine
    ports:
      - "127.0.0.1:5433:5432"
    environment:
      POSTGRES_USER: app
      POSTGRES_PASSWORD: app
      POSTGRES_DB: reports
    volumes:
      - pgdata:/var/lib/postgresql/data
      - ./db/init.sql:/docker-entrypoint-initdb.d/init.sql:ro
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U app -d reports"]
      interval: 3s
      retries: 20
    networks: [aws-local-net]

volumes:
  pgdata:

networks:
  aws-local-net:
    name: aws-local-net
EOF
```
**PowerShell**
```powershell
@'
name: aws-local

services:
  localstack:
    image: localstack/localstack:3.8.1
    ports:
      - "127.0.0.1:4566:4566"
    environment:
      SERVICES: sns,sqs,lambda,logs,iam,sts
      LAMBDA_DOCKER_NETWORK: aws-local-net
      LAMBDA_DOCKER_FLAGS: ${LAMBDA_FLAGS:-}
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock
    networks: [aws-local-net]

  postgres:
    image: postgres:16-alpine
    ports:
      - "127.0.0.1:5433:5432"
    environment:
      POSTGRES_USER: app
      POSTGRES_PASSWORD: app
      POSTGRES_DB: reports
    volumes:
      - pgdata:/var/lib/postgresql/data
      - ./db/init.sql:/docker-entrypoint-initdb.d/init.sql:ro
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U app -d reports"]
      interval: 3s
      retries: 20
    networks: [aws-local-net]

volumes:
  pgdata:

networks:
  aws-local-net:
    name: aws-local-net
'@ | Set-Content -Encoding ascii docker-compose.yml
```
Qué hace cada parte:
- **`image: localstack/localstack:3.8.1`**: la versión está fijada porque las más nuevas pueden exigir una cuenta.
- **`127.0.0.1:4566:4566`**: LocalStack se publica solo en tu máquina (nadie de tu red puede usarlo).
- **`SERVICES`**: solo arranca los servicios que usamos.
- **`/var/run/docker.sock`**: LocalStack ejecuta cada Lambda en **otro contenedor**, por eso necesita hablar con Docker. La ruta es la de *dentro* de Docker: se escribe igual en Windows.
- **`LAMBDA_DOCKER_NETWORK`**: pone los contenedores de las Lambdas en la red `aws-local-net`, la misma de Postgres. Resultado: dentro de una Lambda, la base se llama simplemente `postgres`.
- **`LAMBDA_DOCKER_FLAGS: ${LAMBDA_FLAGS:-}`**: opciones extra con que LocalStack lanza cada contenedor de Lambda. `host.docker.internal` es el nombre de tu máquina visto desde un contenedor, y la Lambda 2 lo necesita para llegar al servicio de avisos. **Docker Desktop (Mac y Windows) ya lo define**, así que aquí se deja vacío (`${LAMBDA_FLAGS:-}` significa "el valor de la variable `LAMBDA_FLAGS`, o vacío si no existe"). **Solo en Linux** hay que declararlo antes de levantar los contenedores:
  ```
  export LAMBDA_FLAGS=--add-host=host.docker.internal:host-gateway
  ```
  No lo uses en Mac/Windows: en Docker Desktop ese flag apunta el nombre a una dirección interna de Docker que **no alcanza tu máquina** (la Lambda fallaría con `fetch failed`).
- **`5433:5432`**: Postgres escucha en 5432 *dentro* del contenedor; hacia fuera se publica en 5433.
- **`POSTGRES_USER/PASSWORD/DB`**: usuario, clave y base. `app/app` son credenciales **solo locales**.
- **`./db/init.sql:/docker-entrypoint-initdb.d/…`**: Postgres ejecuta al crear la base cualquier `.sql` de esa carpeta. Solo corre la primera vez (si cambias el SQL, ejecuta `docker compose down -v` para borrar el volumen).
- **`healthcheck`**: Docker prueba cada 3 s si Postgres acepta conexiones (`pg_isready`).
- **`volumes: pgdata`**: los datos de Postgres sobreviven a un `down`/`up`. LocalStack, en cambio, **no guarda nada entre reinicios**.

Levanta todo y comprueba:

**bash y PowerShell** (idénticos)
```
docker compose up -d --wait
docker compose ps
```
- `up -d`: crea e inicia los contenedores en segundo plano (`-d`, *detached*).
- `--wait`: no devuelve el control hasta que los servicios estén sanos. La primera vez descarga las imágenes y tarda.
- `ps`: lista los contenedores; deben aparecer `localstack` y `postgres` en estado *running/healthy*.

Comprueba LocalStack y la base:

**bash**
```bash
curl -s http://localhost:4566/_localstack/health
docker compose exec -T postgres psql -U app -d reports -c "SELECT name, price FROM products ORDER BY price DESC LIMIT 3"
```
**PowerShell**
```powershell
curl.exe -s http://localhost:4566/_localstack/health
docker compose exec -T postgres psql -U app -d reports -c "SELECT name, price FROM products ORDER BY price DESC LIMIT 3"
```
- La primera línea devuelve un JSON donde `sns`, `sqs` y `lambda` figuran como `available` o `running`.
- La segunda ejecuta `psql` *dentro* del contenedor (no necesitas tener `psql` instalado) y muestra los 3 productos más caros. `-T` desactiva la terminal interactiva (necesario en scripts).

---

## 3. Configurar el CLI hacia LocalStack

El AWS CLI **no sabe nada de LocalStack**: sin más, habla con AWS real. Hay que indicarle otra dirección. La forma más segura es un **perfil** con claves falsas y la dirección de LocalStack (el mismo comando en bash y PowerShell):

```
aws configure set aws_access_key_id     test                   --profile localstack
aws configure set aws_secret_access_key test                   --profile localstack
aws configure set region                us-east-2              --profile localstack
aws configure set endpoint_url          http://localhost:4566  --profile localstack
```
- Escribe en `~/.aws/credentials` y `~/.aws/config` (en Windows, `%USERPROFILE%\.aws\`).
- **Claves `test`/`test`**: LocalStack acepta cualquiera. Y si un día olvidaras activar el perfil, AWS real las **rechazaría**: no hay riesgo de tocar tu cuenta.
- **`endpoint_url`**: a dónde van todos los comandos que usen este perfil.

Actívalo **en esta terminal** y comprueba **siempre** dónde estás antes de crear nada:

**bash**
```bash
export AWS_PROFILE=localstack
export AWS_REGION=us-east-2
aws sts get-caller-identity
ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)
echo "$ACCOUNT_ID"
```
**PowerShell**
```powershell
$env:AWS_PROFILE = "localstack"
$env:AWS_REGION = "us-east-2"
aws sts get-caller-identity
$ACCOUNT_ID = aws sts get-caller-identity --query Account --output text
$ACCOUNT_ID
```
- `AWS_PROFILE`: le dice al CLI qué perfil usar en esta terminal.
- `sts get-caller-identity`: "¿quién soy y en qué cuenta?". **Cuenta `000000000000` = LocalStack.** Si ves un número de 12 dígitos distinto, estás apuntando a AWS real: detente.
- `ACCOUNT_ID`: se usa después para armar ARNs (los identificadores de recursos).

> Cada terminal nueva necesita `AWS_PROFILE` y `AWS_REGION` otra vez (y `ACCOUNT_ID`, si la usas). Escribe siempre `${ACCOUNT_ID}` con llaves dentro de textos: `$ACCOUNT_ID:role` en zsh o PowerShell se interpreta mal.

---

## 4. El servicio de avisos: proyecto Spring Boot con `/api/notice`

En el sistema real, la Lambda 2 publica el aviso en `https://api.claudioviajando.com/api/notice`. En local ese servicio no existe, así que lo **simulamos con un proyecto Spring Boot mínimo** que expone el mismo endpoint:

- `POST /api/notice` recibe `{"text": "..."}` y responde `201`. Si arrancas el servicio con `NOTICE_API_KEY`, exige la cabecera `Authorization: ApiKey <clave>`, como el real.
- `GET /api/notice` lista los avisos recibidos, para que veas qué llegó.

**1. Generar el proyecto** con Spring Initializr (la herramienta oficial de Spring), desde la línea de comandos:

**bash**
```bash
curl -s https://start.spring.io/starter.zip \
  -d type=gradle-project -d language=java -d javaVersion=21 \
  -d groupId=com.example -d artifactId=notice-api -d name=notice-api \
  -d packageName=com.example.notice -d dependencies=web \
  -o notice-api.zip
mkdir notice-api && unzip -q notice-api.zip -d notice-api && rm notice-api.zip
```
**PowerShell**
```powershell
curl.exe -s https://start.spring.io/starter.zip `
  -d type=gradle-project -d language=java -d javaVersion=21 `
  -d groupId=com.example -d artifactId=notice-api -d name=notice-api `
  -d packageName=com.example.notice -d dependencies=web `
  -o notice-api.zip
mkdir notice-api; tar -xf notice-api.zip -C notice-api; Remove-Item notice-api.zip
```
- `curl -s … -d clave=valor -o archivo`: descarga un proyecto ya armado (Gradle, Java 21, con la dependencia `web`) y lo guarda en un zip; `unzip`/`tar -xf` lo extrae.
- La interfaz gráfica equivalente está en https://start.spring.io (Gradle Groovy, Java 21, dependencia *Spring Web*).

> **Si Gradle falla al compilar (típico en Windows y redes corporativas):**
> **(a)** `No matching toolchains found…`: tu JDK no es el 21. Mira `java --version` y, en `build.gradle`, cambia `JavaLanguageVersion.of(21)` por tu versión (Spring Boot pide 17 o más).
> **(b)** `Could not resolve…` / `Could not GET…`: tu red no alcanza Maven Central. Usa un espejo: en `build.gradle` reemplaza el bloque `repositories` por `repositories { maven { url = uri("https://maven-central.storage-download.googleapis.com/maven2/") } }`, y en `settings.gradle`, **antes** de `rootProject.name`, agrega:
> ```groovy
> pluginManagement {
>     repositories {
>         maven { url = uri("https://maven-central.storage-download.googleapis.com/maven2/") }
>         gradlePluginPortal()
>     }
> }
> dependencyResolutionManagement {
>     repositoriesMode.set(RepositoriesMode.PREFER_PROJECT)
>     repositories {
>         maven { url = uri("https://maven-central.storage-download.googleapis.com/maven2/") }
>     }
> }
> ```

**2. El código.** Dos archivos: el controlador y la configuración.

**bash**
```bash
cat > notice-api/src/main/java/com/example/notice/NoticeController.java <<'EOF'
package com.example.notice;

import java.time.Instant;
import java.util.List;
import java.util.Map;
import java.util.UUID;
import java.util.concurrent.CopyOnWriteArrayList;

import org.springframework.beans.factory.annotation.Value;
import org.springframework.http.ResponseEntity;
import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.PostMapping;
import org.springframework.web.bind.annotation.RequestBody;
import org.springframework.web.bind.annotation.RequestHeader;
import org.springframework.web.bind.annotation.RequestMapping;
import org.springframework.web.bind.annotation.RestController;

@RestController
@RequestMapping("/api/notice")
public class NoticeController {

    public record NoticeRequest(String text) {}

    public record Notice(String id, String text, Instant createdAt) {}

    private final List<Notice> notices = new CopyOnWriteArrayList<>();

    // Vacía = sin autenticación (solo para pruebas locales). Con valor, exige "Authorization: ApiKey <valor>".
    @Value("${notice.api-key:}")
    private String apiKey;

    @PostMapping
    public ResponseEntity<Object> create(
            @RequestHeader(value = "Authorization", required = false) String authorization,
            @RequestBody NoticeRequest request) {
        if (!apiKey.isBlank() && !("ApiKey " + apiKey).equals(authorization)) {
            return ResponseEntity.status(401).body(Map.of("error", "API key inválida o ausente"));
        }
        String text = request.text();
        if (text == null || text.isBlank() || text.length() > 2000) {
            return ResponseEntity.badRequest().body(Map.of("error", "text es obligatorio (1 a 2000 caracteres)"));
        }
        Notice notice = new Notice(UUID.randomUUID().toString(), text, Instant.now());
        notices.add(notice);
        // Se registra el texto, nunca la clave.
        System.out.println("AVISO RECIBIDO: " + text.replace('\n', ' '));
        return ResponseEntity.status(201).body(notice);
    }

    @GetMapping
    public List<Notice> list() {
        return notices;
    }
}
EOF

cat > notice-api/src/main/resources/application.properties <<'EOF'
spring.application.name=notice-api
server.port=8082
notice.api-key=${NOTICE_API_KEY:}
EOF
```
**PowerShell**
```powershell
@'
package com.example.notice;

import java.time.Instant;
import java.util.List;
import java.util.Map;
import java.util.UUID;
import java.util.concurrent.CopyOnWriteArrayList;

import org.springframework.beans.factory.annotation.Value;
import org.springframework.http.ResponseEntity;
import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.PostMapping;
import org.springframework.web.bind.annotation.RequestBody;
import org.springframework.web.bind.annotation.RequestHeader;
import org.springframework.web.bind.annotation.RequestMapping;
import org.springframework.web.bind.annotation.RestController;

@RestController
@RequestMapping("/api/notice")
public class NoticeController {

    public record NoticeRequest(String text) {}

    public record Notice(String id, String text, Instant createdAt) {}

    private final List<Notice> notices = new CopyOnWriteArrayList<>();

    // Vacia = sin autenticacion (solo para pruebas locales). Con valor, exige "Authorization: ApiKey <valor>".
    @Value("${notice.api-key:}")
    private String apiKey;

    @PostMapping
    public ResponseEntity<Object> create(
            @RequestHeader(value = "Authorization", required = false) String authorization,
            @RequestBody NoticeRequest request) {
        if (!apiKey.isBlank() && !("ApiKey " + apiKey).equals(authorization)) {
            return ResponseEntity.status(401).body(Map.of("error", "API key invalida o ausente"));
        }
        String text = request.text();
        if (text == null || text.isBlank() || text.length() > 2000) {
            return ResponseEntity.badRequest().body(Map.of("error", "text es obligatorio (1 a 2000 caracteres)"));
        }
        Notice notice = new Notice(UUID.randomUUID().toString(), text, Instant.now());
        notices.add(notice);
        // Se registra el texto, nunca la clave.
        System.out.println("AVISO RECIBIDO: " + text.replace('\n', ' '));
        return ResponseEntity.status(201).body(notice);
    }

    @GetMapping
    public List<Notice> list() {
        return notices;
    }
}
'@ | Set-Content -Encoding ascii notice-api/src/main/java/com/example/notice/NoticeController.java

@'
spring.application.name=notice-api
server.port=8082
notice.api-key=${NOTICE_API_KEY:}
'@ | Set-Content -Encoding ascii notice-api/src/main/resources/application.properties
```
Qué hace el código:
- **`@RestController` + `@RequestMapping("/api/notice")`**: la clase atiende las rutas que empiezan con `/api/notice`.
- **`@PostMapping`**: recibe el JSON `{"text": …}` (Spring lo convierte al `record NoticeRequest`), valida la clave y el texto, guarda el aviso en una lista **en memoria** (se pierde al apagar) y responde `201` con el aviso creado.
- **`@GetMapping`**: devuelve la lista, para inspeccionar.
- **`notice.api-key=${NOTICE_API_KEY:}`**: la clave sale de la variable de entorno `NOTICE_API_KEY`; si no existe, queda vacía y no se exige nada.
- **`server.port=8082`**: el puerto que fijamos arriba.

**3. Arrancarlo.** Es un proceso que **se queda en primer plano**, así que **ábrelo en una segunda terminal** (la primera sigue para la guía). Lo arrancamos con una clave de ejemplo para probar también la autenticación:

**Terminal B, bash**
```bash
cd ~/aws-local/notice-api
NOTICE_API_KEY=clave-local ./gradlew bootRun
```
**Terminal B, PowerShell**
```powershell
cd $HOME\aws-local\notice-api
$env:NOTICE_API_KEY = "clave-local"
.\gradlew.bat bootRun
```
- `NOTICE_API_KEY=clave-local ./gradlew bootRun` (bash) define la variable **solo para ese comando**; en PowerShell se define en la sesión.
- `./gradlew bootRun`: Gradle descarga las dependencias (la primera vez tarda) y arranca el servidor. Está listo cuando aparece `Started NoticeApiApplication`.
- El servicio escucha en **todas las interfaces** (no solo en `127.0.0.1`): es necesario, porque la Lambda lo llamará desde un contenedor. Por eso es **solo para pruebas**: no lo expongas. En Windows puede aparecer un aviso del firewall: acepta solo para redes privadas.

**4. Probarlo** (en la primera terminal): sin clave debe rechazar; con clave, aceptar.

**bash**
```bash
curl -s -o /dev/null -w "sin clave -> HTTP %{http_code}\n" -X POST http://localhost:8082/api/notice \
  -H "Content-Type: application/json" -d '{"text":"prueba"}'

curl -s -w "\ncon clave -> HTTP %{http_code}\n" -X POST http://localhost:8082/api/notice \
  -H "Authorization: ApiKey clave-local" -H "Content-Type: application/json" -d '{"text":"prueba"}'

curl -s http://localhost:8082/api/notice
```
**PowerShell**
```powershell
'{"text":"prueba"}' | Set-Content -Encoding ascii notice-test.json

curl.exe -s -o NUL -w "sin clave -> HTTP %{http_code}\n" -X POST http://localhost:8082/api/notice `
  -H "Content-Type: application/json" --data "@notice-test.json"

curl.exe -s -w "\ncon clave -> HTTP %{http_code}\n" -X POST http://localhost:8082/api/notice `
  -H "Authorization: ApiKey clave-local" -H "Content-Type: application/json" --data "@notice-test.json"

curl.exe -s http://localhost:8082/api/notice
```
- `-X POST`: método HTTP. `-H`: cabecera. `-d`: cuerpo (en PowerShell, `--data "@archivo"` lee el cuerpo de un archivo).
- `-o /dev/null -w "…%{http_code}"`: descarta el cuerpo y muestra solo el código HTTP (`-o NUL` en Windows).
- Esperado: `sin clave -> HTTP 401`, `con clave -> HTTP 201` (con el aviso en JSON) y el `GET` lista ese aviso.

Deja esta terminal B abierta: el resto de la guía depende de ella.

---

## 5. Escribir las dos Lambdas

Una Lambda es una función que AWS ejecuta cuando ocurre algo; aquí, cuando llegan mensajes a una cola. Las dos están en Node.js y usan `pg` (el cliente de Postgres), que hay que **incluir dentro del zip**: Lambda no lo trae.

### 5.1 Lambda 1: guarda la solicitud en Postgres

**bash**
```bash
mkdir lambdas/recorder && cd lambdas/recorder
npm init -y > /dev/null
npm install pg --omit=dev --no-audit --no-fund --silent
cat > index.js <<'EOF'
const { Pool } = require("pg");

// El Pool se crea fuera del handler: se reutiliza entre invocaciones de una misma Lambda "caliente".
const pool = new Pool({
  connectionString: process.env.DATABASE_URL,
  // Supabase exige SSL; en local (Postgres sin SSL) se apaga con DATABASE_SSL=false.
  ssl: process.env.DATABASE_SSL === "false" ? false : { rejectUnauthorized: false },
  max: 1,
});

// SNS entrega a SQS un sobre {"Type":"Notification","Message":"<JSON como texto>",...}.
// SQS lo pone en record.body. El mensaje real es {"type": "...", "body": {...}}.
function parse(record) {
  const envelope = JSON.parse(record.body);
  return JSON.parse(envelope.Message);
}

exports.handler = async (event) => {
  for (const record of event.Records) {
    const { type, body } = parse(record);
    if (type !== "report.requested") {
      throw new Error(`Tipo de mensaje no soportado: ${type}`);
    }
    // ON CONFLICT DO NOTHING: SQS entrega "al menos una vez"; un reintento no duplica la fila.
    await pool.query(
      `INSERT INTO report_requests (id, requester, requester_id, type, params, requested_at)
       VALUES ($1, $2, $3, $4, $5::jsonb, $6)
       ON CONFLICT (id) DO NOTHING`,
      [body.requestId, body.requester, body.requesterId, body.reportType,
       JSON.stringify(body.params ?? {}), body.requestedAt],
    );
    console.log("Solicitud registrada:", body.requestId);
  }
};
EOF
cd ../..
```
**PowerShell**
```powershell
mkdir lambdas/recorder; cd lambdas/recorder
npm init -y | Out-Null
npm install pg --omit=dev --no-audit --no-fund --silent
@'
const { Pool } = require("pg");

// El Pool se crea fuera del handler: se reutiliza entre invocaciones de una misma Lambda "caliente".
const pool = new Pool({
  connectionString: process.env.DATABASE_URL,
  // Supabase exige SSL; en local (Postgres sin SSL) se apaga con DATABASE_SSL=false.
  ssl: process.env.DATABASE_SSL === "false" ? false : { rejectUnauthorized: false },
  max: 1,
});

// SNS entrega a SQS un sobre {"Type":"Notification","Message":"<JSON como texto>",...}.
// SQS lo pone en record.body. El mensaje real es {"type": "...", "body": {...}}.
function parse(record) {
  const envelope = JSON.parse(record.body);
  return JSON.parse(envelope.Message);
}

exports.handler = async (event) => {
  for (const record of event.Records) {
    const { type, body } = parse(record);
    if (type !== "report.requested") {
      throw new Error(`Tipo de mensaje no soportado: ${type}`);
    }
    // ON CONFLICT DO NOTHING: SQS entrega "al menos una vez"; un reintento no duplica la fila.
    await pool.query(
      `INSERT INTO report_requests (id, requester, requester_id, type, params, requested_at)
       VALUES ($1, $2, $3, $4, $5::jsonb, $6)
       ON CONFLICT (id) DO NOTHING`,
      [body.requestId, body.requester, body.requesterId, body.reportType,
       JSON.stringify(body.params ?? {}), body.requestedAt],
    );
    console.log("Solicitud registrada:", body.requestId);
  }
};
'@ | Set-Content -Encoding ascii index.js
cd ../..
```
- `npm init -y`: crea un `package.json` vacío. `npm install pg --omit=dev`: descarga `pg` a `node_modules/` (sin dependencias de desarrollo). `--silent` reduce el ruido.
- **`event.Records`**: el evento que SQS entrega a la Lambda trae un lote (hasta 5 mensajes) en `Records`.
- **`exports.handler`**: la función que Lambda invoca; el nombre `handler` y el archivo `index.js` se declararán al crear la función (`index.handler`).
- **Si el handler lanza un error**, Lambda no borra los mensajes del lote: reaparecen y se reintentan; tras 3 intentos van a la DLQ (sección 7).

### 5.2 Lambda 2: arma el aviso con los 10 productos más caros y lo publica

**bash**
```bash
mkdir lambdas/generator && cd lambdas/generator
npm init -y > /dev/null
npm install pg --omit=dev --no-audit --no-fund --silent
cat > index.js <<'EOF'
const { Pool } = require("pg");

const pool = new Pool({
  connectionString: process.env.DATABASE_URL,
  ssl: process.env.DATABASE_SSL === "false" ? false : { rejectUnauthorized: false },
  max: 1,
});

// Esta Lambda atiende dos tipos de mensaje: el primero (report.requested) y los re-envíos (report.regenerate).
const TYPES = ["report.requested", "report.regenerate"];

function parse(record) {
  const envelope = JSON.parse(record.body);
  return JSON.parse(envelope.Message);
}

async function topProducts() {
  const { rows } = await pool.query(
    "SELECT name, price, stock FROM products WHERE price IS NOT NULL ORDER BY price DESC LIMIT 10",
  );
  return rows;
}

function buildText(request, products) {
  const lines = [
    `Informe ${request.reportType} (${request.requestId}) solicitado por ${request.requester}.`,
    `Los ${products.length} productos más caros:`,
    ...products.map((p, i) => `${i + 1}. ${p.name} - $${Number(p.price).toLocaleString("es-CL")} (stock ${p.stock})`),
  ];
  return lines.join("\n").slice(0, 2000); // la API de avisos acepta hasta 2000 caracteres
}

async function publish(text) {
  const url = process.env.NOTICE_URL;
  const key = process.env.NOTICE_API_KEY;
  if (!url || !key) throw new Error("Faltan NOTICE_URL y/o NOTICE_API_KEY");

  const res = await fetch(url, {
    method: "POST",
    headers: { Authorization: `ApiKey ${key}`, "Content-Type": "application/json" },
    body: JSON.stringify({ text }),
    redirect: "manual", // un redirect convertiría el POST en GET y el aviso no se crearía
    signal: AbortSignal.timeout(8000),
  });
  if (!res.ok) {
    // Se lanza un error: el mensaje se reintenta y, tras 3 intentos, va a la DLQ. La clave nunca se registra.
    throw new Error(`La API de avisos respondió HTTP ${res.status}`);
  }
  console.log(`Aviso publicado (HTTP ${res.status})`);
}

exports.handler = async (event) => {
  for (const record of event.Records) {
    const { type, body } = parse(record);
    if (!TYPES.includes(type)) {
      throw new Error(`Tipo de mensaje no soportado: ${type}`);
    }
    const products = await topProducts();
    await publish(buildText(body, products));
    console.log("Informe generado:", body.requestId, "con", products.length, "productos");
  }
};
EOF
cd ../..
```
**PowerShell**
```powershell
mkdir lambdas/generator; cd lambdas/generator
npm init -y | Out-Null
npm install pg --omit=dev --no-audit --no-fund --silent
@'
const { Pool } = require("pg");

const pool = new Pool({
  connectionString: process.env.DATABASE_URL,
  ssl: process.env.DATABASE_SSL === "false" ? false : { rejectUnauthorized: false },
  max: 1,
});

// Esta Lambda atiende dos tipos de mensaje: el primero (report.requested) y los re-envios (report.regenerate).
const TYPES = ["report.requested", "report.regenerate"];

function parse(record) {
  const envelope = JSON.parse(record.body);
  return JSON.parse(envelope.Message);
}

async function topProducts() {
  const { rows } = await pool.query(
    "SELECT name, price, stock FROM products WHERE price IS NOT NULL ORDER BY price DESC LIMIT 10",
  );
  return rows;
}

function buildText(request, products) {
  const lines = [
    `Informe ${request.reportType} (${request.requestId}) solicitado por ${request.requester}.`,
    `Los ${products.length} productos más caros:`,
    ...products.map((p, i) => `${i + 1}. ${p.name} - $${Number(p.price).toLocaleString("es-CL")} (stock ${p.stock})`),
  ];
  return lines.join("\n").slice(0, 2000); // la API de avisos acepta hasta 2000 caracteres
}

async function publish(text) {
  const url = process.env.NOTICE_URL;
  const key = process.env.NOTICE_API_KEY;
  if (!url || !key) throw new Error("Faltan NOTICE_URL y/o NOTICE_API_KEY");

  const res = await fetch(url, {
    method: "POST",
    headers: { Authorization: `ApiKey ${key}`, "Content-Type": "application/json" },
    body: JSON.stringify({ text }),
    redirect: "manual", // un redirect convertiria el POST en GET y el aviso no se crearia
    signal: AbortSignal.timeout(8000),
  });
  if (!res.ok) {
    // Se lanza un error: el mensaje se reintenta y, tras 3 intentos, va a la DLQ. La clave nunca se registra.
    throw new Error(`La API de avisos respondio HTTP ${res.status}`);
  }
  console.log(`Aviso publicado (HTTP ${res.status})`);
}

exports.handler = async (event) => {
  for (const record of event.Records) {
    const { type, body } = parse(record);
    if (!TYPES.includes(type)) {
      throw new Error(`Tipo de mensaje no soportado: ${type}`);
    }
    const products = await topProducts();
    await publish(buildText(body, products));
    console.log("Informe generado:", body.requestId, "con", products.length, "productos");
  }
};
'@ | Set-Content -Encoding utf8 index.js
cd ../..
```
(En PowerShell este archivo se guarda como UTF-8 porque el texto lleva una tilde, "más". La marca BOM que Windows PowerShell 5.1 agrega no afecta a Node.)

Qué hace:
- **`topProducts`**: la consulta SQL que trae los 10 productos con mayor precio.
- **`buildText`**: arma el texto del aviso (encabezado con el id de la solicitud + lista numerada).
- **`publish`**: hace el `POST` a `NOTICE_URL` con la cabecera `Authorization: ApiKey …`. La URL y la clave **no están en el código**: llegan por variables de entorno de la Lambda (sección 6). Así el mismo código sirve en local y en AWS real.
- **`redirect: "manual"`**: si la URL fuera incorrecta y el servidor redirigiera, `fetch` convertiría el POST en un GET y parecería éxito sin crear nada; con `manual` lo detectamos como error.
- **Cualquier respuesta que no sea 2xx lanza un error**: el mensaje queda sin borrar, se reintenta y termina en la DLQ. Un aviso nunca se da por entregado si no se creó.

### 5.3 Empaquetar (zip)

El zip debe contener `index.js`, `package.json` y `node_modules` **en la raíz** (no dentro de una subcarpeta).

**bash**
```bash
(cd lambdas/recorder  && zip -qr ../recorder.zip  index.js package.json node_modules)
(cd lambdas/generator && zip -qr ../generator.zip index.js package.json node_modules)
ls -l lambdas/*.zip
```
**PowerShell**
```powershell
cd lambdas/recorder;  tar -a -c -f ../recorder.zip  index.js package.json node_modules; cd ../..
cd lambdas/generator; tar -a -c -f ../generator.zip index.js package.json node_modules; cd ../..
Get-ChildItem lambdas/*.zip
```
- `zip -qr archivo.zip …`: `-r` incluye carpetas completas; `-q` sin ruido.
- Los paréntesis `( cd … && zip … )` ejecutan el `cd` en un subproceso, así tu terminal no cambia de carpeta.
- En Windows no existe `zip`: `tar -a -c -f archivo.zip` crea un zip según la extensión del nombre. (Evita `Compress-Archive`: guarda rutas con `\` que Lambda en Linux no entiende bien.)

---

## 6. Desplegar las Lambdas

### 6.1 El rol y las variables de entorno

Toda Lambda necesita un **rol de IAM**: los permisos con que se ejecuta. LocalStack no lo valida (basta un ARN con la forma correcta); en AWS real debe existir (sección 9).

La configuración (URL de la base, URL del servicio de avisos, clave) va en **variables de entorno**, no en el código. Se escribe en archivos JSON:

**bash**
```bash
ROLE_ARN="arn:aws:iam::${ACCOUNT_ID}:role/lambda-role"

cat > env-recorder.json <<'EOF'
{"Variables":{"DATABASE_URL":"postgresql://app:app@postgres:5432/reports","DATABASE_SSL":"false"}}
EOF
cat > env-generator.json <<'EOF'
{"Variables":{"DATABASE_URL":"postgresql://app:app@postgres:5432/reports","DATABASE_SSL":"false","NOTICE_URL":"http://host.docker.internal:8082/api/notice","NOTICE_API_KEY":"clave-local"}}
EOF
```
**PowerShell**
```powershell
$ROLE_ARN = "arn:aws:iam::${ACCOUNT_ID}:role/lambda-role"

@'
{"Variables":{"DATABASE_URL":"postgresql://app:app@postgres:5432/reports","DATABASE_SSL":"false"}}
'@ | Set-Content -Encoding ascii env-recorder.json
@'
{"Variables":{"DATABASE_URL":"postgresql://app:app@postgres:5432/reports","DATABASE_SSL":"false","NOTICE_URL":"http://host.docker.internal:8082/api/notice","NOTICE_API_KEY":"clave-local"}}
'@ | Set-Content -Encoding ascii env-generator.json
```
- **`DATABASE_URL`**: `postgresql://usuario:clave@servidor:puerto/base`. `postgres` es el nombre del servicio del compose (resuelve gracias a `LAMBDA_DOCKER_NETWORK`); el puerto es el **interno**, 5432.
- **`DATABASE_SSL=false`**: el Postgres local no usa SSL (Supabase sí).
- **`NOTICE_URL`**: dónde publica la Lambda 2. `host.docker.internal` es tu máquina vista desde el contenedor de la Lambda.
- **`NOTICE_API_KEY=clave-local`**: la misma clave con la que arrancaste el servicio de avisos.

> Estos archivos tienen claves. Aquí son de juguete, pero el hábito importa: **nunca los subas a git** (la carpeta de trabajo no es un repo) y bórralos al terminar. En AWS real, la clave se guarda como secreto, no en un archivo.

### 6.2 Crear las funciones

**bash**
```bash
aws lambda create-function --function-name report-request-recorder \
  --runtime nodejs20.x --handler index.handler --timeout 15 --memory-size 128 \
  --role "$ROLE_ARN" --zip-file fileb://lambdas/recorder.zip \
  --environment file://env-recorder.json \
  --query FunctionArn --output text

aws lambda create-function --function-name report-generator \
  --runtime nodejs20.x --handler index.handler --timeout 15 --memory-size 128 \
  --role "$ROLE_ARN" --zip-file fileb://lambdas/generator.zip \
  --environment file://env-generator.json \
  --query FunctionArn --output text

aws lambda wait function-active-v2 --function-name report-request-recorder
aws lambda wait function-active-v2 --function-name report-generator
```
**PowerShell**
```powershell
aws lambda create-function --function-name report-request-recorder `
  --runtime nodejs20.x --handler index.handler --timeout 15 --memory-size 128 `
  --role $ROLE_ARN --zip-file fileb://lambdas/recorder.zip `
  --environment file://env-recorder.json `
  --query FunctionArn --output text

aws lambda create-function --function-name report-generator `
  --runtime nodejs20.x --handler index.handler --timeout 15 --memory-size 128 `
  --role $ROLE_ARN --zip-file fileb://lambdas/generator.zip `
  --environment file://env-generator.json `
  --query FunctionArn --output text

aws lambda wait function-active-v2 --function-name report-request-recorder
aws lambda wait function-active-v2 --function-name report-generator
```
- **`--function-name`**: el nombre.
- **`--runtime nodejs20.x`**: el entorno de ejecución (en AWS real, `nodejs22.x`).
- **`--handler index.handler`**: *archivo*`.`*función* que se invoca: `index.js`, función `handler`.
- **`--timeout 15`**: si la función tarda más de 15 s, Lambda la corta. Importante para la cola (sección 7).
- **`--memory-size 128`**: MB de memoria (también define la CPU).
- **`--role`**: el rol de IAM.
- **`--zip-file fileb://…`**: el código. `fileb://` = archivo **binario** (un zip no es texto).
- **`--environment file://…`**: las variables de entorno.
- **`--query FunctionArn --output text`**: **sin esto, el comando imprime la configuración completa, incluidas las variables de entorno con sus claves**. Con `--query` solo se muestra el ARN.
- **`aws lambda wait function-active-v2`**: espera a que la función pase de `Pending` a `Active` (la primera vez LocalStack descarga la imagen de ejecución).

Comprueba:

**bash**
```bash
aws lambda list-functions --query 'Functions[].[FunctionName,Runtime]' --output text
```
**PowerShell**
```powershell
aws lambda list-functions --query 'Functions[].[FunctionName,Runtime]' --output text
```
Deben aparecer las dos. (El `wait` anterior ya garantiza que están activas.)

### 6.3 Probar una Lambda a mano (sin colas todavía)

Antes de conectar nada, invoca la Lambda 2 con un evento falso que imita lo que entregaría SQS. Así separas problemas: si esto funciona, el código, la base y el servicio de avisos están bien.

**bash**
```bash
cat > event-test.json <<'EOF'
{"Records":[{"body":"{\"Type\":\"Notification\",\"Message\":\"{\\\"type\\\":\\\"report.requested\\\",\\\"body\\\":{\\\"requestId\\\":\\\"manual-1\\\",\\\"requester\\\":\\\"prueba@ejemplo.com\\\",\\\"reportType\\\":\\\"top-products\\\"}}\"}"}]}
EOF
aws lambda invoke --function-name report-generator \
  --cli-binary-format raw-in-base64-out --payload file://event-test.json \
  --log-type Tail --query LogResult --output text out.json | base64 --decode
cat out.json
```
**PowerShell**
```powershell
@'
{"Records":[{"body":"{\"Type\":\"Notification\",\"Message\":\"{\\\"type\\\":\\\"report.requested\\\",\\\"body\\\":{\\\"requestId\\\":\\\"manual-1\\\",\\\"requester\\\":\\\"prueba@ejemplo.com\\\",\\\"reportType\\\":\\\"top-products\\\"}}\"}"}]}
'@ | Set-Content -Encoding ascii event-test.json
$log = aws lambda invoke --function-name report-generator `
  --cli-binary-format raw-in-base64-out --payload file://event-test.json `
  --log-type Tail --query LogResult --output text out.json
[Text.Encoding]::UTF8.GetString([Convert]::FromBase64String($log))
Get-Content out.json
```
- **`aws lambda invoke`**: ejecuta la función ahora, sin colas. `--payload`: el evento de entrada. `--cli-binary-format raw-in-base64-out`: manda el payload como texto plano (en CLI v2 el valor por defecto lo exige en base64).
- **`--log-type Tail --query LogResult`**: devuelve las últimas líneas del log, codificadas en base64; `base64 --decode` las muestra legibles. `out.json` es lo que **devolvió** la función.
- El evento tiene tres niveles de JSON anidado (sobre de SQS → sobre de SNS → mensaje): por eso las comillas escapadas.
- **Resultado esperado**: en el log, `Aviso publicado (HTTP 201)` e `Informe generado: manual-1 con 10 productos`; y en la terminal B (el servicio de avisos), `AVISO RECIBIDO: Informe top-products (manual-1) …`. `out.json` mostrará `null`: es normal, la función no devuelve nada (devolver un valor no importa para una cola). Si falla, `out.json` o el log muestran `errorMessage`.

---

## 7. SNS, SQS, filtros y triggers

### 7.1 Cómo decide SNS a qué cola va cada mensaje

Un topic de SNS reparte cada mensaje a **todas** sus suscripciones. Para que cada cola reciba solo lo suyo, cada suscripción lleva una **política de filtro** (`FilterPolicy`) que mira un campo del mensaje: aquí, `type`. El mensaje siempre es JSON con ese campo:

```json
{"type": "report.requested", "body": { ... los datos ... }}
```

| Cola | Lambda | Acepta `type` | Para qué |
|---|---|---|---|
| `report-requests` | recorder (guarda en Postgres) | `report.requested` | Registrar cada solicitud nueva |
| `report-generation` | generator (arma y publica el aviso) | `report.requested` y `report.regenerate` | Generar el informe, ya sea por una solicitud nueva o por un re-envío |

Con eso, el sistema se comporta así:

| Mensaje publicado | Cola recorder | Cola generator | Resultado |
|---|---|---|---|
| `report.requested` | sí | sí | se **guarda** y se **genera** (los dos a la vez: *fan-out*) |
| `report.regenerate` | no | sí | solo se **vuelve a generar** el aviso, sin duplicar la solicitud en la base |
| cualquier otro (`otro`) | no | no | se descarta en silencio |

> **Por qué filtrar por `type` y no tener un topic por cola:** el emisor (el BFF) publica *un hecho* ("se pidió un informe") en un solo lugar y no sabe cuántos consumidores hay. Mañana puedes agregar una tercera cola (por ejemplo, auditoría) sin tocar al emisor: solo creas la cola y su suscripción con el filtro que quieras.

### 7.2 El topic

**bash**
```bash
TOPIC_ARN=$(aws sns create-topic --name report-events --query TopicArn --output text)
echo "$TOPIC_ARN"
```
**PowerShell**
```powershell
$TOPIC_ARN = aws sns create-topic --name report-events --query TopicArn --output text
$TOPIC_ARN
```
- `create-topic`: crea el topic (si ya existe con ese nombre, devuelve el mismo).
- **ARN**: el identificador único de un recurso. Aquí: `arn:aws:sns:us-east-2:000000000000:report-events`; en AWS real, igual con tu número de cuenta.

### 7.3 Cada cola: cuatro pasos

Para **cada** cola: **(1)** una DLQ, **(2)** la cola con su regla de reintentos, **(3)** permiso para que SNS escriba en ella, **(4)** la suscripción con su filtro. Para no repetir el bloque, lo definimos como una función y la llamamos dos veces (con filtros distintos):

**bash**
```bash
create_queue() {
  # $1 = nombre de la cola   $2 = filtro: los tipos de mensaje que acepta (JSON)
  local name="$1" filter="$2"
  local esc=${filter//\"/\\\"}     # el filtro viaja como JSON dentro de JSON: hay que escapar sus comillas

  # 1) La DLQ (dead-letter queue): adonde van los mensajes que fallan demasiadas veces
  local dlq_url dlq_arn
  dlq_url=$(aws sqs create-queue --queue-name "$name-dlq" --attributes MessageRetentionPeriod=1209600 --query QueueUrl --output text)
  dlq_arn=$(aws sqs get-queue-attributes --queue-url "$dlq_url" --attribute-names QueueArn --query Attributes.QueueArn --output text)

  # 2) La cola, con VisibilityTimeout y la regla de reintentos (RedrivePolicy es JSON dentro de JSON: va en un archivo)
  cat > attrs.json <<EOF
{"VisibilityTimeout":"90","RedrivePolicy":"{\"deadLetterTargetArn\":\"$dlq_arn\",\"maxReceiveCount\":\"3\"}"}
EOF
  Q_URL=$(aws sqs create-queue --queue-name "$name" --attributes file://attrs.json --query QueueUrl --output text)
  Q_ARN=$(aws sqs get-queue-attributes --queue-url "$Q_URL" --attribute-names QueueArn --query Attributes.QueueArn --output text)

  # 3) Permiso: SNS (solo este topic) puede escribir en la cola
  cat > policy.json <<EOF
{"Policy":"{\"Version\":\"2012-10-17\",\"Statement\":[{\"Effect\":\"Allow\",\"Principal\":{\"Service\":\"sns.amazonaws.com\"},\"Action\":\"sqs:SendMessage\",\"Resource\":\"$Q_ARN\",\"Condition\":{\"ArnEquals\":{\"aws:SourceArn\":\"$TOPIC_ARN\"}}}]}"}
EOF
  aws sqs set-queue-attributes --queue-url "$Q_URL" --attributes file://policy.json

  # 4) La suscripción: conecta el topic con la cola, con su filtro
  cat > sub-attrs.json <<EOF
{"FilterPolicy":"$esc","FilterPolicyScope":"MessageBody","RawMessageDelivery":"false"}
EOF
  aws sns subscribe --topic-arn "$TOPIC_ARN" --protocol sqs --notification-endpoint "$Q_ARN" \
    --attributes file://sub-attrs.json --query SubscriptionArn --output text
}

create_queue report-requests   '{"type":["report.requested"]}'
REQUESTS_ARN=$Q_ARN
create_queue report-generation '{"type":["report.requested","report.regenerate"]}'
GENERATION_ARN=$Q_ARN
echo "$REQUESTS_ARN"; echo "$GENERATION_ARN"
```
**PowerShell**
```powershell
function New-Queue($Name, $Filter) {
  # $Name = nombre de la cola   $Filter = los tipos de mensaje que acepta (JSON)
  $esc = $Filter.Replace('"', '\"')    # el filtro viaja como JSON dentro de JSON: hay que escapar sus comillas

  # 1) La DLQ (dead-letter queue): adonde van los mensajes que fallan demasiadas veces
  $dlqUrl = aws sqs create-queue --queue-name "$Name-dlq" --attributes MessageRetentionPeriod=1209600 --query QueueUrl --output text
  $dlqArn = aws sqs get-queue-attributes --queue-url $dlqUrl --attribute-names QueueArn --query Attributes.QueueArn --output text

  # 2) La cola, con VisibilityTimeout y la regla de reintentos
  @"
{"VisibilityTimeout":"90","RedrivePolicy":"{\"deadLetterTargetArn\":\"$dlqArn\",\"maxReceiveCount\":\"3\"}"}
"@ | Set-Content -Encoding ascii attrs.json
  $global:Q_URL = aws sqs create-queue --queue-name $Name --attributes file://attrs.json --query QueueUrl --output text
  $global:Q_ARN = aws sqs get-queue-attributes --queue-url $global:Q_URL --attribute-names QueueArn --query Attributes.QueueArn --output text

  # 3) Permiso: SNS (solo este topic) puede escribir en la cola
  @"
{"Policy":"{\"Version\":\"2012-10-17\",\"Statement\":[{\"Effect\":\"Allow\",\"Principal\":{\"Service\":\"sns.amazonaws.com\"},\"Action\":\"sqs:SendMessage\",\"Resource\":\"$($global:Q_ARN)\",\"Condition\":{\"ArnEquals\":{\"aws:SourceArn\":\"$TOPIC_ARN\"}}}]}"}
"@ | Set-Content -Encoding ascii policy.json
  aws sqs set-queue-attributes --queue-url $global:Q_URL --attributes file://policy.json

  # 4) La suscripción: conecta el topic con la cola, con su filtro
  @"
{"FilterPolicy":"$esc","FilterPolicyScope":"MessageBody","RawMessageDelivery":"false"}
"@ | Set-Content -Encoding ascii sub-attrs.json
  aws sns subscribe --topic-arn $TOPIC_ARN --protocol sqs --notification-endpoint $global:Q_ARN `
    --attributes file://sub-attrs.json --query SubscriptionArn --output text
}

New-Queue "report-requests"   '{"type":["report.requested"]}'
$REQUESTS_ARN = $Q_ARN
New-Queue "report-generation" '{"type":["report.requested","report.regenerate"]}'
$GENERATION_ARN = $Q_ARN
$REQUESTS_ARN; $GENERATION_ARN
```
Qué hace cada paso:
1. **DLQ** (`create-queue … -dlq`): una cola aparte donde SQS deposita los mensajes que no se pudieron procesar. `MessageRetentionPeriod=1209600`: se conservan 14 días (en segundos), para que puedas investigarlos.
2. **La cola**, con dos atributos:
   - **`VisibilityTimeout: 90`**: mientras una Lambda procesa un mensaje, queda oculto 90 s. Debe ser **mayor que el timeout de la Lambda** (15 s); si fuera menor, otra Lambda podría tomar el mismo mensaje a la vez.
   - **`RedrivePolicy` con `maxReceiveCount: 3`**: si el mismo mensaje falla 3 veces, SQS lo mueve a la DLQ en lugar de reintentar para siempre.
   - **`get-queue-attributes … QueueArn`** obtiene el ARN de la cola, que necesitan los pasos siguientes.
3. **La política de la cola** (`set-queue-attributes`): autoriza al servicio SNS (`sns.amazonaws.com`) a ejecutar `sqs:SendMessage` en esta cola, **solo si el mensaje viene de nuestro topic** (`aws:SourceArn`). En LocalStack no se exige, pero **en AWS real es obligatoria**: sin ella, SNS no puede escribir y los mensajes se pierden sin avisar.
4. **La suscripción** (`sns subscribe`): conecta el topic con la cola.
   - **`FilterPolicy`**: la regla de la sección 7.1.
   - **`FilterPolicyScope: MessageBody`**: el filtro mira **dentro del JSON del mensaje** (por defecto miraría los *atributos* del mensaje, que no usamos).
   - **`RawMessageDelivery: false`**: SNS entrega el mensaje dentro de un sobre `{"Type":"Notification","Message":"…"}`. Nuestras Lambdas lo desenvuelven (`envelope.Message`). Con `true` llegaría el mensaje a pelo.
- **`local`, `$global:`, `esc`**: detalles de los lenguajes: `local` limita las variables a la función; en PowerShell las dos que necesitamos fuera (`Q_URL` y `Q_ARN`) se declaran `$global:`.

### 7.4 Los triggers: conectar cada cola con su Lambda

Un **trigger** (en AWS, *event source mapping*) hace que Lambda lea la cola por sí sola y ejecute la función con lotes de mensajes.

**bash**
```bash
aws lambda create-event-source-mapping --function-name report-request-recorder \
  --event-source-arn "$REQUESTS_ARN" --batch-size 5 --query '[UUID,State]' --output text

aws lambda create-event-source-mapping --function-name report-generator \
  --event-source-arn "$GENERATION_ARN" --batch-size 5 --query '[UUID,State]' --output text

aws lambda list-event-source-mappings --query 'EventSourceMappings[].[FunctionArn,State]' --output text
```
**PowerShell**
```powershell
aws lambda create-event-source-mapping --function-name report-request-recorder `
  --event-source-arn $REQUESTS_ARN --batch-size 5 --query '[UUID,State]' --output text

aws lambda create-event-source-mapping --function-name report-generator `
  --event-source-arn $GENERATION_ARN --batch-size 5 --query '[UUID,State]' --output text

aws lambda list-event-source-mappings --query 'EventSourceMappings[].[FunctionArn,State]' --output text
```
- **`--event-source-arn`**: la cola que se vigila.
- **`--batch-size 5`**: hasta 5 mensajes por invocación (la `event.Records` del código).
- El último comando debe mostrar ambos en estado **`Enabled`** (justo tras crearlos pueden figurar como `Creating`; repite el comando).
- Cuando la Lambda termina sin error, Lambda borra los mensajes de la cola; si falla, quedan para reintento (y luego, DLQ).

---

## 8. Probar todo de punta a punta

Cada prueba publica en SNS un mensaje con un **identificador único** (`MARK`) y comprueba el resultado buscando ese identificador, para no confundirlo con resultados anteriores.

### 8.1 `report.requested`: debe guardarse **y** generarse

**bash**
```bash
MARK="demo-$(date +%s)"
cat > msg-requested.json <<EOF
{"type":"report.requested","body":{"requestId":"$MARK","requester":"claudio@ejemplo.com","requesterId":"u-1","reportType":"top-products","params":{},"requestedAt":"$(date -u +%Y-%m-%dT%H:%M:%SZ)"}}
EOF
aws sns publish --topic-arn "$TOPIC_ARN" --message file://msg-requested.json --query MessageId --output text
```
**PowerShell**
```powershell
$MARK = "demo-" + [DateTimeOffset]::UtcNow.ToUnixTimeSeconds()
$now = (Get-Date).ToUniversalTime().ToString("yyyy-MM-ddTHH:mm:ssZ")
@"
{"type":"report.requested","body":{"requestId":"$MARK","requester":"claudio@ejemplo.com","requesterId":"u-1","reportType":"top-products","params":{},"requestedAt":"$now"}}
"@ | Set-Content -Encoding ascii msg-requested.json
aws sns publish --topic-arn $TOPIC_ARN --message file://msg-requested.json --query MessageId --output text
```
- **`aws sns publish`**: publica el mensaje en el topic. `--message file://…`: el texto del mensaje, leído de un archivo. Devuelve el `MessageId`.
- Es **exactamente lo que hará el BFF** en AWS real: publicar este JSON en el topic.

Comprueba la **fila en Postgres** (Lambda 1) y el **aviso recibido** (Lambda 2). Las Lambdas tardan unos segundos (la primera vez, más, por el arranque en frío), así que se reintenta hasta 12 veces:

**bash**
```bash
echo "--- Postgres (Lambda 1):"
for i in $(seq 1 12); do
  N=$(docker compose exec -T postgres psql -U app -d reports -t -A -c "SELECT count(*) FROM report_requests WHERE id='$MARK'")
  [ "$N" = "1" ] && break; sleep 5
done
docker compose exec -T postgres psql -U app -d reports -c "SELECT id, requester, type, requested_at FROM report_requests WHERE id='$MARK'"

echo "--- Servicio de avisos (Lambda 2):"
for i in $(seq 1 12); do
  curl -s http://localhost:8082/api/notice | grep -q "$MARK" && break; sleep 5
done
curl -s http://localhost:8082/api/notice | grep -o "Informe top-products ($MARK)[^\"]*" | head -c 300; echo
```
**PowerShell**
```powershell
"--- Postgres (Lambda 1):"
for ($i = 0; $i -lt 12; $i++) {
  $n = docker compose exec -T postgres psql -U app -d reports -t -A -c "SELECT count(*) FROM report_requests WHERE id='$MARK'"
  if ($n -eq "1") { break }; Start-Sleep 5
}
docker compose exec -T postgres psql -U app -d reports -c "SELECT id, requester, type, requested_at FROM report_requests WHERE id='$MARK'"

"--- Servicio de avisos (Lambda 2):"
for ($i = 0; $i -lt 12; $i++) {
  if ((curl.exe -s http://localhost:8082/api/notice) -match $MARK) { break }; Start-Sleep 5
}
(curl.exe -s http://localhost:8082/api/notice) -split '"' | Select-String $MARK | Select-Object -First 1
```
- Esperado: **una fila** en `report_requests` con tu `MARK`, y el aviso `Informe top-products (demo-…) solicitado por claudio@ejemplo.com. Los 10 productos más caros: 1. Notebook Pro 16 …` (también impreso como `AVISO RECIBIDO` en la terminal B).
- Dos Lambdas distintas, dos colas distintas, un solo mensaje publicado: eso es el *fan-out*.

### 8.2 `report.regenerate`: solo debe generarse (sin tocar la base)

**bash**
```bash
MARK2="regen-$(date +%s)"
cat > msg-regenerate.json <<EOF
{"type":"report.regenerate","body":{"requestId":"$MARK2","requester":"claudio@ejemplo.com","requesterId":"u-1","reportType":"top-products","params":{},"requestedAt":"$(date -u +%Y-%m-%dT%H:%M:%SZ)"}}
EOF
aws sns publish --topic-arn "$TOPIC_ARN" --message file://msg-regenerate.json --query MessageId --output text

for i in $(seq 1 12); do
  curl -s http://localhost:8082/api/notice | grep -q "$MARK2" && break; sleep 5
done
echo -n "avisos con $MARK2: "; curl -s http://localhost:8082/api/notice | grep -c "$MARK2"
echo -n "filas en Postgres con $MARK2 (debe ser 0): "
docker compose exec -T postgres psql -U app -d reports -t -A -c "SELECT count(*) FROM report_requests WHERE id='$MARK2'"
```
**PowerShell**
```powershell
$MARK2 = "regen-" + [DateTimeOffset]::UtcNow.ToUnixTimeSeconds()
$now = (Get-Date).ToUniversalTime().ToString("yyyy-MM-ddTHH:mm:ssZ")
@"
{"type":"report.regenerate","body":{"requestId":"$MARK2","requester":"claudio@ejemplo.com","requesterId":"u-1","reportType":"top-products","params":{},"requestedAt":"$now"}}
"@ | Set-Content -Encoding ascii msg-regenerate.json
aws sns publish --topic-arn $TOPIC_ARN --message file://msg-regenerate.json --query MessageId --output text

for ($i = 0; $i -lt 12; $i++) {
  if ((curl.exe -s http://localhost:8082/api/notice) -match $MARK2) { break }; Start-Sleep 5
}
"aviso con ${MARK2}: " + (((curl.exe -s http://localhost:8082/api/notice) -match $MARK2))
"filas en Postgres con $MARK2 (debe ser 0): " + (docker compose exec -T postgres psql -U app -d reports -t -A -c "SELECT count(*) FROM report_requests WHERE id='$MARK2'")
```
Esperado: **1 aviso** y **0 filas**: el filtro de la cola `report-requests` dejó fuera este tipo, y solo `report-generation` lo recibió.

### 8.3 Un tipo que nadie acepta: debe descartarse

**bash**
```bash
MARK3="otro-$(date +%s)"
aws sns publish --topic-arn "$TOPIC_ARN" \
  --message "{\"type\":\"otro\",\"body\":{\"requestId\":\"$MARK3\",\"requester\":\"x\",\"reportType\":\"x\"}}" \
  --query MessageId --output text
sleep 15
echo -n "avisos con $MARK3 (debe ser 0): "; curl -s http://localhost:8082/api/notice | grep -c "$MARK3"
for Q in report-requests report-generation; do
  echo -n "mensajes pendientes en $Q (debe ser 0): "
  aws sqs get-queue-attributes --queue-url "$(aws sqs get-queue-url --queue-name $Q --query QueueUrl --output text)" \
    --attribute-names ApproximateNumberOfMessages --query Attributes.ApproximateNumberOfMessages --output text
done
```
**PowerShell**
```powershell
$MARK3 = "otro-" + [DateTimeOffset]::UtcNow.ToUnixTimeSeconds()
@"
{"type":"otro","body":{"requestId":"$MARK3","requester":"x","reportType":"x"}}
"@ | Set-Content -Encoding ascii msg-otro.json
aws sns publish --topic-arn $TOPIC_ARN --message file://msg-otro.json --query MessageId --output text
Start-Sleep 15
"avisos con $MARK3 (debe ser 0): " + (((curl.exe -s http://localhost:8082/api/notice) -split $MARK3).Count - 1)
foreach ($q in "report-requests", "report-generation") {
  $u = aws sqs get-queue-url --queue-name $q --query QueueUrl --output text
  "mensajes pendientes en ${q} (debe ser 0): " + (aws sqs get-queue-attributes --queue-url $u --attribute-names ApproximateNumberOfMessages --query Attributes.ApproximateNumberOfMessages --output text)
}
```
Esperado: 0 avisos y 0 mensajes pendientes. El mensaje se publicó (SNS lo aceptó), pero ninguna suscripción lo quiso: ni llegó a una cola ni falló nada. **Un filtro mal escrito tiene el mismo síntoma** (todo "funciona" y no llega nada): si algo no llega, compara el `type` publicado con el filtro.

### 8.4 Ver los logs de una Lambda

**bash**
```bash
aws logs filter-log-events --log-group-name /aws/lambda/report-generator \
  --query 'events[].message' --output text | tr '\t' '\n' | grep -E "Aviso publicado|Informe generado" | tail -5
```
**PowerShell**
```powershell
(aws logs filter-log-events --log-group-name /aws/lambda/report-generator --query 'events[].message' --output text) -split "`t" |
  Select-String "Aviso publicado|Informe generado" | Select-Object -Last 5
```
- Cada Lambda escribe en el grupo `/aws/lambda/<nombre>`. `--output text` junta todos los mensajes en **una línea separada por tabuladores**: `tr '\t' '\n'` (o `-split "`t"`) los pone uno por línea.
- Filtra con `grep` y no con `--filter-pattern`: **LocalStack no aplica ese filtro de forma fiable**.
- El grupo de logs se crea **en la primera invocación** de la función: antes no existe.

---

## 9. Pasar a AWS real

Los mismos pasos, con estos cambios. **El código de las Lambdas y del proyecto Spring Boot no cambia**: lo único que cambia es la configuración.

| Qué | Local | AWS real |
|---|---|---|
| Destino del CLI | perfil `localstack` (claves falsas + `endpoint_url`) | tu perfil real, **sin** `endpoint_url`. Comprueba con `aws sts get-caller-identity` que ves **tu** cuenta antes de crear nada |
| Rol de las Lambdas | un ARN inventado | un rol IAM real (abajo) |
| Runtime | `nodejs20.x` | `nodejs22.x` |
| `DATABASE_URL` | `postgresql://app:app@postgres:5432/reports` | la cadena de Supabase (con su contraseña) |
| `DATABASE_SSL` | `false` | no definirla (SSL activo) |
| `NOTICE_URL` | `http://host.docker.internal:8082/api/notice` | `https://api.claudioviajando.com/api/notice` |
| `NOTICE_API_KEY` | `clave-local` | la clave real (solo en secretos, nunca en git ni en archivos) |
| Política de la cola | no se exige | **obligatoria** (el bloque 3 de la sección 7.3 ya la crea) |
| Quién publica en SNS | el `aws sns publish` de la guía | el BFF, con permiso `sns:Publish` en su rol |
| El servicio de avisos | proyecto Spring Boot local | el servicio real, ya desplegado |

> **Verifica el contrato del servicio real.** En local, `/api/notice` recibe `{"text": "…"}` con la cabecera `Authorization: ApiKey …`. La versión anterior de este proyecto publicaba en `https://app.claudioviajando.com/api/saas/notices` con esa misma forma. Antes de apuntar `NOTICE_URL` a `api.claudioviajando.com/api/notice`, confirma con una prueba (un `curl` con tu clave) que esa ruta acepta ese cuerpo y esa cabecera.

**El rol de las Lambdas (en real).** Necesita una política de confianza que deje a Lambda asumirlo y permisos para leer la cola y escribir logs:

**bash**
```bash
cat > trust.json <<'EOF'
{"Version":"2012-10-17","Statement":[{"Effect":"Allow","Principal":{"Service":"lambda.amazonaws.com"},"Action":"sts:AssumeRole"}]}
EOF
aws iam create-role --role-name report-lambda-role --assume-role-policy-document file://trust.json
aws iam attach-role-policy --role-name report-lambda-role --policy-arn arn:aws:iam::aws:policy/service-role/AWSLambdaSQSQueueExecutionRole
```
- **`create-role` + la política de confianza**: "el servicio Lambda puede asumir este rol".
- **`AWSLambdaSQSQueueExecutionRole`**: política administrada por AWS que permite leer/borrar mensajes de SQS y escribir logs en CloudWatch. Sin ella, el trigger no funciona.
- Si la Lambda corre dentro de una VPC o accede a otros servicios, necesita permisos adicionales.

**Orden en real:** rol → funciones (con sus variables como secretos) → topic → colas (con política y DLQ) → suscripciones → triggers → prueba con un `aws sns publish`. Es la misma secuencia de esta guía.

---

## 10. Apagar, limpiar y errores frecuentes

**Apagar** (conserva los datos de Postgres):
```
docker compose down
```
**Borrar todo**, incluida la base (para empezar de cero; el SQL inicial se ejecuta de nuevo):
```
docker compose down -v
```
Y detén el servicio de avisos con `Ctrl+C` en la terminal B. **LocalStack pierde todo al reiniciarse**: topic, colas y Lambdas se recrean repitiendo las secciones 6 y 7. Borra también `env-*.json`.

**Errores frecuentes**

| Síntoma | Causa | Solución |
|---|---|---|
| `aws sts get-caller-identity` muestra una cuenta de 12 dígitos distinta de `000000000000` | Estás apuntando a AWS real | Detente. `export AWS_PROFILE=localstack` (o `$env:AWS_PROFILE`) |
| `Could not connect to the endpoint URL` | LocalStack no está arriba, o el perfil no tiene `endpoint_url` | `docker compose ps`; `aws configure list` |
| La Lambda 1 falla: `getaddrinfo ENOTFOUND postgres` | La Lambda no está en la red del compose | Comprueba `LAMBDA_DOCKER_NETWORK: aws-local-net` y **recrea** el contenedor de LocalStack (`docker compose up -d --force-recreate localstack`), luego repite las secciones 6 y 7 |
| La Lambda 2 falla: `fetch failed` / `ECONNREFUSED` | El servicio de avisos no está arriba; o el firewall bloquea el puerto 8082; o en **Mac/Windows** se definió `LAMBDA_FLAGS` con `--add-host=…host-gateway` (rompe `host.docker.internal`); o en **Linux** falta definirla | Terminal B corriendo y `curl http://localhost:8082/api/notice`; acepta el firewall; Mac/Windows: `unset LAMBDA_FLAGS` y recrea con `docker compose up -d --force-recreate localstack`; Linux: `export LAMBDA_FLAGS=…` y recrea |
| La Lambda 2 falla: `HTTP 401` | `NOTICE_API_KEY` de la Lambda distinta de la del servicio | Misma clave en `env-generator.json` y al arrancar el servicio |
| Publicas y no llega nada, sin errores | El `type` no cumple ningún filtro (sección 8.3) | Compara `type` con la tabla de la sección 7.1 |
| Un mensaje aparece en la DLQ | Falló 3 veces | Mira el log de la Lambda: `errorMessage` |
| `aws logs …` dice que el grupo no existe | La Lambda aún no se ha invocado | Espera a la primera invocación |
| Gradle: `No matching toolchains…` / `Could not resolve…` | JDK distinto o red corporativa | Nota de la sección 4 |
| En PowerShell: `unexpected token` o JSON deformado | Comillas dentro de un argumento | Pasa el JSON siempre por archivo con `file://` |
| Tras `docker compose down`, falta todo en LocalStack | LocalStack no persiste | Repite las secciones 6 y 7 |

**Ver la DLQ** (mensajes que fallaron 3 veces):

**bash**
```bash
aws sqs receive-message --queue-url "$(aws sqs get-queue-url --queue-name report-generation-dlq --query QueueUrl --output text)" \
  --max-number-of-messages 5 --query 'Messages[].Body' --output text
```
**PowerShell**
```powershell
$u = aws sqs get-queue-url --queue-name report-generation-dlq --query QueueUrl --output text
aws sqs receive-message --queue-url $u --max-number-of-messages 5 --query 'Messages[].Body' --output text
```
