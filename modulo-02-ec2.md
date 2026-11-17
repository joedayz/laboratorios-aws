# Módulo 2 — Cómputo EC2
### Servicios: EC2 · Security Groups · AMI · Auto Scaling Group · ALB

---

## TEORÍA

### EC2 — Elastic Compute Cloud
Servidores virtuales en la nube. Pagas por segundo de uso. Tú eliges CPU, RAM, disco y sistema operativo.

### Tipos de instancias más usados
| Familia | Uso | Ejemplo |
|---------|-----|---------|
| t3/t4g | Propósito general económica (burstable CPU) | Dev, web servers |
| m5/m6i | Balance CPU/RAM | Aplicaciones backend |
| c5/c6i | Alta CPU | Procesamiento, APIs |
| r5/r6i | Alta RAM | Bases de datos en memoria |

### AMI — Amazon Machine Image
Plantilla de tu instancia: SO + software + configuraciones. Puedes crear tu propia AMI desde una instancia configurada para reutilizarla.

### Security Groups
Firewall virtual de la instancia. Es **stateful**: si permites el ingreso, la respuesta sale automáticamente. Solo existen reglas **Allow** (sin Deny explícito). Por defecto: todo ingreso bloqueado, todo egreso permitido.

### Auto Scaling Group (ASG)
Ajusta automáticamente la cantidad de instancias según demanda. Define: mínimo, deseado y máximo.

### Application Load Balancer (ALB)
Distribuye tráfico HTTP/HTTPS entre múltiples instancias. Opera en capa 7 (puede enrutar por path, headers, etc).

---

## 🟢 LAB BÁSICO — Lanzar EC2 con Apache y acceder desde browser

**Tiempo estimado: 30 min**

---

### Paso 1 — Crear Key Pair para SSH

```
Consola AWS → EC2
Panel izquierdo → "Network & Security" → "Key Pairs"
Clic "Create key pair"

  Name:   lab-key-ec2
  Type:   RSA
  Format: .pem  (macOS/Linux)

Clic "Create key pair"
✅ El archivo .pem se descarga automáticamente al navegador
```

```bash
# Cambiar permisos del archivo (obligatorio para que SSH funcione)
chmod 400 ~/Downloads/lab-key-ec2.pem
```

---

### Paso 2 — Crear Security Group

```
EC2 → Panel izquierdo → "Security Groups" → "Create security group"

  Security group name: sg-web-lab
  Description:         Acceso SSH y HTTP para laboratorio
  VPC:                 lab-vpc-01  (o default si no tienes la del módulo 1)

  Inbound rules — clic "Add rule" para cada una:
    Regla 1:
      Type:   SSH
      Port:   22
      Source: My IP   ← solo tu IP puede conectarse
    
    Regla 2:
      Type:   HTTP
      Port:   80
      Source: Anywhere-IPv4   (0.0.0.0/0)

  Outbound rules: dejar las reglas por defecto (All traffic, 0.0.0.0/0)

Clic "Create security group"
```

---

### Paso 3 — Lanzar instancia EC2

```
EC2 → "Instances" → "Launch instances"

Name: lab-web-01

--- Application and OS Images ---
  Quick Start → Amazon Linux
  AMI: "Amazon Linux 2023 AMI"  ← buscar esta opción (Free tier eligible)
  Architecture: 64-bit (x86)

--- Instance type ---
  t2.micro o t3.micro  (Free tier eligible)

--- Key pair ---
  Seleccionar: lab-key-ec2

--- Network settings → clic "Edit" ---
  VPC:                     lab-vpc-01
  Subnet:                  lab-subnet-publica-1a
  Auto-assign public IP:   Enable  ← IMPORTANTE

  Firewall (security groups):
    Select existing security group
    Seleccionar: sg-web-lab

--- Configure storage ---
  8 GiB, gp3  (dejar por defecto)
```

**Opción A — Configurar automáticamente con User Data**

En **Advanced details → User data**, pega el siguiente script:

```bash
#!/bin/bash
# User Data: se ejecuta como root al iniciar la instancia por primera vez
set -e
yum update -y
yum install httpd -y
systemctl start httpd
systemctl enable httpd

# Crear página HTML con info del servidor
cat > /var/www/html/index.html << 'EOF'
<!DOCTYPE html>
<html lang="es">
<head>
  <meta charset="UTF-8">
  <title>Servidor EC2</title>
  <style>
    body { font-family: Arial, sans-serif; text-align: center; padding: 60px; background: #f4f4f4; }
    .box { background: white; padding: 30px; border-radius: 10px; display: inline-block; box-shadow: 0 2px 10px rgba(0,0,0,0.1); }
    h1 { color: #232F3E; }
    .info { background: #FF9900; color: white; padding: 10px; border-radius: 5px; margin: 10px 0; }
  </style>
</head>
<body>
  <div class="box">
    <h1>✅ Servidor EC2 en AWS</h1>
    <div class="info">Hostname: HOSTNAME_VAR</div>
    <div class="info">IP Privada: IPPRIV_VAR</div>
    <div class="info">AZ: AZ_VAR</div>
  </div>
</body>
</html>
EOF

# Reemplazar variables con valores reales del servidor
HOSTNAME=$(hostname)
IP_PRIV=$(hostname -I | awk '{print $1}')
TOKEN=$(curl -fsS -X PUT http://169.254.169.254/latest/api/token \
  -H "X-aws-ec2-metadata-token-ttl-seconds: 21600")
AZ=$(curl -fsS -H "X-aws-ec2-metadata-token: $TOKEN" \
  http://169.254.169.254/latest/meta-data/placement/availability-zone)

sed -i "s/HOSTNAME_VAR/$HOSTNAME/g" /var/www/html/index.html
sed -i "s/IPPRIV_VAR/$IP_PRIV/g"   /var/www/html/index.html
sed -i "s/AZ_VAR/$AZ/g"            /var/www/html/index.html
```

**Opción B — Instalar manualmente por SSH:** deja User Data vacío y sigue las
instrucciones del paso 4. También puedes usar esta opción si ya lanzaste la
instancia y olvidaste configurar User Data.

```
Clic "Launch instance"
⏳ Esperar 2-3 minutos
  EC2 → Instances → "lab-web-01"
  Esperar: Instance state = "Running" y Status checks = "2/2 checks passed"
```

---

### Paso 4 — Conectarse por SSH

```
EC2 → Instances → seleccionar "lab-web-01"
Copiar el valor de "Public IPv4 address"  ej: 54.123.45.67
```

```bash
# Desde tu terminal macOS
ssh -i ~/Downloads/lab-key-ec2.pem ec2-user@54.123.45.67

# Primera vez: pregunta confirmación de fingerprint
# Are you sure you want to continue connecting (yes/no)? yes

# Deberías ver el prompt:
#   [ec2-user@ip-10-0-1-xx ~]$
```

**Opción B — Instalar Apache manualmente:** si no usaste User Data, ejecuta
este bloque completo **dentro de la instancia**, después de conectarte por SSH:

```bash
sudo bash <<'INSTALL'
set -e
yum update -y
yum install httpd -y
systemctl start httpd
systemctl enable httpd

cat > /var/www/html/index.html <<'EOF'
<!DOCTYPE html>
<html lang="es">
<head>
  <meta charset="UTF-8">
  <title>Servidor EC2</title>
  <style>
    body { font-family: Arial, sans-serif; text-align: center; padding: 60px; background: #f4f4f4; }
    .box { background: white; padding: 30px; border-radius: 10px; display: inline-block; box-shadow: 0 2px 10px rgba(0,0,0,0.1); }
    h1 { color: #232F3E; }
    .info { background: #FF9900; color: white; padding: 10px; border-radius: 5px; margin: 10px 0; }
  </style>
</head>
<body>
  <div class="box">
    <h1>✅ Servidor EC2 en AWS</h1>
    <div class="info">Hostname: HOSTNAME_VAR</div>
    <div class="info">IP Privada: IPPRIV_VAR</div>
    <div class="info">AZ: AZ_VAR</div>
  </div>
</body>
</html>
EOF

HOSTNAME=$(hostname)
IP_PRIV=$(hostname -I | awk '{print $1}')
TOKEN=$(curl -fsS -X PUT http://169.254.169.254/latest/api/token \
  -H "X-aws-ec2-metadata-token-ttl-seconds: 21600")
AZ=$(curl -fsS -H "X-aws-ec2-metadata-token: $TOKEN" \
  http://169.254.169.254/latest/meta-data/placement/availability-zone)

sed -i "s/HOSTNAME_VAR/$HOSTNAME/g" /var/www/html/index.html
sed -i "s/IPPRIV_VAR/$IP_PRIV/g"   /var/www/html/index.html
sed -i "s/AZ_VAR/$AZ/g"            /var/www/html/index.html
INSTALL
```

Luego, con cualquiera de las dos opciones, verifica desde la instancia:

```bash
# Verificar que Apache está corriendo
sudo systemctl status httpd
# Active: active (running) ✅

# Ver el contenido de la página
cat /var/www/html/index.html

# Salir de la instancia
exit
```

---

### Paso 5 — Verificar desde el navegador

```
Abrir en el navegador:
  http://54.123.45.67   ← IP pública de tu instancia

✅ Deberías ver la página con el hostname, IP privada y AZ del servidor
```

---

## 🟡 LAB INTERMEDIO — AMI + Launch Template + Auto Scaling + ALB

**Tiempo estimado: 75 min**

---

### Paso 1 — Crear AMI desde la instancia configurada

```
EC2 → Instances → seleccionar "lab-web-01"
Menú "Actions" → "Image and templates" → "Create image"

  Image name:        ami-web-apache-v1
  Image description: Amazon Linux 2023 con Apache y pagina de inicio
  No reboot:         DESMARCAR  (queremos snapshot limpio)

Clic "Create image"
⏳ Esperar 3-5 minutos
  EC2 → "AMIs" (panel izquierdo) → estado cambia de "pending" a "available"
```

---

### Paso 2 — Seleccionar dos subnets públicas para el ALB

```
El ALB necesita dos subnets públicas en distintas Zonas de Disponibilidad,
dentro de la misma VPC de la instancia "lab-web-01".

EC2 → Instances → seleccionar "lab-web-01"
  Anotar su VPC ID y Subnet ID.

VPC → "Subnets"
  Filtrar por el VPC ID anotado.
  Seleccionar la subnet de la instancia y otra subnet de una AZ distinta.
  Consultar la columna "Availability Zone" o los detalles de cada subnet.
  Anotar los dos Subnet ID para los pasos 6 y 7.

Para cada subnet, abrir la pestaña "Route table":
  Verificar que exista una ruta 0.0.0.0/0 hacia un Internet Gateway (igw-...).
  Puede ser la tabla principal de la VPC, aunque no haya asociación explícita.

Si usas la VPC default y ya tienes dos subnets que cumplen estas condiciones,
no necesitas crear otra VPC ni otra subnet.
```

Si falta una segunda subnet, créala en la misma VPC y en otra AZ, con un CIDR
disponible dentro de la VPC que no se solape con las demás subnets:

```
VPC → "Subnets" → "Create subnet"
  Elegir la VPC, una AZ diferente y un CIDR disponible.
Clic "Create subnet"

VPC → "Route Tables" → seleccionar la tabla de rutas pública de esa VPC
  "Subnet associations" → "Edit subnet associations"
  Marcar también la nueva subnet → "Save associations"
```

Para `lab-vpc-01` del módulo 1, puedes usar `10.0.3.0/24` si está disponible
y asociarla a `lab-rtb-publica`.

---

### Paso 3 — Crear Security Group para el ALB

```
EC2 → Security Groups → "Create security group"
  Name:        sg-alb-lab
  Description: Acceso HTTP al Load Balancer
  VPC:         la misma VPC de "lab-web-01" (default o lab-vpc-01)

  Inbound rules:
    Type: HTTP, Port: 80, Source: Anywhere-IPv4 (0.0.0.0/0)
  
  Outbound rules: All traffic, 0.0.0.0/0

Clic "Create security group"
```

```
Actualizar sg-web-lab para aceptar tráfico SOLO desde el ALB:
EC2 → Security Groups → "sg-web-lab"
Pestaña "Inbound rules" → "Edit inbound rules"
  Agregar regla:
    Type: HTTP, Port: 80, Source: Custom → seleccionar "sg-alb-lab"
  (Opcional: eliminar la regla HTTP 0.0.0.0/0 anterior)
Clic "Save rules"
```

---

### Paso 4 — Crear Launch Template

```
EC2 → "Launch Templates" → "Create launch template"

  Launch template name:    lt-web-app-v1
  Template version desc:   Apache con pagina de inicio personalizada
  Auto Scaling guidance:   ✅ Marcar checkbox

  Application and OS Images:
    My AMIs → Owned by me → seleccionar "ami-web-apache-v1"

  Instance type: t3.micro

  Key pair: lab-key-ec2

  Network settings:
    Security groups: sg-web-lab
    (NO especificar subnet aquí — el ASG la asigna)

Clic "Create launch template"
```

---

### Paso 5 — Crear Target Group

```
EC2 → "Target Groups" → "Create target group"

  Target type:      Instances
  Target group name: tg-web-lab
  Protocol:          HTTP
  Port:              80
  VPC:               la misma VPC de "lab-web-01"

  Health check settings:
    Protocol:                HTTP
    Path:                    /
    Healthy threshold:       2
    Unhealthy threshold:     3
    Timeout:                 5 seconds
    Interval:                30 seconds
    Success codes:           200

Clic "Next"
  (No agregar instancias — el ASG las registra automáticamente)
Clic "Create target group"
```

---

### Paso 6 — Crear Application Load Balancer

```
EC2 → "Load Balancers" → "Create load balancer"
Seleccionar: "Application Load Balancer" → Clic "Create"

  Load balancer name: alb-web-lab
  Scheme:             Internet-facing
  IP address type:    IPv4

  Network mapping:
    VPC: la misma VPC de "lab-web-01"
    Availability Zones:
      Seleccionar las dos subnets de distintas AZ anotadas en el paso 2.

  Security groups:
    Remover el grupo default
    Agregar: sg-alb-lab

  Listeners and routing:
    Protocol: HTTP  Port: 80
    Default action: Forward to → tg-web-lab

Clic "Create load balancer"
⏳ Esperar ~2-3 minutos hasta estado "Active"

Anotar el DNS name del ALB:
  alb-web-lab-xxxxxxxxxx.us-east-1.elb.amazonaws.com
```

---

### Paso 7 — Crear Auto Scaling Group

```
EC2 → "Auto Scaling Groups" → "Create Auto Scaling group"

--- Step 1: Choose launch template ---
  Auto Scaling group name: asg-web-lab
  Launch template: lt-web-app-v1
  Version: Default (1)
Clic "Next"

--- Step 2: Choose instance launch options ---
  VPC: la misma VPC de "lab-web-01"
  Availability Zones and subnets:
    Seleccionar las dos subnets anotadas en el paso 2.
Clic "Next"

--- Step 3: Configure advanced options ---
  Load balancing: Attach to an existing load balancer
  Choose from your load balancer target groups
  Target groups: tg-web-lab

  Health checks:
    ✅ Turn on Elastic Load Balancing health checks
    Health check grace period: 300 seconds

  Additional settings:
    ✅ Enable group metrics collection within CloudWatch
Clic "Next"

--- Step 4: Configure group size and scaling ---
  Group size:
    Desired capacity: 2
    Minimum capacity: 1
    Maximum capacity: 4

  Automatic scaling:
    ✅ Target tracking scaling policy
    Scaling policy name: politica-cpu-70
    Metric type:         Average CPU utilization
    Target value:        70
    Instance warmup:     300 seconds

Clic "Next" → "Next" → "Create Auto Scaling group"
```

---

### Paso 8 — Verificar funcionamiento

```
EC2 → "Auto Scaling Groups" → "asg-web-lab"
Pestaña "Activity" → Ver el historial (2 instancias lanzándose)
Pestaña "Instance management" → Esperar que ambas sean "InService"
```

```bash
# Verificar balanceo de carga — recargar varias veces
# Desde terminal:
for i in {1..6}; do
  curl -s http://alb-web-lab-xxxxxxxxxx.us-east-1.elb.amazonaws.com | grep "Hostname"
done
# Deberías ver diferentes hostnames alternando (distintas instancias)
```

---

### Paso 9 — Probar escalado con carga de CPU

```bash
# Conectarse a una instancia del ASG via SSH o Instance Connect
# En la instancia:

# Instalar herramienta de stress
sudo yum install stress -y

# Generar carga alta de CPU por 5 minutos
stress --cpu 4 --timeout 300 &

# En la consola AWS, monitorear:
# EC2 → Auto Scaling Groups → asg-web-lab → Pestaña "Activity"
# En ~5 min verás: "Launching a new EC2 instance: i-xxxxxxxx"
# porque la CPU superó el 70% configurado
```

---

## 🔴 LAB AVANZADO — Spot Fleet + Placement Groups + EC2 Instance Connect Endpoint

**Tiempo estimado: 60 min**

---

### Parte 1 — Spot Fleet

```
EC2 → "Spot Requests" → "Request Spot Instances"

  Request type:        Request and Maintain
  Target capacity:     2  (instancias)
  
  Launch parameters:
    AMI: ami-web-apache-v1
    
    Instance types: clic "Select instance types"
      Agregar: t3.micro, t3.small, t3.medium
      (Spot Fleet elige el más barato disponible en ese momento)
    
    Key pair: lab-key-ec2
    Security groups: sg-web-lab
  
  Network:
    VPC: lab-vpc-01
    Subnets: seleccionar ambas subnets públicas
  
  Allocation strategy: Lowest price
  
  Fleet request settings:
    Maintain target capacity: ✅
    Interruption behavior: Terminate

Clic "Launch"
```

```bash
# Comparar precios Spot vs On-Demand desde CLI
aws ec2 describe-spot-price-history \
  --instance-types t3.micro t3.small \
  --product-descriptions "Linux/UNIX" \
  --region us-east-1 \
  --start-time $(date -u -v-1H +%Y-%m-%dT%H:%M:%S) \
  --query 'SpotPriceHistory[*].[Timestamp,InstanceType,SpotPrice,AvailabilityZone]' \
  --output table

# Precio On-Demand t3.micro: $0.0104/hora
# Precio Spot t3.micro:     ~$0.0031/hora  (~70% ahorro)
```

---

### Parte 2 — Placement Groups

**Paso 2.1 — Crear Placement Group Spread**
```
EC2 → Panel izquierdo → "Placement Groups" → "Create placement group"

  Name:         pg-spread-lab
  Strategy:     Spread
  Spread level: Rack

Clic "Create group"
```

**Paso 2.2 — Lanzar instancias en el grupo**
```
EC2 → "Launch instances"
  Name: lab-spread-01
  AMI:  Amazon Linux 2023
  Instance type: t3.micro
  Key pair: lab-key-ec2
  Security group: sg-web-lab
  Number of instances: 2

  Advanced details:
    Placement group name: pg-spread-lab

Clic "Launch instance"
```

```
✅ Las dos instancias quedarán en racks físicos distintos
   Si un rack tiene fallo de hardware, solo afecta una instancia
```

---

### Parte 3 — EC2 Instance Connect Endpoint

Conexión SSH a instancias **sin IP pública, sin bastion host, sin VPN**.

**Paso 3.1 — Security Group para el Endpoint**
```
EC2 → Security Groups → "Create security group"
  Name: sg-eic-endpoint
  VPC:  lab-vpc-01

  Inbound: (ninguna regla)
  Outbound:
    Type: SSH, Port: 22, Destination: Custom → sg-web-lab

Clic "Create security group"
```

**Paso 3.2 — Crear el Endpoint**
```
VPC → "Endpoints" → "Create endpoint"

  Name:              lab-eic-01
  Service category:  EC2 Instance Connect Endpoint
  VPC:               lab-vpc-01
  Security group:    sg-eic-endpoint
  Subnet:            lab-subnet-privada-1a

Clic "Create endpoint"
⏳ Esperar ~2 minutos hasta estado "Available"
```

**Paso 3.3 — Actualizar SG de las instancias**
```
EC2 → Security Groups → sg-web-lab
Inbound rules → "Edit inbound rules"

Modificar regla SSH:
  Antes: SSH, Source: My IP
  Ahora: SSH, Source: Custom → sg-eic-endpoint

Clic "Save rules"
```

**Paso 3.4 — Conectarse sin IP pública**
```bash
# Opción 1: Desde la consola AWS
# EC2 → Instances → seleccionar instancia → "Connect"
# Pestaña "EC2 Instance Connect" → "Connect"
# Se abre una terminal en el browser ✅

# Opción 2: Desde terminal (requiere permisos ec2-instance-connect)
aws ec2-instance-connect ssh \
  --instance-id i-xxxxxxxxxxxxxxxxx \
  --region us-east-1

# ✅ Conexión establecida sin IP pública ni llave .pem expuesta
```

---

## 🧹 LIMPIEZA (en este orden)

```
1.  EC2 → Auto Scaling Groups → "asg-web-lab" → Delete
    (esto termina las instancias del ASG automáticamente)

2.  EC2 → Load Balancers → "alb-web-lab" → Delete load balancer

3.  EC2 → Target Groups → "tg-web-lab" → Delete

4.  EC2 → Spot Requests → seleccionar → Actions → "Cancel request"
    EC2 → Instances → terminar las instancias Spot

5.  EC2 → Launch Templates → "lt-web-app-v1" → Actions → Delete

6.  EC2 → Instances → terminar "lab-web-01" y "lab-spread-01" y cualquier otra

7.  EC2 → Placement Groups → "pg-spread-lab" → Delete

8.  VPC → Endpoints → "lab-eic-01" → Delete

9.  EC2 → AMIs → "ami-web-apache-v1" → Actions → Deregister AMI
    EC2 → Snapshots → eliminar el snapshot creado por la AMI

10. EC2 → Security Groups → eliminar sg-alb-lab, sg-eic-endpoint
    (mantener sg-web-lab si lo usas en módulos siguientes)
```

---

## 📌 COMANDOS CLI DE REFERENCIA RÁPIDA

```bash
# Ver instancias en ejecución con sus IPs
aws ec2 describe-instances \
  --filters "Name=instance-state-name,Values=running" \
  --query 'Reservations[*].Instances[*].[InstanceId,InstanceType,PublicIpAddress,PrivateIpAddress,Tags[?Key==`Name`].Value|[0]]' \
  --output table

# Ver estado del Auto Scaling Group
aws autoscaling describe-auto-scaling-groups \
  --auto-scaling-group-names asg-web-lab \
  --query 'AutoScalingGroups[*].[AutoScalingGroupName,DesiredCapacity,MinSize,MaxSize]' \
  --output table

# Cambiar la capacidad deseada manualmente
aws autoscaling set-desired-capacity \
  --auto-scaling-group-name asg-web-lab \
  --desired-capacity 3

# Terminar instancia específica
aws ec2 terminate-instances --instance-ids i-xxxxxxxxxxxxxxxxx

# Ver AMIs propias
aws ec2 describe-images --owners self \
  --query 'Images[*].[ImageId,Name,CreationDate]' \
  --output table

# Ver historial de actividad del ASG
aws autoscaling describe-scaling-activities \
  --auto-scaling-group-name asg-web-lab \
  --max-items 10 \
  --output table
```
