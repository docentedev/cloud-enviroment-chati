# Guía manual: construir el sistema a mano en LocalStack (y llevarlo a AWS real)

Todo con el AWS CLI, **sin los scripts del proyecto**. La idea es que entiendas cada pieza: cómo el CLI sabe si habla con LocalStack o con AWS real, cómo se crean y conectan SNS, SQS y las Lambdas, cómo probar cada tramo, cómo invocar una Lambda a mano y cómo conectar tu servidor. Los comandos son los **mismos** para LocalStack y para AWS real: lo único que cambia es el perfil del CLI y unos pocos valores, que se listan en la [parte 9](#9-pasar-a-aws-real).

**No necesitas clonar ningún repositorio** para las partes 1 a 7: las Lambdas son ejemplos de pocas líneas que se escriben en la propia guía y solo imprimen en consola. El código real del proyecto es opcional (6.8).

**Funciona en Mac, Linux y Windows.** Cada paso trae dos versiones: **bash** (Mac, Linux, WSL, Git Bash) y **PowerShell** (Windows). Léelas en la sección [Mac, Linux y Windows](#mac-linux-y-windows-cómo-leer-esta-guía).

Arquitectura que se construye (detalle en `GUIA.md`):

```
SNS report-events ─┬─► SQS report-requests   (+DLQ) ──► Lambda report-request-recorder ──► Postgres
 (filtro por type) └─► SQS report-generation (+DLQ) ──► Lambda report-generator ──► API de avisos
```

## Índice
- [Mac, Linux y Windows: cómo leer esta guía](#mac-linux-y-windows-cómo-leer-esta-guía)
1. [Cómo sabe el CLI si va a LocalStack o a AWS real](#1-cómo-sabe-el-cli-si-va-a-localstack-o-a-aws-real)
2. [Levantar LocalStack](#2-levantar-localstack)
3. [Configurar el CLI hacia LocalStack y primera prueba: una Lambda mínima](#3-configurar-el-cli-hacia-localstack-y-primera-prueba-una-lambda-mínima)
4. [SNS y SQS, y conectarlos](#4-sns-y-sqs-y-conectarlos)
5. [Probar el tramo SNS → SQS antes de crear las Lambdas](#5-probar-el-tramo-sns--sqs)
6. [Lambdas y sus triggers](#6-lambdas-y-sus-triggers)
7. [Invocar una Lambda a mano y ver sus logs](#7-invocar-una-lambda-a-mano-y-ver-sus-logs)
8. [Conectar tu servidor a LocalStack](#8-conectar-tu-servidor-a-localstack)
   - [8.1 Ejemplo mínimo: una API Spring Boot de prueba que publica en SNS](#81-ejemplo-mínimo-una-api-spring-boot-de-prueba-que-publica-en-sns-sin-autenticación)
9. [Pasar a AWS real](#9-pasar-a-aws-real)
10. [Inspeccionar, depurar y limpiar](#10-inspeccionar-depurar-y-limpiar)
11. [Errores frecuentes](#11-errores-frecuentes)
12. [Resumen: local frente a real](#12-resumen-local-frente-a-real)

---

## Mac, Linux y Windows: cómo leer esta guía

**Trabaja en una carpeta vacía, fuera de cualquier repositorio** (por ejemplo `mkdir ~/localstack-guia && cd ~/localstack-guia`). La guía crea archivos de apoyo (zips, JSON, y en 6.9 un archivo con tu API key), y un `git add -A` descuidado podría subirlos.

**Qué terminal usar en cada sistema**

| Sistema | Opción | Qué versión de los comandos |
|---|---|---|
| Mac / Linux | Terminal | **bash** |
| Windows | **WSL2** (recomendada) | **bash**, sin cambios |
| Windows | Git Bash | **bash**, con un ajuste (abajo) |
| Windows | PowerShell nativo | **PowerShell** |

**Requisitos en Windows**
- **Docker Desktop** con el backend WSL2 (el valor por defecto). Con él, LocalStack queda en `localhost:4566` tanto desde PowerShell como desde WSL.
- **AWS CLI v2** (`aws --version`), **Node** y **Java 21** instalados en Windows. Si trabajas dentro de WSL, instálalos dentro de WSL: son entornos separados.
- `curl.exe` y `tar.exe` vienen con Windows 10 (1803) y 11.

**Si usas Git Bash**, antes de empezar ejecuta `export MSYS_NO_PATHCONV=1`. Sin eso, Git Bash convierte argumentos que empiezan con `/` en rutas de Windows, y `--log-group-name /aws/lambda/hello` llega al CLI como `C:/Program Files/Git/aws/lambda/hello`.

**Qué cambia en PowerShell** (las versiones PowerShell de esta guía ya lo aplican):

| bash | PowerShell | Por qué |
|---|---|---|
| `export VAR=valor` | `$env:VAR = "valor"` (variable de entorno) o `$VAR = "valor"` (solo de la sesión) | Son cosas distintas: el CLI lee las de entorno |
| `$(comando)` | `$X = comando` | |
| `\` al final de la línea | `` ` `` (acento grave) al final de la línea | Continuación de línea |
| `cat > f <<'EOF' … EOF` | `@' … '@ \| Set-Content -Encoding ascii f` | *Here-string* |
| `--payload '{"a":1}'` (JSON en el comando) | JSON siempre **en un archivo** y `file://archivo` | PowerShell 5.1 deforma las comillas dobles al llamar programas externos |
| `$ACCOUNT_ID:role` | `${ACCOUNT_ID}:role` | `$VAR:texto` se lee como "unidad de disco" y queda vacío |
| `-e A=x,y,z` | `-e "A=x,y,z"` | La coma crea una lista en PowerShell |
| `base64 --decode` | `[Text.Encoding]::UTF8.GetString([Convert]::FromBase64String($x))` | |
| `grep` | `Select-String` | |
| `sleep 10` | `Start-Sleep 10` | |
| `curl` | `curl.exe` | `curl` es un alias de otra cosa en PowerShell 5.1 |
| `a && b` | `a; b` | `&&` solo existe en PowerShell 7 |
| `zip` | `tar -a -c -f archivo.zip …` | No hay `zip`; evita `Compress-Archive` (ver parte 6.8) |
| `unset VAR` | `Remove-Item Env:VAR` | |

**Regla de oro en PowerShell:** todo JSON se escribe en un archivo con `Set-Content -Encoding ascii` y se pasa con `file://`. Así funciona igual en PowerShell 5.1 y en 7. (`-Encoding ascii` evita que Windows PowerShell 5.1 agregue una marca BOM al archivo, que algunos parsers rechazan.)

> **Estado de esta guía:** los bloques **bash** se ejecutaron contra LocalStack 3.8.1 (parte 3.3) y su sintaxis está validada. Los bloques **PowerShell** se validaron en su estructura (here-strings balanceados, JSON válido), pero **no se han ejecutado en una máquina Windows**. Si alguno falla, anota el comando y el error.

---

## 1. Cómo sabe el CLI si va a LocalStack o a AWS real

El AWS CLI **no sabe nada de LocalStack**. Por defecto habla con `https://<servicio>.<región>.amazonaws.com`, o sea con AWS real. Para que hable con otra dirección hay que decírselo explícitamente. Hay cuatro formas, de mayor a menor prioridad:

| Forma | Alcance | Ejemplo |
|---|---|---|
| 1. Opción en el comando | Un solo comando | `aws sns list-topics --endpoint-url http://localhost:4566` |
| 2. Variable de entorno de un servicio | Esa terminal, un servicio | `AWS_ENDPOINT_URL_SNS=http://localhost:4566` |
| 3. Variable de entorno global | Esa terminal, todo | `AWS_ENDPOINT_URL=http://localhost:4566` |
| 4. `endpoint_url` en un **perfil** | Cada vez que uses ese perfil | `aws --profile localstack …` |

Sin ninguna de las cuatro, el CLI va a AWS real con las credenciales que tenga configuradas.

**Mi recomendación: usa un perfil dedicado (la forma 4).** Resuelve los dos peligros a la vez:
- **No hay riesgo de tocar AWS real por error.** El perfil `localstack` solo tiene claves falsas (`test`/`test`): aunque olvides el endpoint, AWS real rechazaría esas credenciales.
- **No hay que repetir `--endpoint-url`** en cada comando.

Es peligrosa la forma 3 si la dejas puesta: una terminal con `AWS_ENDPOINT_URL` definido manda *todos* tus comandos a LocalStack hasta que la cierres, y te puede confundir ("¿por qué no veo mis recursos reales?"). La inversa también es peligrosa: usar tus credenciales reales (el perfil `default`) y olvidar el endpoint crea recursos **en tu cuenta real**.

**Cómo comprobar dónde estás, siempre antes de crear nada** (igual en bash y PowerShell):
```
aws sts get-caller-identity
```
- Cuenta `000000000000` → LocalStack.
- Tu cuenta real (12 dígitos distintos) → AWS real.

Hazlo un hábito: es el equivalente de mirar en qué base de datos estás conectado.

> Alternativa: `pip install awscli-local` instala el comando `awslocal`, que es el `aws` con el endpoint de LocalStack ya puesto. Esta guía usa el `aws` normal para que se vea qué está pasando.

---

## 2. Levantar LocalStack
Necesitas Docker abierto (en Windows, Docker Desktop).

**bash**
```bash
docker run -d --name localstack \
  -p 127.0.0.1:4566:4566 \
  -e SERVICES=sns,sqs,lambda,logs,iam,sts \
  -e LAMBDA_DOCKER_FLAGS="--add-host=host.docker.internal:host-gateway" \
  -v /var/run/docker.sock:/var/run/docker.sock \
  localstack/localstack:3.8.1
```
**PowerShell**
```powershell
docker run -d --name localstack `
  -p 127.0.0.1:4566:4566 `
  -e "SERVICES=sns,sqs,lambda,logs,iam,sts" `
  -e "LAMBDA_DOCKER_FLAGS=--add-host=host.docker.internal:host-gateway" `
  -v /var/run/docker.sock:/var/run/docker.sock `
  localstack/localstack:3.8.1
```
- `-p 127.0.0.1:4566:4566`: todo LocalStack escucha en un solo puerto, y solo en tu máquina.
- `SERVICES=…`: solo los servicios que usamos. (En PowerShell va entre comillas: sin ellas, la coma separa argumentos.)
- `-v /var/run/docker.sock…`: LocalStack ejecuta cada Lambda en **otro contenedor**, por eso necesita hablar con Docker. La ruta es la del *interior* de Docker, así que se escribe igual en Windows.
- `LAMBDA_DOCKER_FLAGS`: permite que las Lambdas alcancen servicios de tu máquina con el nombre `host.docker.internal` (en Linux hace falta; en Docker Desktop ya existe).
- La versión está fijada porque las más recientes pueden exigir un token.

El repo trae lo mismo en `local/docker-compose.yml` (`docker compose -f local/docker-compose.yml up -d`).

Comprueba que responde:

**bash**
```bash
curl -s http://localhost:4566/_localstack/health
```
**PowerShell**
```powershell
curl.exe -s http://localhost:4566/_localstack/health
```
Debe mostrar un JSON con `sns`, `sqs` y `lambda` como `available` o `running`.

**LocalStack no guarda nada entre reinicios.** Si apagas el contenedor (o reinicias Docker), pierdes topic, colas y Lambdas, y hay que recrearlos con esta guía.

---

## 3. Configurar el CLI hacia LocalStack y primera prueba: una Lambda mínima

### 3.1 Configurar el CLI
Crea un perfil `localstack` con claves falsas y el endpoint (el mismo comando en bash y PowerShell):
```
aws configure set aws_access_key_id     test                   --profile localstack
aws configure set aws_secret_access_key test                   --profile localstack
aws configure set region                us-east-2              --profile localstack
aws configure set endpoint_url          http://localhost:4566  --profile localstack
```
Eso escribe en `~/.aws/credentials` y `~/.aws/config` (en Windows, `%USERPROFILE%\.aws\`). Requiere AWS CLI v2.13 o superior (`aws --version`); en una anterior, usa `--endpoint-url` en cada comando.

Actívalo **en esta terminal**:

**bash**
```bash
export AWS_PROFILE=localstack
aws sts get-caller-identity
aws configure list
```
**PowerShell**
```powershell
$env:AWS_PROFILE = "localstack"
aws sts get-caller-identity
aws configure list
```
Debes ver `"Account": "000000000000"`. `aws configure list` muestra de dónde sale cada valor (el perfil). Para volver a AWS real en esa terminal: `unset AWS_PROFILE` (bash) o `Remove-Item Env:AWS_PROFILE` (PowerShell), o apunta a tu perfil real.

### 3.2 Variables que usará el resto de la guía
**bash**
```bash
export AWS_REGION=us-east-2
export ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)
echo "$ACCOUNT_ID"          # 000000000000 en LocalStack
```
**PowerShell**
```powershell
$AWS_REGION = "us-east-2"
$env:AWS_REGION = $AWS_REGION
$ACCOUNT_ID = aws sts get-caller-identity --query Account --output text
$ACCOUNT_ID                 # 000000000000 en LocalStack
```
> Dentro de textos, escribe siempre `${ACCOUNT_ID}` con llaves (en zsh y en PowerShell): `$ACCOUNT_ID:role` se interpreta como un modificador o una unidad de disco, y rompe el ARN.

### 3.3 Primera prueba: una Lambda mínima de punta a punta
Antes de construir el sistema, comprueba que tu entorno funciona con lo más pequeño posible: una Lambda `hello` que solo imprime lo que recibe. Si esto funciona, el resto de la guía también. **La versión bash se probó contra LocalStack 3.8.1** y terminó sin errores.

**1. El código y el zip.**

**bash**
```bash
mkdir hello-lambda && cd hello-lambda
cat > index.js <<'EOF'
exports.handler = async (event) => {
  console.log("Evento recibido:", JSON.stringify(event));
  return { ok: true, recibido: event };
};
EOF
zip hello.zip index.js
```
**PowerShell**
```powershell
mkdir hello-lambda; cd hello-lambda
@'
exports.handler = async (event) => {
  console.log("Evento recibido:", JSON.stringify(event));
  return { ok: true, recibido: event };
};
'@ | Set-Content -Encoding ascii index.js
tar -a -c -f hello.zip index.js
```
(`tar -a -c -f archivo.zip` crea un zip por la extensión del nombre.)

**2. Crear la función** (el rol es solo un ARN con la forma correcta: LocalStack no lo valida) **y esperar a que esté lista:**

**bash**
```bash
aws lambda create-function --function-name hello \
  --runtime nodejs20.x --handler index.handler \
  --role "arn:aws:iam::${ACCOUNT_ID}:role/lambda-role" \
  --zip-file fileb://hello.zip

aws lambda wait function-active-v2 --function-name hello
aws lambda get-function --function-name hello --query 'Configuration.[State,LastUpdateStatus]' --output text
```
**PowerShell**
```powershell
aws lambda create-function --function-name hello `
  --runtime nodejs20.x --handler index.handler `
  --role "arn:aws:iam::${ACCOUNT_ID}:role/lambda-role" `
  --zip-file fileb://hello.zip

aws lambda wait function-active-v2 --function-name hello
aws lambda get-function --function-name hello --query 'Configuration.[State,LastUpdateStatus]' --output text
```
La salida de `create-function` dice `State: Pending`, y la última línea debe decir `Active  Successful`.

**3. Invocarla a mano.**

**bash**
```bash
aws lambda invoke --function-name hello \
  --cli-binary-format raw-in-base64-out --payload '{"nombre":"mundo"}' \
  --log-type Tail --query LogResult --output text out.json | base64 --decode
cat out.json
```
**PowerShell**
```powershell
'{"nombre":"mundo"}' | Set-Content -Encoding ascii payload.json
$log = aws lambda invoke --function-name hello `
  --cli-binary-format raw-in-base64-out --payload file://payload.json `
  --log-type Tail --query LogResult --output text out.json
[Text.Encoding]::UTF8.GetString([Convert]::FromBase64String($log))
Get-Content out.json
```
Resultado esperado:
```
START RequestId: … Version: $LATEST
… INFO  Evento recibido: {"nombre":"mundo"}
END RequestId: …
REPORT RequestId: … Duration: 25.68 ms  Billed Duration: 26 ms  Memory Size: 128 MB …
{"ok":true,"recibido":{"nombre":"mundo"}}
```
Las cuatro primeras líneas son el log de la ejecución; la última, lo que devolvió la función (`out.json`). La primera invocación puede tardar más, porque LocalStack descarga la imagen de Lambda.

**4. Conectarla a una cola (un trigger).** Crea una cola, enlázala a la función y envía un mensaje con una marca única, para poder reconocerlo:

**bash**
```bash
Q_URL=$(aws sqs create-queue --queue-name hello-queue --query QueueUrl --output text)
Q_ARN=$(aws sqs get-queue-attributes --queue-url "$Q_URL" --attribute-names QueueArn --query Attributes.QueueArn --output text)

aws lambda create-event-source-mapping --function-name hello --event-source-arn "$Q_ARN" --batch-size 1
aws lambda list-event-source-mappings --function-name hello --query 'EventSourceMappings[].State' --output text   # Enabled

MARK="marca-$(date +%s)"
aws sqs send-message --queue-url "$Q_URL" --message-body "{\"hola\":\"$MARK\"}"
```
**PowerShell**
```powershell
$Q_URL = aws sqs create-queue --queue-name hello-queue --query QueueUrl --output text
$Q_ARN = aws sqs get-queue-attributes --queue-url $Q_URL --attribute-names QueueArn --query Attributes.QueueArn --output text

aws lambda create-event-source-mapping --function-name hello --event-source-arn $Q_ARN --batch-size 1
aws lambda list-event-source-mappings --function-name hello --query 'EventSourceMappings[].State' --output text   # Enabled

$MARK = "marca-" + [DateTimeOffset]::UtcNow.ToUnixTimeSeconds()
@{ hola = $MARK } | ConvertTo-Json -Compress | Set-Content -Encoding ascii body.json
aws sqs send-message --queue-url $Q_URL --message-body file://body.json
```

**5. Comprobar que la Lambda procesó ese mensaje.** Espera unos 10 segundos y busca la marca en el log:

**bash**
```bash
sleep 10
aws logs filter-log-events --log-group-name /aws/lambda/hello --query 'events[].message' --output text | tr '\t' '\n' | grep "$MARK"
```
**PowerShell**
```powershell
Start-Sleep 10
(aws logs filter-log-events --log-group-name /aws/lambda/hello --query 'events[].message' --output text) -split "`t" | Select-String $MARK
```
`--output text` junta todos los mensajes del log en **una sola línea separada por tabuladores**; `tr '\t' '\n'` (o `-split "`t"` en PowerShell) los separa uno por línea, para mostrar solo el evento que contiene tu marca y no todo el historial. Debe salir algo así:
```
… INFO  Evento recibido: {"Records":[{"messageId":"…","body":"{\"hola\": \"marca-1791891606\"}","attributes":{…},"eventSource":"aws:sqs",…}]}
```
Si aparece esa línea con tu marca, el trigger funciona: SQS invocó la Lambda por sí solo (en la prueba tardó 1 segundo). Fíjate en `"eventSource":"aws:sqs"`: así es el evento real que reciben las Lambdas conectadas a una cola, igual al que fabricas a mano en la parte 7.1. Si no sale nada, espera otros 10 segundos y repite. La cola debe quedar vacía:
```
aws sqs get-queue-attributes --queue-url <URL_DE_LA_COLA> --attribute-names ApproximateNumberOfMessages ApproximateNumberOfMessagesNotVisible --query 'Attributes.[ApproximateNumberOfMessages,ApproximateNumberOfMessagesNotVisible]' --output text
```
(en bash, `--queue-url "$Q_URL"`; en PowerShell, `--queue-url $Q_URL`). Debe responder `0   0`.
> **Busca la marca con `grep` / `Select-String`, no con `--filter-pattern`:** LocalStack no aplica ese filtro de forma fiable y devuelve **todos** los eventos del grupo. Con él, una verificación puede dar "éxito" mostrando logs de una ejecución anterior sin que tu mensaje haya llegado: por eso se busca una marca única en el texto.

**6. Limpiar** (en este orden: primero el mapeo):

**bash**
```bash
UUID=$(aws lambda list-event-source-mappings --function-name hello --query 'EventSourceMappings[0].UUID' --output text)
aws lambda delete-event-source-mapping --uuid "$UUID"
aws lambda delete-function --function-name hello
aws sqs delete-queue --queue-url "$Q_URL"
aws logs delete-log-group --log-group-name /aws/lambda/hello
cd .. && rm -rf hello-lambda
```
**PowerShell**
```powershell
$UUID = aws lambda list-event-source-mappings --function-name hello --query 'EventSourceMappings[0].UUID' --output text
aws lambda delete-event-source-mapping --uuid $UUID
aws lambda delete-function --function-name hello
aws sqs delete-queue --queue-url $Q_URL
aws logs delete-log-group --log-group-name /aws/lambda/hello
cd ..; Remove-Item -Recurse -Force hello-lambda
```

**Qué demuestra cada paso:** el 2 prueba que el CLI llega a LocalStack y que Docker ejecuta Lambdas; el 3, que puedes invocar una función y ver sus logs; el 4 y el 5, que un mensaje en una cola dispara la función sin que escribas código de lectura. Es exactamente el mecanismo que usa el proyecto.

**En AWS real** es la misma secuencia, con tu perfil real y dos cambios: el rol de la Lambda debe existir (parte 9.1) y el runtime sería `nodejs22.x`. Cuidado: tendrías que crear la cola en tu cuenta real.

---

## 4. SNS y SQS, y conectarlos

### 4.1 El topic (SNS)
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
Un **ARN** es el identificador único de un recurso. En LocalStack será `arn:aws:sns:us-east-2:000000000000:report-events`; en AWS real, el mismo con tu número de cuenta.

### 4.2 Cada cola necesita cuatro pasos
Para cada cola: **(1)** su DLQ, **(2)** la cola con la regla de reintentos, **(3)** permiso para que SNS escriba en ella, **(4)** la suscripción con el filtro. Los comandos usan la variable `Q`: ejecuta el bloque completo una vez con `report-requests` y otra con `report-generation`.

**bash**
```bash
Q=report-requests        # luego repite todo el bloque con Q=report-generation

# 1) La DLQ: adonde van los mensajes que fallan demasiadas veces
DLQ_URL=$(aws sqs create-queue --queue-name "$Q-dlq" --attributes MessageRetentionPeriod=1209600 --query QueueUrl --output text)
DLQ_ARN=$(aws sqs get-queue-attributes --queue-url "$DLQ_URL" --attribute-names QueueArn --query Attributes.QueueArn --output text)

# 2) La cola. RedrivePolicy es JSON dentro de JSON, por eso los atributos van en un archivo.
cat > attrs.json <<EOF
{"VisibilityTimeout":"90","RedrivePolicy":"{\"deadLetterTargetArn\":\"$DLQ_ARN\",\"maxReceiveCount\":\"3\"}"}
EOF
Q_URL=$(aws sqs create-queue --queue-name "$Q" --attributes file://attrs.json --query QueueUrl --output text)
Q_ARN=$(aws sqs get-queue-attributes --queue-url "$Q_URL" --attribute-names QueueArn --query Attributes.QueueArn --output text)

# 3) Permiso: SNS (solo este topic) puede escribir en la cola
cat > policy.json <<EOF
{"Policy":"{\"Version\":\"2012-10-17\",\"Statement\":[{\"Effect\":\"Allow\",\"Principal\":{\"Service\":\"sns.amazonaws.com\"},\"Action\":\"sqs:SendMessage\",\"Resource\":\"$Q_ARN\",\"Condition\":{\"ArnEquals\":{\"aws:SourceArn\":\"$TOPIC_ARN\"}}}]}"}
EOF
aws sqs set-queue-attributes --queue-url "$Q_URL" --attributes file://policy.json

# 4) La suscripción: conecta el topic con la cola, con un filtro
aws sns subscribe --topic-arn "$TOPIC_ARN" --protocol sqs --notification-endpoint "$Q_ARN" \
  --attributes '{"FilterPolicy":"{\"type\":[\"report.requested\"]}","FilterPolicyScope":"MessageBody","RawMessageDelivery":"false"}'
```
**PowerShell**
```powershell
$Q = "report-requests"       # luego repite todo el bloque con $Q = "report-generation"

# 1) La DLQ: adonde van los mensajes que fallan demasiadas veces
$DLQ_URL = aws sqs create-queue --queue-name "$Q-dlq" --attributes MessageRetentionPeriod=1209600 --query QueueUrl --output text
$DLQ_ARN = aws sqs get-queue-attributes --queue-url $DLQ_URL --attribute-names QueueArn --query Attributes.QueueArn --output text

# 2) La cola. RedrivePolicy es JSON dentro de JSON, por eso los atributos van en un archivo.
@"
{"VisibilityTimeout":"90","RedrivePolicy":"{\"deadLetterTargetArn\":\"$DLQ_ARN\",\"maxReceiveCount\":\"3\"}"}
"@ | Set-Content -Encoding ascii attrs.json
$Q_URL = aws sqs create-queue --queue-name $Q --attributes file://attrs.json --query QueueUrl --output text
$Q_ARN = aws sqs get-queue-attributes --queue-url $Q_URL --attribute-names QueueArn --query Attributes.QueueArn --output text

# 3) Permiso: SNS (solo este topic) puede escribir en la cola
@"
{"Policy":"{\"Version\":\"2012-10-17\",\"Statement\":[{\"Effect\":\"Allow\",\"Principal\":{\"Service\":\"sns.amazonaws.com\"},\"Action\":\"sqs:SendMessage\",\"Resource\":\"$Q_ARN\",\"Condition\":{\"ArnEquals\":{\"aws:SourceArn\":\"$TOPIC_ARN\"}}}]}"}
"@ | Set-Content -Encoding ascii policy.json
aws sqs set-queue-attributes --queue-url $Q_URL --attributes file://policy.json

# 4) La suscripción: conecta el topic con la cola, con un filtro
@'
{"FilterPolicy":"{\"type\":[\"report.requested\"]}","FilterPolicyScope":"MessageBody","RawMessageDelivery":"false"}
'@ | Set-Content -Encoding ascii sub-attrs.json
aws sns subscribe --topic-arn $TOPIC_ARN --protocol sqs --notification-endpoint $Q_ARN --attributes file://sub-attrs.json
```
(En los *here-strings* de PowerShell con `@"` se sustituyen las variables; con `@'` no. Por eso los dos primeros usan `@"` y el último `@'`.)

Guarda las direcciones de cada cola, que usarás más adelante. Hazlo justo después de cada ejecución del bloque:

**bash**
```bash
# tras Q=report-requests:
REQUESTS_URL=$Q_URL;   REQUESTS_ARN=$Q_ARN
# tras Q=report-generation:
GENERATION_URL=$Q_URL; GENERATION_ARN=$Q_ARN
```
**PowerShell**
```powershell
# tras $Q = "report-requests":
$REQUESTS_URL = $Q_URL;   $REQUESTS_ARN = $Q_ARN
# tras $Q = "report-generation":
$GENERATION_URL = $Q_URL; $GENERATION_ARN = $Q_ARN
```

**Qué significa cada parte:**
- **`maxReceiveCount: 3`:** si la Lambda falla 3 veces con el mismo mensaje, SQS lo mueve a la DLQ en vez de reintentarlo para siempre. Se conserva 14 días.
- **`VisibilityTimeout: 90`:** mientras una Lambda procesa un mensaje, queda oculto 90 s. Debe ser **mayor que el timeout de la Lambda** (15 s). Si fuera menor, otra Lambda podría tomar el mismo mensaje a la vez.
- **`FilterPolicyScope: MessageBody`:** el filtro mira dentro del JSON del mensaje, no en sus atributos. Un mensaje llega solo si su campo `type` vale `report.requested`; los demás se descartan en silencio.
- **`RawMessageDelivery: false`:** SNS envuelve el mensaje en un sobre `{"Type":"Notification","Message":"…"}`. Las Lambdas del proyecto lo desenvuelven. Con `true` llegaría el mensaje tal cual y habría que quitar ese desenvuelto.
- **La política de la cola:** en LocalStack no se exige, pero **en AWS real es obligatoria**: sin ella SNS no puede escribir. Incluirla aquí evita que lo olvides al pasar a real.

Los archivos `attrs.json`, `policy.json` y `sub-attrs.json` se pisan en cada vuelta; puedes borrarlos al terminar.

---

## 5. Probar el tramo SNS → SQS
Prueba esto **antes** de crear las Lambdas: si hay un error de conexión, es más fácil verlo sin ellas.

**Un mensaje que cumple el filtro debe llegar a las dos colas:**

**bash**
```bash
aws sns publish --topic-arn "$TOPIC_ARN" --message '{"type":"report.requested","body":{"requestId":"t1","requester":"yo@ejemplo.com","requesterId":"s1","reportType":"INVENTORY","params":{},"requestedAt":"2026-01-01T00:00:00Z"}}'

aws sqs receive-message --queue-url "$REQUESTS_URL"   --wait-time-seconds 3
aws sqs receive-message --queue-url "$GENERATION_URL" --wait-time-seconds 3
```
**PowerShell**
```powershell
@'
{"type":"report.requested","body":{"requestId":"t1","requester":"yo@ejemplo.com","requesterId":"s1","reportType":"INVENTORY","params":{},"requestedAt":"2026-01-01T00:00:00Z"}}
'@ | Set-Content -Encoding ascii msg.json
aws sns publish --topic-arn $TOPIC_ARN --message file://msg.json

aws sqs receive-message --queue-url $REQUESTS_URL   --wait-time-seconds 3
aws sqs receive-message --queue-url $GENERATION_URL --wait-time-seconds 3
```
Verás el sobre de SNS en el campo `Body`: `{"Type":"Notification","MessageId":…,"Message":"{\"type\":\"report.requested\",…}"}`. Es lo que recibirán las Lambdas.

**Un mensaje de otro tipo no debe llegar:**

**bash**
```bash
aws sns publish --topic-arn "$TOPIC_ARN" --message '{"type":"otro","body":{}}'
aws sqs receive-message --queue-url "$REQUESTS_URL" --wait-time-seconds 3
```
**PowerShell**
```powershell
'{"type":"otro","body":{}}' | Set-Content -Encoding ascii msg-otro.json
aws sns publish --topic-arn $TOPIC_ARN --message file://msg-otro.json
aws sqs receive-message --queue-url $REQUESTS_URL --wait-time-seconds 3
```
La respuesta debe estar vacía: el filtro lo descartó.

**Limpia las colas** para que esos mensajes de prueba no los procesen las Lambdas cuando las crees:

**bash**
```bash
aws sqs purge-queue --queue-url "$REQUESTS_URL"
aws sqs purge-queue --queue-url "$GENERATION_URL"
```
**PowerShell**
```powershell
aws sqs purge-queue --queue-url $REQUESTS_URL
aws sqs purge-queue --queue-url $GENERATION_URL
```
(SQS limita `purge-queue` a una vez por minuto por cola en AWS real.)

---

## 6. Lambdas y sus triggers

Para no depender de ningún repositorio, esta parte usa **dos Lambdas de ejemplo que solo imprimen en consola** lo que reciben. Se llaman como las reales (`report-request-recorder` y `report-generator`) para que el resto de la guía funcione igual, y ambas ejecutan el mismo código de unas pocas líneas. Al final (6.8) se explica cómo cambiarlas por el código real del proyecto, y en 6.9 cómo hacer que una de ellas **publique cada mensaje como un aviso en tu API** para validar el recorrido completo.

### 6.1 El código y el zip
Una Lambda de Node se sube como un `.zip` con su código. Este ejemplo tiene un solo archivo, sin dependencias, así que no hace falta `npm`:

**bash**
```bash
mkdir lambda-demo
cat > lambda-demo/index.js <<'EOF'
// Lambda de ejemplo: imprime en la consola cada mensaje que recibe de SQS.
function leer(texto) {
  try { return JSON.parse(texto); } catch { return texto; }
}

exports.handler = async (event) => {
  const nombre = process.env.AWS_LAMBDA_FUNCTION_NAME;
  const registros = event.Records || [];
  for (const registro of registros) {
    const externo = leer(registro.body);
    // SNS (sin raw delivery) envuelve el mensaje: {"Type":"Notification","Message":"..."}
    const mensaje = externo && externo.Message ? leer(externo.Message) : externo;
    console.log("[" + nombre + "] mensaje recibido:", JSON.stringify(mensaje));
  }
  console.log("[" + nombre + "] lote procesado: " + registros.length + " mensaje(s)");
  return { procesados: registros.length };
};
EOF
(cd lambda-demo && zip ../demo.zip index.js)
```
**PowerShell**
```powershell
mkdir lambda-demo
@'
// Lambda de ejemplo: imprime en la consola cada mensaje que recibe de SQS.
function leer(texto) {
  try { return JSON.parse(texto); } catch { return texto; }
}

exports.handler = async (event) => {
  const nombre = process.env.AWS_LAMBDA_FUNCTION_NAME;
  const registros = event.Records || [];
  for (const registro of registros) {
    const externo = leer(registro.body);
    // SNS (sin raw delivery) envuelve el mensaje: {"Type":"Notification","Message":"..."}
    const mensaje = externo && externo.Message ? leer(externo.Message) : externo;
    console.log("[" + nombre + "] mensaje recibido:", JSON.stringify(mensaje));
  }
  console.log("[" + nombre + "] lote procesado: " + registros.length + " mensaje(s)");
  return { procesados: registros.length };
};
'@ | Set-Content -Encoding ascii lambda-demo/index.js
Push-Location lambda-demo; tar -a -c -f ../demo.zip index.js; Pop-Location
```
Qué hace el código: recorre `event.Records` (cada mensaje de SQS llega ahí, con su texto en `body`), y si el `body` es el sobre de SNS (`{"Type":"Notification","Message":"…"}`) lo desenvuelve. Imprime cada mensaje con el nombre de la función (`AWS_LAMBDA_FUNCTION_NAME`, que AWS define solo), así sabes cuál de las dos lo recibió.

### 6.2 El rol
Toda Lambda se ejecuta con un **rol de IAM**. **LocalStack no valida el rol**: basta con un ARN con la forma correcta, sin crear nada.

**bash**
```bash
ROLE_ARN="arn:aws:iam::${ACCOUNT_ID}:role/lambda-role"
```
**PowerShell**
```powershell
$ROLE_ARN = "arn:aws:iam::${ACCOUNT_ID}:role/lambda-role"
```
En AWS real el rol **sí debe existir**, con permisos. Se explica en la [parte 9](#9-pasar-a-aws-real).

### 6.3 Variables de entorno
Las Lambdas de ejemplo **no necesitan ninguna**. Solo el código real del proyecto las usa (conexión a la base de datos, API de avisos): se explican en la 6.8.

### 6.4 Crear las funciones
La misma `demo.zip` sirve para las dos:

**bash**
```bash
aws lambda create-function --function-name report-request-recorder \
  --runtime nodejs20.x --handler index.handler --timeout 15 --memory-size 128 \
  --role "$ROLE_ARN" --zip-file fileb://demo.zip

aws lambda create-function --function-name report-generator \
  --runtime nodejs20.x --handler index.handler --timeout 15 --memory-size 128 \
  --role "$ROLE_ARN" --zip-file fileb://demo.zip
```
**PowerShell**
```powershell
aws lambda create-function --function-name report-request-recorder `
  --runtime nodejs20.x --handler index.handler --timeout 15 --memory-size 128 `
  --role $ROLE_ARN --zip-file fileb://demo.zip

aws lambda create-function --function-name report-generator `
  --runtime nodejs20.x --handler index.handler --timeout 15 --memory-size 128 `
  --role $ROLE_ARN --zip-file fileb://demo.zip
```
- `--handler index.handler`: el archivo `index.js` y la función exportada `handler`.
- `--timeout 15`: máximo 15 s por ejecución (debe ser menor que el `VisibilityTimeout` de la cola).
- `fileb://`: lee el zip como binario (con `file://` fallaría).

Espera a que estén listas (el mismo comando en bash y PowerShell):
```
aws lambda wait function-active-v2 --function-name report-request-recorder
aws lambda wait function-active-v2 --function-name report-generator
aws lambda get-function --function-name report-generator --query 'Configuration.[State,LastUpdateStatus]' --output text
```
Debe decir `Active  Successful`. La primera vez, LocalStack descarga la imagen de runtime y puede tardar.

### 6.5 Los triggers (event source mappings)
Un *event source mapping* hace que AWS lea una cola por ti y llame a la Lambda con lotes de mensajes. Tú no escribes ningún código de lectura.

**bash**
```bash
aws lambda create-event-source-mapping --function-name report-request-recorder \
  --event-source-arn "$REQUESTS_ARN"   --batch-size 5

aws lambda create-event-source-mapping --function-name report-generator \
  --event-source-arn "$GENERATION_ARN" --batch-size 5
```
**PowerShell**
```powershell
aws lambda create-event-source-mapping --function-name report-request-recorder `
  --event-source-arn $REQUESTS_ARN   --batch-size 5

aws lambda create-event-source-mapping --function-name report-generator `
  --event-source-arn $GENERATION_ARN --batch-size 5
```
Compruébalos (igual en ambos):
```
aws lambda list-event-source-mappings --query 'EventSourceMappings[].[FunctionArn,EventSourceArn,State]' --output table
```
Deben estar `Enabled`. `--batch-size 5` significa que la Lambda recibe hasta 5 mensajes por invocación. Si la Lambda lanza un error, **todo el lote** vuelve a la cola y cuenta como un intento: por eso las Lambdas reales son idempotentes.

### 6.6 Actualizar el código
Tras cambiar `lambda-demo/index.js`, vuelve a generar el zip (6.1) y sube el código; no hace falta recrear nada:

**bash**
```bash
aws lambda update-function-code --function-name report-generator --zip-file fileb://demo.zip
aws lambda wait function-updated --function-name report-generator
```
**PowerShell**
```powershell
aws lambda update-function-code --function-name report-generator --zip-file fileb://demo.zip
aws lambda wait function-updated --function-name report-generator
```
Para cambiar las **variables de entorno** se usa `aws lambda update-function-configuration --function-name <nombre> --environment file://env.json`. Ojo: **reemplaza todas** las variables, no agrega: siempre envía el conjunto completo.

### 6.7 Recrear una Lambda en LocalStack
Si borras una función y la recreas, borra **antes** su mapeo; si no, LocalStack responde `ResourceConflictException` porque el mapeo viejo sigue existiendo:

**bash**
```bash
aws lambda list-event-source-mappings --event-source-arn "$REQUESTS_ARN" --query 'EventSourceMappings[].UUID' --output text
aws lambda delete-event-source-mapping --uuid <UUID>
aws lambda delete-function --function-name report-request-recorder
```
**PowerShell**
```powershell
aws lambda list-event-source-mappings --event-source-arn $REQUESTS_ARN --query 'EventSourceMappings[].UUID' --output text
aws lambda delete-event-source-mapping --uuid <UUID>
aws lambda delete-function --function-name report-request-recorder
```

### 6.8 Opcional: usar el código real del proyecto
Las Lambdas de ejemplo solo imprimen. Si quieres que guarden la solicitud en Postgres y publiquen el aviso, cambia su código por el de los repositorios (necesitas haberlos clonado, parte 2.2 de `GUIA.md`: la carpeta `lambdas/pg`). Es solo reemplazar el código y agregar las variables que ese código usa; las funciones, colas y triggers no cambian.

**1. Empaquetar** (estas sí tienen dependencias, `pg`):

**bash**
```bash
(cd lambdas/pg/report-request-recorder && npm ci --omit=dev && zip -r ../recorder.zip .)
(cd lambdas/pg/report-generator        && npm ci --omit=dev && zip -r ../generator.zip .)
```
**PowerShell**
```powershell
Push-Location lambdas/pg/report-request-recorder; npm ci --omit=dev; tar -a -c -f ../recorder.zip *; Pop-Location
Push-Location lambdas/pg/report-generator;        npm ci --omit=dev; tar -a -c -f ../generator.zip *; Pop-Location
```
> **No uses `Compress-Archive` para esto en Windows PowerShell 5.1:** genera zips con `\` como separador de carpetas, y la Lambda (Linux) no encuentra `node_modules/pg/…`. `tar -a -c -f` produce un zip correcto. Con WSL o Git Bash puedes usar `zip` directamente.

**2. Las variables de entorno**, en archivos (las URLs de conexión tienen símbolos que rompen la sintaxis corta del CLI). Dentro del contenedor de la Lambda, `localhost` es el propio contenedor, por eso se usa `host.docker.internal` para alcanzar el Postgres de tu máquina. Con `NOTIFY_MODE=log`, el generador imprime el aviso en vez de llamar a la API real:

**bash**
```bash
cat > env-recorder.json <<'EOF'
{"Variables":{"DATABASE_URL":"postgresql://USUARIO@host.docker.internal:5432/reports_local","DATABASE_SSL":"false"}}
EOF
cat > env-generator.json <<'EOF'
{"Variables":{"DATABASE_URL":"postgresql://USUARIO@host.docker.internal:5432/reports_local","DATABASE_SSL":"false","NOTIFY_MODE":"log"}}
EOF
```
**PowerShell**
```powershell
@'
{"Variables":{"DATABASE_URL":"postgresql://USUARIO@host.docker.internal:5432/reports_local","DATABASE_SSL":"false"}}
'@ | Set-Content -Encoding ascii env-recorder.json
@'
{"Variables":{"DATABASE_URL":"postgresql://USUARIO@host.docker.internal:5432/reports_local","DATABASE_SSL":"false","NOTIFY_MODE":"log"}}
'@ | Set-Content -Encoding ascii env-generator.json
```
Reemplaza `USUARIO` por tu usuario de Postgres local. Si tu Postgres tiene contraseña: `postgresql://USUARIO:CLAVE@…`.

**3. Reemplazar el código y las variables.** El `--query LastUpdateStatus --output text` del final no es adorno: sin él, la respuesta de `update-function-configuration` **imprime todas las variables de entorno**, incluida la contraseña de la base:

**bash**
```bash
aws lambda update-function-code --function-name report-request-recorder --zip-file fileb://lambdas/pg/recorder.zip
aws lambda wait function-updated --function-name report-request-recorder
aws lambda update-function-configuration --function-name report-request-recorder --environment file://env-recorder.json --query LastUpdateStatus --output text

aws lambda update-function-code --function-name report-generator --zip-file fileb://lambdas/pg/generator.zip
aws lambda wait function-updated --function-name report-generator
aws lambda update-function-configuration --function-name report-generator --environment file://env-generator.json --query LastUpdateStatus --output text
```
**PowerShell**
```powershell
aws lambda update-function-code --function-name report-request-recorder --zip-file fileb://lambdas/pg/recorder.zip
aws lambda wait function-updated --function-name report-request-recorder
aws lambda update-function-configuration --function-name report-request-recorder --environment file://env-recorder.json --query LastUpdateStatus --output text

aws lambda update-function-code --function-name report-generator --zip-file fileb://lambdas/pg/generator.zip
aws lambda wait function-updated --function-name report-generator
aws lambda update-function-configuration --function-name report-generator --environment file://env-generator.json --query LastUpdateStatus --output text
```
Con el código real, el recorder necesita la tabla `report_requests` (SQL en `GUIA.md`, parte 1) y que Postgres acepte conexiones desde Docker; el generador, la tabla `products`. En AWS real las variables cambian (parte 9).

### 6.9 Prueba de punta a punta: SNS → Lambda → `/notices` (con tu API key)
Hasta aquí las Lambdas solo imprimen. En este paso `report-generator` **publica cada mensaje que recibe como un aviso en tu app**, con `POST /api/saas/notices` y tu API key, y compruebas todo el recorrido: SNS → cola → Lambda → API → pantalla de avisos. La otra Lambda (`report-request-recorder`) sigue imprimiendo, para ver que el mismo mensaje llega a las dos.

Necesitas la **URL del sitio de tu app** (la API cuelga de `/api`, que el código agrega solo) y una **API key** (se genera en tu perfil de la app; empieza con `crr_`). La API key es un **secreto**: no la pegues en ningún archivo de un repositorio ni en un chat, y nunca la escribas directamente en un comando (quedaría en tu historial). Los pasos la piden sin mostrarla y no la guardan en ningún sitio salvo en la configuración de la función.

> **Esto crea un aviso real en tu app.** El texto de la prueba es un JSON con la marca `e2e-…` y `PRUEBA_E2E`, fácil de reconocer. La API de avisos no se emula: se llama de verdad, y la Lambda (que corre en un contenedor de Docker) necesita salida a Internet.

**1. Tus datos**, sin que la clave se vea ni quede en el historial:

**bash** (funciona en bash y en zsh)
```bash
export API_NOTIFICATION="https://TU-APP/"
printf 'API key (no se muestra): '; stty -echo; read -r API_TOKEN_INTEGRATION; stty echo; echo
export API_TOKEN_INTEGRATION
```
**PowerShell**
```powershell
$env:API_NOTIFICATION = "https://TU-APP/"
$clave = Read-Host "API key (no se muestra)" -AsSecureString
$env:API_TOKEN_INTEGRATION = [Runtime.InteropServices.Marshal]::PtrToStringAuto([Runtime.InteropServices.Marshal]::SecureStringToBSTR($clave))
```

**2. El código del notificador.** Otra Lambda de ejemplo, también sin dependencias (usa el `fetch` que trae Node 20). Recibe cada mensaje de SQS, lo desenvuelve igual que la anterior y lo publica como aviso:

**bash**
```bash
mkdir lambda-notifier
cat > lambda-notifier/index.js <<'EOF'
// Lambda de ejemplo: publica cada mensaje recibido de SQS como un aviso en la API (POST /api/saas/notices).
function leer(texto) {
  try { return JSON.parse(texto); } catch { return texto; }
}

exports.handler = async (event) => {
  const base = (process.env.API_NOTIFICATION || "").trim().replace(/\/+$/, "");
  const token = process.env.API_TOKEN_INTEGRATION;
  if (!base || !token) throw new Error("Faltan API_NOTIFICATION y/o API_TOKEN_INTEGRATION");
  // La API cuelga de /api: se agrega si no viene ya en la URL
  const url = base + (/\/api$/i.test(base) ? "" : "/api") + "/saas/notices";

  for (const registro of event.Records || []) {
    const externo = leer(registro.body);
    const mensaje = externo && externo.Message ? leer(externo.Message) : externo;
    const texto = (typeof mensaje === "string" ? mensaje : JSON.stringify(mensaje)).slice(0, 2000);

    const res = await fetch(url, {
      method: "POST",
      headers: { Authorization: "ApiKey " + token, "Content-Type": "application/json" },
      body: JSON.stringify({ text: texto }),
      redirect: "manual", // un redirect convertiría el POST en GET y el aviso no se crearía
    });
    // Un error lanza una excepción: el mensaje vuelve a la cola y, tras 3 intentos, va a la DLQ.
    if (!res.ok) throw new Error("La API respondió HTTP " + res.status);
    console.log("Aviso publicado (HTTP " + res.status + "):", texto.slice(0, 120));
  }
};
EOF
(cd lambda-notifier && zip ../notifier.zip index.js)
```
**PowerShell**
```powershell
mkdir lambda-notifier
@'
// Lambda de ejemplo: publica cada mensaje recibido de SQS como un aviso en la API (POST /api/saas/notices).
function leer(texto) {
  try { return JSON.parse(texto); } catch { return texto; }
}

exports.handler = async (event) => {
  const base = (process.env.API_NOTIFICATION || "").trim().replace(/\/+$/, "");
  const token = process.env.API_TOKEN_INTEGRATION;
  if (!base || !token) throw new Error("Faltan API_NOTIFICATION y/o API_TOKEN_INTEGRATION");
  // La API cuelga de /api: se agrega si no viene ya en la URL
  const url = base + (/\/api$/i.test(base) ? "" : "/api") + "/saas/notices";

  for (const registro of event.Records || []) {
    const externo = leer(registro.body);
    const mensaje = externo && externo.Message ? leer(externo.Message) : externo;
    const texto = (typeof mensaje === "string" ? mensaje : JSON.stringify(mensaje)).slice(0, 2000);

    const res = await fetch(url, {
      method: "POST",
      headers: { Authorization: "ApiKey " + token, "Content-Type": "application/json" },
      body: JSON.stringify({ text: texto }),
      redirect: "manual", // un redirect convertiría el POST en GET y el aviso no se crearía
    });
    // Un error lanza una excepción: el mensaje vuelve a la cola y, tras 3 intentos, va a la DLQ.
    if (!res.ok) throw new Error("La API respondió HTTP " + res.status);
    console.log("Aviso publicado (HTTP " + res.status + "):", texto.slice(0, 120));
  }
};
'@ | Set-Content -Encoding ascii lambda-notifier/index.js
Push-Location lambda-notifier; tar -a -c -f ../notifier.zip index.js; Pop-Location
```
La API exige que el texto tenga entre 1 y 2000 caracteres (por eso el `slice`). El token viaja solo en el encabezado `Authorization: ApiKey …` y **nunca se escribe en el log**.

**3. Sustituir el código de `report-generator` y darle las variables.** Las variables van en un archivo que se borra enseguida. Fíjate en el `--query` final: sin él, la respuesta de `update-function-configuration` **imprimiría tu API key**:

**bash**
```bash
printf '{"Variables":{"API_NOTIFICATION":"%s","API_TOKEN_INTEGRATION":"%s"}}' "$API_NOTIFICATION" "$API_TOKEN_INTEGRATION" > env-notifier.json

aws lambda update-function-code --function-name report-generator --zip-file fileb://notifier.zip --query LastUpdateStatus --output text
aws lambda wait function-updated --function-name report-generator
aws lambda update-function-configuration --function-name report-generator --environment file://env-notifier.json --query LastUpdateStatus --output text
aws lambda wait function-updated --function-name report-generator

rm env-notifier.json
```
**PowerShell**
```powershell
@{ Variables = @{ API_NOTIFICATION = $env:API_NOTIFICATION; API_TOKEN_INTEGRATION = $env:API_TOKEN_INTEGRATION } } | ConvertTo-Json -Compress | Set-Content -Encoding ascii env-notifier.json

aws lambda update-function-code --function-name report-generator --zip-file fileb://notifier.zip --query LastUpdateStatus --output text
aws lambda wait function-updated --function-name report-generator
aws lambda update-function-configuration --function-name report-generator --environment file://env-notifier.json --query LastUpdateStatus --output text
aws lambda wait function-updated --function-name report-generator

Remove-Item env-notifier.json
```
Cada `update-…` imprime `InProgress` o `Successful`; los `wait` esperan a que termine.

**4. Enviar un mensaje al SNS**, con una marca única para reconocerlo:

**bash**
```bash
MARK="e2e-$(date +%s)"
aws sns publish --topic-arn "$TOPIC_ARN" \
  --message "{\"type\":\"report.requested\",\"body\":{\"requestId\":\"$MARK\",\"requester\":\"guia@ejemplo.com\",\"reportType\":\"PRUEBA_E2E\"}}"
echo "Busca este texto en /notices: $MARK"
```
**PowerShell**
```powershell
$MARK = "e2e-" + [DateTimeOffset]::UtcNow.ToUnixTimeSeconds()
@{ type = "report.requested"; body = @{ requestId = $MARK; requester = "guia@ejemplo.com"; reportType = "PRUEBA_E2E" } } | ConvertTo-Json -Compress | Set-Content -Encoding ascii msg-e2e.json
aws sns publish --topic-arn $TOPIC_ARN --message file://msg-e2e.json
"Busca este texto en /notices: $MARK"
```

**5. Comprobar el resultado**, en dos sitios.

*En el log de la Lambda*: debe decir que publicó el aviso (HTTP 201 = creado):

**bash**
```bash
sleep 10
aws logs filter-log-events --log-group-name /aws/lambda/report-generator --query 'events[].message' --output text | tr '\t' '\n' | grep "Aviso publicado"
```
**PowerShell**
```powershell
Start-Sleep 10
(aws logs filter-log-events --log-group-name /aws/lambda/report-generator --query 'events[].message' --output text) -split "`t" | Select-String "Aviso publicado"
```
Salida esperada:
```
… INFO  Aviso publicado (HTTP 201): {"type":"report.requested","body":{"requestId":"e2e-1791896500","requester":"guia@ejemplo.com","reportType":"PRUEBA_E2E"}}
```
*En tu app*: abre `https://TU-APP/notices`. Debe aparecer un aviso cuyo texto contiene la marca `e2e-…`. Esa es la validación completa: el mensaje que enviaste a SNS recorrió la cola y la Lambda, y terminó en tu pantalla de avisos.

*Preguntándole a la API* (opcional, sin navegador): la misma API key permite listar tus avisos. Si la marca aparece, el aviso existe en la API:

**bash**
```bash
curl -s -H "Authorization: ApiKey $API_TOKEN_INTEGRATION" "${API_NOTIFICATION%/}/api/saas/notices?mine=1&limit=20" | grep -o "$MARK"
```
**PowerShell**
```powershell
curl.exe -s -H "Authorization: ApiKey $env:API_TOKEN_INTEGRATION" "$($env:API_NOTIFICATION.TrimEnd('/'))/api/saas/notices?mine=1&limit=20" | Select-String $MARK
```
Imprime tu marca (`e2e-…`) si el aviso está. (Estos dos comandos suponen que `API_NOTIFICATION` es la URL del sitio, sin `/api`, como en el paso 1.)

Mientras tanto, `report-request-recorder` imprimió el mismo mensaje en su log (compruébalo con el comando de 7.3), porque SNS lo copió a las dos colas.

Esta secuencia se probó de punta a punta contra LocalStack 3.8.1 y una API de avisos real: el log mostró `Aviso publicado (HTTP 201)`, la API listó el aviso con la marca, y la clave no apareció en ningún log.

**Si algo falla**, el log de la Lambda dice por qué (el mensaje vuelve a la cola, se reintenta 3 veces y luego va a la DLQ, parte 10):

| El log dice | Causa | Solución |
|---|---|---|
| `Faltan API_NOTIFICATION y/o API_TOKEN_INTEGRATION` | Las variables no llegaron a la función | Repetir el paso 3 (y revisar que el paso 1 las dejó definidas) |
| `La API respondió HTTP 401` o `403` | La API key no es válida, o no tiene permiso | Generar otra en tu perfil y repetir 1 y 3 |
| `La API respondió HTTP 404`, `405` o `3xx` | La URL base no es la del sitio de la app | Probarla con el `notify:test` del proyecto, o revisar `API_NOTIFICATION` |
| `fetch failed` / `ENOTFOUND` | El contenedor de la Lambda no sale a Internet, o el dominio no existe | Revisar la red de Docker y la URL |
| No aparece nada en el log | La Lambda aún no corrió, o el trigger no existe | Esperar 10 s más; `list-event-source-mappings` (6.5) |

**6. Limpiar.** El aviso de prueba se borra desde la pantalla de avisos de tu app. La función de LocalStack se elimina junto con el contenedor (`docker rm -f localstack`), o puedes devolverla a la versión que solo imprime subiendo `demo.zip` otra vez (6.6). Cierra la terminal o limpia las variables cuando termines: `unset API_TOKEN_INTEGRATION` (bash) o `Remove-Item Env:API_TOKEN_INTEGRATION` (PowerShell).

**En AWS real** es la misma Lambda y las mismas variables; lo que cambia es el rol (parte 9.1) y que las variables se pasan en `create-function` (con el `--query`, para no imprimirlas). Para producción, guarda la API key en **AWS Secrets Manager** o **SSM Parameter Store** en vez de una variable de entorno de la función.

---

## 7. Invocar una Lambda a mano y ver sus logs

### 7.1 Con un evento falso
Una Lambda conectada a SQS recibe un evento con una lista `Records`, cada uno con el mensaje en `body`. Puedes fabricarlo a mano. El `body` es un **string** que contiene el JSON del mensaje (por eso las comillas internas van escapadas). Las Lambdas del proyecto aceptan el mensaje directo o dentro del sobre de SNS:

**bash**
```bash
cat > event.json <<'EOF'
{"Records":[{"messageId":"m1","body":"{\"type\":\"report.requested\",\"body\":{\"requestId\":\"manual-1\",\"requester\":\"yo@ejemplo.com\",\"requesterId\":\"s1\",\"reportType\":\"INVENTORY\",\"params\":{},\"requestedAt\":\"2026-01-01T00:00:00Z\"}}"}]}
EOF

aws lambda invoke --function-name report-generator \
  --cli-binary-format raw-in-base64-out --payload file://event.json \
  --log-type Tail --query LogResult --output text response.json | base64 --decode

cat response.json
```
**PowerShell**
```powershell
@'
{"Records":[{"messageId":"m1","body":"{\"type\":\"report.requested\",\"body\":{\"requestId\":\"manual-1\",\"requester\":\"yo@ejemplo.com\",\"requesterId\":\"s1\",\"reportType\":\"INVENTORY\",\"params\":{},\"requestedAt\":\"2026-01-01T00:00:00Z\"}}"}]}
'@ | Set-Content -Encoding ascii event.json

$log = aws lambda invoke --function-name report-generator `
  --cli-binary-format raw-in-base64-out --payload file://event.json `
  --log-type Tail --query LogResult --output text response.json
[Text.Encoding]::UTF8.GetString([Convert]::FromBase64String($log))

Get-Content response.json
```
- `--cli-binary-format raw-in-base64-out`: obligatorio en CLI v2 para pasar JSON "tal cual" (si no, lo interpreta como base64 y falla con *Invalid base64*).
- `--log-type Tail`: devuelve las últimas líneas del log de esa invocación, en base64; la decodificación las muestra. (En macOS antiguo, `base64 -D`.)
- `response.json`: lo que devuelve la función.

Si la función **lanza un error**, la invocación igual "funciona" (HTTP 200) pero el resultado trae `FunctionError`. Mira la respuesta completa, sin `--query` (mismo comando en ambos):
```
aws lambda invoke --function-name report-generator --cli-binary-format raw-in-base64-out --payload file://event.json response.json
```
y luego `cat response.json` (o `Get-Content response.json`), que mostrará `{"errorType":"Error","errorMessage":"…"}`. Para invocarla sin esperar el resultado: `--invocation-type Event`.

Esto prueba la Lambda **aislada**, sin colas ni SNS: ideal para depurar su lógica.

### 7.2 Con el flujo real
Publica en SNS y deja que las colas y los triggers hagan su trabajo:

**bash**
```bash
aws sns publish --topic-arn "$TOPIC_ARN" --message '{"type":"report.requested","body":{"requestId":"flujo-1","requester":"yo@ejemplo.com","requesterId":"s1","reportType":"INVENTORY","params":{},"requestedAt":"2026-01-01T00:00:00Z"}}'
```
**PowerShell**
```powershell
@'
{"type":"report.requested","body":{"requestId":"flujo-1","requester":"yo@ejemplo.com","requesterId":"s1","reportType":"INVENTORY","params":{},"requestedAt":"2026-01-01T00:00:00Z"}}
'@ | Set-Content -Encoding ascii msg-flujo.json
aws sns publish --topic-arn $TOPIC_ARN --message file://msg-flujo.json
```
Espera unos segundos y mira los logs (7.3). Con las Lambdas de ejemplo, **cada una imprime el mensaje**, porque el filtro lo deja pasar a las dos colas:
```
[report-request-recorder] mensaje recibido: {"type":"report.requested","body":{"requestId":"flujo-1",…}}
[report-request-recorder] lote procesado: 1 mensaje(s)

[report-generator] mensaje recibido: {"type":"report.requested","body":{"requestId":"flujo-1",…}}
[report-generator] lote procesado: 1 mensaje(s)
```

### 7.3 Logs
Cada Lambda escribe en el grupo `/aws/lambda/<nombre>` (mismo comando en ambos):
```
aws logs tail /aws/lambda/report-generator --since 10m
aws logs tail /aws/lambda/report-request-recorder --since 10m
```
Si `tail` no funciona en tu versión de LocalStack:
```
aws logs filter-log-events --log-group-name /aws/lambda/report-generator --query 'events[].message' --output text
```
> **No uses `--filter-pattern` en LocalStack:** no se aplica de forma fiable y devuelve todos los eventos, incluidos los de ejecuciones anteriores. Para encontrar una ejecución concreta, filtra con `grep` (o `Select-String`) por un dato único (el `requestId` del mensaje).

Con las Lambdas de ejemplo, la prueba es que el mensaje **aparezca impreso en el log** de cada una (ver 7.2). Si usas el código real (6.8), verifica además en la base que el recorder guardó la fila (necesita `psql` en el PATH; en Windows, instala el cliente de PostgreSQL, o usa pgAdmin / DBeaver):
```
psql reports_local -P pager=off -c "SELECT id, requester, type, requested_at FROM report_requests ORDER BY requested_at DESC LIMIT 3;"
```
(la tabla `report_requests` la crea el BFF al arrancar, o créala a mano con el SQL de `GUIA.md`, parte 1).

---

## 8. Conectar tu servidor a LocalStack

**El código del servidor no cambia entre local y producción.** Lo que decide a dónde habla el SDK de AWS son variables de entorno, la misma lógica que el CLI:

| Variable | En local | En AWS real |
|---|---|---|
| `AWS_ENDPOINT_URL` | `http://localhost:4566` | **no se define** |
| `AWS_ACCESS_KEY_ID` / `AWS_SECRET_ACCESS_KEY` | `test` / `test` | **no se definen**: usa el rol de la instancia EC2 |
| `AWS_REGION` | `us-east-2` | `us-east-2` |
| `SNS_TOPIC_ARN` | `arn:aws:sns:us-east-2:000000000000:report-events` | el ARN real |

Todos los SDK modernos respetan `AWS_ENDPOINT_URL` (Java v2 ≥ 2.21, JavaScript v3, Python boto3, Go v2…). El BFF de este proyecto (Spring Boot, SDK v2 2.29) crea sus clientes sin endpoint ni claves en el código, así que basta con el entorno:

**bash**
```bash
cd bff
AWS_ENDPOINT_URL=http://localhost:4566 \
AWS_ACCESS_KEY_ID=test AWS_SECRET_ACCESS_KEY=test AWS_REGION=us-east-2 \
SNS_TOPIC_ARN=arn:aws:sns:us-east-2:000000000000:report-events \
DB_URL=jdbc:postgresql://localhost:5432/reports_local DB_USER=USUARIO DB_PASSWORD= \
SPRING_SECURITY_OAUTH2_RESOURCESERVER_JWT_ISSUER_URI=http://localhost:9000 \
./gradlew bootRun
```
**PowerShell**
```powershell
$env:AWS_ENDPOINT_URL = "http://localhost:4566"
$env:AWS_ACCESS_KEY_ID = "test"
$env:AWS_SECRET_ACCESS_KEY = "test"
$env:AWS_REGION = "us-east-2"
$env:SNS_TOPIC_ARN = "arn:aws:sns:us-east-2:000000000000:report-events"
$env:DB_URL = "jdbc:postgresql://localhost:5432/reports_local"
$env:DB_USER = "USUARIO"
$env:SPRING_SECURITY_OAUTH2_RESOURCESERVER_JWT_ISSUER_URI = "http://localhost:9000"
cd bff
.\gradlew.bat bootRun
```
En Windows no existen variables de entorno **vacías**, así que `DB_PASSWORD` simplemente no se define: el BFF tiene un valor por defecto vacío (`${DB_PASSWORD:}`).

> **Ojo en PowerShell:** esas variables `$env:…` viven mientras esa ventana esté abierta. Si luego usas **la misma ventana** para hablar con AWS real, el CLI usará `test`/`test` y el endpoint local. Cierra la ventana al terminar, o límpialas:
> ```powershell
> Remove-Item Env:AWS_ENDPOINT_URL, Env:AWS_ACCESS_KEY_ID, Env:AWS_SECRET_ACCESS_KEY, Env:SNS_TOPIC_ARN, Env:DB_URL, Env:DB_USER, Env:SPRING_SECURITY_OAUTH2_RESOURCESERVER_JWT_ISSUER_URI
> ```

La variable `…ISSUER_URI` apunta la validación del JWT a un OIDC local en lugar de Cognito (`node scripts/local/mock-oidc.js`, o cualquier servidor OIDC); es la única parte de este paso que usa un script del proyecto. Si prefieres un token real de Cognito, no definas esa variable y envía un `id_token` válido en `Authorization: Bearer`.

**Probar que el servidor publica:** llama a su endpoint, y luego mira las colas o los logs de las Lambdas:

**bash**
```bash
curl -i -X POST http://localhost:8080/api/reports \
  -H "Authorization: Bearer $TOKEN" -H "X-Secret-Gateway: secreto_local" -H "Content-Type: application/json" \
  -d '{"type":"INVENTORY","params":{}}'
```
**PowerShell**
```powershell
'{"type":"INVENTORY","params":{}}' | Set-Content -Encoding ascii report-body.json
curl.exe -i -X POST http://localhost:8080/api/reports `
  -H "Authorization: Bearer $TOKEN" -H "X-Secret-Gateway: secreto_local" -H "Content-Type: application/json" `
  --data "@report-body.json"
```
Debe responder `202 Accepted` con un `requestId`.

**Qué NO debe llegar nunca a producción:** `AWS_ENDPOINT_URL`, las claves `test`, `NOTIFY_MODE=log`, `DATABASE_SSL=false` y el `…ISSUER_URI` local. En el despliegue real, el contenedor del BFF se inicia sin ellas (mira `deploy.yml`: solo define las de producción).

### 8.1 Ejemplo mínimo: una API Spring Boot de prueba que publica en SNS (sin autenticación)
Una API de unas 25 líneas, **sin Spring Security, sin Cognito y sin nada**, con un solo endpoint: lo que recibes en el cuerpo se **publica en el topic de SNS** `report-events`. Con eso se recorre el flujo completo del proyecto: **API → SNS → (filtro) → las dos colas → las dos Lambdas**. Es, simplificado, lo que hace el BFF real con `POST /api/reports`.

> **Solo para pruebas.** Un endpoint abierto que publica en un topic es un agujero de seguridad. Esta API escucha únicamente en `127.0.0.1` (tu máquina) y usa el puerto 8081 para no chocar con el BFF (8080). **No la despliegues así.**

**Requisitos:** Java 21, LocalStack arriba con tu perfil activo (partes 2 y 3), y el flujo ya construido en las partes **4** (topic, colas y suscripciones), **6** (las dos Lambdas con sus triggers) y, si quieres ver el aviso en tu app, **6.9**. Todo en la **misma terminal**, porque se usa la variable `TOPIC_ARN` de la parte 4.1.

**1. Generar el proyecto con Spring Initializr** (la herramienta oficial de Spring para crear proyectos; aquí por línea de comandos, con la dependencia web):

**bash**
```bash
curl -s https://start.spring.io/starter.zip \
  -d type=gradle-project -d language=java -d javaVersion=21 \
  -d groupId=com.example -d artifactId=sns-test-api -d name=sns-test-api \
  -d packageName=com.example.snstest -d dependencies=web \
  -o sns-test-api.zip
mkdir sns-test-api && unzip -q sns-test-api.zip -d sns-test-api
cd sns-test-api
```
**PowerShell**
```powershell
curl.exe -s https://start.spring.io/starter.zip `
  -d type=gradle-project -d language=java -d javaVersion=21 `
  -d groupId=com.example -d artifactId=sns-test-api -d name=sns-test-api `
  -d packageName=com.example.snstest -d dependencies=web `
  -o sns-test-api.zip
mkdir sns-test-api; tar -xf sns-test-api.zip -C sns-test-api
cd sns-test-api
```
Si prefieres una interfaz gráfica, https://start.spring.io ofrece lo mismo: proyecto Gradle (Groovy), Java 21, dependencia "Spring Web".

> **Si Gradle falla al compilar (típico en Windows y en redes corporativas)**, hay dos causas habituales:
>
> **1. Tu JDK no es el 21.** El proyecto generado le pide a Gradle un JDK concreto (*toolchain*): si no lo encuentra, intenta descargarlo, y sin red falla con `No matching toolchains found for requested specification`. Mira cuál tienes con `java --version` y genera el proyecto con esa versión (`-d javaVersion=26`, por ejemplo), o edita en `build.gradle`:
> ```groovy
> java {
>     toolchain {
>         languageVersion = JavaLanguageVersion.of(26)   // el número de tu JDK (Spring Boot pide Java 17 o superior)
>     }
> }
> ```
>
> **2. Tu red no alcanza Maven Central** (`repo.maven.apache.org`) y Gradle falla con `Could not resolve …` o `Could not GET …`. Se soluciona usando un **espejo de Maven Central** en lugar del repositorio por defecto. Esta configuración, con el espejo que aloja Google, se usó en Windows con este mismo proyecto. En `build.gradle`, reemplaza el bloque `repositories` (que trae `mavenCentral()`):
> ```groovy
> repositories {
>     maven {
>         url = uri("https://maven-central.storage-download.googleapis.com/maven2/")
>     }
> }
> ```
> y en `settings.gradle` agrega, **antes** de la línea `rootProject.name`, los dos bloques siguientes. El primero hace lo mismo para los *plugins* de Gradle (el de Spring Boot se descarga por ahí; se conserva `gradlePluginPortal()` como respaldo), y el segundo evita que Gradle ignore el espejo:
> ```groovy
> pluginManagement {
>     repositories {
>         maven {
>             url = uri("https://maven-central.storage-download.googleapis.com/maven2/")
>         }
>         gradlePluginPortal()
>     }
> }
>
> dependencyResolutionManagement {
>     repositoriesMode.set(RepositoriesMode.PREFER_PROJECT)
>     repositories {
>         maven {
>             url = uri("https://maven-central.storage-download.googleapis.com/maven2/")
>         }
>     }
> }
>
> rootProject.name = 'sns-test-api'
> ```
> Si tu red también bloquea el *Gradle Plugin Portal* (`plugins.gradle.org`), pide a tu equipo de TI que lo permita o que provea un espejo propio. El resto del proyecto (dependencias, código) no cambia: el `build.gradle` final queda con `spring-boot-starter-webmvc`, el BOM del SDK de AWS y `software.amazon.awssdk:sns`, tal como en los pasos 1 y 2.

**2. Agregar el SDK de AWS para SNS.** El proyecto generado no lo trae. Se agrega al final de `build.gradle`; el *BOM* (`platform`) fija las versiones de todos los módulos del SDK:

**bash**
```bash
cat >> build.gradle <<'EOF'

dependencies {
    implementation platform('software.amazon.awssdk:bom:2.29.52')
    implementation 'software.amazon.awssdk:sns'
}
EOF
```
**PowerShell**
```powershell
@'

dependencies {
    implementation platform('software.amazon.awssdk:bom:2.29.52')
    implementation 'software.amazon.awssdk:sns'
}
'@ | Add-Content -Encoding ascii build.gradle
```

**3. El código: un cliente de SNS y un endpoint.**

**bash**
```bash
cat > src/main/java/com/example/snstest/SnsConfig.java <<'EOF'
package com.example.snstest;

import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import software.amazon.awssdk.services.sns.SnsClient;

@Configuration
public class SnsConfig {

    // Sin región, claves ni endpoint en el código: todo sale del entorno (AWS_REGION, credenciales y
    // AWS_ENDPOINT_URL). Por eso el mismo código sirve para LocalStack y para AWS real.
    @Bean
    public SnsClient snsClient() {
        return SnsClient.create();
    }
}
EOF

cat > src/main/java/com/example/snstest/SnsTestController.java <<'EOF'
package com.example.snstest;

import java.util.Map;
import org.springframework.beans.factory.annotation.Value;
import org.springframework.http.ResponseEntity;
import org.springframework.web.bind.annotation.PostMapping;
import org.springframework.web.bind.annotation.RequestBody;
import org.springframework.web.bind.annotation.RequestMapping;
import org.springframework.web.bind.annotation.RestController;
import software.amazon.awssdk.services.sns.SnsClient;
import software.amazon.awssdk.services.sns.model.PublishRequest;

@RestController
@RequestMapping("/api/test")
public class SnsTestController {

    private final SnsClient sns;
    private final String topicArn;

    public SnsTestController(SnsClient sns, @Value("${app.sns.topic-arn}") String topicArn) {
        this.sns = sns;
        this.topicArn = topicArn;
    }

    // Lo que llegue en el cuerpo se publica tal cual en el topic de SNS.
    @PostMapping("/sns")
    public ResponseEntity<Map<String, String>> publish(@RequestBody String body) {
        String messageId = sns.publish(PublishRequest.builder()
                .topicArn(topicArn)
                .message(body)
                .build()).messageId();
        return ResponseEntity.accepted().body(Map.of("messageId", messageId, "topicArn", topicArn));
    }
}
EOF

cat > src/main/resources/application.properties <<'EOF'
spring.application.name=sns-test-api
server.port=8081
server.address=127.0.0.1
app.sns.topic-arn=${SNS_TOPIC_ARN}
EOF
```
**PowerShell**
```powershell
@'
package com.example.snstest;

import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import software.amazon.awssdk.services.sns.SnsClient;

@Configuration
public class SnsConfig {

    // Sin región, claves ni endpoint en el código: todo sale del entorno (AWS_REGION, credenciales y
    // AWS_ENDPOINT_URL). Por eso el mismo código sirve para LocalStack y para AWS real.
    @Bean
    public SnsClient snsClient() {
        return SnsClient.create();
    }
}
'@ | Set-Content -Encoding ascii src/main/java/com/example/snstest/SnsConfig.java

@'
package com.example.snstest;

import java.util.Map;
import org.springframework.beans.factory.annotation.Value;
import org.springframework.http.ResponseEntity;
import org.springframework.web.bind.annotation.PostMapping;
import org.springframework.web.bind.annotation.RequestBody;
import org.springframework.web.bind.annotation.RequestMapping;
import org.springframework.web.bind.annotation.RestController;
import software.amazon.awssdk.services.sns.SnsClient;
import software.amazon.awssdk.services.sns.model.PublishRequest;

@RestController
@RequestMapping("/api/test")
public class SnsTestController {

    private final SnsClient sns;
    private final String topicArn;

    public SnsTestController(SnsClient sns, @Value("${app.sns.topic-arn}") String topicArn) {
        this.sns = sns;
        this.topicArn = topicArn;
    }

    // Lo que llegue en el cuerpo se publica tal cual en el topic de SNS.
    @PostMapping("/sns")
    public ResponseEntity<Map<String, String>> publish(@RequestBody String body) {
        String messageId = sns.publish(PublishRequest.builder()
                .topicArn(topicArn)
                .message(body)
                .build()).messageId();
        return ResponseEntity.accepted().body(Map.of("messageId", messageId, "topicArn", topicArn));
    }
}
'@ | Set-Content -Encoding ascii src/main/java/com/example/snstest/SnsTestController.java

@'
spring.application.name=sns-test-api
server.port=8081
server.address=127.0.0.1
app.sns.topic-arn=${SNS_TOPIC_ARN}
'@ | Set-Content -Encoding ascii src/main/resources/application.properties
```
- **`SnsClient.create()`** no recibe nada: el SDK busca por sí solo la región (`AWS_REGION`), las credenciales y, desde la versión 2.21, el endpoint (`AWS_ENDPOINT_URL`). Es lo que permite cambiar de LocalStack a AWS real sin tocar el código.
- **`@RequestBody String`** recibe el cuerpo crudo, sin interpretarlo como JSON: así la API no necesita ninguna librería de JSON y publica exactamente lo que le mandes.
- **`publish` sin atributos de mensaje:** el filtro de las suscripciones mira el campo `type` **dentro** del JSON del cuerpo (parte 4.2), no atributos.
- **`server.address=127.0.0.1`** hace que solo tu máquina pueda llamarla.

**4. Arrancarla**, apuntando a LocalStack y al topic. Hazlo **en la misma terminal de la parte 4.1**, donde existe `TOPIC_ARN`. Es un proceso que no termina: déjalo corriendo y abre otra terminal para los pasos siguientes (la primera vez Gradle descarga sus dependencias y puede tardar):

**bash**
```bash
[[ "$TOPIC_ARN" == arn:aws:sns:* ]] || { echo "TOPIC_ARN no es un ARN válido (repite la parte 4.1)"; false; } && \
AWS_ENDPOINT_URL=http://localhost:4566 AWS_ACCESS_KEY_ID=test AWS_SECRET_ACCESS_KEY=test \
AWS_REGION=us-east-2 SNS_TOPIC_ARN="$TOPIC_ARN" \
./gradlew bootRun
```
**PowerShell**
```powershell
if ($TOPIC_ARN -notlike "arn:aws:sns:*") { throw "TOPIC_ARN no es un ARN válido (repite la parte 4.1)" }
$env:AWS_ENDPOINT_URL = "http://localhost:4566"
$env:AWS_ACCESS_KEY_ID = "test"
$env:AWS_SECRET_ACCESS_KEY = "test"
$env:AWS_REGION = "us-east-2"
$env:SNS_TOPIC_ARN = $TOPIC_ARN
.\gradlew.bat bootRun
```
Cuando veas `Started SnsTestApiApplication in … seconds`, está lista. Para los pasos siguientes, la otra terminal necesita el perfil `localstack` activo (parte 3.1).

**La primera línea del bloque comprueba que `TOPIC_ARN` sea un ARN válido** antes de arrancar. Es una protección contra un error engañoso: si esa variable no existe en tu terminal, la API arrancaría igual, pero cada llamada fallaría con un `500` y este mensaje en su log:
```
InvalidParameterException: Invalid parameter: TopicArn Reason: An ARN must have at least 6 elements, not 1
```
("un ARN tiene al menos 6 elementos, y esto tiene 1": un ARN real son seis partes separadas por `:`, y un valor vacío no tiene ninguna). Si ves `TOPIC_ARN no es un ARN válido`, la variable está vacía o el topic no existe. Para recuperarla:

**bash**
```bash
TOPIC_ARN=$(aws sns list-topics --query "Topics[?contains(TopicArn,'report-events')].TopicArn | [0]" --output text)
echo "$TOPIC_ARN"
```
**PowerShell**
```powershell
$TOPIC_ARN = aws sns list-topics --query "Topics[?contains(TopicArn,'report-events')].TopicArn | [0]" --output text
$TOPIC_ARN
```
Si imprime `None`, el topic no existe: repite las partes 4 y 6 (LocalStack no guarda nada entre reinicios).

**5. Publicar un mensaje por la API.** En otra terminal. Es el mensaje con la forma que esperan las Lambdas del proyecto (`type: report.requested`), con una marca única para reconocerlo:

**bash**
```bash
MARK="api-$(date +%s)"
curl -i -X POST http://localhost:8081/api/test/sns \
  -H "Content-Type: application/json" \
  -d "{\"type\":\"report.requested\",\"body\":{\"requestId\":\"$MARK\",\"requester\":\"yo@ejemplo.com\",\"reportType\":\"INVENTORY\"}}"
```
**PowerShell**
```powershell
$MARK = "api-" + [DateTimeOffset]::UtcNow.ToUnixTimeSeconds()
@{ type = "report.requested"; body = @{ requestId = $MARK; requester = "yo@ejemplo.com"; reportType = "INVENTORY" } } | ConvertTo-Json -Compress | Set-Content -Encoding ascii msg-api.json
curl.exe -i -X POST http://localhost:8081/api/test/sns `
  -H "Content-Type: application/json" `
  --data "@msg-api.json"
```
La API responde `202 Accepted` con `{"messageId":"…","topicArn":"arn:aws:sns:us-east-2:000000000000:report-events"}`. Ese `messageId` es el que SNS asignó al mensaje.

**6. Comprobar que el mensaje recorrió todo el flujo.** SNS lo copió a las dos colas, y los triggers invocaron a las dos Lambdas. Busca tu marca en el log de cada una:

**bash**
```bash
for FN in report-request-recorder report-generator; do
  echo "--- $FN:"
  for i in $(seq 1 12); do
    aws logs filter-log-events --log-group-name /aws/lambda/$FN --query 'events[].message' --output text 2>/dev/null | tr '\t' '\n' | grep "$MARK" && break
    sleep 5
  done
done
```
**PowerShell**
```powershell
foreach ($fn in "report-request-recorder", "report-generator") {
  "--- $fn`:"
  for ($i = 0; $i -lt 12; $i++) {
    $hit = (aws logs filter-log-events --log-group-name /aws/lambda/$fn --query 'events[].message' --output text 2>$null) -split "`t" | Select-String $MARK
    if ($hit) { $hit; break }
    Start-Sleep 5
  }
}
```
**Por qué se reintenta en vez de esperar un tiempo fijo:** el grupo de logs de una Lambda **no existe hasta su primera ejecución**, y esa primera ejecución en un LocalStack recién creado es un *arranque en frío* (LocalStack levanta el contenedor de la función) que puede tardar 10 a 30 segundos. Mientras tanto, `filter-log-events` falla con `The specified log group does not exist`. El bucle prueba cada 5 s, hasta 60 s, y se detiene al encontrar la marca; las siguientes ejecuciones son casi instantáneas.

Cada Lambda debe imprimir una línea con tu marca (`[report-request-recorder] mensaje recibido: {"type":"report.requested","body":{"requestId":"api-…"…`). Si hiciste la 6.9, `report-generator` además publica el mensaje como aviso en tu app: míralo en `https://TU-APP/notices`, o con el comando opcional de 6.9, paso 5.

**7. Ver el filtro en acción.** Publica un mensaje cuyo `type` **no** cumple el filtro de las suscripciones; SNS lo descarta y no llega a ninguna cola ni Lambda:

**bash**
```bash
curl -i -X POST http://localhost:8081/api/test/sns \
  -H "Content-Type: application/json" \
  -d '{"type":"otro","body":{"nota":"este no debe llegar"}}'
sleep 8
aws logs filter-log-events --log-group-name /aws/lambda/report-generator --query 'events[].message' --output text | tr '\t' '\n' | grep -c "este no debe llegar"
```
**PowerShell**
```powershell
'{"type":"otro","body":{"nota":"este no debe llegar"}}' | Set-Content -Encoding ascii msg-otro-api.json
curl.exe -i -X POST http://localhost:8081/api/test/sns -H "Content-Type: application/json" --data "@msg-otro-api.json"
Start-Sleep 8
((aws logs filter-log-events --log-group-name /aws/lambda/report-generator --query 'events[].message' --output text) -split "`t" | Select-String "este no debe llegar" | Measure-Object).Count
```
La API responde igual `202` (publicar en SNS siempre funciona), pero el conteo final es `0`: ninguna Lambda recibió el mensaje. (Ese `0` solo significa algo porque en el paso 6 las dos Lambdas ya corrieron y sus logs existen: con un grupo de logs inexistente, el conteo también daría `0`, pero acompañado de un error.) Esa es la diferencia con enviar directo a una cola: SNS decide a quién le llega cada mensaje.

**8. Limpiar.** Detén la API con Ctrl+C y borra el proyecto:

**bash**
```bash
cd .. && rm -rf sns-test-api sns-test-api.zip
```
**PowerShell**
```powershell
cd ..; Remove-Item -Recurse -Force sns-test-api, sns-test-api.zip
```

**Para AWS real, esta API cambiaría así** (el código no):
- **Se quitan** `AWS_ENDPOINT_URL` y las claves `test`: el servidor usa el rol de la instancia EC2.
- **`SNS_TOPIC_ARN`** pasa a ser el ARN real del topic (`aws sns list-topics`, o el valor que devolvió `create-topic`).
- **El rol necesita permiso** `sns:Publish` sobre ese topic (parte 9.3): es exactamente el permiso que ya usa el BFF del proyecto.
- **El endpoint no puede quedar abierto:** en un servidor real iría detrás de autenticación (Cognito en el BFF del proyecto), y el mensaje no lo armaría quien llama, sino el servidor a partir del JWT: así el BFF decide quién es el `requester`.

---

## 9. Pasar a AWS real
Con la guía hecha en local, pasar a real consiste en **repetir los mismos comandos con otro perfil** y cambiar lo que se lista aquí.

**Antes de empezar, confirma dónde estás:**

**bash**
```bash
unset AWS_PROFILE              # o: export AWS_PROFILE=<tu-perfil-real>
aws sts get-caller-identity    # debe mostrar TU cuenta real, no 000000000000
aws configure list
```
**PowerShell**
```powershell
Remove-Item Env:AWS_PROFILE    # o: $env:AWS_PROFILE = "<tu-perfil-real>"
aws sts get-caller-identity    # debe mostrar TU cuenta real, no 000000000000
aws configure list
```

| Qué | LocalStack | AWS real |
|---|---|---|
| Perfil del CLI | `localstack` (claves `test`, con endpoint) | tu perfil real, **sin** `endpoint_url` |
| Cuenta en los ARN | `000000000000` | tu número de cuenta (`$ACCOUNT_ID` lo recoge solo) |
| Rol de la Lambda | un ARN ficticio basta | **debe existir**, con permisos (abajo) |
| Política de la cola para SNS | opcional | **obligatoria** (ya está en el paso 4) |
| Runtime | `nodejs20.x` | `nodejs22.x` |
| `DATABASE_URL` de las Lambdas | `…@host.docker.internal:5432/…` | la de Supabase |
| `DATABASE_SSL` | `false` | no se define (SSL activo) |
| `NOTIFY_MODE` en el generador | `log` | no se define; en su lugar, `API_NOTIFICATION` y `API_TOKEN_INTEGRATION` |
| Credenciales del servidor | claves `test` | el rol de la instancia EC2 |

### 9.1 Los roles de las Lambdas (solo en AWS real)
Un rol por Lambda, con lo mínimo: escribir logs y leer **su** cola.

**bash**
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
sleep 12      # el rol tarda unos segundos en propagarse; sin esperar, create-function puede fallar
```
**PowerShell**
```powershell
@'
{"Version":"2012-10-17","Statement":[{"Effect":"Allow","Principal":{"Service":"lambda.amazonaws.com"},"Action":"sts:AssumeRole"}]}
'@ | Set-Content -Encoding ascii trust.json

$pares = @(
  @{ fn = "report-request-recorder"; q = "report-requests" },
  @{ fn = "report-generator";        q = "report-generation" }
)
foreach ($p in $pares) {
  $rol = "$($p.fn)-role"
  aws iam create-role --role-name $rol --assume-role-policy-document file://trust.json
  aws iam attach-role-policy --role-name $rol --policy-arn arn:aws:iam::aws:policy/service-role/AWSLambdaBasicExecutionRole
  @"
{"Version":"2012-10-17","Statement":[{"Effect":"Allow","Action":["sqs:ReceiveMessage","sqs:DeleteMessage","sqs:GetQueueAttributes"],"Resource":"arn:aws:sqs:${AWS_REGION}:${ACCOUNT_ID}:$($p.q)"}]}
"@ | Set-Content -Encoding ascii "policy-$($p.q).json"
  aws iam put-role-policy --role-name $rol --policy-name "consume-$($p.q)" --policy-document "file://policy-$($p.q).json"
}
Start-Sleep 12    # el rol tarda unos segundos en propagarse; sin esperar, create-function puede fallar
```
Y en la parte 6.4, usa `--role "arn:aws:iam::${ACCOUNT_ID}:role/report-request-recorder-role"` (y `…/report-generator-role` para el otro), con `--runtime nodejs22.x`. Si usas el código real, las variables de producción son las de la parte 6.8 con los valores reales (9.2).

### 9.2 Variables del generador en real
**bash**
```bash
cat > env-generator.json <<'EOF'
{"Variables":{"API_NOTIFICATION":"https://TU-APP/","API_TOKEN_INTEGRATION":"crr_...","DATABASE_URL":"postgresql://USUARIO:CLAVE@HOST:5432/postgres"}}
EOF
```
**PowerShell**
```powershell
@'
{"Variables":{"API_NOTIFICATION":"https://TU-APP/","API_TOKEN_INTEGRATION":"crr_...","DATABASE_URL":"postgresql://USUARIO:CLAVE@HOST:5432/postgres"}}
'@ | Set-Content -Encoding ascii env-generator.json
```
Bórralo al terminar (`rm env-generator.json`, o `Remove-Item env-generator.json`): tiene secretos. Y ojo: `create-function` y `update-function-configuration` **imprimen las variables de entorno** (con el token y la contraseña) en su respuesta; agrega `--query FunctionArn --output text` (o `--query LastUpdateStatus --output text`) para que no se muestren.

### 9.3 El servidor en AWS real
- El servidor corre en una EC2 con un **rol de instancia**. Ese rol necesita permiso para publicar en el topic:

**bash**
```bash
aws iam put-role-policy --role-name <ROL_DE_LA_EC2> --policy-name publish-report-events --policy-document \
  "{\"Version\":\"2012-10-17\",\"Statement\":[{\"Effect\":\"Allow\",\"Action\":\"sns:Publish\",\"Resource\":\"$TOPIC_ARN\"}]}"
```
**PowerShell**
```powershell
@"
{"Version":"2012-10-17","Statement":[{"Effect":"Allow","Action":"sns:Publish","Resource":"$TOPIC_ARN"}]}
"@ | Set-Content -Encoding ascii publish-policy.json
aws iam put-role-policy --role-name <ROL_DE_LA_EC2> --policy-name publish-report-events --policy-document file://publish-policy.json
```
- Si corre dentro de un contenedor Docker, el servicio de metadatos de la instancia debe aceptar 2 saltos de red, o el contenedor no obtiene las credenciales del rol (mismo comando en ambos):
```
aws ec2 describe-instances --instance-ids <INSTANCE_ID> --query 'Reservations[].Instances[].MetadataOptions.HttpPutResponseHopLimit'
aws ec2 modify-instance-metadata-options --instance-id <INSTANCE_ID> --http-put-response-hop-limit 2    # solo si no es 2 o más
```

### 9.4 Después: GitHub Actions mantiene el código
La primera vez creas todo a mano (esta guía). Después, los workflows de cada repo actualizan el **código** con `update-function-code`; no recrean nada. Por eso los nombres de las funciones deben coincidir con los secrets `LAMBDA_RECORDER_FUNCTION_NAME` y `LAMBDA_GENERATOR_FUNCTION_NAME` (ver `GUIA.md`, parte 5).

### 9.5 Prueba de humo en real
Es el mismo `publish` de la parte 7.2, con tu perfil real. Ojo: en real el generador **publica un aviso de verdad** en la API de avisos.
```
aws sns publish --topic-arn <TOPIC_ARN> --message file://msg-flujo.json
aws logs tail /aws/lambda/report-generator --since 5m
```

---

## 10. Inspeccionar, depurar y limpiar

**Ver qué existe** (mismo comando en bash y PowerShell, salvo la variable):
```
aws sns list-topics
aws sqs list-queues
aws lambda list-functions --query 'Functions[].[FunctionName,Runtime,State]' --output table
aws sns list-subscriptions-by-topic --topic-arn <TOPIC_ARN> --query 'Subscriptions[].[Endpoint,SubscriptionArn]' --output table
```
(`<TOPIC_ARN>`: en bash `"$TOPIC_ARN"`; en PowerShell `$TOPIC_ARN`.)

**Cuántos mensajes hay en una cola** (la cifra es aproximada):
```
aws sqs get-queue-attributes --queue-url <URL_DE_LA_COLA> --attribute-names ApproximateNumberOfMessages ApproximateNumberOfMessagesNotVisible
```
`NotVisible` son los que una Lambda está procesando ahora.

**Mirar la DLQ** (mensajes que fallaron 3 veces) y entender por qué:

**bash**
```bash
DLQ_URL=$(aws sqs get-queue-url --queue-name report-requests-dlq --query QueueUrl --output text)
aws sqs receive-message --queue-url "$DLQ_URL" --wait-time-seconds 3
```
**PowerShell**
```powershell
$DLQ_URL = aws sqs get-queue-url --queue-name report-requests-dlq --query QueueUrl --output text
aws sqs receive-message --queue-url $DLQ_URL --wait-time-seconds 3
```
Combina el contenido con el log de la Lambda para ver el error.

**Reenviar la DLQ a su cola** (cuando ya arreglaste el problema), solo en AWS real:
```
aws sqs start-message-move-task --source-arn <ARN_DE_LA_DLQ> --destination-arn <ARN_DE_LA_COLA>
```
O purga lo que no necesites: `aws sqs purge-queue --queue-url <URL_DE_LA_DLQ>`.

**Ver los logs de LocalStack** (cuando algo no arranca): `docker logs --tail 100 localstack`

**Limpiar en LocalStack** (equivale a empezar de cero, porque no guarda estado): `docker rm -f localstack`

**Limpiar un recurso concreto** (orden inverso a la creación):
```
aws lambda delete-event-source-mapping --uuid <UUID>
aws lambda delete-function --function-name report-generator
aws sns unsubscribe --subscription-arn <SUBSCRIPTION_ARN>
aws sqs delete-queue --queue-url <URL_DE_LA_COLA>
aws sns delete-topic --topic-arn <TOPIC_ARN>
```

---

## 11. Errores frecuentes

| Mensaje o síntoma | Causa | Solución |
|---|---|---|
| `Could not connect to the endpoint URL: "http://localhost:4566/"` | LocalStack no está corriendo | `docker ps` y repetir la parte 2 |
| Los comandos no encuentran tus recursos reales, o ves `000000000000` sin querer | Perfil o endpoint equivocado | `aws sts get-caller-identity`; revisa `AWS_PROFILE` y `AWS_ENDPOINT_URL` |
| `The security token included in the request is invalid` en AWS real | Estás usando las claves `test` contra AWS real | Quita `AWS_PROFILE` o selecciona tu perfil real; en PowerShell, revisa que no queden `$env:AWS_ACCESS_KEY_ID` de la parte 8 |
| `Invalid base64` al usar `invoke --payload` | Falta `--cli-binary-format raw-in-base64-out` | Agregarlo (CLI v2) |
| `ResourceConflictException` al crear un mapeo | Quedó un mapeo anterior | Borrarlo (parte 6.7) |
| `InvalidParameterValueException: The role defined for the function cannot be assumed` (AWS real) | El rol no existe o aún no se propaga | Crear el rol (9.1) y esperar 10 segundos |
| El mensaje no llega a una cola | El filtro no coincide, o falta la política de la cola | `receive-message` de la parte 5; revisa el `type` y el scope `MessageBody` |
| El aviso no aparece en `/notices` y el log dice `La API respondió HTTP …` (6.9) | La API rechazó la publicación | Ver la tabla de 6.9, paso 5 |
| (código real, 6.8) La Lambda falla con `ECONNREFUSED` al conectarse a Postgres (local) | Usa `localhost`, que dentro del contenedor es él mismo | `host.docker.internal` en `DATABASE_URL` |
| (código real, 6.8) La Lambda falla con `no pg_hba.conf entry` o por SSL (local) | Postgres local no usa SSL | `DATABASE_SSL=false` |
| (código real, 6.8) `relation "report_requests" does not exist` | La tabla aún no existe | Crearla con el SQL de `GUIA.md` (parte 1), o arrancar el BFF |
| Gradle (8.1) falla con `Could not resolve …` / `Could not GET …` (red corporativa, Windows) | La red no alcanza Maven Central (o el Gradle Plugin Portal) | Usar el espejo de Maven Central (nota del paso 1 de 8.1) |
| Gradle (8.1) falla con `No matching toolchains found for requested specification` | El JDK instalado no es el que pide `build.gradle` | Alinear `languageVersion` con `java --version` (nota del paso 1 de 8.1) |
| La API Spring Boot (8.1) responde `500` y su log dice `Invalid parameter: TopicArn Reason: An ARN must have at least 6 elements, not 1` | `SNS_TOPIC_ARN` llegó **vacía**: la variable `TOPIC_ARN` no existía en la terminal donde arrancaste la API, o el topic no existe (LocalStack se reinició) | Recuperar la variable con `aws sns list-topics …` (8.1, paso 4), o repetir las partes 4 y 6 si no existe; reiniciar la API en esa misma terminal |
| La API (8.1) responde `202` pero ninguna Lambda imprime el mensaje | El `type` del mensaje no es `report.requested` (el filtro lo descartó), o faltan los triggers | Revisar el cuerpo enviado (paso 5) y `list-event-source-mappings` (6.5) |
| El mensaje reaparece en la cola una y otra vez | La Lambda lanza un error | `aws logs tail` y corregir; tras 3 intentos va a la DLQ |
| `sqs receive-message` devuelve vacío tras publicar | Aún no llegó, o ya lo consumió una Lambda | Prueba antes de crear los mapeos (parte 5) |
| **Windows (Git Bash):** `--log-group-name` falla con una ruta `C:/Program Files/Git/…` | Git Bash convierte argumentos que empiezan con `/` | `export MSYS_NO_PATHCONV=1` |
| **Windows (PowerShell):** `Invalid JSON` o `Expecting property name enclosed in double quotes` | PowerShell 5.1 quita las comillas dobles del JSON pasado en el comando | Escribir el JSON en un archivo y usar `file://` |
| **Windows (PowerShell):** un ARN sale incompleto, p. ej. `arn:aws:iam::` sin la cuenta | `$ACCOUNT_ID:role` se lee como una unidad de disco | Usar `${ACCOUNT_ID}` con llaves |
| **Windows (PowerShell):** `docker run` rechaza `-e SERVICES=sns,sqs,…` | La coma separa argumentos | Poner el valor entre comillas: `-e "SERVICES=sns,sqs,…"` |
| **Windows:** la Lambda falla con `Cannot find module 'pg'` | El zip se hizo con `Compress-Archive` (rutas con `\`) | Reempaquetar con `tar -a -c -f ../x.zip *` |
| **Windows (PowerShell 5.1):** `El token '&&' no es un separador de instrucciones válido` | `&&` solo existe en PowerShell 7 | Separar con `;` |
| **Windows (PowerShell):** `curl` imprime algo raro o pide parámetros | `curl` es un alias de `Invoke-WebRequest` | Usar `curl.exe` |
| **Windows (WSL):** el CLI no encuentra LocalStack | Docker Desktop sin la integración con WSL | Docker Desktop → Settings → Resources → WSL integration |

---

## 12. Resumen: local frente a real

```
LOCAL                                   AWS REAL
─────                                   ────────
AWS_PROFILE=localstack                  sin AWS_PROFILE (o tu perfil real)
 └ endpoint http://localhost:4566        └ sin endpoint_url
 └ claves test / test                    └ tus credenciales
cuenta 000000000000                     tu cuenta
rol de Lambda: ARN ficticio             rol real con permisos (9.1)
política de cola: opcional              política de cola: obligatoria
nodejs20.x                              nodejs22.x
DB: host.docker.internal, sin SSL       DB: Supabase, con SSL
NOTIFY_MODE=log                         API_NOTIFICATION + API_TOKEN_INTEGRATION
servidor: AWS_ENDPOINT_URL + claves     servidor: rol de la instancia, sin esas variables
```

**La regla de oro:** los comandos y el código son idénticos; lo que cambia es **a quién le hablas** (decidido por el perfil o por variables de entorno) y **con qué identidad** (claves de mentira o rol real). Antes de cualquier comando que cree o borre, ejecuta `aws sts get-caller-identity`.
