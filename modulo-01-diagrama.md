# Módulo 1 — Diagrama de arquitectura completo

Este diagrama resume todo lo construido a lo largo del Módulo 1 (IAM + VPC + Route 53), incluyendo los 3 niveles de laboratorio: Básico, Intermedio y Avanzado.

```mermaid
graph TB
    Internet((("🌐 Internet")))

    subgraph R53["Route 53"]
        direction TB
        PubApex["📍 Zona Pública<br/>joedayz123.com<br/><i>(producción, apunta a CloudFront — NO TOCAR)</i>"]
        LabZone["📍 Zona Pública Lab<br/>lab.joedayz123.com<br/>Registro: app (Failover: primary/secondary)"]
        PrivZone["📍 Zona Privada<br/>lab.interno<br/>servidor01 → 10.0.1.100<br/>⚠️ Resolver privado no resuelve en este sandbox"]
    end

    subgraph AWS["Cuenta AWS — us-east-1"]
        direction TB

        subgraph VPC1["VPC lab-vpc-01 — 10.0.0.0/16"]
            direction TB
            IGW1["🌐 Internet Gateway"]
            NAT["🔁 NAT Gateway<br/>nat-034c8989514aee8c7"]
            S3EP["🔗 S3 Gateway Endpoint<br/>vpce-024eaacec1f0c0c6b<br/>Policy: Full Access"]

            subgraph PubSub1["Subnet pública 10.0.1.0/24"]
                EC2Pub["🖥️ lab-web-01<br/>10.0.1.33 (IP pública)<br/>Apache httpd<br/>SG: HTTP 80, SSH 22, ICMP desde 10.0.0.0/16"]
            end

            subgraph PrivSub1["Subnet privada 10.0.2.0/24"]
                EC2Priv1["🖥️ lab-web-privada<br/>10.0.2.32 (sin IP pública)<br/>Rol IAM: lab-ssm-s3-privada<br/>(SSM + S3FullAccess)"]
            end
        end

        subgraph VPC2["VPC lab-vpc-02 — 10.1.0.0/16"]
            direction TB
            SSMEP["🔗 VPC Interface Endpoints<br/>ssm · ssmmessages · ec2messages<br/>Private DNS habilitado"]

            subgraph PrivSub2["Subnet privada 10.1.1.0/24"]
                EC2Priv2["🖥️ EC2 VPC2<br/>10.1.1.174<br/>Rol IAM: SystemsManagerEC2Role<br/>SG: ICMP desde 10.0.0.0/16"]
            end
        end

        Peering{{"🔀 VPC Peering Connection<br/>pcx-0eabdfb8338f035e0 — Active<br/>Ruta 10.1.0.0/16 ↔ 10.0.0.0/16"}}

        S3Bucket[("🪣 S3 Bucket")]
        SSMSvc["⚙️ AWS Systems Manager<br/>(Session Manager)"]
    end

    %% Conexiones VPC1
    Internet <--> IGW1
    IGW1 <--> EC2Pub
    EC2Pub --> NAT
    NAT --> IGW1
    EC2Priv1 -. "tráfico internet" .-> NAT
    EC2Priv1 == "aws s3 ls / cp" ==> S3EP
    S3EP --> S3Bucket
    EC2Priv1 -. "SSM vía NAT" .-> SSMSvc

    %% Conexiones VPC2
    EC2Priv2 == "SSM vía PrivateLink" ==> SSMEP
    SSMEP -.-> SSMSvc

    %% Peering
    VPC1 === Peering === VPC2
    EC2Pub -. "ping 10.1.1.174 ✅" .-> EC2Priv2

    %% DNS
    EC2Pub -. "dig servidor01.lab.interno @10.0.0.2" .-> PrivZone
    Internet -.-> PubApex
    Internet -.-> LabZone

    style PrivZone fill:#ffe0e0,stroke:#cc0000
    style PubApex fill:#fff3cd,stroke:#cc9900
    style VPC1 fill:#e8f4fd,stroke:#0066cc
    style VPC2 fill:#e8f4fd,stroke:#0066cc
    style R53 fill:#f0f0ff,stroke:#6666cc
```

## Leyenda y estado de cada componente

| # | Componente | Detalle | Estado |
|---|---|---|---|
| 1 | `lab-vpc-01` (10.0.0.0/16) | VPC principal con subnet pública + privada, IGW, NAT Gateway | ✅ Funcionando |
| 2 | `lab-vpc-02` (10.1.0.0/16) | VPC secundaria, solo subnet privada, sin IGW/NAT | ✅ Funcionando |
| 3 | VPC Peering (`pcx-...`) | Conecta ambas VPCs, rutas agregadas en ambos lados | ✅ Activa, ping exitoso |
| 4 | EC2 pública `lab-web-01` | Apache httpd mostrando hostname/IP/AZ | ✅ Funcionando |
| 5 | EC2 privada VPC1 `lab-web-privada` | Acceso sin SSH vía **SSM Session Manager** (a través de NAT) | ✅ Funcionando |
| 6 | EC2 privada VPC2 | Acceso sin SSH vía **SSM Session Manager** (a través de VPC Interface Endpoints, sin salida a internet) | ✅ Funcionando |
| 7 | S3 Gateway Endpoint | Acceso a S3 desde subnet privada sin pasar por NAT/Internet | ✅ Funcionando |
| 8 | Zona pública `joedayz123.com` | Dominio de producción (apunta a CloudFront) — **no se tocó** | ✅ Intacto |
| 9 | Zona pública `lab.joedayz123.com` | Usada para Health Check + Failover Routing (`app` primary/secondary) | ✅ Funcionando |
| 10 | Zona privada `lab.interno` | Registro `servidor01 → 10.0.1.100` correctamente configurado y asociado a la VPC | ⚠️ El Resolver de la VPC (10.0.0.2) no resuelve zonas privadas en este entorno sandbox — descartado como error de configuración tras diagnóstico exhaustivo (zona, VPC, DHCP options, reglas de Resolver, DNS Firewall, caché, propagación — todo validado) |

## Notas para la clase

- El punto **10 (DNS privado)** es el único pendiente: toda la configuración es correcta según el estándar de AWS, pero el entorno de laboratorio (sandbox) no refleja la resolución. Es un buen caso de estudio de **metodología de troubleshooting** (se descartó causa por causa con evidencia de CLI/CloudTrail) aunque no se haya resuelto el síntoma final.
- El acceso a instancias privadas **sin keypair ni SSH** se logró con **AWS Systems Manager Session Manager**, usando dos variantes de conectividad de red:
  - **VPC1**: saliendo por NAT Gateway (requiere solo el rol IAM).
  - **VPC2**: 100% privado, sin salida a internet, usando VPC Interface Endpoints (`ssm`, `ssmmessages`, `ec2messages`).
