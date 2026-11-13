# Módulo 1 — Fundamentos AWS y red de la solución
### Servicios: IAM · VPC · Route 53

---

## TEORÍA

### IAM — Identity and Access Management
Controla **quién puede hacer qué** en tu cuenta AWS.
- **Usuario**: persona o app con credenciales propias (usuario + contraseña / access keys)
- **Grupo**: colección de usuarios que comparten permisos
- **Rol**: identidad temporal que asumen servicios (EC2, Lambda) o usuarios externos
- **Política**: documento JSON con reglas Allow/Deny sobre acciones y recursos

### VPC — Virtual Private Cloud
Es tu **red privada dentro de AWS**. Tú defines el rango de IPs, subnets, rutas y accesos.
- **Subnet pública**: tiene ruta al Internet Gateway → instancias pueden recibir tráfico de internet
- **Subnet privada**: sin ruta directa → solo tráfico interno o salida via NAT Gateway
- **Internet Gateway (IGW)**: puerta de entrada/salida a internet para subnets públicas
- **NAT Gateway**: permite que instancias privadas salgan a internet pero nadie entra a ellas

### Route 53 — DNS Administrado
Traduce nombres de dominio a IPs.
- **Registro A**: nombre → IPv4
- **Registro CNAME**: nombre → otro nombre (no funciona en el apex del dominio, ej: `ejemplo.com`)
- **Registro Alias**: como CNAME pero nativo AWS, funciona en el apex y no cobra por query
- **TTL**: segundos que los resolvers cachean el registro antes de volver a consultar

---

## 🟢 LAB BÁSICO — Crear usuario IAM + VPC + DNS privado

**Tiempo estimado: 45 min**

---

### PARTE 1 — Crear usuario IAM con política de solo lectura S3

**Paso 1.1 — Abrir IAM**
```
Consola AWS (console.aws.amazon.com)
Barra de búsqueda superior → escribir "IAM" → clic en IAM
```

**Paso 1.2 — Crear el usuario**
```
Panel izquierdo → "Users" → botón "Create user"

  User name: lab-usuario-01
  
  ✅ Provide user access to the AWS Management Console: marcar
  Console password: Custom password → escribir: Lab@2024!
  Users must create a new password at next sign-in: desmarcar

Clic "Next"
```

**Paso 1.3 — Asignar permisos**
```
Permissions options: "Attach policies directly"
En la barra de búsqueda: AmazonS3ReadOnlyAccess
Marcar la casilla de "AmazonS3ReadOnlyAccess"

Clic "Next" → "Create user"
```

**Paso 1.4 — Crear Access Key para CLI**
```
Clic en el usuario "lab-usuario-01"
Pestaña "Security credentials"
Sección "Access keys" → "Create access key"

Seleccionar: "Command Line Interface (CLI)"
Marcar el checkbox de confirmación
Clic "Next"
  Description tag: lab-key-cli
Clic "Create access key"

⚠️  IMPORTANTE: Hacer clic en "Download .csv file"
    No podrás ver la Secret Key después de cerrar esta ventana
```

**Paso 1.5 — Probar con AWS CLI**
```bash
# Verificar que tienes AWS CLI instalado
aws --version
# Debe mostrar: aws-cli/2.x.x ...

# Configurar perfil con las credenciales del CSV
aws configure --profile lab-usuario-01

# Cuando pida ingresar:
AWS Access Key ID [None]: AKIA...  (del CSV)
AWS Secret Access Key [None]: xxxx  (del CSV)
Default region name [None]: us-east-1
Default output format [None]: json

# Probar — listar buckets S3 (funciona con ReadOnly)
aws s3 ls --profile lab-usuario-01

# Probar acción NO permitida — crear bucket (debe fallar)
aws s3 mb s3://test-acceso-$(date +%s) --profile lab-usuario-01
# Error esperado: An error occurred (AccessDenied) ...
# ✅ Confirma que la política ReadOnly funciona correctamente
```

---

### PARTE 2 — Crear VPC con subnet pública

**Paso 2.1 — Ir a VPC**
```
Barra de búsqueda → "VPC" → clic en VPC
Verificar región arriba a la derecha: N. Virginia (us-east-1)
```

**Paso 2.2 — Crear la VPC**
```
Panel izquierdo → "Your VPCs" → "Create VPC"

  Seleccionar: "VPC only"  (no "VPC and more" — lo hacemos manual para aprender)
  
  Name tag:          lab-vpc-01
  IPv4 CIDR block:   10.0.0.0/16
  IPv6 CIDR block:   No IPv6 CIDR block
  Tenancy:           Default

Clic "Create VPC"
✅ Aparece: "You successfully created vpc-xxxxxxxxx"
```

**Paso 2.3 — Crear subnet pública**
```
Panel izquierdo → "Subnets" → "Create subnet"

  VPC ID: seleccionar "lab-vpc-01"
  
  Subnet settings:
    Subnet name:              lab-subnet-publica-1a
    Availability Zone:        us-east-1a
    IPv4 subnet CIDR block:   10.0.1.0/24

Clic "Create subnet"
```

**Paso 2.4 — Crear Internet Gateway**
```
Panel izquierdo → "Internet Gateways" → "Create internet gateway"

  Name tag: lab-igw-01

Clic "Create internet gateway"

Una vez creado (estado: detached):
  Botón "Actions" → "Attach to VPC"
  Available VPCs: seleccionar "lab-vpc-01"
  Clic "Attach internet gateway"
  
✅ Estado cambia a "attached"
```

**Paso 2.5 — Crear tabla de rutas y asociarla a la subnet**
```
Panel izquierdo → "Route Tables" → "Create route table"

  Name:  lab-rtb-publica
  VPC:   lab-vpc-01

Clic "Create route table"

Con la tabla seleccionada → pestaña "Routes" → "Edit routes":
  Clic "Add route"
    Destination: 0.0.0.0/0
    Target:      Internet Gateway → seleccionar "lab-igw-01"
  Clic "Save changes"

Pestaña "Subnet associations" → "Edit subnet associations":
  Marcar "lab-subnet-publica-1a"
  Clic "Save associations"

✅ Subnet pública ahora tiene ruta a internet
```

---

### PARTE 3 — Crear DNS privado con Route 53

> Si no tienes dominio propio, usa una **Private Hosted Zone** para practicar DNS interno sin costo adicional.

**Paso 3.1 — Crear Hosted Zone privada**
```
Barra de búsqueda → "Route 53" → clic en Route 53
Panel izquierdo → "Hosted zones" → "Create hosted zone"

  Domain name:  lab.interno
  Description:  Zona DNS privada para laboratorio
  Type:         Private hosted zone
  
  VPCs to associate:
    Region: us-east-1
    VPC:    lab-vpc-01

Clic "Create hosted zone"
```

**Paso 3.2 — Crear registro A**
```
Dentro de la hosted zone "lab.interno"
Clic "Create record"

  Record name:  servidor01
  Record type:  A
  Value:        10.0.1.100
  TTL (seconds): 300

Clic "Create records"
✅ El FQDN será: servidor01.lab.interno → resuelve a 10.0.1.100
```

**Verificación (desde instancia EC2 dentro de la VPC):**
```bash
# Instalar herramientas DNS
sudo yum install bind-utils -y

# Consultar el registro privado
dig servidor01.lab.interno

# Salida esperada:
# ;; ANSWER SECTION:
# servidor01.lab.interno. 300 IN A 10.0.1.100

nslookup servidor01.lab.interno
# Salida esperada:
# Server: 10.0.0.2
# Name: servidor01.lab.interno
# Address: 10.0.1.100
```

---

## 🟡 LAB INTERMEDIO — Política IAM personalizada + NAT Gateway + Weighted Routing

**Tiempo estimado: 60 min**

---

### PARTE 1 — Política IAM granular

**Paso 1.1 — Crear la política en JSON**
```
IAM → "Policies" → "Create policy"
Clic en pestaña "JSON" (esquina superior)
Borrar todo el contenido y pegar:
```

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "ListarTodosLosBuckets",
      "Effect": "Allow",
      "Action": "s3:ListAllMyBuckets",
      "Resource": "*"
    },
    {
      "Sid": "AccesoSoloBucketApp",
      "Effect": "Allow",
      "Action": [
        "s3:GetObject",
        "s3:PutObject",
        "s3:DeleteObject",
        "s3:ListBucket"
      ],
      "Resource": [
        "arn:aws:s3:::REEMPLAZAR-CON-TU-BUCKET",
        "arn:aws:s3:::REEMPLAZAR-CON-TU-BUCKET/*"
      ]
    }
  ]
}
```

```
⚠️  Reemplazar "REEMPLAZAR-CON-TU-BUCKET" con el nombre real de tu bucket S3

Clic "Next"
  Policy name: pol-s3-bucket-especifico
  Description: Solo acceso al bucket de la aplicacion
Clic "Create policy"
```

**Paso 1.2 — Crear rol IAM para EC2**
```
IAM → "Roles" → "Create role"

  Trusted entity type: AWS service
  Service or use case:  EC2
  Clic "Next"

  Search: pol-s3-bucket-especifico
  Marcar la casilla
  Clic "Next"

  Role name:  rol-ec2-acceso-s3
  Description: Permite a EC2 acceder al bucket de la app
Clic "Create role"
```

**Paso 1.3 — Adjuntar rol a la instancia EC2**
```
EC2 → Instances → seleccionar "lab-web-01"
Menú "Actions" → "Security" → "Modify IAM role"
  IAM role: rol-ec2-acceso-s3
Clic "Update IAM role"
```

```bash
# Verificar desde la instancia EC2 (no necesitas access keys — usa el rol)
ssh -i ~/Downloads/lab-key-ec2.pem ec2-user@54.xxx.xxx.xxx

# La instancia usa el rol automáticamente
aws s3 ls                          # lista todos los buckets ✓
aws s3 ls s3://tu-bucket-nombre    # lista el bucket específico ✓
aws s3 ls s3://otro-bucket         # AccessDenied ✓ (política funciona)
```

---

### PARTE 2 — Subnet privada + NAT Gateway

**Paso 2.1 — Crear subnet privada**
```
VPC → "Subnets" → "Create subnet"

  VPC ID:   lab-vpc-01
  
  Subnet name:            lab-subnet-privada-1a
  Availability Zone:      us-east-1a
  IPv4 CIDR block:        10.0.2.0/24

Clic "Create subnet"
```

**Paso 2.2 — Asignar Elastic IP para NAT Gateway**
```
VPC → "Elastic IPs" → "Allocate Elastic IP address"
  Network Border Group: us-east-1
Clic "Allocate"
Anotar la IP asignada (ej: 54.x.x.x)
```

**Paso 2.3 — Crear NAT Gateway**
```
VPC → "NAT Gateways" → "Create NAT gateway"

  Name:               lab-nat-01
  Subnet:             lab-subnet-publica-1a   ← OBLIGATORIO: subnet PÚBLICA
  Connectivity type:  Public
  Elastic IP:         seleccionar la IP del paso anterior

Clic "Create NAT gateway"
⏳ Esperar ~2 minutos hasta estado "Available"
```

**Paso 2.4 — Tabla de rutas para subnet privada**
```
VPC → "Route Tables" → "Create route table"
  Name: lab-rtb-privada
  VPC:  lab-vpc-01
Clic "Create route table"

Pestaña "Routes" → "Edit routes" → "Add route":
  Destination: 0.0.0.0/0
  Target:      NAT Gateway → "lab-nat-01"
Clic "Save changes"

Pestaña "Subnet associations" → "Edit subnet associations":
  Marcar "lab-subnet-privada-1a"
Clic "Save associations"
```

---

### PARTE 3 — Route 53 Weighted Routing

**Paso 3.1 — Prerequisito: necesitas dos IPs de servidores**
```
Para el lab usamos estas IPs de ejemplo (reemplazar con tus IPs reales de EC2):
  Servidor A (versión 1): 54.100.100.10
  Servidor B (versión 2): 54.200.200.20
```

**Paso 3.2 — Crear los dos registros ponderados**
```
Route 53 → "Hosted zones" → abrir tu zona pública

REGISTRO 1 — 70% del tráfico:
Clic "Create record"
  Record name:    app
  Record type:    A
  Value:          54.100.100.10
  TTL:            60
  Routing policy: Weighted
  Weight:         70
  Record ID:      servidor-v1
Clic "Create records"

REGISTRO 2 — 30% del tráfico:
Clic "Create record" nuevamente
  Record name:    app
  Record type:    A
  Value:          54.200.200.20
  TTL:            60
  Routing policy: Weighted
  Weight:         30
  Record ID:      servidor-v2
Clic "Create records"
```

**Paso 3.3 — Verificar la distribución**
```bash
# Ejecutar múltiples consultas y ver la distribución
for i in {1..10}; do
  dig +short app.TU-DOMINIO.com
done

# Resultado esperado: ~7 respuestas con IP de v1, ~3 con IP de v2
```

---

## 🔴 LAB AVANZADO — VPC Peering + VPC Endpoint S3 + Route 53 Resolver

**Tiempo estimado: 90 min**

---

### PARTE 1 — VPC Peering

**Paso 1.1 — Crear segunda VPC**
```
VPC → "Your VPCs" → "Create VPC"
  Name:  lab-vpc-02
  CIDR:  10.1.0.0/16   ← DEBE ser diferente a lab-vpc-01 (10.0.0.0/16)
Clic "Create VPC"
```

```
VPC → "Subnets" → "Create subnet"
  VPC:   lab-vpc-02
  Name:  lab-vpc-02-subnet-1a
  AZ:    us-east-1a
  CIDR:  10.1.1.0/24
Clic "Create subnet"
```

**Paso 1.2 — Crear Peering Connection**
```
VPC → "Peering Connections" → "Create peering connection"

  Name:                   lab-peering-01
  VPC ID (Requester):     seleccionar "lab-vpc-01" (10.0.0.0/16)
  Account:                My account
  Region:                 This Region (us-east-1)
  VPC ID (Accepter):      seleccionar "lab-vpc-02" (10.1.0.0/16)

Clic "Create peering connection"
Estado: "Pending Acceptance"
```

**Paso 1.3 — Aceptar la conexión**
```
Con la conexión seleccionada:
Menú "Actions" → "Accept request" → "Accept request"
✅ Estado cambia a "Active"
```

**Paso 1.4 — Agregar rutas en AMBAS VPCs**
```
VPC 1 — tabla "lab-rtb-publica":
  Pestaña Routes → Edit routes → Add route
    Destination: 10.1.0.0/16
    Target: Peering Connection → lab-peering-01
  Save changes

VPC 2 — tabla de rutas por defecto de lab-vpc-02:
  Pestaña Routes → Edit routes → Add route
    Destination: 10.0.0.0/16
    Target: Peering Connection → lab-peering-01
  Save changes
```

**Paso 1.5 — Verificar conectividad**
```bash
# Lanzar una EC2 en cada VPC
# Desde instancia en VPC-01, hacer ping a instancia en VPC-02:
ping 10.1.1.X

# Si falla, verificar Security Groups:
# La SG de la instancia en VPC-02 debe permitir ICMP desde 10.0.0.0/16
# Inbound: All ICMP - IPv4, Source: 10.0.0.0/16
```

---

### PARTE 2 — VPC Gateway Endpoint para S3

Permite acceder a S3 desde subnets privadas **sin pasar por internet ni NAT Gateway**.

**Paso 2.1 — Crear el Endpoint**
```
VPC → "Endpoints" → "Create endpoint"

  Name:              lab-endpoint-s3
  Service category:  AWS services
  
  Services: escribir "s3" en la búsqueda
    Seleccionar: "com.amazonaws.us-east-1.s3"  tipo "Gateway"
    (Hay dos opciones s3: elegir la de tipo Gateway, no Interface)
  
  VPC: lab-vpc-01
  
  Route tables: marcar "lab-rtb-privada"
  
  Policy: Full Access

Clic "Create endpoint"
```

**Paso 2.2 — Verificar ruta automática**
```
VPC → Route Tables → "lab-rtb-privada"
Pestaña "Routes"

Deberías ver una ruta nueva añadida automáticamente:
  Destination: pl-xxxxxxxx (prefix list de S3)
  Target:      vpce-xxxxxxxxxx  ← el Gateway Endpoint
```

**Paso 2.3 — Probar desde instancia en subnet privada**
```bash
# Conectar a instancia EC2 en subnet privada
# (usar EC2 Instance Connect o bastion host)

# El tráfico a S3 pasa por el endpoint, no por internet
aws s3 ls                           # listar buckets ✓
aws s3 cp archivo.txt s3://mi-bucket/  # subir archivo ✓

# Para confirmar que usa el endpoint (no internet):
# Deshabilitar el NAT Gateway y verificar que S3 sigue funcionando
```

---

### PARTE 3 — Route 53 Resolver Outbound Endpoint

Permite resolver DNS de servidores **on-premise** desde instancias AWS.

**Paso 3.1 — Crear Security Group para el Resolver**
```
EC2 → Security Groups → "Create security group"
  Name: sg-resolver-outbound
  VPC:  lab-vpc-01

  Inbound rules: (ninguna)

  Outbound rules:
    Type: DNS (UDP), Port: 53, Destination: Custom → 192.168.1.0/24
    Type: DNS (TCP), Port: 53, Destination: Custom → 192.168.1.0/24
    (192.168.1.0/24 = rango IP de tu red on-premise)
```

**Paso 3.2 — Crear el Endpoint**
```
Route 53 → Panel izquierdo → "Resolver" → "Outbound endpoints"
Clic "Create outbound endpoint"

  Name:            lab-resolver-outbound
  VPC:             lab-vpc-01
  Security group:  sg-resolver-outbound

  IP addresses:
    AZ 1: us-east-1a → Subnet: lab-subnet-privada-1a → Use auto-assigned IP
    AZ 2: us-east-1b → Subnet: (crear subnet privada 1b antes si no existe) → auto

Clic "Submit"
⏳ Esperar ~2 minutos hasta estado "Operational"
```

**Paso 3.3 — Crear regla de reenvío**
```
Route 53 → Resolver → "Rules" → "Create rule"

  Name:         regla-dns-onpremise
  Rule type:    Forward
  Domain name:  empresa.local

  VPCs to associate: lab-vpc-01

  Target IP addresses:
    IP: 192.168.1.53   Port: 53   Protocol: UDP
    (IP del servidor DNS on-premise)

Clic "Save"
```

**Resultado:** Desde cualquier EC2 en `lab-vpc-01`, las consultas DNS a `*.empresa.local` se envían al DNS on-premise.

---

## 🧹 LIMPIEZA (en este orden para evitar errores)

```
1.  Route 53 → Resolver → Rules → eliminar "regla-dns-onpremise"
2.  Route 53 → Resolver → Outbound endpoints → eliminar "lab-resolver-outbound"
3.  Route 53 → Hosted zones → eliminar todos los registros → eliminar la zona
4.  VPC → Endpoints → eliminar "lab-endpoint-s3"
5.  VPC → Peering Connections → eliminar "lab-peering-01"
6.  VPC → NAT Gateways → eliminar "lab-nat-01"
    ⏳ Esperar que el estado sea "Deleted" antes de continuar
7.  VPC → Elastic IPs → "Release" la IP asignada al NAT
8.  VPC → Subnets → eliminar todas las subnets creadas
9.  VPC → Internet Gateways → Detach → Delete "lab-igw-01"
10. VPC → Route Tables → eliminar lab-rtb-publica y lab-rtb-privada
11. VPC → Your VPCs → eliminar lab-vpc-01 y lab-vpc-02
12. IAM → Roles → eliminar "rol-ec2-acceso-s3"
13. IAM → Policies → eliminar "pol-s3-bucket-especifico"
14. IAM → Users → eliminar "lab-usuario-01"
```

---

## 📌 COMANDOS CLI DE REFERENCIA RÁPIDA

```bash
# Listar VPCs
aws ec2 describe-vpcs \
  --query 'Vpcs[*].[VpcId,CidrBlock,Tags[?Key==`Name`].Value|[0]]' \
  --output table

# Listar subnets de una VPC
aws ec2 describe-subnets \
  --filters "Name=vpc-id,Values=vpc-xxxxxxxxxx" \
  --query 'Subnets[*].[SubnetId,CidrBlock,AvailabilityZone,Tags[?Key==`Name`].Value|[0]]' \
  --output table

# Listar usuarios IAM
aws iam list-users \
  --query 'Users[*].[UserName,CreateDate]' \
  --output table

# Verificar políticas de un usuario
aws iam list-attached-user-policies --user-name lab-usuario-01

# Ver estado de NAT Gateways
aws ec2 describe-nat-gateways \
  --query 'NatGateways[*].[NatGatewayId,State,SubnetId]' \
  --output table
```
