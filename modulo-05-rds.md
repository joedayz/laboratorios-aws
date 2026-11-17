# Módulo 5 — Bases de datos administradas RDS
### Servicios: RDS · Multi-AZ · Read Replicas · RDS Proxy · Aurora

---

## TEORÍA

### RDS — Relational Database Service
Base de datos relacional administrada. AWS se encarga de: backups automáticos, parches, failover, escalado de almacenamiento. Tú solo administras el esquema y los datos.

**Motores soportados:** MySQL, PostgreSQL, MariaDB, Oracle, SQL Server, Aurora MySQL, Aurora PostgreSQL

### Multi-AZ
RDS mantiene una réplica **síncrona** en otra AZ. Si la instancia principal falla, el failover es automático (~60-120 segundos). El endpoint DNS no cambia — las aplicaciones no necesitan reconfigurarse.

### Read Replica
Réplica **asíncrona** de solo lectura. Sirve para escalar lecturas (descargar consultas del primary). Puede estar en la misma región, en otra región, o incluso en otra cuenta.

### RDS Proxy
Pool de conexiones gestionado entre la aplicación y RDS. Crítico para Lambda (que abre miles de conexiones simultáneas y satura la base de datos sin proxy).

### Aurora
Motor de base de datos propio de AWS. Compatible con MySQL y PostgreSQL. Almacenamiento distribuido en 6 copias a través de 3 AZs. Hasta 5x más rápido que MySQL estándar.

---

## 🟢 LAB BÁSICO — Crear RDS MySQL y conectarse desde EC2

**Tiempo estimado: 40 min**
**Prerequisito:** VPC lab-vpc-01 con subnets privadas del Módulo 1

---

### Paso 1 — Crear Subnet Group para RDS

```
RDS necesita un Subnet Group que indique en qué subnets puede colocar las instancias.
El grupo debe tener subnets en al menos 2 AZs.

Primero crear una segunda subnet privada (si no existe):
  VPC → Subnets → "Create subnet"
    VPC:   lab-vpc-01
    Name:  lab-subnet-privada-1b
    AZ:    us-east-1b
    CIDR:  10.0.4.0/24
  Clic "Create subnet"
```

```
Consola → buscar "RDS" → clic en RDS
Panel izquierdo → "Subnet groups" → "Create DB subnet group"

  Name:        lab-rds-subnet-group
  Description: Subnets privadas para RDS
  VPC:         lab-vpc-01

  Add subnets:
    us-east-1a → seleccionar "lab-subnet-privada-1a"
    us-east-1b → seleccionar "lab-subnet-privada-1b"

Clic "Create"
```

---

### Paso 2 — Crear Security Group para RDS

```
EC2 → Security Groups → "Create security group"

  Name:        sg-rds-lab
  Description: Acceso MySQL solo desde EC2
  VPC:         lab-vpc-01

  Inbound rules:
    Type:   MySQL/Aurora
    Port:   3306
    Source: Custom → seleccionar "sg-web-lab"
    (Solo instancias con sg-web-lab pueden conectarse)

  Outbound rules: All traffic (dejar por defecto)

Clic "Create security group"
```

---

### Paso 3 — Crear instancia RDS MySQL

```
RDS → Panel izquierdo → "Databases" → "Create database"

  Choose a database creation method:
    Standard create

  Engine options:
    Engine type: MySQL
    Engine Version: MySQL 8.0.xx (la más reciente disponible)

  Templates:
    Free tier   ← para el laboratorio

  Settings:
    DB instance identifier: lab-mysql-01
    Master username:        admin
    Master password:        Lab@12345!   ← cambiar en producción
    Confirm password:       Lab@12345!

  Instance configuration:
    DB instance class: db.t3.micro   (Free tier)

  Storage:
    Storage type: gp3
    Allocated storage: 20 GiB
    Enable storage autoscaling: ✅
    Maximum storage threshold: 100 GiB

  Connectivity:
    Compute resource: Don't connect to an EC2 compute resource
    VPC: lab-vpc-01
    DB subnet group: lab-rds-subnet-group
    Public access: No   ← IMPORTANTE: RDS en subnet privada
    VPC security group:
      Remove "default"
      Add: sg-rds-lab

  Availability Zone: us-east-1a

  Additional configuration:
    Initial database name: labapp
    Backup retention: 7 days
    ✅ Enable automated backups
    Backup window: 03:00 UTC  (1am hora Lima)
    
    ✅ Enable Enhanced monitoring
    Monitoring role: (AWS crea uno automáticamente)
    Granularity: 60 seconds

  ⚠️  Multi-AZ: NO (Free tier no lo soporta)
      En el lab de Intermedio sí lo habilitaremos

Clic "Create database"
⏳ Esperar 5-10 minutos hasta estado "Available"
```

---

### Paso 4 — Conectarse a RDS desde EC2

```
Mientras espera RDS:
  EC2 → Instances → "lab-web-01" → "Connect"
  Usar EC2 Instance Connect o SSH
```

```bash
# Dentro de la instancia EC2:

# Instalar el cliente MariaDB compatible con MySQL en Amazon Linux 2023
sudo dnf install mariadb105 -y

# Obtener el endpoint de RDS:
#   RDS → Databases → "lab-mysql-01"
#   Pestaña "Connectivity & security"
#   Copiar el "Endpoint" ej: lab-mysql-01.xxxxxxxxxx.us-east-1.rds.amazonaws.com

# Conectarse (reemplazar el endpoint con el tuyo)
mysql -h lab-mysql-01.xxxxxxxxxx.us-east-1.rds.amazonaws.com \
      -u admin \
      -p
# Ingresar la contraseña: Lab@12345!

# Deberías ver:
# Welcome to the MySQL monitor. Commands end with ; or \g.
# mysql>
```

```sql
-- Dentro del cliente MySQL:

-- Ver las bases de datos
SHOW DATABASES;

-- Usar la base de datos del lab
USE labapp;

-- Crear tabla de empleados
CREATE TABLE empleados (
    id          INT AUTO_INCREMENT PRIMARY KEY,
    nombre      VARCHAR(100) NOT NULL,
    cargo       VARCHAR(100),
    salario     DECIMAL(10,2),
    fecha_ingreso DATE,
    creado_en   TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- Insertar datos de prueba
INSERT INTO empleados (nombre, cargo, salario, fecha_ingreso) VALUES
  ('Ana García',    'Desarrolladora', 5000.00, '2023-01-15'),
  ('Carlos López',  'DevOps',         6000.00, '2022-06-01'),
  ('María Torres',  'DBA',            5500.00, '2023-03-20'),
  ('Juan Pérez',    'QA',             4500.00, '2024-01-10');

-- Ver los datos
SELECT * FROM empleados;

-- Consulta con filtro
SELECT nombre, salario 
FROM empleados 
WHERE salario > 5000
ORDER BY salario DESC;

-- Ver información del servidor RDS
SELECT VERSION();
SHOW VARIABLES LIKE 'max_connections';

-- Salir
EXIT;
```

---

### Paso 5 — Verificar backups automáticos

```
RDS → "lab-mysql-01"
Pestaña "Maintenance & backups"

Sección "Automated backups":
  Muestra el período de retención (7 días)
  Muestra la ventana de backup configurada

Sección "Snapshots":
  Puedes crear un snapshot manual:
  Actions → "Take snapshot"
    Snapshot identifier: snap-mysql-lab-manual-01
  Clic "Take snapshot"
  ⏳ Estado: "creating" → "available"
```

---

## 🟡 LAB INTERMEDIO — Multi-AZ + Read Replica + RDS Proxy

**Tiempo estimado: 70 min**

---

### Paso 1 — Habilitar Multi-AZ sin downtime

```
RDS → "lab-mysql-01"
Clic "Modify"

  Availability & durability:
    Deployment options: Multi-AZ DB instance
    ✅ Create a standby instance (recommended for production usage)

  Scheduled changes:
    When to apply changes: Apply immediately
    ⚠️  "Apply immediately" causa ~60-120 segundos de downtime
       En producción: usar "During next maintenance window"

Clic "Continue" → "Modify DB instance"
⏳ Estado: "modifying" → puede tardar 5-10 minutos → "available"
```

---

### Paso 2 — Crear Read Replica

```
RDS → seleccionar "lab-mysql-01"
Menú "Actions" → "Create read replica"

  DB instance identifier:     lab-mysql-01-replica-1
  AWS Region:                 us-east-1 (misma región para el lab)
  
  DB instance class:          db.t3.micro
  
  Availability Zone:          us-east-1b   (diferente AZ al primary)
  
  Storage:
    Storage type: gp3
    Allocated storage: 20 GiB

  Connectivity:
    VPC: lab-vpc-01
    Subnet group: lab-rds-subnet-group
    Public access: No
    Security group: sg-rds-lab
  
  Encryption: (heredará del primary)

Clic "Create read replica"
⏳ Esperar 5-10 minutos hasta estado "Available"
```

```bash
# Probar la Read Replica desde EC2
# Obtener el endpoint de la réplica desde la consola RDS

mysql -h lab-mysql-01-replica-1.xxxxxxxxxx.us-east-1.rds.amazonaws.com \
      -u admin \
      -p

-- En la réplica: solo SELECT funciona
SELECT * FROM labapp.empleados;   -- ✅ funciona

-- Intentar escribir (debe fallar)
INSERT INTO labapp.empleados (nombre) VALUES ('Test');
-- Error: ERROR 1290 (HY000): The MySQL server is running with the
--        --read-only option so it cannot execute this statement  ✅
EXIT;
```

---

### Paso 3 — Simular Failover Multi-AZ

```
RDS → seleccionar "lab-mysql-01"
Menú "Actions" → "Reboot"

  ✅ Reboot with failover   ← esto activa el failover a la standby

Clic "Confirm"
⏳ La instancia queda en "rebooting" por ~60-120 segundos

Durante el failover, en la instancia EC2:
```

```bash
# Ejecutar este script antes del failover para medir el tiempo de interrupción
while true; do
  RESULT=$(mysql -h lab-mysql-01.xxxxxxxxxx.us-east-1.rds.amazonaws.com \
    -u admin -pLab@12345! \
    -e "SELECT NOW(), @@hostname;" \
    --connect-timeout=5 \
    2>&1)
  echo "$(date '+%H:%M:%S') - $RESULT"
  sleep 2
done

# Verás las conexiones fallando durante el failover
# y luego reconectando automáticamente en la standby (misma URL de endpoint)
# Ctrl+C para parar el script
```

---

### Paso 4 — Crear RDS Proxy

```
RDS → Panel izquierdo → "Proxies" → "Create proxy"

  Proxy identifier:    lab-rds-proxy
  Engine family:       MySQL
  
  Idle client connection timeout: 1800 seconds (30 min)

  Connectivity:
    Database:   lab-mysql-01   ← el primary (no la réplica)
    
    Connection pool maximum connections: 100   (% del max_connections de RDS)

  Authentication:
    Secrets Manager secret:
      "Create a new secret"
        Secret name: rds-proxy-credentials-lab
        Username:    admin
        Password:    Lab@12345!
      Clic "Create secret" (se crea en Secrets Manager)
    
    IAM authentication: Allowed

  Connectivity:
    VPC: lab-vpc-01
    Subnets: lab-subnet-privada-1a, lab-subnet-privada-1b
    VPC security group: sg-rds-lab

Clic "Create proxy"
⏳ El proxy tarda ~5 minutos en quedar "Available"
```

```bash
# Obtener el endpoint del proxy desde la consola RDS → Proxies
# ej: lab-rds-proxy.proxy-xxxxxxxxxxxx.us-east-1.rds.amazonaws.com

# Conectarse via el Proxy (mismo usuario/contraseña)
mysql -h lab-rds-proxy.proxy-xxxxxxxxxxxx.us-east-1.rds.amazonaws.com \
      -u admin \
      -p
# Ingresar: Lab@12345!

SELECT * FROM labapp.empleados;
EXIT;

# ✅ Funciona igual pero las conexiones van a través del pool del proxy
# Beneficio: múltiples conexiones simultáneas de Lambda se "comparten"
# reduciendo la carga sobre RDS
```

---

## 🔴 LAB AVANZADO — Aurora Serverless v2 + Performance Insights + Blue/Green

**Tiempo estimado: 75 min**

---

### Parte 1 — Crear Aurora MySQL Cluster

```
RDS → "Create database"

  Engine: Amazon Aurora
  Edition: Amazon Aurora MySQL-Compatible Edition
  Engine version: Aurora MySQL 3.x.x (compatible with MySQL 8.0)

  Templates: Dev/Test  (o Production para ver todas las opciones)

  Settings:
    DB cluster identifier: lab-aurora-cluster
    Master username:       admin
    Master password:       Lab@12345!

  Cluster storage configuration:
    Aurora Standard  (para el lab)

  Instance configuration:
    DB instance class:
      ✅ Serverless v2
      Minimum ACUs: 0.5   (escala hasta 0 en inactividad)
      Maximum ACUs: 4

  Availability & durability:
    Create an Aurora Replica or Reader node: ✅
    (crea writer + reader automáticamente)

  Connectivity:
    VPC: lab-vpc-01
    Subnet group: lab-rds-subnet-group
    Public access: No
    Security group: sg-rds-lab

  Additional configuration:
    Initial database name: auroraapp
    Backup retention: 7 days
    ✅ Enable Enhanced monitoring

Clic "Create database"
⏳ Esperar 10-15 minutos — se crean dos instancias (writer + reader)
```

```bash
# Obtener los endpoints del cluster Aurora:
#   RDS → "lab-aurora-cluster"
#   "Writer endpoint":  lab-aurora-cluster.cluster-xxxx.us-east-1.rds.amazonaws.com
#   "Reader endpoint":  lab-aurora-cluster.cluster-ro-xxxx.us-east-1.rds.amazonaws.com

# Conectarse al writer (lectura + escritura)
mysql -h lab-aurora-cluster.cluster-xxxx.us-east-1.rds.amazonaws.com \
      -u admin -p

USE auroraapp;
CREATE TABLE productos (
    id    INT AUTO_INCREMENT PRIMARY KEY,
    nombre VARCHAR(100),
    precio DECIMAL(10,2)
);
INSERT INTO productos VALUES (NULL,'Laptop',2500),(NULL,'Mouse',25),(NULL,'Teclado',80);
SELECT * FROM productos;
EXIT;

# Conectarse al reader (solo lectura — para escalar queries de reportes)
mysql -h lab-aurora-cluster.cluster-ro-xxxx.us-east-1.rds.amazonaws.com \
      -u admin -p
SELECT * FROM auroraapp.productos;   -- ✅ funciona
EXIT;
```

---

### Parte 2 — Performance Insights

```
RDS → seleccionar "lab-mysql-01" (o la instancia Aurora)
Pestaña "Monitoring" → "Performance Insights"
  (Si no está habilitado: Actions → Modify → Enable Performance Insights)

En Performance Insights:
  Verás la gráfica de "DB Load" con carga por SQL
```

```bash
# Generar carga para ver en Performance Insights
# Desde EC2:

for i in {1..200}; do
  mysql -h lab-mysql-01.xxxxxxxxxx.us-east-1.rds.amazonaws.com \
    -u admin -pLab@12345! \
    -e "SELECT SLEEP(0.5), COUNT(*) FROM information_schema.columns c1, information_schema.columns c2 LIMIT 1000;" \
    labapp 2>/dev/null &
done

wait
echo "Carga completada"
```

```
Volver a Performance Insights en la consola:
- Verás el pico de carga en la gráfica
- En la sección "Top SQL" verás la query que más carga genera
- En "Top waits" verás si hay cuellos de botella en I/O, locks, etc.
✅ Herramienta fundamental para detectar slow queries en producción
```

---

### Parte 3 — Blue/Green Deployment

```
⚠️  Blue/Green solo está disponible para RDS MySQL, PostgreSQL y Aurora.
    Sirve para actualizar la versión del motor sin downtime.

RDS → seleccionar "lab-mysql-01"
Menú "Actions" → "Create Blue/Green deployment"

  Blue/Green deployment identifier: lab-bg-deployment-01

  Green environment settings:
    DB engine version: (seleccionar una versión más nueva de MySQL 8.0.x)
    DB instance class: db.t3.micro (mantener igual)

Clic "Create Blue/Green deployment"
⏳ El "Green" se crea como clon del "Blue" — tarda 5-10 minutos
   Durante este tiempo, Green recibe los cambios del Blue via replicación
```

```
Una vez que Green está "Available":
  Actions → "Switch over"

  ⚠️  Esto es el momento de downtime (~30-60 segundos)
      En producción, hacerlo en horario de bajo tráfico

  AWS realizará:
  1. Esperar que Green esté al día con Blue
  2. Redirigir el endpoint DNS a Green
  3. Blue queda como "old" (puedes eliminarlo o hacer rollback)

Clic "Switch over" → Confirm
✅ Ahora la instancia activa tiene la versión nueva del motor
```

---

## 🧹 LIMPIEZA (en este orden)

```
1.  RDS → Proxies → "lab-rds-proxy" → Actions → Delete proxy

2.  RDS → Databases:
    - Eliminar "lab-mysql-01-replica-1" (Read Replica)
    - Eliminar "lab-mysql-01" (Primary)
      ⚠️  Desmarcar "Create final snapshot" solo en labs
      Escribir "delete me" para confirmar
    
    - Eliminar instancias Aurora del cluster "lab-aurora-cluster"
      Primero el Reader, luego el Writer, luego el Cluster

3.  RDS → Subnet groups → "lab-rds-subnet-group" → Delete

4.  Secrets Manager → "rds-proxy-credentials-lab" → Delete secret
    (Período de recuperación: 7 días mínimo)

5.  EC2 → Security Groups → "sg-rds-lab" → Delete

6.  RDS → Snapshots → eliminar snapshots manuales del lab
    (Los automated backups se eliminan automáticamente al borrar la instancia)
```

---

## 📌 COMANDOS CLI DE REFERENCIA RÁPIDA

```bash
# Listar instancias RDS
aws rds describe-db-instances \
  --query 'DBInstances[*].[DBInstanceIdentifier,DBInstanceStatus,Engine,Endpoint.Address]' \
  --output table

# Ver endpoint de una instancia
aws rds describe-db-instances \
  --db-instance-identifier lab-mysql-01 \
  --query 'DBInstances[0].Endpoint.Address' \
  --output text

# Crear snapshot manual
aws rds create-db-snapshot \
  --db-instance-identifier lab-mysql-01 \
  --db-snapshot-identifier snap-mysql-lab-$(date +%Y%m%d)

# Ver snapshots
aws rds describe-db-snapshots \
  --db-instance-identifier lab-mysql-01 \
  --query 'DBSnapshots[*].[DBSnapshotIdentifier,Status,SnapshotCreateTime]' \
  --output table

# Iniciar failover Multi-AZ manualmente
aws rds reboot-db-instance \
  --db-instance-identifier lab-mysql-01 \
  --force-failover

# Ver métricas de conexiones en CloudWatch
aws cloudwatch get-metric-statistics \
  --namespace AWS/RDS \
  --metric-name DatabaseConnections \
  --dimensions Name=DBInstanceIdentifier,Value=lab-mysql-01 \
  --start-time $(date -u -v-1H +%Y-%m-%dT%H:%M:%S) \
  --end-time $(date -u +%Y-%m-%dT%H:%M:%S) \
  --period 300 \
  --statistics Average \
  --output table
```
