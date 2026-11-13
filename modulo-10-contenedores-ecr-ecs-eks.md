# Módulo 10 — Contenedores en AWS
### Servicios: ECR · ECS (Fargate) · EKS

---

## TEORÍA

### ¿Por qué contenedores?
Un contenedor empaqueta la aplicación con todas sus dependencias en una unidad portable. Funciona igual en tu laptop, en staging y en producción. Docker es la tecnología de contenedores más usada.

### ECR — Elastic Container Registry
Registro privado de imágenes Docker administrado por AWS. Integrado con IAM para control de acceso. Escanea vulnerabilidades automáticamente al hacer push.

**Componentes:**
- **Registry**: tu cuenta tiene un registry privado por región
- **Repository**: almacena las imágenes de una aplicación (como un repo de Git pero para imágenes)
- **Image**: versión específica de tu app empaquetada
- **Tag**: etiqueta de la imagen (`latest`, `1.0.0`, commit SHA)

### ECS — Elastic Container Service
Orquestador de contenedores administrado por AWS. Tú defines qué correr (Task Definition) y AWS se encarga del resto.

**Conceptos clave:**
- **Task Definition**: "receta" de tu contenedor — imagen, CPU, RAM, puertos, variables de entorno, volúmenes
- **Task**: instancia en ejecución de una Task Definition (es el contenedor corriendo)
- **Service**: mantiene N tasks corriendo, las reemplaza si fallan, se integra con ALB
- **Cluster**: agrupación lógica de tasks y services

**Modos de lanzamiento:**
- **Fargate**: sin servidores — AWS gestiona la infraestructura. Pagas por CPU/RAM usada por el contenedor
- **EC2**: tú gestionas las instancias EC2 del cluster. Más control, más trabajo

### EKS — Elastic Kubernetes Service
Kubernetes administrado. AWS gestiona el **control plane** (master nodes). Tú gestionas los **worker nodes** (EC2 o Fargate).

**Conceptos Kubernetes:**
- **Pod**: unidad mínima — uno o más contenedores que comparten red y almacenamiento
- **Deployment**: gestiona replicas de Pods, hace rolling updates
- **Service**: expone los Pods internamente o externamente (LoadBalancer)
- **Namespace**: aislamiento lógico dentro del cluster
- **Node**: máquina (EC2) donde corren los Pods
- **kubectl**: CLI para interactuar con Kubernetes

**ECS vs EKS:**
| | ECS | EKS |
|---|-----|-----|
| Curva de aprendizaje | Baja | Alta |
| Control | Menor | Mayor |
| Ecosistema | AWS nativo | Kubernetes estándar |
| Costo | Fargate: por uso | $0.10/hora control plane + workers |
| Mejor para | Apps simples, equipos pequeños | Apps complejas, múltiples equipos |

---

## 🐳 PODMAN — Alternativa a Docker sin daemon

### ¿Qué es Podman?
Podman es una herramienta de contenedores **daemonless** (sin proceso de fondo) y **rootless** (sin necesidad de root). Es compatible con Docker a nivel de comandos — en la mayoría de casos solo cambias `docker` por `podman`.

**Diferencias clave vs Docker:**
| | Docker | Podman |
|---|--------|--------|
| Daemon | Requiere dockerd corriendo | No necesita daemon |
| Root | Requiere root (o grupo docker) | Puede correr sin root |
| Compatibilidad | Estándar de facto | Compatible con Docker CLI |
| Compose | docker-compose | podman-compose o podman play kube |
| Instalación macOS | Docker Desktop (pago en empresas) | Libre y gratuito |

### Instalar Podman en macOS

```bash
# Con Homebrew
brew install podman

# Inicializar la máquina virtual de Podman (necesaria en macOS)
podman machine init
podman machine start

# Verificar instalación
podman --version
# podman version 5.x.x

podman info | grep -E "version|os|arch"
```

### Alias para usar Podman como Docker

```bash
# Agregar a tu ~/.zshrc o ~/.bash_profile para usar "docker" y que llame a podman
echo 'alias docker=podman' >> ~/.zshrc
source ~/.zshrc

# Verificar que el alias funciona
docker --version
# podman version 5.x.x  (responde podman pero con el alias)
```

---

### PODMAN — Comandos equivalentes a Docker

Todos los comandos del lab funcionan igual con `podman`. Esta tabla es la referencia rápida:

| Operación | Docker | Podman |
|-----------|--------|--------|
| Construir imagen | `docker build -t nombre .` | `podman build -t nombre .` |
| Listar imágenes | `docker images` | `podman images` |
| Correr contenedor | `docker run -d -p 3000:3000 img` | `podman run -d -p 3000:3000 img` |
| Listar contenedores | `docker ps` | `podman ps` |
| Ver logs | `docker logs ID` | `podman logs ID` |
| Shell en contenedor | `docker exec -it ID sh` | `podman exec -it ID sh` |
| Detener | `docker stop ID` | `podman stop ID` |
| Eliminar | `docker rm ID` | `podman rm ID` |
| Etiquetar imagen | `docker tag img:tag dest:tag` | `podman tag img:tag dest:tag` |
| Push a registry | `docker push img` | `podman push img` |
| Pull de registry | `docker pull img` | `podman pull img` |
| Login en ECR | `docker login ...` | `podman login ...` |
| Eliminar imagen | `docker rmi img` | `podman rmi img` |
| Ver info del sistema | `docker info` | `podman info` |
| Inspeccionar | `docker inspect ID` | `podman inspect ID` |
| Copiar archivos | `docker cp ID:/path ./` | `podman cp ID:/path ./` |

---

### PODMAN — Flujo completo con ECR

```bash
# Variables — definir con comillas para evitar caracteres invisibles
AWS_REGION="us-east-1"
AWS_ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text | tr -d '[:space:]')
ECR_URI="${AWS_ACCOUNT_ID}.dkr.ecr.${AWS_REGION}.amazonaws.com"
REPO_NAME="lab-contenedor"

# ⚠️  VERIFICAR antes de continuar — deben verse exactamente así, sin espacios ni caracteres extra
echo "Account ID : '${AWS_ACCOUNT_ID}'"
echo "ECR URI    : '${ECR_URI}'"
echo "Repo Name  : '${REPO_NAME}'"
echo "Full URI   : '${ECR_URI}/${REPO_NAME}'"
# Si algún valor se ve raro (doble nombre, caracteres extraños) → redefinir manualmente:
# export AWS_ACCOUNT_ID="123456789012"
# export ECR_URI="123456789012.dkr.ecr.us-east-1.amazonaws.com"
# export REPO_NAME="lab-contenedor"

# PASO 1 — Construir la imagen
podman build -t lab-contenedor:1.0.0 .

# PASO 2 — Probar localmente
podman run -d \
  --name mi-app-local \
  -p 3000:3000 \
  -e APP_VERSION=1.0.0 \
  lab-contenedor:1.0.0

curl http://localhost:3000/health
# {"status":"healthy"} ✅

podman stop mi-app-local
podman rm mi-app-local

# PASO 3 — Login en ECR con Podman
# ⚠️  Diferencia: Podman usa --tls-verify=false en algunos entornos corporativos
aws ecr get-login-password --region $AWS_REGION | \
  podman login \
    --username AWS \
    --password-stdin \
    $ECR_URI
# Login Succeeded! ✅

# PASO 4 — Etiquetar y hacer push a ECR
# ⚠️  Usar la URI completa hardcodeada en el tag para evitar errores de variables
# Reemplazar 123456789012 con tu AWS Account ID real (el que mostraron los echo del paso anterior)
FULL_URI="${ECR_URI}/${REPO_NAME}"

podman tag lab-contenedor:1.0.0 ${FULL_URI}:1.0.0
podman tag lab-contenedor:1.0.0 ${FULL_URI}:latest

# Verificar ANTES de hacer push — ambas líneas deben mostrar "lab-contenedor" sin nada extra
podman images | grep "${AWS_ACCOUNT_ID}"
# Salida esperada:
# xxx.dkr.ecr.us-east-1.amazonaws.com/lab-contenedor  1.0.0   sha256:xxx ...
# xxx.dkr.ecr.us-east-1.amazonaws.com/lab-contenedor  latest  sha256:xxx ...

podman push ${FULL_URI}:1.0.0
podman push $ECR_URI/$REPO_NAME:latest

# Verificar en ECR
aws ecr list-images \
  --repository-name $REPO_NAME \
  --query 'imageIds[*].[imageTag]' \
  --output table
```

---

### PODMAN — Características exclusivas

```bash
# Correr como rootless (sin sudo, más seguro)
podman run --userns=keep-id -d -p 3000:3000 lab-contenedor:1.0.0

# Ver contenedores de TODOS los usuarios del sistema
podman ps --all --external

# Generar un archivo Kubernetes YAML desde un contenedor corriendo
# (muy útil para migrar de Docker a Kubernetes)
podman generate kube mi-app-local > mi-app-pod.yaml
cat mi-app-pod.yaml
# Genera un Pod Spec de Kubernetes listo para usar con kubectl apply

# Correr un Pod Kubernetes directamente con Podman (sin k8s)
podman play kube mi-app-pod.yaml

# Podman Compose (alternativa a docker-compose)
# Instalar:
pip3 install podman-compose

# Crear docker-compose.yml de ejemplo:
cat > docker-compose.yml << 'EOF'
version: "3"
services:
  app:
    build: .
    ports:
      - "3000:3000"
    environment:
      - APP_VERSION=1.0.0
      - NODE_ENV=development
  redis:
    image: redis:alpine
    ports:
      - "6379:6379"
EOF

# Levantar con podman-compose (compatible con docker-compose.yml)
podman-compose up -d
podman-compose ps
podman-compose logs app
podman-compose down

# Ver sistema de almacenamiento de Podman
podman system df
# TYPE          TOTAL   ACTIVE  SIZE    RECLAIMABLE
# Images        3       1       450MB   300MB (66%)
# Containers    1       0       0B      0B
# Local Volumes 0       0       0B      0B

# Limpiar recursos no utilizados
podman system prune -f

# Gestión de la máquina virtual (macOS)
podman machine list     # listar VMs de Podman
podman machine stop     # detener la VM
podman machine start    # iniciar la VM
podman machine ssh      # conectar por SSH a la VM
podman machine rm       # eliminar la VM
```

---

### PODMAN — Multi-plataforma (build para Linux en Mac ARM)

```bash
# Si tu Mac es Apple Silicon (M1/M2/M3) y ECS/EKS usa x86_64
# debes construir la imagen para la plataforma correcta

# Ver plataforma de tu Mac
uname -m
# arm64  (Apple Silicon) o x86_64 (Intel)

# Construir imagen para plataforma específica con Podman
podman build \
  --platform linux/amd64 \
  -t $ECR_URI/$REPO_NAME:1.0.0-amd64 \
  .

# Construir para múltiples plataformas (manifest list)
podman manifest create $ECR_URI/$REPO_NAME:multi-arch

podman build --platform linux/amd64 \
  -t $ECR_URI/$REPO_NAME:1.0.0-amd64 . && \
  podman manifest add $ECR_URI/$REPO_NAME:multi-arch \
  $ECR_URI/$REPO_NAME:1.0.0-amd64

podman build --platform linux/arm64 \
  -t $ECR_URI/$REPO_NAME:1.0.0-arm64 . && \
  podman manifest add $ECR_URI/$REPO_NAME:multi-arch \
  $ECR_URI/$REPO_NAME:1.0.0-arm64

# Push del manifest multi-arch a ECR
podman manifest push $ECR_URI/$REPO_NAME:multi-arch

# ✅ ECR y ECS elegirán automáticamente la arquitectura correcta
```

---

## 🟢 LAB BÁSICO — Docker/Podman + ECR + primer Task en ECS Fargate

**Tiempo estimado: 50 min**
**Prerequisito:** Docker **o** Podman instalado en tu laptop, AWS CLI v2 configurado

> 💡 Todos los comandos `docker` de este lab pueden ejecutarse con `podman` sin cambios.

---

### Paso 1 — Verificar herramienta de contenedores

```bash
# Opción A: Docker
docker --version
# Docker version 24.x.x
docker run hello-world

# Opción B: Podman
podman --version
# podman version 5.x.x
podman run hello-world
# Should print: "Hello from Docker!" (Podman usa la misma imagen)
```

---

### Paso 2 — Crear la aplicación de ejemplo

```bash
mkdir ~/lab-contenedor && cd ~/lab-contenedor

# Crear la app Node.js
cat > app.js << 'EOF'
const http = require('http');
const os   = require('os');

const PORT = process.env.PORT || 3000;

const server = http.createServer((req, res) => {
  if (req.url === '/health') {
    res.writeHead(200, { 'Content-Type': 'application/json' });
    res.end(JSON.stringify({ status: 'healthy' }));
    return;
  }

  res.writeHead(200, { 'Content-Type': 'application/json' });
  res.end(JSON.stringify({
    mensaje:    'Hola desde contenedor AWS!',
    hostname:   os.hostname(),
    plataforma: os.platform(),
    version:    process.env.APP_VERSION || '1.0.0',
    timestamp:  new Date().toISOString()
  }));
});

server.listen(PORT, () => {
  console.log(`Servidor corriendo en puerto ${PORT}`);
  console.log(`Hostname: ${os.hostname()}`);
});
EOF

# Crear package.json mínimo
cat > package.json << 'EOF'
{
  "name": "lab-contenedor",
  "version": "1.0.0",
  "main": "app.js",
  "scripts": {
    "start": "node app.js"
  }
}
EOF
```

---

### Paso 3 — Crear Dockerfile

```bash
cat > Dockerfile << 'EOF'
# Imagen base oficial Node.js (alpine = imagen mínima ~50MB)
FROM node:18-alpine

# Directorio de trabajo dentro del contenedor
WORKDIR /app

# Copiar archivos de dependencias primero (optimización de capas)
COPY package*.json ./

# Instalar dependencias (sin devDependencies)
RUN npm install --only=production

# Copiar el código de la aplicación
COPY app.js ./

# Puerto que expone la app
EXPOSE 3000

# Healthcheck integrado en la imagen
HEALTHCHECK \
  --interval=30s \
  --timeout=5s \
  --start-period=10s \
  --retries=3 \
  CMD wget -q --spider http://localhost:3000/health || exit 1

# No correr como root (buena práctica de seguridad)
USER node

# Comando de inicio
CMD ["node", "app.js"]
EOF
```

---

### Paso 4 — Construir y probar la imagen localmente

```bash
# ── Con Docker ──────────────────────────────────────────────
docker build -t lab-contenedor:1.0.0 .

docker images | grep lab-contenedor
# lab-contenedor   1.0.0   sha256:xxx   30 seconds ago   120MB

docker run -d \
  --name mi-app-local \
  -p 3000:3000 \
  -e APP_VERSION=1.0.0 \
  lab-contenedor:1.0.0

docker ps
curl http://localhost:3000/
curl http://localhost:3000/health

docker logs mi-app-local
docker stop mi-app-local && docker rm mi-app-local

# ── Con Podman (mismos comandos, cambia el prefijo) ──────────
podman build -t lab-contenedor:1.0.0 .

podman images | grep lab-contenedor

podman run -d \
  --name mi-app-local \
  -p 3000:3000 \
  -e APP_VERSION=1.0.0 \
  lab-contenedor:1.0.0

podman ps
curl http://localhost:3000/
curl http://localhost:3000/health

podman logs mi-app-local
podman stop mi-app-local && podman rm mi-app-local
```

---

### Paso 5 — Crear repositorio ECR y subir la imagen

```bash
# Variables (reemplazar con tu región y cuenta)
AWS_REGION="us-east-1"
AWS_ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text | tr -d '[:space:]')
ECR_URI="${AWS_ACCOUNT_ID}.dkr.ecr.${AWS_REGION}.amazonaws.com"
REPO_NAME="lab-contenedor"

# ⚠️  VERIFICAR que las variables no tienen caracteres invisibles antes de continuar
echo "Account ID : '${AWS_ACCOUNT_ID}'"
echo "ECR URI    : '${ECR_URI}'"
echo "Repo Name  : '${REPO_NAME}'"
echo "Full URI   : '${ECR_URI}/${REPO_NAME}'"
# Si algún valor se ve concatenado o raro → redefinir manualmente:
# export AWS_ACCOUNT_ID="123456789012"
# export ECR_URI="123456789012.dkr.ecr.us-east-1.amazonaws.com"
# export REPO_NAME="lab-contenedor"
```

```
Consola → buscar "ECR" → Elastic Container Registry
Clic "Create repository"

  Visibility:         Private
  Repository name:    lab-contenedor
  Tag immutability:   Disabled
  Scan on push:       ✅ Enable
  Encryption:         AES-256

Clic "Create repository"
```

```bash
# Login en ECR (el token dura 12 horas)

# ── Con Docker ──────────────────────────────────────────────
aws ecr get-login-password --region $AWS_REGION | \
  docker login \
    --username AWS \
    --password-stdin \
    $ECR_URI
# Login Succeeded ✅

# ── Con Podman ───────────────────────────────────────────────
aws ecr get-login-password --region $AWS_REGION | \
  podman login \
    --username AWS \
    --password-stdin \
    $ECR_URI
# Login Succeeded! ✅

# ⚠️  CONSTRUIR la URI completa una sola vez antes de taggear
# Esto evita el problema de concatenación cuando $REPO_NAME tiene caracteres invisibles
FULL_URI="${ECR_URI}/${REPO_NAME}"
echo "Push URI: '${FULL_URI}'"
# Debe mostrar EXACTAMENTE (sin nada extra):
# Push URI: '123456789012.dkr.ecr.us-east-1.amazonaws.com/lab-contenedor'

# Etiquetar usando ${FULL_URI} con llaves — más seguro que $ECR_URI/$REPO_NAME
docker tag lab-contenedor:1.0.0 ${FULL_URI}:1.0.0    # docker
docker tag lab-contenedor:1.0.0 ${FULL_URI}:latest   # docker

podman tag lab-contenedor:1.0.0 ${FULL_URI}:1.0.0    # podman
podman tag lab-contenedor:1.0.0 ${FULL_URI}:latest   # podman

# ⚠️  VERIFICAR los tags antes del push — si el nombre no dice solo "lab-contenedor" hay problema
docker images | grep "${AWS_ACCOUNT_ID}"   # docker
podman images | grep "${AWS_ACCOUNT_ID}"   # podman
# Salida esperada (dos líneas con el mismo repositorio):
# 123456789012.dkr.ecr.us-east-1.amazonaws.com/lab-contenedor  1.0.0   sha256:xxx ...
# 123456789012.dkr.ecr.us-east-1.amazonaws.com/lab-contenedor  latest  sha256:xxx ...

# Subir la imagen a ECR
docker push ${FULL_URI}:1.0.0    # docker
docker push ${FULL_URI}:latest   # docker

podman push ${FULL_URI}:1.0.0    # podman
podman push ${FULL_URI}:latest   # podman

# Verificar que está en ECR
aws ecr list-images \
  --repository-name $REPO_NAME \
  --query 'imageIds[*].[imageTag]' \
  --output table --no-cli-pager
# +--------+
# | 1.0.0  |
# | latest |
# +--------+
```

```
Verificar en la consola:
ECR → "lab-contenedor"
Deberías ver las dos imágenes ✅
Clic en cada imagen → pestaña "Vulnerabilities" para ver el scan
```

---

### Paso 6 — Crear Cluster ECS (Fargate)

```
Consola → buscar "ECS" → Elastic Container Service
Panel izquierdo → "Clusters" → "Create cluster"

  Cluster name:  lab-cluster
  
  Infrastructure:
    ✅ AWS Fargate (serverless)   ← sin gestionar servidores
    ☐ Amazon EC2 instances       ← desmarcar para este lab básico
  
  Monitoring:
    ✅ Use Container Insights     (métricas en CloudWatch)

Clic "Create"
⏳ Esperar ~30 segundos
```

---

### Paso 7 — Crear Task Definition

```
ECS → "Task definitions" → "Create new task definition"

--- Step 1: Configure task definition and containers ---
  Task definition family:  lab-tarea-web
  
  Launch type:  AWS Fargate
  OS/Architecture: Linux/X86_64
  
  Task size:
    CPU:    0.25 vCPU   (256 CPU units — mínimo en Fargate)
    Memory: 0.5 GB     (512 MB — mínimo en Fargate)
  
  Task role:      (dejar vacío por ahora)
  Task execution role:
    "Create new role"
    Name: ecsTaskExecutionRole
    (AWS crea este rol automáticamente con permisos para ECR y CloudWatch)

  Container:
    Name:   lab-app
    Image URI:  [pegar la URI de ECR]
      ej: 123456789012.dkr.ecr.us-east-1.amazonaws.com/lab-contenedor:latest
    
    Container port:   3000
    Protocol:         TCP
    Port name:        lab-app-3000-tcp
    App protocol:     HTTP
    
    Environment variables:
      Key: APP_VERSION   Value: 1.0.0
    
    Log collection:
      ✅ Use log collection
      Log driver: awslogs (CloudWatch)
      Log group: /ecs/lab-tarea-web   (se crea automáticamente)
      Log stream prefix: ecs

Clic "Create"
```

---

### Paso 8 — Ejecutar la Task en el Cluster

```
ECS → "Clusters" → "lab-cluster"
Pestaña "Tasks" → "Run new task"

  Compute configuration:
    Capacity provider strategy: Fargate
    
  Deployment configuration:
    Family: lab-tarea-web
    Revision: LATEST
    
  Networking:
    VPC:      lab-vpc-01   (o default)
    Subnets:  lab-subnet-publica-1a
    
    Security group:
      Create a new security group
        Name: sg-ecs-task-lab
        Inbound: Custom TCP, Port 3000, Source: Anywhere (0.0.0.0/0)
    
    Public IP: ✅ ENABLED   (para poder acceder directamente en este lab)

Clic "Create"
⏳ Estado: PROVISIONING → PENDING → RUNNING (~30-60 segundos)
```

```bash
# Obtener la IP pública de la task
# ECS → Clusters → lab-cluster → Tasks
# Clic en la Task → Pestaña "Configuration"
# Sección "Network" → ver "Public IP"

# Probar la app:
curl http://PUBLIC-IP:3000/
# {"mensaje":"Hola desde contenedor AWS!","hostname":"aws-fargate-xxx",...}
```

---

## 🟡 LAB INTERMEDIO — ECS Service + ALB + Auto Scaling + Variables desde Secrets Manager

**Tiempo estimado: 75 min**

---

### Paso 1 — Crear Security Groups para el Service

```
EC2 → Security Groups → "Create security group"
  Name: sg-ecs-service
  VPC:  lab-vpc-01
  
  Inbound:
    Type: Custom TCP, Port: 3000, Source: sg-alb-lab
    (Solo el ALB puede enviar tráfico a los contenedores)
  
  Outbound: All traffic, 0.0.0.0/0
Clic "Create security group"
```

---

### Paso 2 — Crear Target Group para ECS

```
EC2 → Target Groups → "Create target group"

  Target type:       IP   ← IMPORTANTE: Fargate usa IPs, no instancias
  Target group name: tg-ecs-lab
  Protocol:          HTTP
  Port:              3000
  VPC:               lab-vpc-01
  
  Health check:
    Protocol:   HTTP
    Path:       /health
    Codes:      200
    Interval:   30 sec
    Threshold:  2 healthy / 3 unhealthy

Clic "Next" → "Create target group"
(No agregar targets — ECS los registra automáticamente)
```

---

### Paso 3 — Agregar Listener al ALB existente

```
EC2 → Load Balancers → "alb-web-lab"
Pestaña "Listeners and rules"

Si ya tiene un listener en puerto 80:
  Agregar regla para enrutar /api/* a ECS:
  Clic en el listener → "Add rule"
    Name: regla-ecs
    Conditions: Path is /api/*
    Action: Forward → tg-ecs-lab
  Clic "Save"

Si no tiene listener:
  "Add listener"
    Protocol: HTTP   Port: 8080
    Default action: Forward → tg-ecs-lab
  Clic "Add"
```

---

### Paso 4 — Guardar secreto en Secrets Manager

```
Consola → buscar "Secrets Manager" → clic
Clic "Store a new secret"

  Secret type: Other type of secret
  
  Key/value pairs:
    Key: DB_PASSWORD   Value: MiPasswordRDS123!
    Key: API_KEY       Value: mi-api-key-secreta-xyz
  
  Encryption key: aws/secretsmanager (default)
  
  Clic "Next"
  
  Secret name:  /lab/ecs/app-secrets
  Description:  Secretos de la app ECS del lab
  
  Clic "Next" → "Next" → "Store"

Anotar el ARN del secreto:
  arn:aws:secretsmanager:us-east-1:123456789012:secret:/lab/ecs/app-secrets-xxxxxx
```

---

### Paso 5 — Actualizar Task Definition con secretos y más recursos

```bash
# Crear nueva revisión de la Task Definition con secretos
# Es más fácil hacerlo desde la consola o con CLI

# Desde CLI — crear nueva revisión de la task definition
# Primero copiar la definición actual y modificarla:

cat > task-definition-v2.json << 'EOF'
{
  "family": "lab-tarea-web",
  "requiresCompatibilities": ["FARGATE"],
  "networkMode": "awsvpc",
  "cpu": "512",
  "memory": "1024",
  "executionRoleArn": "arn:aws:iam::ACCOUNT_ID:role/ecsTaskExecutionRole",
  "containerDefinitions": [
    {
      "name": "lab-app",
      "image": "ACCOUNT_ID.dkr.ecr.us-east-1.amazonaws.com/lab-contenedor:latest",
      "portMappings": [
        {
          "containerPort": 3000,
          "protocol": "tcp"
        }
      ],
      "environment": [
        {
          "name": "APP_VERSION",
          "value": "2.0.0"
        },
        {
          "name": "NODE_ENV",
          "value": "production"
        }
      ],
      "secrets": [
        {
          "name": "DB_PASSWORD",
          "valueFrom": "arn:aws:secretsmanager:us-east-1:ACCOUNT_ID:secret:/lab/ecs/app-secrets-xxxxxx:DB_PASSWORD::"
        },
        {
          "name": "API_KEY",
          "valueFrom": "arn:aws:secretsmanager:us-east-1:ACCOUNT_ID:secret:/lab/ecs/app-secrets-xxxxxx:API_KEY::"
        }
      ],
      "logConfiguration": {
        "logDriver": "awslogs",
        "options": {
          "awslogs-group": "/ecs/lab-tarea-web",
          "awslogs-region": "us-east-1",
          "awslogs-stream-prefix": "ecs"
        }
      },
      "healthCheck": {
        "command": ["CMD-SHELL", "wget -q --spider http://localhost:3000/health || exit 1"],
        "interval": 30,
        "timeout": 5,
        "retries": 3,
        "startPeriod": 10
      }
    }
  ]
}
EOF
```

```bash
# Reemplazar los placeholders
AWS_ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)
sed -i "s/ACCOUNT_ID/$AWS_ACCOUNT_ID/g" task-definition-v2.json

# Registrar la nueva Task Definition
aws ecs register-task-definition \
  --cli-input-json file://task-definition-v2.json \
  --query 'taskDefinition.[family,revision,status]' \
  --output table
# +------------------+----------+---------+
# | lab-tarea-web    |    2     | ACTIVE  |
# +------------------+----------+---------+
```

---

### Paso 6 — Dar permisos al rol de ejecución para leer secretos

```
IAM → Roles → "ecsTaskExecutionRole"
"Add permissions" → "Attach policies"
  SecretsManagerReadWrite  (o crear política más restrictiva)
Clic "Add permissions"

Política más restrictiva (mejor práctica):
"Add permissions" → "Create inline policy"
```

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "secretsmanager:GetSecretValue",
        "secretsmanager:DescribeSecret"
      ],
      "Resource": "arn:aws:secretsmanager:us-east-1:ACCOUNT_ID:secret:/lab/ecs/*"
    }
  ]
}
```

```
Policy name: pol-ecs-secrets-lab
Clic "Create policy"
```

---

### Paso 7 — Crear ECS Service con ALB

```
ECS → "Clusters" → "lab-cluster"
Pestaña "Services" → "Create"

--- Deployment configuration ---
  Application type:  Service   (mantiene N tasks corriendo continuamente)
  Task definition:
    Family:    lab-tarea-web
    Revision:  2 (LATEST)
  
  Service name:     lab-servicio-web
  
  Desired tasks:    2   (2 contenedores siempre corriendo)

--- Deployment options ---
  Deployment type:   Rolling update
  Min running tasks: 100%  (nunca baja de 2 durante el deploy)
  Max running tasks: 200%  (puede subir hasta 4 durante el deploy)

--- Networking ---
  VPC:     lab-vpc-01
  Subnets: lab-subnet-privada-1a, lab-subnet-privada-1b
  Security group: sg-ecs-service   (el que creaste en Paso 1)
  Public IP: Disabled   (están en subnet privada, el ALB hace de front)

--- Load balancing ---
  Load balancer type:   Application Load Balancer
  Load balancer:        alb-web-lab
  Container:            lab-app  3000:3000
  Listener:             HTTP:80 (o el que configuraste)
  Target group:         tg-ecs-lab

--- Service auto scaling ---
  ✅ Use service auto scaling
  
  Minimum number of tasks:  1
  Maximum number of tasks:  6
  
  Scaling policy:
    Policy name:      escalar-por-cpu
    Policy type:      Target tracking
    ECS service metric: ECSServiceAverageCPUUtilization
    Target value:     70
    Scale-out cooldown:  120 seconds
    Scale-in cooldown:   300 seconds

Clic "Create"
⏳ Esperar ~2-3 minutos hasta que el service tenga 2 tasks "RUNNING"
```

---

### Paso 8 — Verificar el Service

```
ECS → "Clusters" → "lab-cluster"
Pestaña "Services" → "lab-servicio-web"

Pestaña "Tasks":
  Deberías ver 2 tasks con estado RUNNING ✅

Pestaña "Events":
  Muestra el historial: "service lab-servicio-web has reached a steady state"

Pestaña "Health and metrics":
  Ver CPU y memoria de los contenedores
```

```bash
# Probar el servicio via el ALB (el service está en subnet privada)
curl http://alb-web-lab-xxxxxxx.us-east-1.elb.amazonaws.com/
# Reload varias veces — el hostname cambia entre los 2 contenedores ✅

# Ver logs de los contenedores en CloudWatch
aws logs tail /ecs/lab-tarea-web --follow
# Ctrl+C para parar
```

---

### Paso 9 — Rolling Update (nuevo deploy sin downtime)

```bash
# Modificar la app para simular una nueva versión
sed -i 's/version: .*/version: "2.0.0 - rolling update",/' ~/lab-contenedor/app.js

cd ~/lab-contenedor
AWS_ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text | tr -d '[:space:]')
ECR_URI="${AWS_ACCOUNT_ID}.dkr.ecr.us-east-1.amazonaws.com"
REPO_NAME="lab-contenedor"
FULL_URI="${ECR_URI}/${REPO_NAME}"

# Verificar antes de continuar
echo "Push URI: '${FULL_URI}'"

# ── Con Docker ──────────────────────────────────────────────
docker build -t ${FULL_URI}:2.0.0 .
docker build -t ${FULL_URI}:latest .
aws ecr get-login-password --region us-east-1 | \
  docker login --username AWS --password-stdin $ECR_URI
docker push ${FULL_URI}:2.0.0
docker push ${FULL_URI}:latest

# ── Con Podman ───────────────────────────────────────────────
podman build -t ${FULL_URI}:2.0.0 .
podman build -t ${FULL_URI}:latest .
aws ecr get-login-password --region us-east-1 | \
  podman login --username AWS --password-stdin $ECR_URI
podman push ${FULL_URI}:2.0.0
podman push ${FULL_URI}:latest
```

```
Forzar nuevo deployment del Service:
ECS → "lab-cluster" → "lab-servicio-web"
Clic "Update service"
  ✅ Force new deployment
  Revision: LATEST
Clic "Update"

ECS desplegará la nueva versión:
1. Lanza 2 tasks nuevas (con imagen latest = 2.0.0)
2. Espera que pasen health checks
3. Registra las nuevas tasks en el ALB
4. Drena las conexiones de las tasks viejas
5. Termina las tasks viejas

Durante este proceso el servicio NO tiene downtime ✅
Monitorear en:
  ECS → Service → Pestaña "Deployments"
  Verás "PRIMARY" (nuevo) y "ACTIVE" (viejo) simultáneamente
```

---

## 🔴 LAB AVANZADO — EKS con eksctl + Deployments + Services + HPA + Ingress

**Tiempo estimado: 90 min**
**Prerequisito:** eksctl, kubectl, helm instalados en tu laptop

---

### Prerequisito — Instalar herramientas

```bash
# macOS — instalar eksctl
brew tap weaveworks/tap
brew install weaveworks/tap/eksctl
eksctl version

# macOS — instalar kubectl
brew install kubectl
kubectl version --client

# macOS — instalar helm (gestor de paquetes de Kubernetes)
brew install helm
helm version

# Verificar AWS CLI configurado
aws sts get-caller-identity
```

---

### Paso 1 — Crear cluster EKS con eksctl

```bash
# ⚠️  COSTO: EKS cobra $0.10/hora por el control plane (~$0.40 para este lab)
# Además cobran las instancias EC2 de los worker nodes

# Crear archivo de configuración del cluster
cat > cluster-config.yaml << 'EOF'
apiVersion: eksctl.io/v1alpha5
kind: ClusterConfig

metadata:
  name:    lab-eks-cluster
  region:  us-east-1
  version: "1.29"

availabilityZones:
  - us-east-1a
  - us-east-1b

# Worker nodes gestionados (Managed Node Group)
managedNodeGroups:
  - name: workers-lab
    instanceType: t3.small   # mínimo recomendado para Kubernetes
    minSize:      1
    desiredCapacity: 2
    maxSize:      4
    
    amiFamily: AmazonLinux2
    
    labels:
      environment: lab
    
    tags:
      Name: eks-worker-lab
    
    # SSH access (opcional — para debug)
    ssh:
      allow: true
      publicKeyName: lab-key-ec2   # el key pair del módulo 2

# Logging del control plane
cloudWatch:
  clusterLogging:
    enableTypes:
      - api
      - audit
      - authenticator
EOF

# Crear el cluster (tarda 15-20 minutos)
eksctl create cluster -f cluster-config.yaml

# Verás en la salida:
# [ℹ]  creating EKS cluster "lab-eks-cluster" in "us-east-1" region...
# [ℹ]  creating Managed nodegroup...
# [✔]  all EKS cluster resources for "lab-eks-cluster" have been created
```

```bash
# Verificar acceso al cluster
kubectl cluster-info
# Kubernetes control plane is running at https://xxx.gr7.us-east-1.eks.amazonaws.com

# Ver los worker nodes
kubectl get nodes
# NAME                            STATUS   ROLES    AGE   VERSION
# ip-192-168-1-10.ec2.internal    Ready    <none>   2m    v1.29.x
# ip-192-168-1-20.ec2.internal    Ready    <none>   2m    v1.29.x

# Ver namespaces
kubectl get namespaces
# default    Active
# kube-system Active
# kube-public Active
```

---

### Paso 2 — Crear Namespace para el lab

```bash
# Crear namespace separado para el lab
kubectl create namespace lab-app
# namespace/lab-app created

# Ver todos los namespaces ahora
kubectl get namespaces
# NAME          STATUS
# default       Active
# kube-system   Active
# lab-app       Active  ✅
```

---

### Paso 3 — Crear Secret con credenciales de ECR

```bash
# Para que Kubernetes pueda descargar imágenes de ECR privado
# necesita un imagePullSecret con las credenciales

AWS_ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)
AWS_REGION="us-east-1"
ECR_URI="${AWS_ACCOUNT_ID}.dkr.ecr.${AWS_REGION}.amazonaws.com"

# Obtener token de ECR
ECR_PASSWORD=$(aws ecr get-login-password --region $AWS_REGION)

# Crear el secret en Kubernetes
kubectl create secret docker-registry ecr-secret \
  --docker-server=$ECR_URI \
  --docker-username=AWS \
  --docker-password=$ECR_PASSWORD \
  --namespace=lab-app

# Verificar
kubectl get secrets -n lab-app
# NAME         TYPE                    DATA
# ecr-secret   kubernetes.io/dockerconfigjson   1
```

---

### Paso 4 — Crear Deployment

```bash
# Crear el archivo de Deployment de Kubernetes
cat > deployment.yaml << 'EOF'
apiVersion: apps/v1
kind: Deployment
metadata:
  name:      lab-app-deployment
  namespace: lab-app
  labels:
    app:     lab-app
    version: "1.0.0"
spec:
  # Número de réplicas (Pods) corriendo
  replicas: 2
  
  # Qué Pods gestiona este Deployment
  selector:
    matchLabels:
      app: lab-app
  
  # Estrategia de actualización
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxUnavailable: 0    # nunca dejar Pods no disponibles
      maxSurge:       1    # crear 1 Pod extra durante el update
  
  # Plantilla del Pod
  template:
    metadata:
      labels:
        app: lab-app
    spec:
      # Credenciales para ECR
      imagePullSecrets:
        - name: ecr-secret
      
      containers:
        - name: lab-app
          # Reemplazar ACCOUNT_ID con el tuyo
          image: ACCOUNT_ID.dkr.ecr.us-east-1.amazonaws.com/lab-contenedor:latest
          
          ports:
            - containerPort: 3000
          
          env:
            - name:  APP_VERSION
              value: "1.0.0-k8s"
            - name:  NODE_ENV
              value: "production"
          
          # Límites de recursos (obligatorio en producción)
          resources:
            requests:
              cpu:    "100m"   # 0.1 CPU core
              memory: "128Mi"  # 128 MB RAM
            limits:
              cpu:    "500m"   # 0.5 CPU core
              memory: "256Mi"  # 256 MB RAM
          
          # Health checks de Kubernetes
          readinessProbe:  # ¿está listo para recibir tráfico?
            httpGet:
              path: /health
              port: 3000
            initialDelaySeconds: 5
            periodSeconds:       10
            failureThreshold:    3
          
          livenessProbe:   # ¿sigue vivo? (si falla, Kubernetes lo reinicia)
            httpGet:
              path: /health
              port: 3000
            initialDelaySeconds: 15
            periodSeconds:       20
            failureThreshold:    3
EOF

# Reemplazar ACCOUNT_ID con el ID real
AWS_ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)
sed -i "s/ACCOUNT_ID/$AWS_ACCOUNT_ID/g" deployment.yaml

# Aplicar el Deployment
kubectl apply -f deployment.yaml

# Ver el estado del Deployment
kubectl get deployments -n lab-app
# NAME                 READY   UP-TO-DATE   AVAILABLE   AGE
# lab-app-deployment   2/2     2            2           30s ✅

# Ver los Pods
kubectl get pods -n lab-app
# NAME                                READY   STATUS    RESTARTS   AGE
# lab-app-deployment-xxx-yyy          1/1     Running   0          30s
# lab-app-deployment-xxx-zzz          1/1     Running   0          30s

# Ver detalle de un Pod
kubectl describe pod -n lab-app lab-app-deployment-xxx-yyy
# Verás: eventos, imagen, estado, recursos, health checks

# Ver logs de un Pod
kubectl logs -n lab-app -l app=lab-app --tail=20
# Servidor corriendo en puerto 3000
```

---

### Paso 5 — Crear Service para exponer los Pods

```bash
# Un Service balancea carga entre los Pods y da una IP estable

cat > service.yaml << 'EOF'
apiVersion: v1
kind: Service
metadata:
  name:      lab-app-service
  namespace: lab-app
  labels:
    app: lab-app
spec:
  type: LoadBalancer  # crea un NLB en AWS automáticamente
  
  # Mapea tráfico al puerto 3000 de los Pods con label app=lab-app
  selector:
    app: lab-app
  
  ports:
    - name:       http
      port:       80        # puerto del Service (externo)
      targetPort: 3000      # puerto del contenedor (interno)
      protocol:   TCP
EOF

kubectl apply -f service.yaml

# Ver el Service — esperar que tenga EXTERNAL-IP (~2 minutos)
kubectl get service lab-app-service -n lab-app --watch
# NAME              TYPE           CLUSTER-IP      EXTERNAL-IP                        PORT(S)
# lab-app-service   LoadBalancer   10.100.200.1    <pending>                          80:3xxxx/TCP
# lab-app-service   LoadBalancer   10.100.200.1    aaa-bbb.us-east-1.elb.amazonaws.com 80:3xxxx/TCP ✅

# Ctrl+C cuando aparezca la EXTERNAL-IP

# Probar el servicio
EXTERNAL_IP=$(kubectl get service lab-app-service -n lab-app \
  -o jsonpath='{.status.loadBalancer.ingress[0].hostname}')
echo "URL: http://$EXTERNAL_IP"

curl http://$EXTERNAL_IP/
# {"mensaje":"Hola desde contenedor AWS!","hostname":"lab-app-deployment-xxx-yyy",...}

# Recargar varias veces — el hostname cambia entre los 2 Pods ✅
for i in {1..4}; do curl -s http://$EXTERNAL_IP/ | python3 -m json.tool | grep hostname; done
```

---

### Paso 6 — Horizontal Pod Autoscaler (HPA)

```bash
# El HPA escala automáticamente el número de Pods según CPU

# Prerequisito: instalar el Metrics Server en EKS
kubectl apply -f https://github.com/kubernetes-sigs/metrics-server/releases/latest/download/components.yaml

# Esperar que el metrics-server esté listo
kubectl get deployment metrics-server -n kube-system
# READY: 1/1 ✅

# Crear el HPA
cat > hpa.yaml << 'EOF'
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name:      lab-app-hpa
  namespace: lab-app
spec:
  # Qué Deployment gestiona
  scaleTargetRef:
    apiVersion: apps/v1
    kind:       Deployment
    name:       lab-app-deployment
  
  minReplicas: 2    # mínimo de Pods
  maxReplicas: 8    # máximo de Pods
  
  metrics:
    # Escalar cuando CPU supere el 70%
    - type: Resource
      resource:
        name: cpu
        target:
          type:               Utilization
          averageUtilization: 70
    
    # Escalar cuando memoria supere el 80%
    - type: Resource
      resource:
        name: memory
        target:
          type:               Utilization
          averageUtilization: 80
EOF

kubectl apply -f hpa.yaml

# Ver el HPA
kubectl get hpa -n lab-app
# NAME          REFERENCE                TARGETS         MINPODS   MAXPODS   REPLICAS
# lab-app-hpa   Deployment/lab-app-dep   cpu: 5%/70%     2         8         2

# Ver uso actual de recursos de los Pods
kubectl top pods -n lab-app
# NAME                            CPU(cores)   MEMORY(bytes)
# lab-app-deployment-xxx-yyy      3m           32Mi
# lab-app-deployment-xxx-zzz      2m           31Mi
```

---

### Paso 7 — Rolling Update en Kubernetes

```bash
# Simular una nueva versión — cambiar la imagen del Deployment

# Opción 1: actualizar la imagen directamente
kubectl set image deployment/lab-app-deployment \
  lab-app=$ECR_URI/lab-contenedor:2.0.0 \
  -n lab-app

# Ver el rollout en tiempo real
kubectl rollout status deployment/lab-app-deployment -n lab-app
# Waiting for deployment "lab-app-deployment" rollout to finish:
# 1 out of 2 new replicas have been updated...
# 2 out of 2 new replicas have been updated...
# deployment "lab-app-deployment" successfully rolled out ✅

# Ver historial de revisiones
kubectl rollout history deployment/lab-app-deployment -n lab-app
# REVISION  CHANGE-CAUSE
# 1         <none>
# 2         <none>

# Hacer rollback a la versión anterior si algo falla
kubectl rollout undo deployment/lab-app-deployment -n lab-app
# deployment.apps/lab-app-deployment rolled back ✅

# O rollback a una revisión específica
kubectl rollout undo deployment/lab-app-deployment --to-revision=1 -n lab-app
```

---

### Paso 8 — ConfigMap y Secrets en Kubernetes

```bash
# ConfigMap — variables de entorno no sensibles
cat > configmap.yaml << 'EOF'
apiVersion: v1
kind: ConfigMap
metadata:
  name:      lab-app-config
  namespace: lab-app
data:
  NODE_ENV:    "production"
  APP_VERSION: "1.0.0"
  LOG_LEVEL:   "info"
  PORT:        "3000"
EOF

kubectl apply -f configmap.yaml

# Secret — datos sensibles (base64 encoded)
# Generar valores en base64
DB_PASS_B64=$(echo -n "MiPasswordDB123!" | base64)
API_KEY_B64=$(echo -n "mi-api-key-secreta"  | base64)

cat > secret.yaml << EOF
apiVersion: v1
kind: Secret
metadata:
  name:      lab-app-secret
  namespace: lab-app
type: Opaque
data:
  DB_PASSWORD: $DB_PASS_B64
  API_KEY:     $API_KEY_B64
EOF

kubectl apply -f secret.yaml

# Verificar (los valores aparecen como base64, no en claro)
kubectl get secret lab-app-secret -n lab-app -o yaml
# data:
#   DB_PASSWORD: TWlQYXNzd29yZERCMTIzIQ==
#   API_KEY:     bWktYXBpLWtleS1zZWNyZXRh

# Decodificar para verificar
kubectl get secret lab-app-secret -n lab-app \
  -o jsonpath='{.data.DB_PASSWORD}' | base64 --decode
# MiPasswordDB123!  ✅
```

---

### Paso 9 — Instalar AWS Load Balancer Controller (Ingress)

```bash
# El AWS Load Balancer Controller permite usar ALB como Ingress en EKS
# Ingress = enrutamiento HTTP avanzado (paths, headers, hostnames)

# Paso 9.1 — Crear IAM Policy para el controller
curl -o iam-policy.json \
  https://raw.githubusercontent.com/kubernetes-sigs/aws-load-balancer-controller/main/docs/install/iam_policy.json

aws iam create-policy \
  --policy-name AWSLoadBalancerControllerIAMPolicy \
  --policy-document file://iam-policy.json

# Paso 9.2 — Crear Service Account con IRSA (IAM Roles for Service Accounts)
AWS_ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)

eksctl create iamserviceaccount \
  --cluster=lab-eks-cluster \
  --namespace=kube-system \
  --name=aws-load-balancer-controller \
  --role-name AmazonEKSLoadBalancerControllerRole \
  --attach-policy-arn=arn:aws:iam::${AWS_ACCOUNT_ID}:policy/AWSLoadBalancerControllerIAMPolicy \
  --approve

# Paso 9.3 — Instalar el controller con Helm
helm repo add eks https://aws.github.io/eks-charts
helm repo update

helm install aws-load-balancer-controller eks/aws-load-balancer-controller \
  -n kube-system \
  --set clusterName=lab-eks-cluster \
  --set serviceAccount.create=false \
  --set serviceAccount.name=aws-load-balancer-controller

# Verificar que el controller está corriendo
kubectl get deployment -n kube-system aws-load-balancer-controller
# READY: 2/2 ✅
```

```bash
# Crear Ingress con rutas
cat > ingress.yaml << 'EOF'
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name:      lab-app-ingress
  namespace: lab-app
  annotations:
    kubernetes.io/ingress.class:                    alb
    alb.ingress.kubernetes.io/scheme:               internet-facing
    alb.ingress.kubernetes.io/target-type:          ip
    alb.ingress.kubernetes.io/healthcheck-path:     /health
    alb.ingress.kubernetes.io/healthcheck-port:     "3000"
spec:
  rules:
    - http:
        paths:
          - path:     /
            pathType: Prefix
            backend:
              service:
                name: lab-app-service
                port:
                  number: 80
EOF

kubectl apply -f ingress.yaml

# Esperar que se cree el ALB (~2 minutos)
kubectl get ingress -n lab-app --watch
# NAME              CLASS   HOSTS   ADDRESS                                     PORTS
# lab-app-ingress   <none>  *       k8s-labapp-xxx.us-east-1.elb.amazonaws.com  80 ✅

# Probar via el ALB del Ingress
INGRESS_HOST=$(kubectl get ingress lab-app-ingress -n lab-app \
  -o jsonpath='{.status.loadBalancer.ingress[0].hostname}')
curl http://$INGRESS_HOST/
```

---

## 🧹 LIMPIEZA (en este orden — MUY IMPORTANTE con EKS)

```bash
# --- KUBERNETES (eliminar en este orden para evitar que ALBs queden huérfanos) ---

# 1. Eliminar Ingress PRIMERO (libera el ALB)
kubectl delete -f ingress.yaml

# 2. Esperar que el ALB del Ingress sea eliminado en AWS
# EC2 → Load Balancers → verificar que el ALB de EKS desapareció (~2 min)

# 3. Eliminar Service (libera el NLB)
kubectl delete -f service.yaml

# 4. Esperar que el NLB sea eliminado en AWS (~2 min)

# 5. Eliminar HPA, Deployment, ConfigMap, Secret
kubectl delete -f hpa.yaml
kubectl delete -f deployment.yaml
kubectl delete -f configmap.yaml
kubectl delete -f secret.yaml

# 6. Eliminar namespace (elimina todo lo que queda dentro)
kubectl delete namespace lab-app

# 7. Desinstalar AWS Load Balancer Controller
helm uninstall aws-load-balancer-controller -n kube-system

# --- ELIMINAR EL CLUSTER EKS (lo más caro) ---
# 8. Eliminar el cluster EKS con eksctl
eksctl delete cluster \
  --name lab-eks-cluster \
  --region us-east-1
# ⏳ Tarda 10-15 minutos

# --- ECS ---
# 9. ECS → Services → "lab-servicio-web" → "Delete service"
#    Esperar que las tasks se detengan

# 10. ECS → Clusters → "lab-cluster" → "Delete cluster"

# 11. EC2 → Target Groups → "tg-ecs-lab" → Delete

# --- ECR ---
# 12. ECR → "lab-contenedor" → Delete repository

# --- Secrets Manager ---
# 13. Secrets Manager → "/lab/ecs/app-secrets" → Delete secret
#     (período de recuperación: mínimo 7 días)

# --- IAM ---
# 14. IAM → Policies → "AWSLoadBalancerControllerIAMPolicy" → Delete
# 15. IAM → Roles → "AmazonEKSLoadBalancerControllerRole" → Delete

# --- Security Groups ---
# 16. EC2 → Security Groups → "sg-ecs-task-lab", "sg-ecs-service" → Delete
```

---

## 📌 COMANDOS CLI DE REFERENCIA RÁPIDA

```bash
# ===== DOCKER =====
docker build -t nombre:tag .             # construir imagen
docker run -d -p 3000:3000 nombre:tag   # correr contenedor en background
docker ps                                # contenedores corriendo
docker logs CONTAINER_ID                 # ver logs
docker exec -it CONTAINER_ID sh          # abrir shell en contenedor
docker stop CONTAINER_ID                 # detener
docker rm CONTAINER_ID                   # eliminar

# ===== PODMAN (mismos comandos — solo cambia el prefijo) =====
podman build -t nombre:tag .
podman run -d -p 3000:3000 nombre:tag
podman ps
podman logs CONTAINER_ID
podman exec -it CONTAINER_ID sh
podman stop CONTAINER_ID
podman rm CONTAINER_ID

# Comandos exclusivos de Podman
podman machine start                     # iniciar la VM (macOS)
podman machine stop                      # detener la VM (macOS)
podman generate kube CONTAINER           # generar YAML de Kubernetes desde contenedor
podman play kube pod.yaml                # correr un Pod Kubernetes localmente
podman system prune -f                   # limpiar imágenes y contenedores no usados
podman system df                         # ver uso de disco

# ===== ECR =====
# Login con Docker
aws ecr get-login-password --region us-east-1 | \
  docker login --username AWS --password-stdin \
  $(aws sts get-caller-identity --query Account --output text).dkr.ecr.us-east-1.amazonaws.com

# Login con Podman
aws ecr get-login-password --region us-east-1 | \
  podman login --username AWS --password-stdin \
  $(aws sts get-caller-identity --query Account --output text).dkr.ecr.us-east-1.amazonaws.com

# Listar repositorios
aws ecr describe-repositories \
  --query 'repositories[*].[repositoryName,repositoryUri]' \
  --output table

# Listar imágenes
aws ecr list-images --repository-name lab-contenedor --output table

# ===== ECS =====
# Ver clusters
aws ecs list-clusters --output table

# Ver services en un cluster
aws ecs list-services --cluster lab-cluster --output table

# Ver tasks corriendo
aws ecs list-tasks --cluster lab-cluster --output table

# Forzar nuevo deployment
aws ecs update-service \
  --cluster lab-cluster \
  --service lab-servicio-web \
  --force-new-deployment

# Ver logs de ECS en CloudWatch
aws logs tail /ecs/lab-tarea-web --follow

# ===== KUBECTL (EKS) =====
kubectl get all -n lab-app               # ver todos los recursos del namespace
kubectl get pods -n lab-app -o wide      # Pods con más detalle (nodo, IP)
kubectl describe pod POD_NAME -n lab-app # detalle completo de un Pod
kubectl logs POD_NAME -n lab-app -f      # seguir logs en tiempo real
kubectl exec -it POD_NAME -n lab-app -- sh  # shell en Pod
kubectl scale deployment lab-app-deployment --replicas=3 -n lab-app  # escalar
kubectl get events -n lab-app --sort-by=.metadata.creationTimestamp  # eventos
```

---

## 📊 COMPARACIÓN DE SERVICIOS DE CONTENEDORES AWS

```
┌─────────────────────────────────────────────────────────────────┐
│              ¿Cuándo usar cada servicio?                        │
├──────────────┬──────────────────────────────────────────────────┤
│ ECR          │ Siempre — registro de imágenes para ECS y EKS    │
├──────────────┼──────────────────────────────────────────────────┤
│ ECS Fargate  │ ✅ Equipo pequeño                                │
│              │ ✅ No quieres gestionar nodos                    │
│              │ ✅ Apps simples o microservicios                  │
│              │ ✅ Rápido para comenzar                          │
├──────────────┼──────────────────────────────────────────────────┤
│ ECS EC2      │ ✅ Control total sobre los nodos                 │
│              │ ✅ Instancias Spot para ahorrar costos            │
│              │ ✅ Cargas de trabajo específicas de hardware      │
├──────────────┼──────────────────────────────────────────────────┤
│ EKS          │ ✅ Equipo con experiencia en Kubernetes           │
│              │ ✅ Ecosistema Kubernetes (Helm, Istio, Argo CD)  │
│              │ ✅ Portabilidad multi-cloud                       │
│              │ ✅ Cargas de trabajo complejas con muchos servicios│
└──────────────┴──────────────────────────────────────────────────┘
```
