# Módulo 9 — Integración End-to-End, Operación y Cierre
### Servicios: EC2 · EBS · RDS · S3 · Route 53 · CodePipeline · CloudWatch · CDK

---

## TEORÍA

### Arquitectura de 3 capas
La arquitectura clásica para aplicaciones web en AWS:
- **Capa 1 — Web / Load Balancer**: ALB en subnets públicas, recibe tráfico de internet
- **Capa 2 — Aplicación**: EC2 en subnets privadas (Auto Scaling Group)
- **Capa 3 — Datos**: RDS en subnets privadas, solo accesible desde la capa de app

### CloudWatch — Monitoreo
- **Metrics**: datos numéricos de servicios AWS (CPU, conexiones, latencia)
- **Logs**: texto de logs de aplicaciones y servicios
- **Alarms**: notificación o acción automática cuando una métrica supera un umbral
- **Dashboards**: vista consolidada de métricas relevantes

### Well-Architected Framework — 6 Pilares
1. **Excelencia operacional**: automatizar, anticipar fallas, aprender de errores
2. **Seguridad**: cifrado, mínimo privilegio, trazabilidad
3. **Confiabilidad**: recuperación automática, escalar horizontalmente
4. **Eficiencia de rendimiento**: uso de recursos correcto para cada carga
5. **Optimización de costos**: eliminar desperdicio, usar pricing adecuado
6. **Sostenibilidad**: minimizar impacto ambiental

---

## 🟢 LAB BÁSICO — Arquitectura 3 capas + Dashboard CloudWatch

**Tiempo estimado: 60 min**
**Objetivo:** Conectar todos los servicios de los módulos anteriores y agregar monitoreo

---

### Paso 1 — Verificar que todos los recursos del lab existen

```bash
# Verificar VPC
aws ec2 describe-vpcs \
  --filters "Name=tag:Name,Values=lab-vpc-01" \
  --query 'Vpcs[0].[VpcId,CidrBlock]' \
  --output text

# Verificar EC2
aws ec2 describe-instances \
  --filters "Name=tag:Name,Values=lab-web-01" "Name=instance-state-name,Values=running" \
  --query 'Reservations[0].Instances[0].[InstanceId,PublicIpAddress]' \
  --output text

# Verificar RDS
aws rds describe-db-instances \
  --db-instance-identifier lab-mysql-01 \
  --query 'DBInstances[0].[DBInstanceIdentifier,DBInstanceStatus,Endpoint.Address]' \
  --output text

# Verificar S3
aws s3 ls | grep lab-s3
```

```
Si algún recurso no existe: volver al módulo correspondiente y crearlo.
Arquitectura mínima necesaria para este laboratorio:
  ✅ lab-vpc-01 con subnets públicas y privadas
  ✅ lab-web-01 (EC2 en subnet pública o privada)
  ✅ lab-mysql-01 (RDS en subnet privada)
  ✅ lab-s3-TUAPELLIDO-2024 (bucket S3)
  ✅ alb-web-lab (Application Load Balancer)
  ✅ asg-web-lab (Auto Scaling Group)
```

---

### Paso 2 — Conectar la app EC2 con RDS y S3

```bash
# Conectarse a la instancia EC2
ssh -i ~/Downloads/lab-key-ec2.pem ec2-user@54.xxx.xxx.xxx
# O usar Instance Connect desde la consola

# Instalar cliente MySQL (si no está instalado)
sudo yum install mysql -y

# Probar conexión con RDS
mysql -h lab-mysql-01.xxxxxxxxxx.us-east-1.rds.amazonaws.com \
      -u admin -pLab@12345! \
      -e "SELECT 'Conexion RDS OK' AS status; USE labapp; SELECT COUNT(*) as total FROM empleados;"
# ✅ Debe mostrar: Conexion RDS OK | total: 4

# Probar acceso a S3 (con el rol IAM adjunto a la instancia)
aws s3 ls s3://lab-s3-TUAPELLIDO-2024/
# ✅ Debe listar los objetos del bucket
aws s3 cp s3://lab-s3-TUAPELLIDO-2024/index.html /tmp/index-s3.html
cat /tmp/index-s3.html
# ✅ Debe mostrar el contenido del archivo
```

---

### Paso 3 — Habilitar CloudWatch Agent en EC2

```bash
# Instalar el CloudWatch Agent
sudo yum install amazon-cloudwatch-agent -y

# Crear configuración del agente
sudo mkdir -p /opt/aws/amazon-cloudwatch-agent/etc/

sudo cat > /opt/aws/amazon-cloudwatch-agent/etc/amazon-cloudwatch-agent.json << 'EOF'
{
  "agent": {
    "metrics_collection_interval": 60,
    "run_as_user": "root"
  },
  "logs": {
    "logs_collected": {
      "files": {
        "collect_list": [
          {
            "file_path": "/var/log/httpd/access_log",
            "log_group_name": "/ec2/lab-web-01/apache-access",
            "log_stream_name": "{instance_id}",
            "timestamp_format": "%d/%b/%Y:%H:%M:%S %z"
          },
          {
            "file_path": "/var/log/httpd/error_log",
            "log_group_name": "/ec2/lab-web-01/apache-error",
            "log_stream_name": "{instance_id}"
          }
        ]
      }
    }
  },
  "metrics": {
    "namespace": "LabApp/EC2",
    "metrics_collected": {
      "cpu": {
        "measurement": ["cpu_usage_idle", "cpu_usage_user", "cpu_usage_system"],
        "metrics_collection_interval": 60,
        "totalcpu": true
      },
      "disk": {
        "measurement": ["used_percent"],
        "metrics_collection_interval": 60,
        "resources": ["/", "/datos"]
      },
      "mem": {
        "measurement": ["mem_used_percent"],
        "metrics_collection_interval": 60
      }
    }
  }
}
EOF

# Iniciar el agente con la configuración
sudo /opt/aws/amazon-cloudwatch-agent/bin/amazon-cloudwatch-agent-ctl \
  -a fetch-config \
  -m ec2 \
  -s \
  -c file:/opt/aws/amazon-cloudwatch-agent/etc/amazon-cloudwatch-agent.json

# Verificar que está corriendo
sudo systemctl status amazon-cloudwatch-agent
# Active: active (running) ✅

# Habilitar inicio automático
sudo systemctl enable amazon-cloudwatch-agent
```

---

### Paso 4 — Crear Dashboard CloudWatch

```
Consola → CloudWatch → "Dashboards" → "Create dashboard"

  Dashboard name: dashboard-lab-completo
Clic "Create dashboard"

Agregar widgets:
```

**Widget 1 — CPU del EC2:**
```
"Add widget" → Line → "Next"
Metrics:
  EC2 → Per-Instance Metrics
  Instance: lab-web-01
  Metric: CPUUtilization
  Period: 1 minute
  Statistic: Average
Title: "CPU - lab-web-01"
Clic "Create widget"
```

**Widget 2 — Requests del ALB:**
```
"Add widget" → Line → "Next"
Metrics:
  ApplicationELB → Per AppELB Metrics
  LoadBalancer: alb-web-lab
  Metrics: RequestCount, HTTPCode_ELB_4XX_Count, HTTPCode_ELB_5XX_Count
Title: "Requests ALB"
Clic "Create widget"
```

**Widget 3 — Conexiones a RDS:**
```
"Add widget" → Line → "Next"
Metrics:
  RDS → Per-Database Metrics
  DBInstanceIdentifier: lab-mysql-01
  Metrics: DatabaseConnections, CPUUtilization, FreeableMemory
Title: "RDS lab-mysql-01"
Clic "Create widget"
```

**Widget 4 — Instancias del Auto Scaling Group:**
```
"Add widget" → Number → "Next"
Metrics:
  EC2 → By Auto Scaling Group
  AutoScalingGroupName: asg-web-lab
  Metric: GroupInServiceInstances
Title: "Instancias en servicio"
Clic "Create widget"
```

```
Organizar los widgets en la grilla → "Save dashboard"
✅ Tienes un dashboard completo de tu arquitectura
```

---

## 🟡 LAB INTERMEDIO — Pipeline CI/CD Completo + Alarmas + SSM + CloudTrail

**Tiempo estimado: 75 min**

---

### Paso 1 — Verificar el pipeline end-to-end

```
En este punto deberías tener:
  ✅ pipeline-lab-node: GitHub → CodeBuild → CodeDeploy → EC2

Si no existe, seguir los pasos del Módulo 8.

Hacer un cambio de código para disparar el pipeline completo:
```

```bash
# En ~/lab-app en tu laptop:
# Modificar app.js para agregar nueva ruta
cat >> app.js << 'EOF'

app.get('/info', (req, res) => {
  res.json({
    servicio: 'Lab App AWS',
    version:  '2.0.0',
    modulos:  ['VPC', 'EC2', 'EBS', 'S3', 'RDS', 'Route53', 'CodeBuild', 'CodeDeploy'],
    build:    process.env.CODEBUILD_BUILD_ID || 'local'
  });
});
EOF

git add app.js
git commit -m "feat: agregar ruta /info con version 2.0.0"
git push

# En la consola AWS, observar:
# CodePipeline → "pipeline-lab-node" → el pipeline se dispara automáticamente
# Source: descarga el commit → ✅
# Build: ejecuta tests → ✅
# Deploy: despliega en EC2 → ✅
```

```bash
# Verificar que la nueva ruta fue desplegada
curl http://alb-web-lab-xxxxxxxxx.us-east-1.elb.amazonaws.com/info
# {"servicio":"Lab App AWS","version":"2.0.0",...}  ✅
```

---

### Paso 2 — Crear Alarma de escalado con acción SNS

```
CloudWatch → "Alarms" → "Create alarm"

  Select metric:
    EC2 → By Auto Scaling Group
    AutoScalingGroupName: asg-web-lab
    Metric: CPUUtilization
  
  Clic "Select metric"
  
  Period: 1 minute
  Statistics: Average
  
  Conditions:
    Threshold type: Static
    Whenever CPUUtilization is: Greater than
    Value: 80
  
  Additional configuration:
    Datapoints to alarm: 2 out of 2
    Missing data treatment: Treat missing data as not breaching
  
  Notification:
    Alarm state trigger: In alarm
    SNS topic: (crear nuevo)
      Topic name: alarma-cpu-alta
      Email: tu@email.com
  
  Alarm name: alarma-cpu-asg-por-encima-80
Clic "Create alarm"
⚠️  Ir al email y confirmar la suscripción SNS
```

---

### Paso 3 — SSM Session Manager (acceso seguro sin SSH)

```
Session Manager es una alternativa más segura a SSH:
- No necesitas puertos abiertos (ni el 22)
- No necesitas administrar llaves SSH
- Todas las sesiones se registran en CloudWatch/S3
- Funciona con el rol IAM "AmazonSSMManagedInstanceCore"

Prerequisito: la instancia debe tener:
  1. El rol IAM con política AmazonSSMManagedInstanceCore
  2. El SSM Agent instalado (viene preinstalado en Amazon Linux 2023)
```

```bash
# Verificar que el SSM Agent está corriendo en la instancia
sudo systemctl status amazon-ssm-agent
# Active: active (running) ✅
```

```
Desde la consola, iniciar sesión SSM:
  EC2 → Instances → seleccionar "lab-web-01"
  Clic "Connect"
  Pestaña "Session Manager"
  Clic "Connect"
  → Se abre una terminal en el browser sin usar SSH

Desde CLI:
```

```bash
# Desde tu laptop con AWS CLI
aws ssm start-session \
  --target i-xxxxxxxxxxxxxxxxx \
  --region us-east-1
# ✅ Sesión interactiva sin llaves SSH

# Dentro de la sesión SSM:
whoami           # ssm-user
ls /var/www/app  # ver los archivos de la app
pm2 list         # ver procesos Node.js
exit
```

```
Ver historial de sesiones:
  Systems Manager → Session Manager → "Session history"
  Verás todas las sesiones: quién conectó, cuándo, duración
```

---

### Paso 4 — Habilitar CloudTrail (auditoría)

```
Consola → buscar "CloudTrail" → clic en CloudTrail

Si ya existe un trail: saltar al paso de verificación.
Si no existe:

  Clic "Create trail"
  
  Trail name: lab-trail-auditoria
  
  Storage location:
    ✅ Create new S3 bucket
    Bucket name: lab-cloudtrail-TUAPELLIDO-2024
  
  Log file SSE-KMS encryption: Desmarcar (para simplificar el lab)
  
  CloudWatch Logs:
    ✅ Enabled
    Log group: /cloudtrail/lab-audit
    IAM Role: (crear nuevo automáticamente)
  
  Events:
    ✅ Management events (acciones sobre recursos AWS)
    ✅ Read events
    ✅ Write events
    
    Data events: (no necesario para el lab)
    
  Clic "Create trail"
```

```bash
# Realizar algunas acciones para generar eventos
aws ec2 describe-instances --region us-east-1
aws s3 ls
aws iam list-users

# Ver los eventos en CloudTrail (demora ~15 minutos en aparecer)
# CloudTrail → "Event history"
# Filtrar por:
#   Event name: DescribeInstances
#   User name: (tu usuario IAM)
```

---

## 🔴 LAB AVANZADO — Arquitectura completa con CDK + WAF + GuardDuty + X-Ray

**Tiempo estimado: 90 min**

---

### Parte 1 — Desplegar arquitectura completa con AWS CDK

```bash
# Crear directorio del proyecto CDK
mkdir ~/lab-arquitectura-cdk && cd ~/lab-arquitectura-cdk
cdk init app --language typescript
```

```typescript
// lib/arquitectura-stack.ts
import * as cdk from 'aws-cdk-lib';
import * as ec2 from 'aws-cdk-lib/aws-ec2';
import * as rds from 'aws-cdk-lib/aws-rds';
import * as elbv2 from 'aws-cdk-lib/aws-elasticloadbalancingv2';
import * as autoscaling from 'aws-cdk-lib/aws-autoscaling';
import * as s3 from 'aws-cdk-lib/aws-s3';
import * as cloudwatch from 'aws-cdk-lib/aws-cloudwatch';
import * as cloudwatch_actions from 'aws-cdk-lib/aws-cloudwatch-actions';
import * as sns from 'aws-cdk-lib/aws-sns';
import * as sns_subs from 'aws-cdk-lib/aws-sns-subscriptions';
import { Construct } from 'constructs';

export class ArquitecturaStack extends cdk.Stack {
  constructor(scope: Construct, id: string, props?: cdk.StackProps) {
    super(scope, id, props);

    // =====================
    // CAPA 1: RED (VPC)
    // =====================
    const vpc = new ec2.Vpc(this, 'VpcPrincipal', {
      vpcName: 'cdk-vpc-lab',
      maxAzs:  2,
      natGateways: 1,   // 1 NAT Gateway (en producción: uno por AZ)
      subnetConfiguration: [
        {
          name:               'Public',
          subnetType:         ec2.SubnetType.PUBLIC,
          cidrMask:           24,
        },
        {
          name:               'Private',
          subnetType:         ec2.SubnetType.PRIVATE_WITH_EGRESS,
          cidrMask:           24,
        },
        {
          name:               'Isolated',
          subnetType:         ec2.SubnetType.PRIVATE_ISOLATED,
          cidrMask:           24,
        },
      ],
    });

    // =====================
    // CAPA 3: BASE DE DATOS (RDS Aurora)
    // =====================
    const dbSecurityGroup = new ec2.SecurityGroup(this, 'DbSG', {
      vpc,
      description: 'SG para RDS Aurora',
    });

    const auroraCluster = new rds.DatabaseCluster(this, 'AuroraCluster', {
      engine: rds.DatabaseClusterEngine.auroraMysql({
        version: rds.AuroraMysqlEngineVersion.VER_3_04_0,
      }),
      writer: rds.ClusterInstance.serverlessV2('writer', {
        scaleWithWriter: true,
      }),
      readers: [
        rds.ClusterInstance.serverlessV2('reader'),
      ],
      serverlessV2MinCapacity: 0.5,
      serverlessV2MaxCapacity: 4,
      vpc,
      vpcSubnets: { subnetType: ec2.SubnetType.PRIVATE_ISOLATED },
      securityGroups: [dbSecurityGroup],
      defaultDatabaseName: 'appdb',
      backup: {
        retention:        cdk.Duration.days(7),
        preferredWindow:  '03:00-04:00',
      },
      deletionProtection: false,  // en producción: true
      removalPolicy:      cdk.RemovalPolicy.DESTROY,  // solo para labs
    });

    // =====================
    // CAPA 2: APLICACIÓN (EC2 Auto Scaling)
    // =====================
    const appSecurityGroup = new ec2.SecurityGroup(this, 'AppSG', {
      vpc,
      description: 'SG para instancias de aplicacion',
    });

    // Permitir que la app acceda a RDS
    dbSecurityGroup.addIngressRule(
      appSecurityGroup,
      ec2.Port.tcp(3306),
      'App accede a MySQL'
    );

    const asg = new autoscaling.AutoScalingGroup(this, 'AppASG', {
      vpc,
      vpcSubnets: { subnetType: ec2.SubnetType.PRIVATE_WITH_EGRESS },
      securityGroup: appSecurityGroup,
      instanceType: ec2.InstanceType.of(ec2.InstanceClass.T3, ec2.InstanceSize.MICRO),
      machineImage: ec2.MachineImage.latestAmazonLinux2023(),
      minCapacity: 1,
      desiredCapacity: 2,
      maxCapacity: 4,
      cooldown: cdk.Duration.seconds(300),
    });

    // Escalar por CPU
    asg.scaleOnCpuUtilization('ScaleCpu', {
      targetUtilizationPercent: 70,
      cooldown: cdk.Duration.seconds(300),
    });

    // =====================
    // CAPA 1: WEB (ALB)
    // =====================
    const albSecurityGroup = new ec2.SecurityGroup(this, 'AlbSG', {
      vpc,
      description: 'SG para ALB',
      allowAllOutbound: true,
    });
    albSecurityGroup.addIngressRule(
      ec2.Peer.anyIpv4(),
      ec2.Port.tcp(80),
      'HTTP publico'
    );
    albSecurityGroup.addIngressRule(
      ec2.Peer.anyIpv4(),
      ec2.Port.tcp(443),
      'HTTPS publico'
    );

    const alb = new elbv2.ApplicationLoadBalancer(this, 'ALB', {
      vpc,
      internetFacing: true,
      securityGroup: albSecurityGroup,
      vpcSubnets: { subnetType: ec2.SubnetType.PUBLIC },
    });

    const targetGroup = new elbv2.ApplicationTargetGroup(this, 'TargetGroup', {
      vpc,
      port: 3000,
      protocol: elbv2.ApplicationProtocol.HTTP,
      targets: [asg],
      healthCheck: {
        path: '/health',
        healthyHttpCodes: '200',
        interval: cdk.Duration.seconds(30),
        timeout:  cdk.Duration.seconds(5),
        healthyThresholdCount:   2,
        unhealthyThresholdCount: 3,
      },
    });

    alb.addListener('HttpListener', {
      port: 80,
      defaultTargetGroups: [targetGroup],
    });

    // Permitir ALB → App
    appSecurityGroup.addIngressRule(
      albSecurityGroup,
      ec2.Port.tcp(3000),
      'ALB a App'
    );

    // =====================
    // ALMACENAMIENTO: S3
    // =====================
    const appBucket = new s3.Bucket(this, 'AppBucket', {
      bucketName:       `cdk-app-bucket-${this.account}-${this.region}`,
      versioned:        true,
      encryption:       s3.BucketEncryption.S3_MANAGED,
      blockPublicAccess: s3.BlockPublicAccess.BLOCK_ALL,
      removalPolicy:    cdk.RemovalPolicy.DESTROY,  // solo para labs
      autoDeleteObjects: true,
      lifecycleRules: [
        {
          id:         'mover-a-ia-30-dias',
          transitions: [{
            storageClass:        s3.StorageClass.INFREQUENT_ACCESS,
            transitionAfter:     cdk.Duration.days(30),
          }],
        },
      ],
    });

    // Dar acceso al ASG al bucket
    appBucket.grantReadWrite(asg.role);

    // =====================
    // MONITOREO: CloudWatch
    // =====================
    const alertaTopic = new sns.Topic(this, 'AlertaTopic', {
      topicName: 'cdk-alertas-app',
    });
    alertaTopic.addSubscription(
      new sns_subs.EmailSubscription('tu@email.com')
    );

    // Alarma de CPU del ASG
    const alarmaCpu = new cloudwatch.Alarm(this, 'AlarmaCpu', {
      alarmName:          'cdk-cpu-alta-asg',
      metric:             new cloudwatch.Metric({
        namespace:        'AWS/EC2',
        metricName:       'CPUUtilization',
        dimensionsMap:    { AutoScalingGroupName: asg.autoScalingGroupName },
        statistic:        'Average',
        period:           cdk.Duration.minutes(1),
      }),
      threshold:          80,
      evaluationPeriods:  2,
      comparisonOperator: cloudwatch.ComparisonOperator.GREATER_THAN_THRESHOLD,
      alarmDescription:   'CPU del ASG por encima del 80%',
    });
    alarmaCpu.addAlarmAction(new cloudwatch_actions.SnsAction(alertaTopic));

    // =====================
    // OUTPUTS
    // =====================
    new cdk.CfnOutput(this, 'AlbDns', {
      value:       alb.loadBalancerDnsName,
      description: 'DNS del Application Load Balancer',
    });

    new cdk.CfnOutput(this, 'RdsEndpoint', {
      value:       auroraCluster.clusterEndpoint.hostname,
      description: 'Endpoint del cluster Aurora',
    });

    new cdk.CfnOutput(this, 'BucketName', {
      value:       appBucket.bucketName,
      description: 'Nombre del bucket S3',
    });
  }
}
```

```bash
# Instalar dependencias CDK
npm install

# Verificar que el stack compila sin errores
cdk synth

# Ver qué recursos va a crear (diff)
cdk diff

# Desplegar (tarda ~15-20 minutos)
cdk deploy

# Al finalizar verás los outputs:
# ArquitecturaStack.AlbDns = cdk-alb-xxx.us-east-1.elb.amazonaws.com
# ArquitecturaStack.RdsEndpoint = cdk-aurora-xxx.cluster-xxx.us-east-1.rds.amazonaws.com
# ArquitecturaStack.BucketName = cdk-app-bucket-123456789012-us-east-1
```

---

### Parte 2 — Habilitar WAF en el ALB

```
Consola → buscar "WAF" → AWS WAF

Clic "Create web ACL"

  Name: waf-lab-alb
  Resource type: Regional resources (ALB, API GW...)
  Region: us-east-1

  Associated AWS resources:
    Clic "Add AWS resources"
    Resource type: Application Load Balancer
    Seleccionar: cdk-ALB-xxx (el ALB del CDK)
  
  Add rules:
    Clic "Add managed rule groups"
    
    AWS managed rule groups:
      ✅ Core rule set (AWSManagedRulesCommonRuleSet)
         Protege contra: SQL injection, XSS, Path traversal, etc.
      
      ✅ Amazon IP reputation list (AWSManagedRulesAmazonIpReputationList)
         Bloquea IPs conocidas como maliciosas
      
      ✅ SQL database (AWSManagedRulesSQLiRuleSet)
         Protección adicional contra SQL injection
    
    Clic "Add rules"
  
  Default action: Allow   (bloquear solo lo que indican las reglas)
  
  Set rule priority: (dejar el orden por defecto)
  
  Configure metrics:
    CloudWatch metrics: ✅ Enable
    Sampling requests:  ✅ Enable
  
  Clic "Create web ACL"
```

```bash
# Probar que el WAF está funcionando
# Intentar una petición con SQL injection (debe ser bloqueada)
curl "http://cdk-alb-xxx.us-east-1.elb.amazonaws.com/?id=1' OR 1=1--"
# Respuesta esperada: HTTP 403 Forbidden (bloqueado por WAF) ✅

# Petición legítima (debe pasar)
curl "http://cdk-alb-xxx.us-east-1.elb.amazonaws.com/"
# HTTP 200 ✅
```

---

### Parte 3 — Habilitar GuardDuty

```
Consola → buscar "GuardDuty" → clic en GuardDuty
Clic "Get started" → "Enable GuardDuty"

  GuardDuty comienza a analizar:
  - CloudTrail logs
  - VPC Flow Logs
  - DNS logs
  - EKS logs (si tienes EKS)

  Costo: se cobra según el volumen de datos procesados
  El primer mes tiene 30 días de prueba gratuita ✅
```

```
Generar un hallazgo de prueba para ver cómo funciona:
  GuardDuty → Panel izquierdo → "Settings"
  Sección "Sample findings"
  Clic "Generate sample findings"

  → GuardDuty crea findings de ejemplo de todos los tipos
  
  GuardDuty → "Findings"
  Verás hallazgos simulados como:
  - UnauthorizedAccess:EC2/SSHBruteForce
  - CryptoCurrency:EC2/BitcoinTool.B
  - Recon:EC2/PortScan
  
  ✅ En producción, estos serían ataques reales que GuardDuty detectaría
```

---

### Parte 4 — AWS Well-Architected Tool

```
Consola → buscar "Well-Architected Tool" → clic

Clic "Define workload"

  Name:         lab-arquitectura-completa
  Description:  Arquitectura del programa de formación AWS - 9 módulos
  Review owner: (tu nombre)
  Environment:  Pre-production
  
  Regions: us-east-1
  
  Industry type: Internet of Things / Technology
  Industry:      Technology

Clic "Create workload"
```

```
Con el workload creado:
  Clic "Start reviewing" → seleccionar "AWS Well-Architected Framework"

  El tool tiene 6 pilares con preguntas:
  
  Pilar 1 — Excelencia Operacional:
    OPS 1: ¿Cómo determinas las prioridades de tu organización?
    OPS 2: ¿Cómo estructuras tu organización para soportar los resultados del negocio?
    ... (12 preguntas en total)
    
    Responder cada pregunta basándose en lo que implementamos:
    - Usamos IaC con CDK: seleccionar "Infrastructure as code"
    - Usamos CI/CD con CodePipeline: seleccionar "Automate deployment of changes"
    - Tenemos monitoreo con CloudWatch: seleccionar "Monitor workload resources"
  
  Ir respondiendo cada pilar...
```

```
Al finalizar la revisión:
  "Well-Architected Tool" → tu workload → "Generate report"
  
  El reporte muestra:
  - Número de High Risks (riesgos altos a corregir)
  - Número de Medium Risks
  - Recomendaciones específicas por pilar
  
  Ejemplo de recomendaciones que aparecerán en el lab:
  ⚠️  "No encryption at rest configured" → corregir con KMS
  ⚠️  "No multi-region backup strategy" → corregir con CRR en S3
  ✅  "Automated deployment pipeline in use"
  ✅  "Auto Scaling configured"
```

---

## 🧹 LIMPIEZA FINAL — Todos los recursos del programa

```
⚠️  ANTES DE LIMPIAR: revisar que ya no necesitas ningún recurso

Orden correcto para evitar errores de dependencias:

--- MÓDULO 9 ---
1.  WAF → Web ACLs → "waf-lab-alb" → Delete web ACL
2.  GuardDuty → Settings → Suspend GuardDuty (o Disable si terminas el trial)
3.  CDK: cd ~/lab-arquitectura-cdk && cdk destroy
4.  CloudWatch → Dashboards → "dashboard-lab-completo" → Delete
5.  CloudTrail → Trails → "lab-trail-auditoria" → Delete
6.  S3 → "lab-cloudtrail-TUAPELLIDO-2024" → Empty → Delete

--- MÓDULOS 7 y 8 ---
7.  CodePipeline → "pipeline-lab-node" → Delete
8.  CodeDeploy → Applications → eliminar todas
9.  CodeBuild → Projects → "lab-build-node" → Delete
10. ECR → Repositories → "lab-app-repo" → Delete
11. CodeArtifact → Domains → "lab-domain" → Delete repos → Delete domain

--- MÓDULO 6 ---
12. Route 53 → Health checks → eliminar todos
13. Route 53 → Hosted zones → eliminar registros → eliminar zonas

--- MÓDULO 5 ---
14. RDS → Proxies → eliminar proxy
15. RDS → Databases → eliminar Read Replicas → Primary → Aurora Cluster
16. RDS → Subnet groups → eliminar

--- MÓDULO 4 ---
17. S3 → Vaciar y eliminar todos los buckets del lab

--- MÓDULO 3 ---
18. EC2 → Snapshots → eliminar snapshots del lab
19. EC2 → Volumes → Detach → Delete los volúmenes EBS del lab
20. AWS Backup → Plans → eliminar plan → eliminar backup vault (si está vacío)
21. EC2 → Lifecycle Manager → eliminar políticas DLM

--- MÓDULOS 1 y 2 ---
22. EC2 → Auto Scaling Groups → "asg-web-lab" → Delete
23. EC2 → Load Balancers → "alb-web-lab" → Delete
24. EC2 → Target Groups → "tg-web-lab" → Delete
25. EC2 → Launch Templates → "lt-web-app-v1" → Delete
26. EC2 → Instances → Terminate todas las instancias del lab
27. EC2 → AMIs → Deregister → Delete snapshots asociados
28. VPC → NAT Gateways → Delete → esperar "Deleted"
29. VPC → Elastic IPs → Release
30. VPC → Endpoints → Delete todos
31. VPC → Subnets → Delete todas las del lab
32. VPC → Internet Gateways → Detach → Delete
33. VPC → Route Tables → Delete todas (excepto la main)
34. VPC → Peering Connections → Delete
35. VPC → Your VPCs → Delete lab-vpc-01 y lab-vpc-02

--- IAM ---
36. IAM → Roles → eliminar todos los roles del lab
37. IAM → Policies → eliminar todas las políticas custom del lab
38. IAM → Users → eliminar "lab-usuario-01"

--- KMS ---
39. KMS → Customer managed keys → Schedule deletion (mínimo 7 días)

--- SNS ---
40. SNS → Topics → eliminar todos los del lab
    SNS → Subscriptions → eliminar todas

--- SSM/Parameter Store ---
41. Systems Manager → Parameter Store → eliminar parámetros /lab/...

--- CloudWatch ---
42. CloudWatch → Log groups → eliminar grupos del lab:
    /codebuild/lab-build-node
    /ec2/lab-web-01/*
    /cloudtrail/lab-audit
    /aws/lambda/procesar-imagen-s3
43. CloudWatch → Alarms → eliminar todas las alarmas del lab
```

---

## 📊 RESUMEN FINAL — Lo que construimos en 9 módulos

```
╔══════════════════════════════════════════════════════════════╗
║          ARQUITECTURA COMPLETA - PROGRAMA AWS 36H            ║
╠══════════════════════════════════════════════════════════════╣
║                                                              ║
║  Internet → Route 53 → ALB (WAF) → EC2 ASG → RDS Aurora     ║
║               │                       │           │          ║
║               │                   S3 Bucket   CloudWatch     ║
║               │                       │                      ║
║          CloudTrail               CodePipeline               ║
║          GuardDuty              (GitHub→Build→Deploy)        ║
║                                                              ║
║  Módulo 1:  VPC, Subnets, IGW, NAT, IAM, Route 53           ║
║  Módulo 2:  EC2, AMI, Security Groups, ASG, ALB              ║
║  Módulo 3:  EBS Volumes, Snapshots, DLM, AWS Backup          ║
║  Módulo 4:  S3, Versioning, Lifecycle, CRR, Lambda Events    ║
║  Módulo 5:  RDS Multi-AZ, Read Replica, Aurora Serverless    ║
║  Módulo 6:  Route 53 Failover, Weighted, Health Checks       ║
║  Módulo 7:  CodeBuild, CodeArtifact, ECR, Docker             ║
║  Módulo 8:  CodePipeline, CodeDeploy Blue/Green              ║
║  Módulo 9:  CloudWatch, WAF, GuardDuty, CDK, WAF             ║
╚══════════════════════════════════════════════════════════════╝
```

---

## 📌 COMANDOS CLI DE REFERENCIA RÁPIDA

```bash
# Ver todos los recursos del lab (resumen)
echo "=== VPCs ==="
aws ec2 describe-vpcs \
  --query 'Vpcs[*].[VpcId,CidrBlock,Tags[?Key==`Name`].Value|[0]]' \
  --output table

echo "=== EC2 Instances ==="
aws ec2 describe-instances \
  --filters "Name=instance-state-name,Values=running" \
  --query 'Reservations[*].Instances[*].[InstanceId,InstanceType,Tags[?Key==`Name`].Value|[0]]' \
  --output table

echo "=== RDS ==="
aws rds describe-db-instances \
  --query 'DBInstances[*].[DBInstanceIdentifier,DBInstanceStatus,Engine]' \
  --output table

echo "=== S3 Buckets ==="
aws s3 ls

echo "=== Load Balancers ==="
aws elbv2 describe-load-balancers \
  --query 'LoadBalancers[*].[LoadBalancerName,State.Code,DNSName]' \
  --output table

echo "=== Auto Scaling Groups ==="
aws autoscaling describe-auto-scaling-groups \
  --query 'AutoScalingGroups[*].[AutoScalingGroupName,DesiredCapacity,MinSize,MaxSize]' \
  --output table

echo "=== CodePipeline ==="
aws codepipeline list-pipelines \
  --query 'pipelines[*].[name,created]' \
  --output table

# Estimar costo mensual de los recursos activos
# Usar: https://calculator.aws/pricing/2/estimator
```
