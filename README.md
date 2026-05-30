# ContactosCRUD — AWS Lambda + DynamoDB

> **Actividad Práctica — 30% de la Nota Final**  
> Desarrollo de Aplicación CRUD con AWS (Capa Gratuita) y Amazon DynamoDB

| Integrante | Rol |
|---|---|
| RIOS GONZALEZ JUAN MIGUEL | Desarrollo Backend (Lambda) |
| NIETO CARDEÑO BREYNEER | Infraestructura AWS (IAM / API Gateway) |
| GARCIA ILIAS ALEXA TATIANA | Frontend & Documentación |

---

## Arquitectura

```
┌─────────────────────────────────────────────────────────────┐
│                        CLIENTE (Browser)                     │
│               index.html — S3 Static Hosting                 │
└───────────────────────────┬─────────────────────────────────┘
                            │ HTTPS
                            ▼
┌─────────────────────────────────────────────────────────────┐
│              Amazon API Gateway (REST API)                    │
│   /contactos          →  GET (listar), POST (crear)          │
│   /contactos/{id}     →  GET, PUT (actualizar), DELETE        │
└───────────────────────────┬─────────────────────────────────┘
                            │ Invoca
                            ▼
┌─────────────────────────────────────────────────────────────┐
│            AWS Lambda  (Node.js 18.x · 128 MB)               │
│   • Enrutamiento por método HTTP                             │
│   • Validación de entradas                                   │
│   • Manejo de errores con mensajes claros                    │
│   • @aws-sdk/client-dynamodb v3                              │
└───────────────────────────┬─────────────────────────────────┘
                            │ SDK v3
                            ▼
┌─────────────────────────────────────────────────────────────┐
│         Amazon DynamoDB  (On-Demand · capa gratuita)         │
│   Tabla: Contactos                                           │
│   Partition Key: contactoId (String / UUID)                  │
│   GSI: email-index  → búsqueda por email O(1)                │
└─────────────────────────────────────────────────────────────┘

Servicios AWS utilizados (todos en capa gratuita):
  • Lambda         → 1 M req/mes + 400 000 GB-s gratis
  • API Gateway    → 1 M llamadas REST/mes gratis (primer año)
  • DynamoDB       → 25 GB + 25 WCU + 25 RCU gratis (siempre)
  • S3             → 5 GB / 20 000 GET / 2 000 PUT gratis
  • IAM            → Siempre gratuito
  • CloudFormation → Siempre gratuito
```

---

## Operaciones CRUD implementadas

| Operación | Método HTTP | Endpoint | Descripción |
|---|---|---|---|
| **Create** | `POST` | `/contactos` | Crea un contacto con validación completa |
| **Read** (lista) | `GET` | `/contactos?filtro=x` | Lista todos, con filtro y orden por fecha |
| **Read** (uno) | `GET` | `/contactos/{id}` | Obtiene un contacto por su UUID |
| **Update** | `PUT` | `/contactos/{id}` | Actualiza campos con ConditionExpression |
| **Delete** | `DELETE` | `/contactos/{id}` | Elimina con verificación previa de existencia |

---

## Guía de Despliegue paso a paso

### Prerrequisitos

- Cuenta AWS activa (capa gratuita)
- [AWS CLI](https://docs.aws.amazon.com/cli/latest/userguide/install-cliv2.html) instalado y configurado
- Node.js 18+ instalado localmente

```bash
aws configure
# Ingresa: Access Key ID, Secret Access Key, Region (us-east-1), formato (json)
```

---

### Paso 1 — Crear la tabla DynamoDB

```bash
aws dynamodb create-table \
  --table-name Contactos \
  --attribute-definitions \
      AttributeName=contactoId,AttributeType=S \
      AttributeName=email,AttributeType=S \
  --key-schema AttributeName=contactoId,KeyType=HASH \
  --billing-mode PAY_PER_REQUEST \
  --global-secondary-indexes '[{
    "IndexName": "email-index",
    "KeySchema": [{"AttributeName":"email","KeyType":"HASH"}],
    "Projection": {"ProjectionType":"ALL"}
  }]' \
  --region us-east-1
```

Verificar creación:
```bash
aws dynamodb describe-table --table-name Contactos --query "Table.TableStatus"
# Debe devolver: "ACTIVE"
```

---

### Paso 2 — Empaquetar el código Lambda

```bash
cd lambda/
npm install
zip -r ../lambda-function.zip . -x "*.git*"
cd ..
```

---

### Paso 3 — Crear el rol IAM (mínimo privilegio)

```bash
# Crear rol
aws iam create-role \
  --role-name CrudContactosLambdaRole \
  --assume-role-policy-document '{
    "Version":"2012-10-17",
    "Statement":[{
      "Effect":"Allow",
      "Principal":{"Service":"lambda.amazonaws.com"},
      "Action":"sts:AssumeRole"
    }]
  }'

# Adjuntar política de logs
aws iam attach-role-policy \
  --role-name CrudContactosLambdaRole \
  --policy-arn arn:aws:iam::aws:policy/service-role/AWSLambdaBasicExecutionRole

# Crear y adjuntar política mínima para DynamoDB
aws iam put-role-policy \
  --role-name CrudContactosLambdaRole \
  --policy-name DynamoDBMinimalPolicy \
  --policy-document '{
    "Version":"2012-10-17",
    "Statement":[{
      "Effect":"Allow",
      "Action":["dynamodb:PutItem","dynamodb:GetItem","dynamodb:UpdateItem",
                "dynamodb:DeleteItem","dynamodb:Scan","dynamodb:Query"],
      "Resource":[
        "arn:aws:dynamodb:us-east-1:*:table/Contactos",
        "arn:aws:dynamodb:us-east-1:*:table/Contactos/index/*"
      ]
    }]
  }'
```

---

### Paso 4 — Crear la función Lambda

```bash
# Obtener el ARN del rol recién creado
ROLE_ARN=$(aws iam get-role --role-name CrudContactosLambdaRole \
  --query "Role.Arn" --output text)

# Crear la función
aws lambda create-function \
  --function-name crud-contactos \
  --runtime nodejs18.x \
  --handler index.handler \
  --role $ROLE_ARN \
  --zip-file fileb://lambda-function.zip \
  --timeout 15 \
  --memory-size 128 \
  --environment "Variables={TABLE_NAME=Contactos,AWS_NODEJS_CONNECTION_REUSE_ENABLED=1}" \
  --region us-east-1
```

Probar localmente antes de crear la API:
```bash
aws lambda invoke \
  --function-name crud-contactos \
  --payload '{"httpMethod":"GET","path":"/contactos"}' \
  --cli-binary-format raw-in-base64-out \
  response.json && cat response.json
```

---

### Paso 5 — Crear API Gateway

```bash
# Crear la API
API_ID=$(aws apigateway create-rest-api \
  --name "crud-contactos-api" \
  --endpoint-configuration types=REGIONAL \
  --query "id" --output text)

echo "API ID: $API_ID"

# Obtener el resource raíz
ROOT_ID=$(aws apigateway get-resources \
  --rest-api-id $API_ID \
  --query "items[0].id" --output text)

# Crear recurso /contactos
CONTACTOS_ID=$(aws apigateway create-resource \
  --rest-api-id $API_ID \
  --parent-id $ROOT_ID \
  --path-part contactos \
  --query "id" --output text)

# Crear recurso /contactos/{contactoId}
ID_RESOURCE=$(aws apigateway create-resource \
  --rest-api-id $API_ID \
  --parent-id $CONTACTOS_ID \
  --path-part "{contactoId}" \
  --query "id" --output text)

LAMBDA_ARN=$(aws lambda get-function \
  --function-name crud-contactos \
  --query "Configuration.FunctionArn" --output text)

# Función para crear método + integración Lambda
create_method() {
  RESOURCE=$1; METHOD=$2
  aws apigateway put-method \
    --rest-api-id $API_ID --resource-id $RESOURCE \
    --http-method $METHOD --authorization-type NONE

  aws apigateway put-integration \
    --rest-api-id $API_ID --resource-id $RESOURCE \
    --http-method $METHOD --type AWS_PROXY \
    --integration-http-method POST \
    --uri "arn:aws:apigateway:us-east-1:lambda:path/2015-03-31/functions/${LAMBDA_ARN}/invocations"
}

# Métodos en /contactos
create_method $CONTACTOS_ID GET
create_method $CONTACTOS_ID POST
create_method $CONTACTOS_ID OPTIONS

# Métodos en /contactos/{contactoId}
create_method $ID_RESOURCE GET
create_method $ID_RESOURCE PUT
create_method $ID_RESOURCE DELETE
create_method $ID_RESOURCE OPTIONS

# Permiso para que API Gateway invoque Lambda
ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)
aws lambda add-permission \
  --function-name crud-contactos \
  --statement-id apigw-invoke \
  --action lambda:InvokeFunction \
  --principal apigateway.amazonaws.com \
  --source-arn "arn:aws:execute-api:us-east-1:${ACCOUNT_ID}:${API_ID}/*"

# Deploy
aws apigateway create-deployment \
  --rest-api-id $API_ID \
  --stage-name prod

echo "URL de la API:"
echo "https://${API_ID}.execute-api.us-east-1.amazonaws.com/prod/contactos"
```

---

### Paso 6 — Configurar el frontend y subir a S3

```bash
# 1. Editar frontend/index.html:
#    Reemplaza la variable API_BASE_URL con la URL del paso anterior

# 2. Crear bucket S3 (nombre único)
BUCKET="contactos-crud-$(date +%s)"
aws s3 mb s3://$BUCKET --region us-east-1

# 3. Habilitar hosting estático
aws s3 website s3://$BUCKET \
  --index-document index.html \
  --error-document index.html

# 4. Hacer el bucket público (solo para demo académico)
aws s3api put-bucket-policy --bucket $BUCKET --policy '{
  "Version":"2012-10-17",
  "Statement":[{
    "Effect":"Allow",
    "Principal":"*",
    "Action":"s3:GetObject",
    "Resource":"arn:aws:s3:::'"$BUCKET"'/*"
  }]
}'

# 5. Subir el frontend
aws s3 cp frontend/index.html s3://$BUCKET/

echo "URL del sitio:"
echo "http://${BUCKET}.s3-website-us-east-1.amazonaws.com"
```

---

### (Alternativa) Despliegue con CloudFormation

```bash
aws cloudformation deploy \
  --template-file iac/template.yaml \
  --stack-name crud-contactos \
  --capabilities CAPABILITY_NAMED_IAM \
  --region us-east-1
```

---

## Pruebas con curl

```bash
BASE="https://TU_API_ID.execute-api.us-east-1.amazonaws.com/prod/contactos"

# CREATE
curl -X POST $BASE \
  -H "Content-Type: application/json" \
  -d '{"nombre":"Juan Rios","email":"juan@email.com","telefono":"+57 300 1234567","empresa":"Universidad"}'

# READ (lista)
curl $BASE

# READ (filtro)
curl "$BASE?filtro=juan"

# UPDATE (cambia {ID} por el contactoId retornado)
curl -X PUT $BASE/{ID} \
  -H "Content-Type: application/json" \
  -d '{"telefono":"+57 310 9876543"}'

# DELETE
curl -X DELETE $BASE/{ID}
```

---

## Seguridad implementada

| Medida | Implementación |
|---|---|
| Credenciales | IAM Role asignado a Lambda (sin hardcodear keys) |
| Mínimo privilegio | Solo los 6 permisos DynamoDB necesarios |
| Validación de entradas | Regex en backend (email, teléfono, longitud nombre) |
| ConditionExpression | Previene race conditions en UPDATE y DELETE |
| HTTPS | Forzado por API Gateway por defecto |
| CORS | Configurado con cabeceras explícitas |
| XSS | Función `esc()` en frontend previene inyección HTML |

---

## Estructura del repositorio

```
crud-contactos-aws/
├── lambda/
│   ├── index.js          # Handler Lambda — todas las operaciones CRUD
│   └── package.json      # Dependencias (AWS SDK v3)
├── frontend/
│   └── index.html        # SPA completa (HTML + CSS + JS)
├── iac/
│   └── template.yaml     # CloudFormation — infraestructura como código
└── README.md             # Esta documentación
```

---

## Decisiones técnicas

**¿Por qué Lambda + API Gateway en lugar de EC2?**  
Lambda en la capa gratuita ofrece 1 millón de invocaciones mensuales gratuitas indefinidamente, sin necesidad de mantener un servidor encendido. EC2 con t2.micro solo es gratuito el primer año.

**¿Por qué DynamoDB On-Demand (PAY_PER_REQUEST)?**  
El modo on-demand escala automáticamente a 0 cuando no hay tráfico, garantizando que nunca se excedan los límites de la capa gratuita (25 WCU / 25 RCU en modo provisionado son suficientes para producción ligera, pero on-demand es más seguro para picos).

**¿Por qué AWS SDK v3 y no v2?**  
El SDK v3 usa módulos individuales (tree-shaking), reduciendo el tamaño del paquete Lambda de ~50 MB (v2) a ~5 MB, lo que mejora los cold starts y está incluido en el runtime Node.js 18.

**¿Por qué UUID como Partition Key y no email?**  
Permite múltiples registros futuros por usuario (ej. contactos duplicados en organizaciones distintas) y distribuye la carga uniformemente en DynamoDB. El email se indexa en un GSI para búsquedas eficientes.

**¿Por qué ConditionExpression en UPDATE y DELETE?**  
Previene race conditions: si dos usuarios intentan eliminar el mismo contacto simultáneamente, solo uno tiene éxito y el otro recibe un error claro (409 Conflict) en lugar de un error genérico.



*Integrantes: RIOS GONZALEZ JUAN MIGUEL · NIETO CARDEÑO BREYNEER · GARCIA ILIAS ALEXA TATIANA*
