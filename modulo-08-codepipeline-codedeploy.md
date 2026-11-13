# Módulo 8 — Pipeline y despliegue (CodePipeline + CodeDeploy)
### Servicios: CodePipeline · CodeDeploy · GitHub

---

## TEORÍA

### CodePipeline
Orquestador de CI/CD que conecta etapas en secuencia: **Source → Build → Test → Deploy**. Cada cambio en el código dispara el pipeline automáticamente. Puede conectar múltiples proveedores (GitHub, CodeBuild, CodeDeploy, Lambda, ECS, etc.).

### CodeDeploy
Automatiza el despliegue de aplicaciones en:
- **EC2/On-premise**: instala y actualiza software en instancias
- **Lambda**: gestiona versiones y aliases con tráfico controlado
- **ECS**: actualiza Tasks Definitions en Fargate y EC2

**Estrategias de despliegue:**
- **In-Place**: actualiza las instancias existentes una a una (downtime posible)
- **Blue/Green**: crea un entorno nuevo → prueba → mueve todo el tráfico → elimina el viejo

**appspec.yml**: define los pasos de despliegue: qué archivos copiar, qué scripts ejecutar antes/después.

---

## 🟢 LAB BÁSICO — Pipeline GitHub → CodeBuild → CodeDeploy en EC2

**Tiempo estimado: 60 min**
**Prerequisito:** App del Módulo 7 en GitHub, instancia EC2 lab-web-01, Auto Scaling Group

---

### Paso 1 — Preparar la instancia EC2 para CodeDeploy

```bash
# La instancia EC2 necesita el CodeDeploy Agent instalado y el rol IAM correcto

# En la instancia EC2 (vía SSH o Instance Connect):
# Instalar el CodeDeploy Agent
sudo yum update -y
sudo yum install ruby wget -y

# Descargar el instalador del agente (para us-east-1)
cd /tmp
wget https://aws-codedeploy-us-east-1.s3.us-east-1.amazonaws.com/latest/install
chmod +x ./install

# Instalar el agente
sudo ./install auto

# Verificar que el agente está corriendo
sudo systemctl status codedeploy-agent
# Active: active (running) ✅

# Habilitar el agente para que inicie con el sistema
sudo systemctl enable codedeploy-agent
```

---

### Paso 2 — Crear rol IAM para EC2 con permisos CodeDeploy

```
IAM → Roles → "Create role"

  Trusted entity: AWS service → EC2
  Clic "Next"

  Policies a adjuntar:
    AmazonEC2RoleforAWSCodeDeploy   (permite al agente descargar artefactos de S3)
    AmazonSSMManagedInstanceCore    (acceso a SSM — opcional pero recomendado)

Clic "Next"
  Role name: rol-ec2-codedeploy
Clic "Create role"
```

```
Adjuntar el rol a la instancia EC2:
EC2 → Instances → "lab-web-01"
Actions → Security → "Modify IAM role"
  IAM role: rol-ec2-codedeploy
Clic "Update IAM role"
```

---

### Paso 3 — Crear appspec.yml en el repositorio

```bash
# En ~/lab-app, crear el archivo appspec.yml
cat > appspec.yml << 'EOF'
version: 0.0
os: linux

files:
  - source: app.js
    destination: /var/www/app/
  - source: package.json
    destination: /var/www/app/
  - source: package-lock.json
    destination: /var/www/app/

permissions:
  - object: /var/www/app/
    owner: ec2-user
    group: ec2-user
    mode: 755
    type:
      - directory
  - object: /var/www/app/app.js
    owner: ec2-user
    group: ec2-user
    mode: 755
    type:
      - file

hooks:
  ApplicationStop:
    - location: scripts/stop-app.sh
      timeout: 30
      runas: ec2-user
  
  BeforeInstall:
    - location: scripts/before-install.sh
      timeout: 60
      runas: root
  
  AfterInstall:
    - location: scripts/install-dependencies.sh
      timeout: 120
      runas: ec2-user
  
  ApplicationStart:
    - location: scripts/start-app.sh
      timeout: 30
      runas: ec2-user
  
  ValidateService:
    - location: scripts/validate-service.sh
      timeout: 30
      runas: ec2-user
EOF
```

```bash
# Crear los scripts de deploy
mkdir scripts

# Script: detener la aplicación
cat > scripts/stop-app.sh << 'EOF'
#!/bin/bash
# Detener la app si está corriendo (PM2 o node directamente)
if [ -f /var/www/app/app.pid ]; then
  PID=$(cat /var/www/app/app.pid)
  kill $PID 2>/dev/null || true
  rm -f /var/www/app/app.pid
fi
# Si usamos PM2:
pm2 stop lab-app 2>/dev/null || true
echo "Aplicacion detenida"
EOF

# Script: preparar el directorio antes de instalar
cat > scripts/before-install.sh << 'EOF'
#!/bin/bash
# Crear directorio de la app si no existe
mkdir -p /var/www/app
chown -R ec2-user:ec2-user /var/www/app

# Instalar Node.js si no está instalado
if ! command -v node &> /dev/null; then
  curl -fsSL https://rpm.nodesource.com/setup_18.x | bash -
  yum install -y nodejs
fi

# Instalar PM2 globalmente (gestor de procesos Node.js)
npm install -g pm2 2>/dev/null || true
echo "Preparacion completada"
EOF

# Script: instalar dependencias de la app
cat > scripts/install-dependencies.sh << 'EOF'
#!/bin/bash
cd /var/www/app
npm ci --only=production
echo "Dependencias instaladas"
EOF

# Script: iniciar la aplicación
cat > scripts/start-app.sh << 'EOF'
#!/bin/bash
cd /var/www/app
export PORT=3000
export NODE_ENV=production
# Iniciar con PM2 (gestiona restarts automáticos)
pm2 start app.js --name lab-app --env production
pm2 save   # guardar lista de procesos para restaurar al reiniciar
echo "Aplicacion iniciada con PM2"
EOF

# Script: validar que la app responde correctamente
cat > scripts/validate-service.sh << 'EOF'
#!/bin/bash
# Esperar que la app esté lista
sleep 5

# Verificar que responde en el puerto 3000
HTTP_CODE=$(curl -s -o /dev/null -w "%{http_code}" http://localhost:3000/health)

if [ "$HTTP_CODE" -eq 200 ]; then
  echo "Validacion exitosa: app responde con HTTP 200"
  exit 0
else
  echo "FALLO: app no responde (HTTP $HTTP_CODE)"
  exit 1
fi
EOF

# Dar permisos de ejecución a todos los scripts
chmod +x scripts/*.sh

# Actualizar buildspec.yml para incluir scripts y appspec en artefactos
cat > buildspec.yml << 'EOF'
version: 0.2

phases:
  install:
    runtime-versions:
      nodejs: 18
  pre_build:
    commands:
      - npm ci
  build:
    commands:
      - npm test

artifacts:
  files:
    - app.js
    - package.json
    - package-lock.json
    - appspec.yml
    - scripts/**
  discard-paths: no

cache:
  paths:
    - node_modules/**/*
EOF

git add .
git commit -m "deploy: agregar appspec.yml y scripts de CodeDeploy"
git push
```

---

### Paso 4 — Crear Aplicación y Deployment Group en CodeDeploy

```
Consola → buscar "CodeDeploy" → clic en CodeDeploy
Panel izquierdo → "Applications" → "Create application"

  Application name: lab-app-ec2
  Compute platform: EC2/On-premises

Clic "Create application"
```

```
Dentro de "lab-app-ec2" → "Create deployment group"

  Deployment group name: dg-ec2-lab
  
  Service role:
    Create new role: NO — Crear previamente:
    
    IAM → Create role → AWS service → CodeDeploy
      Use case: CodeDeploy
      Policy: AWSCodeDeployRole   (se adjunta automáticamente)
      Role name: rol-codedeploy-ec2
    
    Luego en el Deployment Group:
    Service role: rol-codedeploy-ec2

  Deployment type: In-place   (básico — Blue/Green en intermedio)

  Environment configuration:
    ✅ Amazon EC2 instances
    Tag group 1:
      Key:   Name
      Value: lab-web-01

  Deployment settings:
    Deployment configuration: CodeDeployDefault.OneAtATime
    (En producción: HalfAtATime para menos riesgo)

  Load balancer:
    ✅ Enable load balancing: NO   (básico sin ALB)

Clic "Create deployment group"
```

---

### Paso 5 — Crear el Pipeline en CodePipeline

```
Consola → buscar "CodePipeline" → clic en CodePipeline
Clic "Create pipeline"

--- Step 1: Pipeline settings ---
  Pipeline name:          pipeline-lab-node
  Pipeline type:          V2   (más funciones)
  Execution mode:         SUPERSEDED
  Service role:
    New service role
    Role name: AWSCodePipelineServiceRole-us-east-1-pipeline-lab-node
Clic "Next"

--- Step 2: Add source stage ---
  Source provider:   GitHub (Version 2)
  
  Connection:
    Clic "Connect to GitHub"
    Connection name: github-lab-connection
    → Se abre ventana de GitHub → "Install a new app"
    → Seleccionar tu repositorio → Authorize
    → "Connect"
  
  Repository name:   TU-USUARIO/lab-app-aws
  Default branch:    main
  Output artifact format: CodePipeline default
  
  ✅ Start the pipeline on source code change
Clic "Next"

--- Step 3: Add build stage ---
  Build provider: AWS CodeBuild
  Region: US East (N. Virginia)
  Project name: lab-build-node
  Build type: Single build
Clic "Next"

--- Step 4: Add deploy stage ---
  Deploy provider: AWS CodeDeploy
  Region: US East (N. Virginia)
  Application name: lab-app-ec2
  Deployment group: dg-ec2-lab
Clic "Next"

--- Step 5: Review ---
  Revisar toda la configuración
Clic "Create pipeline"
```

---

### Paso 6 — Observar el pipeline ejecutarse

```
El pipeline arranca automáticamente al crearse.
Verás las 3 etapas: Source → Build → Deploy

Monitorear cada etapa:
  Source:  clic en "Details" → ver el commit que disparó el pipeline
  Build:   clic en "Details" → abre CodeBuild con los logs en tiempo real
  Deploy:  clic en "Details" → abre CodeDeploy con el estado del deployment
```

```
CodeDeploy → Applications → "lab-app-ec2" → Deployments
Seleccionar el deployment activo
  
  Pestaña "Deployment lifecycle events":
    ApplicationStop:   SUCCEEDED ✅
    BeforeInstall:     SUCCEEDED ✅
    Install:           SUCCEEDED ✅
    AfterInstall:      SUCCEEDED ✅
    ApplicationStart:  SUCCEEDED ✅
    ValidateService:   SUCCEEDED ✅
  
  Estado final: "Succeeded" ✅
```

```bash
# Verificar desde la instancia EC2 que la app está corriendo con PM2
pm2 list
# Debe mostrar "lab-app" con status "online"

curl http://localhost:3000/
# {"mensaje":"API funcionando correctamente",...}
```

---

## 🟡 LAB INTERMEDIO — Blue/Green + Canary + Rollback + Aprobación Manual

**Tiempo estimado: 70 min**

---

### Paso 1 — Configurar Blue/Green con ALB

```
En CodeDeploy, necesitas el ASG del Módulo 2 (asg-web-lab) y el ALB (alb-web-lab).

CodeDeploy → Applications → "lab-app-ec2"
Clic "Create deployment group" (un grupo nuevo para Blue/Green)

  Name:         dg-ec2-bluegreen
  Service role: rol-codedeploy-ec2

  Deployment type: Blue/green

  Environment configuration:
    ✅ Automatically copy Amazon EC2 Auto Scaling group
    Auto Scaling group: asg-web-lab

  Deployment settings:
    ✅ Route traffic immediately after a successful deployment
    
    Terminate the original instances:
      ✅ Terminate the original instances in the deployment group: 60 minutes
      (CodeDeploy espera 60 min antes de terminar las instancias Blue)

  Load balancer:
    ✅ Enable load balancing
    Load balancer: Choose from Application load balancers
    Load balancer: alb-web-lab
    Target group: tg-web-lab

Clic "Create deployment group"
```

---

### Paso 2 — Deployment configuration Canary

```
Usaremos la configuración canary: primero 10% del tráfico → esperar → luego 100%

CodeDeploy → "Deployment configurations" → "Create deployment configuration"

  Config name:              MiCanary10Percent5Minutes
  Compute platform:         EC2/On-Premises
  
  Deployment type:          In-place
  Configuration type:       Canary
  
  Canary settings:
    Percentage:   10   (% de instancias en el primer batch)
    Interval:     5    (minutos entre el primer y segundo batch)

Clic "Create deployment configuration"
```

```
Actualizar el deployment group para usar esta configuración:
CodeDeploy → "dg-ec2-bluegreen" → Edit
  Deployment configuration: MiCanary10Percent5Minutes
Clic "Save changes"
```

---

### Paso 3 — Configurar Rollback automático con CloudWatch

```
CodeDeploy → Deployment groups → "dg-ec2-bluegreen" → Edit

  Advanced → Rollbacks:
    ✅ Roll back when a deployment fails
    ✅ Roll back when alarm thresholds are met
    
    Alarms:
      Clic "Add alarm"
      
      Primero crear la alarma en CloudWatch:
        CloudWatch → "Alarms" → "Create alarm"
        Select metric → EC2 → By Auto Scaling Group
          Metric name: CPUUtilization
          ASG: asg-web-lab
        
        Conditions:
          Threshold: Greater than 80
          Period: 1 minute
          Consecutive periods: 2
        
        Actions:
          En alarm: (sin acción por ahora — CodeDeploy la detecta)
          
        Alarm name: alarma-cpu-asg-lab
        Clic "Create alarm"
      
      De vuelta en CodeDeploy:
      Alarm name: alarma-cpu-asg-lab

Clic "Save changes"
```

**Simular rollback:**
```bash
# En una instancia del ASG, generar alta CPU
stress --cpu 4 --timeout 300

# CloudWatch detectará CPU > 80% después de 2 minutos
# Si hay un deploy activo → se activa rollback automático
# En la consola CodeDeploy → Deployments → ver estado "Stopped" con motivo "CWAlarm"
```

---

### Paso 4 — Agregar Aprobación Manual en CodePipeline

```
CodePipeline → "pipeline-lab-node" → Edit

Agregar etapa entre Build y Deploy:
  Clic "Add stage" (entre la etapa Build y Deploy)
    Stage name: Aprobacion-QA

  Dentro de la nueva etapa → "Add action group":
    Action name:     aprobar-deploy-produccion
    Action provider: Manual approval
    
    SNS topic ARN:   (crear un topic SNS primero)
      SNS → "Create topic"
        Type: Standard
        Name: notif-aprobacion-deploy
      Clic "Create topic"
      
      Crear suscripción:
        Protocol: Email
        Endpoint: tu@email.com
      Confirmar email de suscripción
    
    SNS topic ARN: arn:aws:sns:us-east-1:ACCOUNT:notif-aprobacion-deploy
    
    Review URL:  (opcional: URL de ambiente de QA para revisar)
    Comments:    Revisar en QA antes de aprobar: http://alb-web-lab-xxx.amazonaws.com
    
Clic "Done" → "Save"
```

**Flujo de aprobación:**
```
1. Pipeline llega a la etapa "Aprobacion-QA"
2. Pipeline se detiene y envía email via SNS al equipo de QA
3. El email contiene un link directo para aprobar o rechazar
4. Si se aprueba → continúa al deploy
5. Si se rechaza → pipeline falla con el mensaje de rechazo
6. Las aprobaciones tienen TTL de 7 días — si nadie aprueba, expira

También puedes aprobar desde CLI:
```

```bash
# Aprobar desde CLI (sin abrir la consola)
aws codepipeline put-approval-result \
  --pipeline-name pipeline-lab-node \
  --stage-name Aprobacion-QA \
  --action-name aprobar-deploy-produccion \
  --result "summary=Revisado en QA, todo OK,status=Approved" \
  --token XXXXXXXX   # obtener el token del email o desde la consola
```

---

## 🔴 LAB AVANZADO — Lambda Canary + Pipeline Multi-región + Pipeline como Código CDK

**Tiempo estimado: 90 min**

---

### Parte 1 — CodeDeploy para Lambda con Canary

```bash
# Crear función Lambda de ejemplo
cat > lambda-function.py << 'EOF'
import json

def handler(event, context):
    return {
        'statusCode': 200,
        'headers': {'Content-Type': 'application/json'},
        'body': json.dumps({
            'mensaje': 'Lambda version 2.0',
            'version': '2.0'
        })
    }
EOF

# Comprimir para subir
zip lambda-v2.zip lambda-function.py
```

```
Lambda → "Create function"
  Name: lab-lambda-deploy
  Runtime: Python 3.12
  Architecture: x86_64
Clic "Create function"

Subir el código:
  "Upload from" → ".zip file" → seleccionar lambda-v2.zip
Clic "Save"

Crear alias:
  Lambda → "lab-lambda-deploy" → Pestaña "Aliases" → "Create alias"
    Name: live
    Version: $LATEST  (o la versión publicada)
  Clic "Save"
```

**appspec.yml para Lambda:**
```yaml
version: 0.0
Resources:
  - LambdaFunction:
      Type: AWS::Lambda::Function
      Properties:
        Name: lab-lambda-deploy
        Alias: live
        CurrentVersion: "1"
        TargetVersion: "2"
Hooks:
  - BeforeAllowTraffic: lab-lambda-pre-traffic
  - AfterAllowTraffic: lab-lambda-post-traffic
```

```
CodeDeploy → Applications → "Create application"
  Name: lab-lambda-app
  Platform: AWS Lambda

Deployment Group:
  Name: dg-lambda-canary
  Service role: rol-codedeploy-ec2 (o crear uno específico para Lambda)
  Deployment config: CodeDeployDefault.LambdaCanary10Percent5Minutes

Realizar deployment:
  CodeDeploy → "lab-lambda-app" → "Create deployment"
    Revision location: (S3 bucket con el appspec.yml)
    Deployment group: dg-lambda-canary
  Clic "Deploy"
```

---

### Parte 2 — Pipeline Multi-región con CDK (TypeScript)

```bash
# En tu laptop, crear el proyecto CDK
mkdir ~/lab-pipeline-cdk && cd ~/lab-pipeline-cdk
npm install -g aws-cdk
cdk init app --language typescript
npm install @aws-cdk/pipelines
```

```typescript
// lib/pipeline-stack.ts
import * as cdk from 'aws-cdk-lib';
import * as codeconnections from 'aws-cdk-lib/aws-codeconnections';
import * as codepipeline from 'aws-cdk-lib/aws-codepipeline';
import * as codepipeline_actions from 'aws-cdk-lib/aws-codepipeline-actions';
import * as codebuild from 'aws-cdk-lib/aws-codebuild';
import * as sns from 'aws-cdk-lib/aws-sns';
import * as sns_subscriptions from 'aws-cdk-lib/aws-sns-subscriptions';
import { Construct } from 'constructs';

export class PipelineStack extends cdk.Stack {
  constructor(scope: Construct, id: string, props?: cdk.StackProps) {
    super(scope, id, props);

    // --- ARTEFACTOS ---
    const sourceOutput  = new codepipeline.Artifact('SourceOutput');
    const buildOutput   = new codepipeline.Artifact('BuildOutput');

    // --- PROYECTO CODEBUILD ---
    const buildProject = new codebuild.PipelineProject(this, 'BuildProject', {
      projectName: 'cdk-build-project',
      environment: {
        buildImage: codebuild.LinuxBuildImage.AMAZON_LINUX_2_5,
        computeType: codebuild.ComputeType.SMALL,
      },
      buildSpec: codebuild.BuildSpec.fromObject({
        version: '0.2',
        phases: {
          install:   { commands: ['npm ci'] },
          build:     { commands: ['npm test', 'npm run build'] },
        },
        artifacts: { files: ['**/*'], 'exclude-paths': ['node_modules/**'] },
      }),
    });

    // --- SNS para aprobación ---
    const approvalTopic = new sns.Topic(this, 'ApprovalTopic', {
      topicName: 'cdk-pipeline-approval',
    });
    approvalTopic.addSubscription(
      new sns_subscriptions.EmailSubscription('tu@email.com')
    );

    // --- PIPELINE ---
    const pipeline = new codepipeline.Pipeline(this, 'AppPipeline', {
      pipelineName: 'cdk-multi-region-pipeline',
      crossAccountKeys: false,
    });

    // Etapa 1: Source
    pipeline.addStage({
      stageName: 'Source',
      actions: [
        new codepipeline_actions.CodeStarConnectionsSourceAction({
          actionName:     'GitHub_Source',
          owner:          'TU-USUARIO-GITHUB',
          repo:           'lab-app-aws',
          branch:         'main',
          connectionArn:  'arn:aws:codeconnections:us-east-1:ACCOUNT:connection/XXXX',
          output:         sourceOutput,
        }),
      ],
    });

    // Etapa 2: Build
    pipeline.addStage({
      stageName: 'Build',
      actions: [
        new codepipeline_actions.CodeBuildAction({
          actionName: 'Build_Test',
          project:    buildProject,
          input:      sourceOutput,
          outputs:    [buildOutput],
        }),
      ],
    });

    // Etapa 3: Aprobación manual
    pipeline.addStage({
      stageName: 'Aprobacion',
      actions: [
        new codepipeline_actions.ManualApprovalAction({
          actionName:        'Aprobar_Deploy',
          notificationTopic: approvalTopic,
          additionalInformation: 'Revisar QA antes de aprobar el deploy a produccion',
        }),
      ],
    });

    // Etapa 4: Deploy us-east-1
    pipeline.addStage({
      stageName: 'Deploy_US_East_1',
      actions: [
        new codepipeline_actions.CodeDeployEcsDeployAction({
          actionName:      'Deploy_ECS_US_East_1',
          deploymentGroup: /* tu ECS deployment group */ undefined as any,
          taskDefinitionTemplateInput: buildOutput,
          appSpecTemplateInput:        buildOutput,
        }),
      ],
    });

    // Output: ARN del pipeline
    new cdk.CfnOutput(this, 'PipelineArn', {
      value:       pipeline.pipelineArn,
      description: 'ARN del pipeline CDK',
    });
  }
}
```

```bash
# Desplegar el stack CDK
cdk bootstrap   # solo la primera vez por cuenta/región
cdk synth       # verificar que compila sin errores
cdk deploy      # desplegar el stack en AWS
```

---

## 🧹 LIMPIEZA (en este orden)

```
1.  CodePipeline → "pipeline-lab-node" → Delete pipeline

2.  CodeDeploy → Applications:
    "lab-app-ec2" → Delete application
    "lab-lambda-app" → Delete application

3.  Lambda → "lab-lambda-deploy" → Delete function

4.  CloudWatch → Alarms → "alarma-cpu-asg-lab" → Delete

5.  SNS → Topics → "notif-aprobacion-deploy" → Delete topic
    SNS → Subscriptions → eliminar la suscripción de email

6.  IAM → Roles → "rol-codedeploy-ec2" → Delete

7.  CDK (si desplegaste):
    cd ~/lab-pipeline-cdk
    cdk destroy
```

---

## 📌 COMANDOS CLI DE REFERENCIA RÁPIDA

```bash
# Ver pipelines
aws codepipeline list-pipelines \
  --query 'pipelines[*].[name,created]' \
  --output table

# Ver estado del pipeline
aws codepipeline get-pipeline-state \
  --name pipeline-lab-node \
  --query 'stageStates[*].[stageName,latestExecution.status]' \
  --output table

# Iniciar pipeline manualmente
aws codepipeline start-pipeline-execution \
  --name pipeline-lab-node

# Ver deployments de CodeDeploy
aws deploy list-deployments \
  --application-name lab-app-ec2 \
  --deployment-group-name dg-ec2-lab \
  --query 'deployments[0:5]' \
  --output table

# Ver detalle de un deployment
aws deploy get-deployment \
  --deployment-id d-XXXXXXXXX \
  --query 'deploymentInfo.[status,createTime,completeTime]' \
  --output table

# Aprobar etapa manual desde CLI
aws codepipeline put-approval-result \
  --pipeline-name pipeline-lab-node \
  --stage-name Aprobacion-QA \
  --action-name aprobar-deploy-produccion \
  --result "summary=OK desde CLI,status=Approved" \
  --token XXXXX

# Ver historial de actividad del deployment
aws deploy list-deployment-instances \
  --deployment-id d-XXXXXXXXX \
  --output table
```
