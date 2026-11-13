# Módulo 7 — Artefactos y build (CodeArtifact + CodeBuild)
### Servicios: CodeArtifact · CodeBuild · ECR

---

## TEORÍA

### CodeArtifact
Repositorio de paquetes administrado. Almacena paquetes de **npm, pip, Maven, NuGet** de forma privada. Puede actuar como proxy hacia repositorios públicos (npmjs.com, PyPI) para cachear dependencias y controlar qué paquetes entran a tu organización.

### CodeBuild
Servicio de compilación completamente administrado. No necesitas servidores de build (Jenkins, etc.). Define el proceso en un archivo **buildspec.yml** dentro de tu repositorio.

**Fases del buildspec:**
- **install**: instalar runtime y herramientas
- **pre_build**: preparar el entorno (login a ECR, instalar dependencias)
- **build**: compilar, testear, construir imágenes Docker
- **post_build**: publicar artefactos, enviar notificaciones

### ECR — Elastic Container Registry
Registro privado de imágenes Docker administrado por AWS. Integrado con ECS, EKS y CodeBuild sin configuración adicional de autenticación.

---

## 🟢 LAB BÁSICO — Crear dominio CodeArtifact + primer proyecto CodeBuild

**Tiempo estimado: 45 min**
**Prerequisito:** Tener código en GitHub o CodeCommit. Usaremos una app Node.js simple.

---

### Paso 1 — Preparar repositorio de código

```bash
# En tu laptop, crear una app Node.js de ejemplo
mkdir ~/lab-app && cd ~/lab-app

# Inicializar proyecto npm
npm init -y

# Instalar dependencias de ejemplo
npm install express

# Crear archivo principal
cat > app.js << 'EOF'
const express = require('express');
const app = express();

app.get('/', (req, res) => {
  res.json({
    mensaje: 'API funcionando correctamente',
    version: process.env.APP_VERSION || '1.0.0',
    servidor: require('os').hostname()
  });
});

app.get('/health', (req, res) => {
  res.status(200).json({ status: 'healthy' });
});

const PORT = process.env.PORT || 3000;
app.listen(PORT, () => {
  console.log(`Servidor corriendo en puerto ${PORT}`);
});
EOF

# Crear archivo de test
mkdir test
cat > test/app.test.js << 'EOF'
const assert = require('assert');

// Test simple
describe('Tests de la aplicacion', () => {
  it('deberia pasar este test basico', () => {
    assert.strictEqual(1 + 1, 2);
  });
  
  it('deberia tener variable de entorno NODE_ENV', () => {
    process.env.NODE_ENV = 'test';
    assert.strictEqual(process.env.NODE_ENV, 'test');
  });
});
EOF

# Agregar script de test en package.json
# Editar manualmente package.json y cambiar "scripts":
cat > package.json << 'EOF'
{
  "name": "lab-app",
  "version": "1.0.0",
  "description": "Aplicacion de laboratorio AWS",
  "main": "app.js",
  "scripts": {
    "start": "node app.js",
    "test": "mocha test/*.test.js --timeout 5000"
  },
  "dependencies": {
    "express": "^4.18.2"
  },
  "devDependencies": {
    "mocha": "^10.2.0"
  }
}
EOF
```

```bash
# Subir código a GitHub
git init
echo "node_modules/" > .gitignore
echo ".aws/" >> .gitignore
git add .
git commit -m "feat: aplicacion inicial Node.js"

# Crear repo en GitHub y hacer push:
# (Reemplazar con tu usuario de GitHub)
git remote add origin https://github.com/TU-USUARIO/lab-app-aws.git
git branch -M main
git push -u origin main
```

---

### Paso 2 — Crear Dominio y Repositorio en CodeArtifact

```
Consola → buscar "CodeArtifact" → clic en CodeArtifact
Clic "Create domain"

  Domain name: lab-domain
  Encryption key: (dejar default — AWS Managed Key)

Clic "Create domain"
```

```
Dentro del dominio → "Create repository"

  Repository name:   lab-npm-repo
  Description:       Repositorio npm privado del lab
  
  Public upstream repositories:
    ✅ npm-store   ← proxy hacia npmjs.com
    (esto permite que CodeArtifact cachee paquetes públicos de npm)

Clic "Next"
  Domain: lab-domain
Clic "Create repository"
```

---

### Paso 3 — Crear buildspec.yml

```bash
# En tu directorio ~/lab-app, crear el archivo:
cat > buildspec.yml << 'EOF'
version: 0.2

phases:
  install:
    runtime-versions:
      nodejs: 18
    commands:
      - echo "=== FASE INSTALL: configurando entorno ==="
      - node --version
      - npm --version
      
  pre_build:
    commands:
      - echo "=== FASE PRE-BUILD: instalando dependencias ==="
      - npm ci   # más rápido y confiable que npm install en CI
      - echo "Dependencias instaladas correctamente"
      
  build:
    commands:
      - echo "=== FASE BUILD: ejecutando tests ==="
      - npm test
      - echo "=== Tests pasaron correctamente ==="
      
  post_build:
    commands:
      - echo "=== FASE POST-BUILD: empaquetando artefactos ==="
      - echo "Build completado en: $(date)"

artifacts:
  files:
    - app.js
    - package.json
    - package-lock.json
    - '**/*'
  exclude-paths:
    - node_modules/**
  name: lab-app-$(date +%Y-%m-%d)
  
cache:
  paths:
    - node_modules/**/*   # cache de dependencias entre builds
EOF

git add buildspec.yml
git commit -m "ci: agregar buildspec.yml para CodeBuild"
git push
```

---

### Paso 4 — Crear proyecto CodeBuild

```
Consola → buscar "CodeBuild" → clic en CodeBuild
Clic "Create build project"

--- Project configuration ---
  Project name:   lab-build-node
  Description:    Build y test de la aplicacion Node.js

--- Source ---
  Source provider: GitHub
  Repository:      Repository in my GitHub account
  
  Conectar con GitHub:
    Clic "Manage default source credential"
    Credential type: OAuth app
    Clic "Connect to GitHub"
    → Se abre ventana de GitHub para autorizar
    → Autorizar CodeBuild → Close

  GitHub repository: TU-USUARIO/lab-app-aws
  
  Source version: main   (rama principal)

--- Primary source webhook events ---
  ✅ Rebuild every time a code change is pushed to this repository
  
  Event types:
    ✅ PUSH   (build en cada push)
    ✅ PULL_REQUEST_MERGED

--- Environment ---
  Environment image: Managed image
  Operating system:  Amazon Linux 2023
  Runtime:           Standard
  Image:             aws/codebuild/amazonlinux2-x86_64-standard:5.0
  Image version:     Always use the latest image for this runtime version
  Environment type:  Linux
  
  Privileged: NO   (no necesitamos Docker en este lab básico)
  
  Service role:
    ✅ New service role
    Role name: codebuild-lab-build-node-service-role

--- Buildspec ---
  Build specifications: Use a buildspec file
  Buildspec name:       buildspec.yml   (en la raíz del repo)

--- Artifacts ---
  Type: Amazon S3
  Bucket name: lab-s3-TUAPELLIDO-2024   (el bucket del módulo 4)
  Path: artifacts/
  Namespace type: Build ID
  Packaging: ZIP

--- Cache ---
  Type: Amazon S3
  Bucket: lab-s3-TUAPELLIDO-2024
  Cache path prefix: cache/node-modules/

--- Logs ---
  CloudWatch logs: ✅
  Group name:  /codebuild/lab-build-node
  Stream name: (dejar vacío — genera automáticamente)

Clic "Create build project"
```

---

### Paso 5 — Ejecutar el primer build

```
En el proyecto "lab-build-node"
Clic "Start build"
  Source version: main
Clic "Start build"

Monitorear el build:
  Pestaña "Phase details" → ver cada fase ejecutándose
  Pestaña "Build logs" → ver los logs en tiempo real
```

```
✅ Estado esperado de cada fase:
  DOWNLOAD_SOURCE:    SUCCEEDED
  INSTALL:            SUCCEEDED
  PRE_BUILD:          SUCCEEDED
  BUILD:              SUCCEEDED (con los tests pasando)
  POST_BUILD:         SUCCEEDED
  UPLOAD_ARTIFACTS:   SUCCEEDED
  FINALIZING:         SUCCEEDED

Estado final: SUCCEEDED (marca verde)
```

---

## 🟡 LAB INTERMEDIO — CodeArtifact upstream + Variables desde Parameter Store + Cache

**Tiempo estimado: 60 min**

---

### Paso 1 — Almacenar variables en Parameter Store

```
Consola → buscar "Systems Manager" → Parameter Store
Clic "Create parameter"

PARÁMETRO 1 — Variable de entorno:
  Name:  /lab/app/version
  Type:  String
  Value: 1.2.0
Clic "Create parameter"

PARÁMETRO 2 — Contraseña (SecureString):
  Name:  /lab/app/db-password
  Type:  SecureString
  KMS key source: My current account → alias/aws/ssm
  Value: MiPasswordSegura123!
Clic "Create parameter"
```

---

### Paso 2 — Configurar CodeBuild para usar CodeArtifact + Parameter Store

```bash
# Actualizar buildspec.yml para usar CodeArtifact y Parameter Store
cat > buildspec.yml << 'EOF'
version: 0.2

env:
  parameter-store:
    APP_VERSION: /lab/app/version
    DB_PASSWORD: /lab/app/db-password
  variables:
    NODE_ENV: test

phases:
  install:
    runtime-versions:
      nodejs: 18
    commands:
      - echo "=== INSTALL: configurando CodeArtifact ==="
      # Login en CodeArtifact para obtener token temporal
      - CODEARTIFACT_TOKEN=$(aws codeartifact get-authorization-token \
          --domain lab-domain \
          --domain-owner $(aws sts get-caller-identity --query Account --output text) \
          --query authorizationToken \
          --output text)
      # Configurar npm para usar CodeArtifact en lugar de npmjs.com
      - npm config set registry \
          https://lab-domain-$(aws sts get-caller-identity --query Account --output text).d.codeartifact.us-east-1.amazonaws.com/npm/lab-npm-repo/
      - npm config set //lab-domain-$(aws sts get-caller-identity --query Account --output text).d.codeartifact.us-east-1.amazonaws.com/npm/lab-npm-repo/:_authToken $CODEARTIFACT_TOKEN
      
  pre_build:
    commands:
      - echo "=== PRE-BUILD: version $APP_VERSION ==="
      - echo "DB Password (no imprimir en prod): [PROTEGIDO]"
      - npm ci
      
  build:
    commands:
      - echo "=== BUILD: ejecutando tests ==="
      - npm test
      - echo "Build version $APP_VERSION completado"
      
  post_build:
    commands:
      - echo "=== POST-BUILD ==="
      - echo "Artefacto version $APP_VERSION listo"

artifacts:
  files:
    - '**/*'
  exclude-paths:
    - node_modules/**

cache:
  paths:
    - node_modules/**/*
EOF

git add buildspec.yml
git commit -m "ci: agregar CodeArtifact y Parameter Store"
git push
```

---

### Paso 3 — Dar permisos al rol de CodeBuild

```
IAM → Roles → buscar "codebuild-lab-build-node-service-role"
Clic en el rol → "Add permissions" → "Attach policies"

Agregar estas políticas:
  1. AWSCodeArtifactAdminAccess   (acceso a CodeArtifact)
  2. AmazonSSMReadOnlyAccess      (leer Parameter Store)

Clic "Add permissions"
```

```
También necesitamos una política inline para el token de CodeArtifact:
En el rol → "Add permissions" → "Create inline policy"
Pegar:
```

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "codeartifact:GetAuthorizationToken",
        "codeartifact:GetRepositoryEndpoint",
        "codeartifact:ReadFromRepository"
      ],
      "Resource": "*"
    },
    {
      "Effect": "Allow",
      "Action": "sts:GetServiceBearerToken",
      "Resource": "*",
      "Condition": {
        "StringEquals": {
          "sts:AWSServiceName": "codeartifact.amazonaws.com"
        }
      }
    }
  ]
}
```

```
Policy name: pol-codebuild-codeartifact
Clic "Create policy"
```

---

### Paso 4 — Medir diferencia de tiempo con/sin caché

```
Primera ejecución (sin caché):
  CodeBuild → "lab-build-node" → "Start build"
  Anotar el tiempo total (en "Phase details")
  Verificar fase INSTALL + PRE_BUILD — probablemente ~40-60 segundos

Segunda ejecución (con caché de node_modules):
  Después que el primer build sube el caché a S3
  CodeBuild → "Start build" nuevamente
  La fase PRE_BUILD (npm ci) debería ser ~10-15 segundos
  
✅ La diferencia puede ser de 30-50 segundos en proyectos reales
   En proyectos grandes (cientos de dependencias) puede ser de varios minutos
```

---

## 🔴 LAB AVANZADO — Docker build + Push a ECR + Webhook + Reporte JUnit

**Tiempo estimado: 75 min**

---

### Paso 1 — Crear repositorio ECR

```
Consola → buscar "ECR" → Elastic Container Registry
Clic "Create repository"

  Visibility: Private
  Repository name: lab-app-repo
  
  Tag immutability: Disabled   (permitir sobreescribir tags en lab)
  
  Scan on push: ✅ Enable   (escanea vulnerabilidades al subir imagen)
  
  Encryption: AES-256 (default)

Clic "Create repository"
Anotar el URI:
  123456789012.dkr.ecr.us-east-1.amazonaws.com/lab-app-repo
```

---

### Paso 2 — Crear Dockerfile

```bash
# En ~/lab-app, crear el Dockerfile
cat > Dockerfile << 'EOF'
# Imagen base oficial de Node.js (versión slim para menor tamaño)
FROM node:18-alpine

# Directorio de trabajo dentro del contenedor
WORKDIR /app

# Copiar package.json primero (optimización de capas Docker)
COPY package*.json ./

# Instalar solo dependencias de producción
RUN npm ci --only=production

# Copiar el resto del código
COPY app.js ./

# Puerto que expone la aplicación
EXPOSE 3000

# Healthcheck de la imagen
HEALTHCHECK --interval=30s --timeout=5s --start-period=5s --retries=3 \
  CMD wget -q --spider http://localhost:3000/health || exit 1

# Usuario no-root para seguridad
USER node

# Comando de inicio
CMD ["node", "app.js"]
EOF
```

---

### Paso 3 — Instalar Mocha y actualizar tests para reporte JUnit

```bash
# Actualizar package.json para incluir reporter JUnit
cat > package.json << 'EOF'
{
  "name": "lab-app",
  "version": "1.0.0",
  "description": "Aplicacion de laboratorio AWS",
  "main": "app.js",
  "scripts": {
    "start": "node app.js",
    "test": "mocha test/*.test.js --reporter mocha-junit-reporter --timeout 5000"
  },
  "dependencies": {
    "express": "^4.18.2"
  },
  "devDependencies": {
    "mocha": "^10.2.0",
    "mocha-junit-reporter": "^2.2.1"
  }
}
EOF
```

---

### Paso 4 — Buildspec avanzado con Docker + ECR + Reporte JUnit

```bash
# Crear el buildspec avanzado
cat > buildspec.yml << 'EOF'
version: 0.2

env:
  variables:
    AWS_ACCOUNT_ID: ""   # se rellena con comando en pre_build
    IMAGE_REPO:          lab-app-repo
    IMAGE_TAG:           latest
  
phases:
  install:
    runtime-versions:
      nodejs: 18
    commands:
      - echo "=== INSTALL ==="
      - node --version
      - docker --version
      
  pre_build:
    commands:
      - echo "=== PRE-BUILD: login en ECR ==="
      - AWS_ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)
      - ECR_URI=$AWS_ACCOUNT_ID.dkr.ecr.$AWS_DEFAULT_REGION.amazonaws.com
      - IMAGE_URI=$ECR_URI/$IMAGE_REPO
      
      # Login en ECR
      - aws ecr get-login-password --region $AWS_DEFAULT_REGION | \
          docker login --username AWS --password-stdin $ECR_URI
      
      # Instalar dependencias para tests (incluyendo devDependencies)
      - npm ci
      
  build:
    commands:
      - echo "=== BUILD: ejecutando tests ==="
      
      # Ejecutar tests con reporter JUnit
      - MOCHA_FILE=test-results/junit.xml npm test || true
      # El "|| true" evita que el build falle si hay tests fallidos
      # (queremos que el reporte se genere aunque haya fallos)
      
      - echo "=== BUILD: construyendo imagen Docker ==="
      - docker build -t $IMAGE_URI:$IMAGE_TAG .
      - docker build -t $IMAGE_URI:$CODEBUILD_RESOLVED_SOURCE_VERSION .
      
  post_build:
    commands:
      - echo "=== POST-BUILD: publicando imagen en ECR ==="
      - docker push $IMAGE_URI:$IMAGE_TAG
      - docker push $IMAGE_URI:$CODEBUILD_RESOLVED_SOURCE_VERSION
      
      # Crear archivo con la URI de la imagen (usado por CodeDeploy/ECS)
      - printf '[{"name":"lab-app","imageUri":"%s"}]' \
          $IMAGE_URI:$CODEBUILD_RESOLVED_SOURCE_VERSION > imagedefinitions.json
      
      - echo "Imagen publicada:"
      - echo "  $IMAGE_URI:$IMAGE_TAG"
      - echo "  $IMAGE_URI:$CODEBUILD_RESOLVED_SOURCE_VERSION"

reports:
  lab-test-report:
    files:
      - 'test-results/junit.xml'
    file-format: JUNITXML

artifacts:
  files:
    - imagedefinitions.json
    - Dockerfile
  discard-paths: yes
EOF

# Crear directorio de resultados de tests
mkdir -p test-results

git add .
git commit -m "feat: agregar Docker build y reporte JUnit"
git push
```

---

### Paso 5 — Habilitar Docker y dar permisos ECR al rol de CodeBuild

```
CodeBuild → "lab-build-node" → "Edit" → "Environment"
  
  Privileged: ✅ Enable   ← OBLIGATORIO para builds con Docker
  
Clic "Update environment"
```

```
IAM → Roles → "codebuild-lab-build-node-service-role"
"Add permissions" → "Attach policies"
  AmazonEC2ContainerRegistryPowerUser   ← permisos para ECR
Clic "Add permissions"
```

---

### Paso 6 — Configurar Webhook (build automático en Pull Request)

```
CodeBuild → "lab-build-node" → "Edit" → "Source"

Primary source webhook events:
  ✅ Rebuild every time a code change is pushed

Filter groups — "Add filter group":
  Grupo 1 (builds en PRs):
    Event type: PULL_REQUEST_CREATED
    Event type: PULL_REQUEST_UPDATED
    Event type: PULL_REQUEST_REOPENED
  
  Agregar filtro dentro del grupo:
    HEAD_REF: ^refs/heads/   (cualquier rama)

  "Add filter group" (segundo grupo para push a main):
    Event type: PUSH
    HEAD_REF: ^refs/heads/main$   (solo rama main)

Clic "Update source"
```

---

### Paso 7 — Ejecutar build y ver reporte JUnit

```
CodeBuild → "lab-build-node" → "Start build"

Una vez completado:
  Pestaña "Reports"
  Clic en el reporte "lab-test-report"
  
  Ver:
  - Total de tests: X
  - Pasaron: X
  - Fallaron: 0
  - Duración: X ms
  - Detalle de cada test ✅
```

---

### Paso 8 — Verificar imagen en ECR

```
ECR → "lab-app-repo"
Deberías ver:
  Imagen con tag "latest"
  Imagen con tag = commit SHA de Git

Clic en la imagen:
  Pestaña "Vulnerabilities":
    Si el scan detectó vulnerabilidades → aparecen aquí
    Con la imagen node:18-alpine deberías tener pocas o ninguna ✅
```

```bash
# Probar la imagen localmente (si tienes Docker instalado)
# Reemplazar con tu URI real de ECR

aws ecr get-login-password --region us-east-1 | \
  docker login --username AWS --password-stdin \
  123456789012.dkr.ecr.us-east-1.amazonaws.com

docker pull 123456789012.dkr.ecr.us-east-1.amazonaws.com/lab-app-repo:latest

docker run -p 3000:3000 \
  123456789012.dkr.ecr.us-east-1.amazonaws.com/lab-app-repo:latest

# En otra terminal:
curl http://localhost:3000/
# {"mensaje":"API funcionando correctamente","version":"1.0.0","servidor":"..."}
curl http://localhost:3000/health
# {"status":"healthy"}
```

---

## 🧹 LIMPIEZA (en este orden)

```
1.  CodeBuild → "lab-build-node" → "Delete build project"

2.  ECR → "lab-app-repo" → "Delete repository"
    (eliminar imágenes primero si pide confirmación)

3.  CodeArtifact → Domains → "lab-domain"
    → Repositories → "lab-npm-repo" → Delete repository
    → Delete domain

4.  Parameter Store → eliminar:
    /lab/app/version
    /lab/app/db-password

5.  IAM → Roles → "codebuild-lab-build-node-service-role" → Delete

6.  S3 → bucket "lab-s3-TUAPELLIDO-2024" → carpeta "artifacts/" → eliminar contenido

7.  CloudWatch → Log groups → "/codebuild/lab-build-node" → Delete
```

---

## 📌 COMANDOS CLI DE REFERENCIA RÁPIDA

```bash
# Iniciar build desde CLI
aws codebuild start-build \
  --project-name lab-build-node \
  --query 'build.id' \
  --output text

# Ver estado de un build
aws codebuild batch-get-builds \
  --ids BUILD-ID-AQUI \
  --query 'builds[0].[currentPhase,buildStatus]' \
  --output table

# Login en ECR
aws ecr get-login-password --region us-east-1 | \
  docker login --username AWS --password-stdin \
  $(aws sts get-caller-identity --query Account --output text).dkr.ecr.us-east-1.amazonaws.com

# Listar imágenes en ECR
aws ecr list-images \
  --repository-name lab-app-repo \
  --query 'imageIds[*].[imageTag,imageDigest]' \
  --output table

# Login en CodeArtifact para npm
aws codeartifact login \
  --tool npm \
  --domain lab-domain \
  --repository lab-npm-repo \
  --domain-owner $(aws sts get-caller-identity --query Account --output text)

# Ver proyectos CodeBuild
aws codebuild list-projects \
  --output table

# Ver historial de builds
aws codebuild list-builds-for-project \
  --project-name lab-build-node \
  --query 'ids[0:5]' \
  --output table
```
