# 📚 Laboratorios AWS — Programa completo

> Cada módulo tiene teoría directa + demos paso a paso en 3 niveles

---

## Índice de módulos

| # | Módulo | Servicios | Archivo |
|---|--------|-----------|---------|
| 1 | Fundamentos AWS y red | IAM · VPC · Route 53 | [modulo-01](./modulo-01-iam-vpc-route53.md) |
| 2 | Cómputo EC2 | EC2 · AMI · ASG · ALB | [modulo-02](./modulo-02-ec2.md) |
| 3 | Almacenamiento en bloque | EBS · Snapshots · DLM | [modulo-03](./modulo-03-ebs.md) |
| 4 | Almacenamiento de objetos | S3 · Políticas · Replicación | [modulo-04](./modulo-04-s3.md) |
| 5 | Bases de datos | RDS · Aurora · Multi-AZ | [modulo-05](./modulo-05-rds.md) |
| 6 | DNS y enrutamiento | Route 53 · Health Checks | [modulo-06](./modulo-06-route53.md) |
| 7 | Artefactos y build | CodeBuild · CodeArtifact · ECR | [modulo-07](./modulo-07-codebuild-codeartifact.md) |
| 8 | Pipeline y despliegue | CodePipeline · CodeDeploy | [modulo-08](./modulo-08-codepipeline-codedeploy.md) |
| 9 | Integración end-to-end | Todo + CloudWatch · CDK · WAF | [modulo-09](./modulo-09-integracion-end-to-end.md) |
| 10 | Contenedores | ECR · ECS Fargate · EKS | [modulo-10](./modulo-10-contenedores-ecr-ecs-eks.md) |

---

## Estructura de cada módulo

```
TEORÍA    → conceptos clave en lenguaje directo
🟢 BÁSICO  → crear servicios desde cero en la consola, comandos exactos
🟡 MEDIO   → configurar HA, seguridad y automatización
🔴 AVANZADO → arquitecturas de producción con IaC y CI/CD
🧹 LIMPIEZA → eliminar recursos en el orden correcto
📌 CLI     → comandos de referencia rápida
```

---

## Prerequisitos

- Cuenta AWS (preferiblemente nueva o con créditos)
- AWS CLI v2 instalado y configurado
- Node.js 18+ instalado (módulos 7-8)
- Docker instalado (módulos 7 y 10)
- Cuenta GitHub (módulos 7-8)
- `eksctl` y `kubectl` instalados (módulo 10 avanzado)
- `helm` instalado (módulo 10 avanzado)
- Región recomendada: **us-east-1** (N. Virginia)

```bash
# Instalar herramientas del módulo 10 en macOS
brew tap weaveworks/tap
brew install weaveworks/tap/eksctl kubectl helm
```

## ⚠️  Costos estimados por módulo (modo lab, recursos mínimos)

| Módulo | Recursos | Costo aprox/4h |
|--------|----------|----------------|
| 1 | VPC, IAM, Route 53 | $0.00 |
| 2 | EC2 t3.micro, ALB | ~$0.50 |
| 3 | EBS 20GB, Snapshots | ~$0.10 |
| 4 | S3 | ~$0.01 |
| 5 | RDS db.t3.micro | ~$0.80 |
| 6 | Route 53 Hosted Zone | ~$0.50/mes |
| 7 | CodeBuild, ECR | ~$0.10 |
| 8 | CodePipeline | $1.00/mes por pipeline |
| 9 | Integración | suma de anteriores |
| 10 (ECS) | Fargate 2 tasks 0.25vCPU | ~$0.10 |
| 10 (EKS) | Control plane + 2× t3.small | ~$1.50 ⚠️ |

> ⚠️  **EKS cobra $0.10/hora solo por el control plane**. Eliminar el cluster al terminar el lab.

**Eliminar todos los recursos al finalizar cada sesión para evitar cargos.**
