# Módulo 4 — Almacenamiento de objetos S3
### Servicios: S3 · Políticas · Versionado · Ciclo de vida · Replicación

---

## TEORÍA

### S3 — Simple Storage Service
Almacenamiento de objetos infinitamente escalable. No es un disco (no puedes montar S3 como unidad). Cada archivo es un **objeto** con una **key** (nombre/ruta), **value** (contenido) y **metadata**.

### Clases de almacenamiento
| Clase | Latencia | Uso | Costo relativo |
|-------|----------|-----|----------------|
| S3 Standard | Milisegundos | Acceso frecuente | $$$ |
| S3 Intelligent-Tiering | Milisegundos | Patrón de acceso variable | $$ |
| S3 Standard-IA | Milisegundos | Acceso infrecuente, pero rápido cuando lo necesitas | $$ |
| S3 Glacier Instant Retrieval | Milisegundos | Archivos raramente accedidos | $ |
| S3 Glacier Flexible Retrieval | Minutos-horas | Backups a largo plazo | $ |
| S3 Glacier Deep Archive | Horas | Archivos de auditoría / compliance | $¢ |

### Versionado
Cada vez que sobreescribes un objeto, S3 guarda todas las versiones. Puedes restaurar cualquier versión anterior. Los objetos eliminados reciben un "Delete Marker" (no se borran físicamente hasta que eliminas el marcador).

### Bucket Policies
Políticas en JSON adjuntas al **bucket** (no al usuario IAM). Controlan acceso desde cualquier entidad: usuarios, roles, servicios, cuentas externas.

---

## 🟢 LAB BÁSICO — Crear bucket + versionado + website estático

**Tiempo estimado: 35 min**

---

### Paso 1 — Crear bucket S3

```
Consola AWS → buscar "S3" → clic en S3
Clic "Create bucket"

  Bucket name: lab-s3-TUAPELLIDO-2024
              ⚠️  El nombre debe ser ÚNICO globalmente en todo AWS
              ⚠️  Solo letras minúsculas, números y guiones

  AWS Region: us-east-1 (N. Virginia)

  Object Ownership: ACLs disabled (Recommended)

  Block Public Access settings:
    ✅ Block all public access  ← dejar marcado por ahora

  Bucket Versioning: Disable  ← lo habilitamos en el paso 3

  Default encryption:
    Server-side encryption: Enable
    Encryption type: SSE-S3  (clave administrada por AWS)
    Bucket Key: Enable  (reduce costos de KMS)

Clic "Create bucket"
✅ Bucket creado
```

---

### Paso 2 — Subir archivos al bucket

```bash
# Crear archivos de prueba localmente en tu laptop
mkdir ~/lab-s3
echo "<h1>Version 1 - Hola desde S3!</h1>" > ~/lab-s3/index.html
echo "Texto de prueba version 1" > ~/lab-s3/datos.txt

# Subir via AWS CLI
aws s3 cp ~/lab-s3/index.html s3://lab-s3-TUAPELLIDO-2024/
aws s3 cp ~/lab-s3/datos.txt  s3://lab-s3-TUAPELLIDO-2024/

# Verificar que están en el bucket
aws s3 ls s3://lab-s3-TUAPELLIDO-2024/
# Salida:
# 2024-11-13 10:00:00         40 datos.txt
# 2024-11-13 10:00:00         36 index.html
```

```
Desde la consola:
  S3 → seleccionar el bucket → ver los objetos ✅
  Clic en "index.html" → Pestaña "Details" → ver el Object URL
```

---

### Paso 3 — Habilitar versionado

```
S3 → bucket "lab-s3-TUAPELLIDO-2024"
Pestaña "Properties"
Sección "Bucket Versioning" → "Edit"
  Versioning: Enable
Clic "Save changes"
```

```bash
# Subir versión 2 del mismo archivo (sobreescribir)
echo "<h1>Version 2 - Actualizado!</h1>" > ~/lab-s3/index.html
aws s3 cp ~/lab-s3/index.html s3://lab-s3-TUAPELLIDO-2024/

# Subir versión 3
echo "<h1>Version 3 - La mas reciente</h1>" > ~/lab-s3/index.html
aws s3 cp ~/lab-s3/index.html s3://lab-s3-TUAPELLIDO-2024/
```

```
Ver versiones en la consola:
  S3 → bucket → clic en "index.html"
  Pestaña "Versions" → deberías ver 3 versiones ✅
  Cada versión tiene su propio Version ID
```

```bash
# Ver versiones via CLI
aws s3api list-object-versions \
  --bucket lab-s3-TUAPELLIDO-2024 \
  --prefix index.html \
  --query 'Versions[*].[VersionId,LastModified,IsLatest]' \
  --output table

# Restaurar versión anterior — descargar una versión específica
aws s3api get-object \
  --bucket lab-s3-TUAPELLIDO-2024 \
  --key index.html \
  --version-id REEMPLAZAR-CON-VERSION-ID \
  ~/lab-s3/index-version1-restaurado.html

cat ~/lab-s3/index-version1-restaurado.html
# Muestra: <h1>Version 1 - Hola desde S3!</h1>  ✅
```

---

### Paso 4 — Habilitar Static Website Hosting

```
S3 → bucket "lab-s3-TUAPELLIDO-2024"
Pestaña "Properties"
Sección "Static website hosting" → "Edit"

  Static website hosting: Enable
  Hosting type: Host a static website
  Index document: index.html
  Error document: error.html   (opcional)

Clic "Save changes"
```

```
⚠️  Para que el sitio sea accesible públicamente:

Pestaña "Permissions"
Sección "Block Public Access" → "Edit"
  Desmarcar "Block all public access"
  Clic "Save changes" → escribir "confirm" → Confirm

Sección "Bucket policy" → "Edit"
Pegar la siguiente política:
```

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "PublicReadGetObject",
      "Effect": "Allow",
      "Principal": "*",
      "Action": "s3:GetObject",
      "Resource": "arn:aws:s3:::lab-s3-TUAPELLIDO-2024/*"
    }
  ]
}
```

```
⚠️  Reemplazar "lab-s3-TUAPELLIDO-2024" con el nombre real de tu bucket

Clic "Save changes"

Ir a: Pestaña "Properties" → "Static website hosting"
Copiar el "Bucket website endpoint"
  ej: http://lab-s3-TUAPELLIDO-2024.s3-website-us-east-1.amazonaws.com

Abrir en el navegador → ✅ Deberías ver la página "Version 3 - La mas reciente"
```

---

## 🟡 LAB INTERMEDIO — Bucket Policy por VPC + Lifecycle Rules + Object Lock

**Tiempo estimado: 55 min**

---

### Paso 1 — Crear segundo bucket (privado, solo acceso desde VPC)

```
S3 → "Create bucket"
  Bucket name: lab-s3-privado-TUAPELLIDO-2024
  Region: us-east-1
  Block all public access: ✅ (todo bloqueado)
  Versioning: Enable
Clic "Create bucket"
```

**Bucket Policy que restringe acceso solo desde la VPC:**
```
S3 → "lab-s3-privado-TUAPELLIDO-2024"
Pestaña "Permissions" → "Bucket policy" → "Edit"
Pegar:
```

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "DenegarAccesoFueraDeVPC",
      "Effect": "Deny",
      "Principal": "*",
      "Action": "s3:*",
      "Resource": [
        "arn:aws:s3:::lab-s3-privado-TUAPELLIDO-2024",
        "arn:aws:s3:::lab-s3-privado-TUAPELLIDO-2024/*"
      ],
      "Condition": {
        "StringNotEquals": {
          "aws:sourceVpc": "vpc-xxxxxxxxxx"
        }
      }
    }
  ]
}
```

```
⚠️  Reemplazar:
  "lab-s3-privado-TUAPELLIDO-2024" con el nombre real de tu bucket
  "vpc-xxxxxxxxxx" con el VPC ID de lab-vpc-01

Clic "Save changes"

Verificación:
  Intentar acceder al bucket desde fuera de la VPC → AccessDenied ✅
  Desde instancia EC2 en lab-vpc-01 → acceso permitido ✅
```

```bash
# Probar desde EC2 en lab-vpc-01:
aws s3 ls s3://lab-s3-privado-TUAPELLIDO-2024/
# ✅ Funciona (dentro de la VPC)

# Probar desde tu laptop (fuera de la VPC):
aws s3 ls s3://lab-s3-privado-TUAPELLIDO-2024/
# Error: An error occurred (AccessDenied) ... ✅ Política funciona
# NOTA: Si tienes el VPC Endpoint de S3, el acceso desde EC2 va por ahí
```

---

### Paso 2 — Configurar Lifecycle Rules

```
S3 → "lab-s3-TUAPELLIDO-2024"
Pestaña "Management"
Sección "Lifecycle rules" → "Create lifecycle rule"

  Lifecycle rule name: optimizar-costos-almacenamiento
  Rule scope: Apply to all objects in the bucket

  Lifecycle rule actions:
    ✅ Transition current versions of objects between storage classes
    ✅ Expire current versions of objects
    ✅ Delete expired object delete markers or incomplete multipart uploads

  Transition current versions:
    Agregar transición 1:
      Transition after: 30 days
      Transition to: S3 Standard-IA   (acceso infrecuente)
    
    Agregar transición 2:
      Transition after: 90 days
      Transition to: S3 Glacier Instant Retrieval
    
    Agregar transición 3:
      Transition after: 365 days
      Transition to: S3 Glacier Deep Archive

  Expire current versions:
    Expire after: 730 days   (2 años)

  Delete expired delete markers: ✅
  Incomplete multipart uploads: Delete after 7 days

Clic "Create rule"
```

---

### Paso 3 — Crear bucket con Object Lock (WORM)

> Object Lock solo puede habilitarse al **crear** el bucket, no después.

```
S3 → "Create bucket"
  Bucket name: lab-s3-worm-TUAPELLIDO-2024
  Region: us-east-1

  Object Ownership: ACLs disabled
  Block all public access: ✅

  Bucket Versioning: Enable   ← OBLIGATORIO para Object Lock

  Advanced settings:
    Object Lock: Enable   ← aquí está la opción
    ⚠️  Leer la advertencia y confirmar

Clic "Create bucket"
```

```
S3 → "lab-s3-worm-TUAPELLIDO-2024"
Pestaña "Properties"
Sección "Object Lock" → "Edit"

  Default retention:  Enable
  Retention mode:     Compliance   
    (Governance permite excepciones con permisos especiales)
    (Compliance NO permite eliminar aunque seas admin durante el período)
  Retention period:   1 days   (usar 1 día para el lab, en producción sería más)

Clic "Save changes"
```

```bash
# Subir un objeto al bucket WORM
echo "Documento importante - no borrar" > ~/lab-s3/documento-worm.txt
aws s3 cp ~/lab-s3/documento-worm.txt s3://lab-s3-worm-TUAPELLIDO-2024/

# Intentar eliminar el objeto (debe fallar)
aws s3 rm s3://lab-s3-worm-TUAPELLIDO-2024/documento-worm.txt
# Error esperado:
# An error occurred (AccessDenied) when calling the DeleteObject operation
# ✅ Object Lock en modo Compliance protege el archivo
```

---

### Paso 4 — S3 Transfer Acceleration

```
S3 → "lab-s3-TUAPELLIDO-2024"
Pestaña "Properties"
Sección "Transfer acceleration" → "Edit"
  Transfer acceleration: Enable
Clic "Save changes"
```

```bash
# Comparar velocidad normal vs acelerada
# La URL de Transfer Acceleration es diferente:
# Normal:       lab-s3-TUAPELLIDO-2024.s3.amazonaws.com
# Acelerada:    lab-s3-TUAPELLIDO-2024.s3-accelerate.amazonaws.com

# Crear archivo de prueba de 100MB
dd if=/dev/urandom of=~/lab-s3/archivo-grande.bin bs=1M count=100

# Subida normal
time aws s3 cp ~/lab-s3/archivo-grande.bin \
  s3://lab-s3-TUAPELLIDO-2024/test-normal.bin

# Subida con Transfer Acceleration
time aws s3 cp ~/lab-s3/archivo-grande.bin \
  s3://lab-s3-TUAPELLIDO-2024/test-acelerado.bin \
  --endpoint-url https://s3-accelerate.amazonaws.com

# Comparar los tiempos en "real" de cada comando
# ⚠️  Transfer Acceleration beneficia más desde ubicaciones alejadas (ej: Lima → us-east-1)
```

---

## 🔴 LAB AVANZADO — CRR Replication + S3 Event → Lambda + S3 Select

**Tiempo estimado: 75 min**

---

### Parte 1 — Cross-Region Replication (CRR)

**Paso 1.1 — Crear bucket destino en otra región**
```
Cambiar región a: sa-east-1 (São Paulo)

S3 → "Create bucket"
  Name:    lab-s3-replica-TUAPELLIDO-2024
  Region:  sa-east-1
  Versioning: Enable   ← OBLIGATORIO para destino de replicación
  Default encryption: Enable, SSE-S3
Clic "Create bucket"

Volver a región: us-east-1
```

**Paso 1.2 — Configurar regla de replicación en el bucket origen**
```
S3 → "lab-s3-TUAPELLIDO-2024"
Pestaña "Management"
Sección "Replication rules" → "Create replication rule"

  Replication rule name: replicacion-dr-sao-paulo
  Status: Enabled
  
  Source bucket:
    Rule scope: Apply to all objects in the bucket

  Destination:
    Choose a bucket in this account: ✅
    Bucket name: lab-s3-replica-TUAPELLIDO-2024
    (Si pide región, confirmar: sa-east-1)

  IAM role:
    Create new role: ✅  (AWS crea el rol automáticamente)
    Name: s3-replication-role-lab

  Additional replication options:
    ✅ Replication Time Control (RTC) — garantiza replicación en 15 minutos
    ✅ Replication metrics and notifications

Clic "Save"

Pregunta: "Do you want to replicate existing objects?"
  Seleccionar: "No" (para el lab)
  Clic "Submit"
```

**Paso 1.3 — Verificar replicación**
```bash
# Subir objeto nuevo en us-east-1
echo "Objeto para replicar - $(date)" > ~/lab-s3/objeto-replica.txt
aws s3 cp ~/lab-s3/objeto-replica.txt s3://lab-s3-TUAPELLIDO-2024/

# Esperar ~30-60 segundos y verificar en sa-east-1
aws s3 ls s3://lab-s3-replica-TUAPELLIDO-2024/ \
  --region sa-east-1
# ✅ Deberías ver "objeto-replica.txt" en la réplica
```

```
Verificar estado de replicación de un objeto:
  S3 → bucket origen → clic en "objeto-replica.txt"
  Pestaña "Details" → sección "Replication status"
  Debe mostrar: "COMPLETED" ✅
```

---

### Parte 2 — Notificación S3 → Lambda

**Paso 2.1 — Crear la función Lambda**
```
Consola → buscar "Lambda" → clic en Lambda
Clic "Create function"

  Author from scratch
  Function name:   procesar-imagen-s3
  Runtime:         Python 3.12
  Architecture:    x86_64
  
  Permissions:
    Create a new role with basic Lambda permissions
    (Luego le agregaremos permisos S3)

Clic "Create function"
```

```
En el editor de código, reemplazar TODO el contenido con:
```

```python
import json
import urllib.parse
import boto3
import os

s3 = boto3.client('s3')

def lambda_handler(event, context):
    """
    Se ejecuta automáticamente cuando se sube un objeto a S3.
    Registra el evento y obtiene metadata del objeto.
    """
    
    # Extraer información del evento S3
    bucket = event['Records'][0]['s3']['bucket']['name']
    key    = urllib.parse.unquote_plus(
                 event['Records'][0]['s3']['object']['key'],
                 encoding='utf-8'
             )
    size   = event['Records'][0]['s3']['object']['size']
    
    print(f"=== Nuevo objeto subido a S3 ===")
    print(f"Bucket: {bucket}")
    print(f"Objeto: {key}")
    print(f"Tamaño: {size} bytes")
    
    # Obtener metadata del objeto
    try:
        response = s3.head_object(Bucket=bucket, Key=key)
        content_type = response['ContentType']
        print(f"Tipo de contenido: {content_type}")
    except Exception as e:
        print(f"Error obteniendo metadata: {str(e)}")
    
    return {
        'statusCode': 200,
        'body': json.dumps({
            'mensaje': f'Objeto procesado: {key}',
            'bucket': bucket,
            'tamaño': f'{size} bytes'
        })
    }
```

```
Clic "Deploy" para guardar los cambios
```

**Paso 2.2 — Dar permisos S3 a Lambda**
```
En la función Lambda:
Pestaña "Configuration" → "Permissions"
Clic en el nombre del rol (ej: procesar-imagen-s3-role-xxxxxxx)

Se abre IAM → Roles
Clic "Add permissions" → "Attach policies"
Buscar: AmazonS3ReadOnlyAccess
Marcar → Clic "Add permissions"
```

**Paso 2.3 — Configurar notificación en S3**
```
S3 → "lab-s3-TUAPELLIDO-2024"
Pestaña "Properties"
Sección "Event notifications" → "Create event notification"

  Event name:        notificacion-nuevo-objeto
  
  Event types:
    ✅ All object create events  (s3:ObjectCreated:*)

  Destination:
    Lambda function
    Lambda function: seleccionar "procesar-imagen-s3"

Clic "Save changes"
✅ S3 ahora puede invocar tu función Lambda
```

**Paso 2.4 — Probar la notificación**
```bash
# Subir un archivo
echo "Archivo de prueba para Lambda" > ~/lab-s3/test-lambda.txt
aws s3 cp ~/lab-s3/test-lambda.txt s3://lab-s3-TUAPELLIDO-2024/
```

```
Ver los logs de Lambda:
  Lambda → "procesar-imagen-s3"
  Pestaña "Monitor" → "View CloudWatch logs"
  
  CloudWatch Logs → Log groups → "/aws/lambda/procesar-imagen-s3"
  Abrir el log stream más reciente
  
  Deberías ver:
  === Nuevo objeto subido a S3 ===
  Bucket: lab-s3-TUAPELLIDO-2024
  Objeto: test-lambda.txt
  Tamaño: 31 bytes  ✅
```

---

### Parte 3 — S3 Select (consultar CSV sin descargarlo completo)

```bash
# Crear un archivo CSV grande de ejemplo
cat > ~/lab-s3/empleados.csv << 'EOF'
id,nombre,departamento,salario,ciudad
1,Ana Garcia,Tecnologia,5000,Lima
2,Carlos Lopez,RRHH,3500,Bogota
3,Maria Torres,Tecnologia,6000,Lima
4,Juan Perez,Finanzas,4500,Santiago
5,Sofia Ramirez,Tecnologia,5500,Lima
6,Pedro Martinez,RRHH,3800,Bogota
7,Laura Sanchez,Finanzas,4800,Santiago
8,Miguel Rodriguez,Tecnologia,7000,Lima
9,Carmen Flores,Marketing,4000,Lima
10,Roberto Diaz,Tecnologia,6500,Bogota
EOF

# Subir el CSV a S3
aws s3 cp ~/lab-s3/empleados.csv s3://lab-s3-TUAPELLIDO-2024/datos/empleados.csv
```

```bash
# Consultar SOLO los empleados de Tecnologia sin descargar todo el CSV
aws s3api select-object-content \
  --bucket lab-s3-TUAPELLIDO-2024 \
  --key datos/empleados.csv \
  --expression "SELECT * FROM S3Object WHERE departamento = 'Tecnologia'" \
  --expression-type SQL \
  --input-serialization '{"CSV":{"FileHeaderInfo":"USE","RecordDelimiter":"\n","FieldDelimiter":","}}' \
  --output-serialization '{"CSV":{"RecordDelimiter":"\n"}}' \
  /dev/stdout

# Resultado esperado:
# 1,Ana Garcia,Tecnologia,5000,Lima
# 3,Maria Torres,Tecnologia,6000,Lima
# 5,Sofia Ramirez,Tecnologia,5500,Lima
# 8,Miguel Rodriguez,Tecnologia,7000,Lima
# 10,Roberto Diaz,Tecnologia,6500,Bogota

# Consulta mas compleja: empleados de Tecnologia con salario > 5500
aws s3api select-object-content \
  --bucket lab-s3-TUAPELLIDO-2024 \
  --key datos/empleados.csv \
  --expression "SELECT nombre, salario FROM S3Object WHERE departamento = 'Tecnologia' AND CAST(salario AS DECIMAL) > 5500" \
  --expression-type SQL \
  --input-serialization '{"CSV":{"FileHeaderInfo":"USE","RecordDelimiter":"\n","FieldDelimiter":","}}' \
  --output-serialization '{"CSV":{"RecordDelimiter":"\n"}}' \
  /dev/stdout
```

---

## 🧹 LIMPIEZA (en este orden)

```
1.  S3 → Buckets → vaciar cada bucket antes de eliminarlo:
    Seleccionar bucket → "Empty" → escribir "permanently delete" → confirmar

    Orden de vaciado:
    a) lab-s3-worm-TUAPELLIDO-2024  
       (esperar que expire el Object Lock — 1 día si pusiste 1 día)
    b) lab-s3-replica-TUAPELLIDO-2024  (en sa-east-1 — cambiar región)
    c) lab-s3-privado-TUAPELLIDO-2024
    d) lab-s3-TUAPELLIDO-2024

2.  Una vez vaciados, eliminar los buckets:
    Seleccionar bucket → "Delete" → escribir nombre del bucket → confirmar

3.  Lambda → "procesar-imagen-s3" → Actions → Delete function

4.  IAM → Roles → eliminar:
    - s3-replication-role-lab
    - procesar-imagen-s3-role-xxxxxxx

5.  CloudWatch → Log groups → eliminar "/aws/lambda/procesar-imagen-s3"
```

---

## 📌 COMANDOS CLI DE REFERENCIA RÁPIDA

```bash
# Listar todos tus buckets S3
aws s3 ls

# Listar objetos en un bucket
aws s3 ls s3://NOMBRE-BUCKET/ --recursive

# Subir archivo
aws s3 cp archivo.txt s3://NOMBRE-BUCKET/

# Subir carpeta completa
aws s3 sync ./carpeta-local/ s3://NOMBRE-BUCKET/carpeta-remota/

# Descargar archivo
aws s3 cp s3://NOMBRE-BUCKET/archivo.txt ./

# Eliminar objeto
aws s3 rm s3://NOMBRE-BUCKET/archivo.txt

# Ver versiones de un objeto
aws s3api list-object-versions \
  --bucket NOMBRE-BUCKET \
  --prefix nombre-archivo.txt

# Ver tamaño total del bucket
aws s3 ls s3://NOMBRE-BUCKET --recursive --human-readable --summarize | tail -2

# Ver configuración de replicación
aws s3api get-bucket-replication --bucket NOMBRE-BUCKET

# Ver política del bucket
aws s3api get-bucket-policy --bucket NOMBRE-BUCKET | python3 -m json.tool
```
