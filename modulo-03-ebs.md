# Módulo 3 — Almacenamiento en bloque EBS
### Servicios: EBS · Snapshots · Volúmenes · DLM · AWS Backup

---

## TEORÍA

### EBS — Elastic Block Store
Disco virtual persistente que adjuntas a EC2. Persiste aunque la instancia se apague (a diferencia del Instance Store que es efímero).

### Tipos de volúmenes
| Tipo | Descripción | Cuándo usarlo |
|------|-------------|---------------|
| gp3 | SSD propósito general | La mayoría de workloads (default recomendado) |
| io2 / io2 Block Express | SSD alta performance | Bases de datos críticas (Oracle, SAP) |
| st1 | HDD throughput | Big data, logs secuenciales |
| sc1 | HDD cold | Archivos de acceso infrecuente |

**gp3 vs gp2**: en gp3 configuras IOPS y throughput independientemente del tamaño. Es más barato y mejor. Siempre elige gp3.

### Snapshots
Copias incrementales del volumen guardadas en S3 (AWS lo gestiona). El primer snapshot copia todo; los siguientes solo los bloques que cambiaron.

### Data Lifecycle Manager (DLM)
Automatiza la creación, copia y eliminación de snapshots según una política que defines.

---

## 🟢 LAB BÁSICO — Crear, adjuntar, formatear y hacer snapshot

**Tiempo estimado: 35 min**
**Prerequisito:** Instancia EC2 "lab-web-01" del Módulo 2 en ejecución

---

### Paso 1 — Verificar la AZ de tu EC2

```
EC2 → Instances → seleccionar "lab-web-01"
Pestaña "Details" → ver el campo "Availability Zone"
Anotar el valor: ej. us-east-1a

⚠️  CRÍTICO: el volumen EBS DEBE crearse en la MISMA AZ que la instancia.
    EBS no puede cruzar zonas de disponibilidad directamente.
```

---

### Paso 2 — Crear volumen EBS gp3

```
EC2 → Panel izquierdo → "Elastic Block Store" → "Volumes"
Clic "Create volume"

  Volume type:        gp3
  Size (GiB):         20
  IOPS:               3000   (mínimo incluido gratis en gp3)
  Throughput (MiB/s): 125    (mínimo incluido gratis en gp3)
  Availability Zone:  us-east-1a   ← MISMA AZ que tu EC2

  Tags:
    Key:   Name
    Value: vol-datos-lab

Clic "Create volume"
Estado: "creating" → cambia a "available" en ~10 segundos
```

---

### Paso 3 — Adjuntar volumen a la instancia

```
Seleccionar el volumen "vol-datos-lab" → menú "Actions" → "Attach volume"

  Instance: buscar y seleccionar "lab-web-01"
  Device name: /dev/sdf

Clic "Attach volume"
Estado cambia a "in-use" ✅
```

---

### Paso 4 — Formatear y montar el volumen (dentro de EC2)

```bash
# Conectarse a la instancia
ssh -i ~/Downloads/lab-key-ec2.pem ec2-user@54.xxx.xxx.xxx

# PASO 4.1 — Ver los discos del sistema
lsblk
# Salida esperada:
# NAME     MAJ:MIN RM SIZE RO TYPE MOUNTPOINT
# xvda     202:0    0   8G  0 disk
# └─xvda1  202:1    0   8G  0 part /
# xvdf     202:80   0  20G  0 disk    ← el volumen nuevo sin montar

# NOTA: En instancias tipo Nitro (t3, m5, c5...) puede aparecer como:
# nvme0n1  (disco raíz)
# nvme1n1  (tu volumen nuevo)
# En ese caso reemplazar /dev/xvdf por /dev/nvme1n1 en todos los comandos

# PASO 4.2 — Verificar que el disco está vacío (sin sistema de archivos)
sudo file -s /dev/xvdf
# Salida: /dev/xvdf: data  → vacío, listo para formatear

# PASO 4.3 — Crear sistema de archivos ext4
sudo mkfs.ext4 /dev/xvdf
# Salida esperada:
# mke2fs 1.46.x
# Creating filesystem with 5242880 4k blocks and 1310720 inodes
# Writing inode tables: done
# Writing superblocks and filesystem accounting information: done

# PASO 4.4 — Crear directorio de punto de montaje
sudo mkdir /datos

# PASO 4.5 — Montar el volumen
sudo mount /dev/xvdf /datos

# PASO 4.6 — Verificar que está montado correctamente
df -h
# Salida: debes ver una línea como:
# /dev/xvdf        20G   24M   19G   1% /datos  ✅

# PASO 4.7 — Escribir un archivo de prueba
echo "Archivo de prueba - $(date)" | sudo tee /datos/prueba.txt
cat /datos/prueba.txt
```

---

### Paso 5 — Hacer que el montaje persista al reiniciar

```bash
# Sin esto, al reiniciar la instancia el volumen no se monta automáticamente

# Obtener el UUID del disco (más seguro que usar /dev/xvdf que puede cambiar)
sudo blkid /dev/xvdf
# Salida:
# /dev/xvdf: UUID="a1b2c3d4-e5f6-7890-abcd-ef1234567890" TYPE="ext4"
# Copiar el UUID

# Editar el archivo /etc/fstab
sudo nano /etc/fstab

# Agregar esta línea al FINAL del archivo (reemplazar UUID con el tuyo):
UUID=a1b2c3d4-e5f6-7890-abcd-ef1234567890  /datos  ext4  defaults,nofail  0  2
#    ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^   ^^^^^^  ^^^^  ^^^^^^^^^^^^^^^^
#    UUID del volumen                              mount   fs    opciones

# Guardar en nano: Ctrl+O → Enter → Ctrl+X

# Verificar que el fstab está correcto (IMPORTANTE antes de reiniciar)
sudo mount -a
# Si no hay ningún error en la salida → el archivo es correcto ✅
# Si hay error → revisar el UUID y la sintaxis antes de reiniciar

# Verificar que sigue montado
df -h /datos
```

---

### Paso 6 — Crear Snapshot

```
EC2 → "Volumes" → seleccionar "vol-datos-lab"
Menú "Actions" → "Create snapshot"

  Description: Snapshot inicial vol-datos-lab 20GB
  
  Tags:
    Key:   Name
    Value: snap-datos-lab-v1

Clic "Create snapshot"
```

```
EC2 → "Snapshots" (panel izquierdo)
Esperar que el estado cambie de "pending" a "completed"
⏳ Para un volumen de 20GB con pocos datos: ~1-2 minutos
```

---

## 🟡 LAB INTERMEDIO — Modificar en caliente + DLM + Restaurar en otra AZ

**Tiempo estimado: 50 min**

---

### Paso 1 — Modificar volumen sin apagar la instancia

```
EC2 → "Volumes" → seleccionar "vol-datos-lab"
Menú "Actions" → "Modify volume"

  Volume type:  gp3   (sin cambio)
  Size:         30 GiB   (aumentar de 20 → 30)
  IOPS:         6000      (aumentar de 3000 → 6000)
  Throughput:   250 MiB/s (aumentar de 125 → 250)

Clic "Modify" → confirmar → "Yes"
Estado: "modifying" → "optimizing" → "completed"
⏳ Puede tardar 5-15 minutos
```

```bash
# El bloque ya es de 30GB pero el sistema de archivos aún ve 20GB
# Verificar con lsblk — debe mostrar 30G
lsblk

# Extender el sistema de archivos para usar todo el espacio nuevo
sudo resize2fs /dev/xvdf

# Verificar que creció
df -h /datos
# Ahora debe mostrar ~30G disponibles ✅

# Los datos existentes NO se pierden
cat /datos/prueba.txt  # sigue ahí ✅
```

---

### Paso 2 — Medir rendimiento del volumen con fio

```bash
# Instalar fio (benchmark de discos)
sudo yum install fio -y

# --- Test de ESCRITURA ALEATORIA (IOPS) ---
sudo fio \
  --name=test-iops-escritura \
  --filename=/datos/fio-test \
  --rw=randwrite \
  --bs=4k \
  --size=512m \
  --numjobs=4 \
  --runtime=30 \
  --group_reporting \
  --output-format=normal

# Buscar en la salida: "WRITE: IOPS=" (deberías ver ~6000 IOPS)

# --- Test de LECTURA ALEATORIA (IOPS) ---
sudo fio \
  --name=test-iops-lectura \
  --filename=/datos/fio-test \
  --rw=randread \
  --bs=4k \
  --size=512m \
  --numjobs=4 \
  --runtime=30 \
  --group_reporting

# Buscar en la salida: "READ: IOPS="

# --- Test de ESCRITURA SECUENCIAL (Throughput) ---
sudo fio \
  --name=test-throughput \
  --filename=/datos/fio-test \
  --rw=write \
  --bs=1m \
  --size=512m \
  --numjobs=1 \
  --runtime=30 \
  --group_reporting

# Buscar: "WRITE: bw=" (deberías ver ~250 MiB/s)

# Limpiar el archivo de test
sudo rm /datos/fio-test
```

---

### Paso 3 — Configurar Data Lifecycle Manager (DLM)

```
EC2 → Panel izquierdo → "Elastic Block Store" → "Lifecycle Manager"
Clic "Create lifecycle policy"

  Policy type: EBS snapshot policy
  
  Description: Backup diario de volumen de datos lab
  
  Target resources:
    Resource types: Volumes
    Target resource tags:
      Key:   Name
      Value: vol-datos-lab

  IAM role: AWSDataLifecycleManagerDefaultRole
    (Si aparece error "Role not found": seleccionar "Create new role"
     → AWS crea el rol automáticamente)

  Schedule:
    Schedule name:   diario-7-dias
    Frequency:       Daily
    Every:           24  hours
    Starting at:     02:00  UTC   (9pm Perú | 10pm Colombia)
    Retention type:  Count
    Retain:          7   (guarda los últimos 7 snapshots, borra los más viejos)

  Optional settings:
    ✅ Copy tags from source
    ✅ Enable cross-region copy (opcional: seleccionar sa-east-1 para DR)

  Policy status: Enabled

Clic "Create policy"
✅ DLM creará un snapshot automático a las 2:00 UTC cada día
```

---

### Paso 4 — Restaurar snapshot en diferente AZ

```
Escenario: quieres usar el snapshot en una instancia ubicada en us-east-1b

EC2 → "Snapshots" → seleccionar "snap-datos-lab-v1"
Menú "Actions" → "Create volume from snapshot"

  Volume type:        gp3
  Size:               30 GiB   (puede ser igual o mayor que el original)
  IOPS:               3000
  Availability Zone:  us-east-1b   ← diferente AZ

  Tags:
    Key:   Name
    Value: vol-datos-lab-1b

Clic "Create volume"
⏳ Estado: "creating" → "available" en ~1-2 minutos
```

```bash
# Adjuntar el nuevo volumen a una EC2 en us-east-1b
# EC2 → Volumes → "vol-datos-lab-1b" → Actions → Attach volume
# Seleccionar instancia en us-east-1b → /dev/sdf → Attach

# Desde la instancia en us-east-1b:
# NOTA: NO formatear — los datos ya están del snapshot
sudo mkdir /datos-restaurados
sudo mount /dev/xvdf /datos-restaurados

# Verificar que los datos del snapshot están ahí
ls /datos-restaurados
cat /datos-restaurados/prueba.txt
# ✅ Deberías ver el archivo creado en la instancia original
```

---

## 🔴 LAB AVANZADO — Cifrado KMS + Copy Cross-Region + AWS Backup

**Tiempo estimado: 60 min**

---

### Parte 1 — Habilitar cifrado EBS por defecto con KMS

**Paso 1.1 — Crear CMK (Customer Managed Key) en KMS**
```
Barra de búsqueda → "KMS" → Key Management Service
Panel izquierdo → "Customer managed keys" → "Create key"

  Key type:   Symmetric
  Key usage:  Encrypt and decrypt
  Clic "Next"

  Alias:  alias/lab-ebs-key
  Description: Clave para cifrado de volúmenes EBS del lab
  Clic "Next"

  Key administrators: seleccionar tu usuario/rol de administrador
  Clic "Next"

  Key users: seleccionar tu usuario/rol
  Clic "Next" → "Finish"
```

**Paso 1.2 — Habilitar cifrado por defecto**
```
EC2 → Panel izquierdo → "Account attributes" (en Settings)
  Buscar la sección "EBS encryption"
  Clic "Manage"

  ✅ Enable (activar)
  Default encryption key: seleccionar "alias/lab-ebs-key"
  Clic "Update EBS encryption"

✅ A partir de ahora, todos los volúmenes EBS nuevos en esta región
   se cifrarán automáticamente con tu CMK
```

**Paso 1.3 — Crear volumen cifrado y verificar**
```
EC2 → Volumes → "Create volume"
  Type: gp3, Size: 10 GiB, AZ: us-east-1a
  
  Encryption:
    ✅ Encrypt this volume
    KMS key: alias/lab-ebs-key

Clic "Create volume"

Seleccionar el volumen → Pestaña "Description"
Buscar campo "Encrypted" → debe mostrar "Yes" ✅
```

---

### Parte 2 — Copiar Snapshot entre regiones

```
EC2 → "Snapshots" → seleccionar "snap-datos-lab-v1"
Menú "Actions" → "Copy snapshot"

  Destination Region:   us-west-2  (Oregon)
  Description:          Copia DR de snap-datos-lab-v1
  
  Encryption:
    ✅ Encrypt this snapshot
    Master key:  (en us-west-2 necesitas una KMS key — AWS Managed Key es suficiente)
    Usar: "aws/ebs"  (clave administrada por AWS)

Clic "Copy snapshot"
⏳ La copia tardará 5-15 minutos dependiendo del tamaño

Verificar en us-west-2:
  Cambiar región a "us-west-2" en la consola
  EC2 → Snapshots → debe aparecer la copia ✅
```

---

### Parte 3 — Plan de backup con AWS Backup

```
Barra de búsqueda → "AWS Backup" → clic en AWS Backup

Panel izquierdo → "Backup plans" → "Create backup plan"

  Seleccionar: "Build a new plan"
  Backup plan name: plan-lab-ebs-rds

  Backup rule:
    Rule name:           diario-30-dias
    Backup vault:        Default
    Backup frequency:    Daily
    Backup window:
      Start time:  03:00  UTC
      Duration:    2 hours
    Lifecycle:
      Transition to cold storage: Never
      Expire (delete backup):     After 30 days
    
    Copy to destination:
      ✅ Create a copy in destination Region
      Region: us-west-2
      Vault:  Default  (en us-west-2)

  Clic "Create plan"
```

```
Asignar recursos al plan:
  Panel izquierdo → "Backup plans" → seleccionar "plan-lab-ebs-rds"
  Pestaña "Resource assignments" → "Assign resources"

  Resource assignment name: recursos-lab
  IAM role: Default role (AWSBackupDefaultServiceRole)
  
  Assign by:
    Resource type: EBS
    
    Include specific resource IDs:
      Agregar el Volume ID de "vol-datos-lab" (ej: vol-xxxxxxxxxxxxxxxxx)
  
  Clic "Assign resources"
```

```
Verificar el backup:
  AWS Backup → "Protected resources"
  Debe aparecer el volumen EBS ✅
  
  AWS Backup → "Jobs" → "Backup jobs"
  El primer backup se ejecuta según la ventana configurada
  También puedes ejecutar uno manualmente:
    Seleccionar el recurso → "Create on-demand backup"
```

---

### Parte 4 — CloudWatch Metrics de EBS

```bash
# Ver métricas de un volumen EBS via CLI
aws cloudwatch get-metric-statistics \
  --namespace AWS/EBS \
  --metric-name VolumeReadOps \
  --dimensions Name=VolumeId,Value=vol-xxxxxxxxxxxxxxxxx \
  --start-time $(date -u -v-1H +%Y-%m-%dT%H:%M:%S) \
  --end-time $(date -u +%Y-%m-%dT%H:%M:%S) \
  --period 300 \
  --statistics Average \
  --output table

# Métricas clave a monitorear:
# VolumeReadOps    - operaciones de lectura
# VolumeWriteOps   - operaciones de escritura
# VolumeReadBytes  - bytes leídos
# VolumeWriteBytes - bytes escritos
# VolumeTotalReadTime  - latencia de lectura
# VolumeTotalWriteTime - latencia de escritura
# BurstBalance     - solo gp2: créditos de IOPS disponibles (gp3 no lo necesita)
```

```
Crear Dashboard en CloudWatch:
  CloudWatch → "Dashboards" → "Create dashboard"
  Dashboard name: dashboard-ebs-lab

  Add widget → Line → Select metric
    Namespace: EBS
    By Volume: seleccionar tu volumen
    Métricas: VolumeReadOps, VolumeWriteOps

  Clic "Update" → "Save dashboard"
```

---

## 🧹 LIMPIEZA (en este orden)

```
1.  AWS Backup → Backup plans → "plan-lab-ebs-rds"
    → Resource assignments → eliminar asignación
    → Eliminar el plan

2.  AWS Backup → Protected resources → verificar que quedó vacío

3.  EC2 → Snapshots → eliminar todos los snapshots del lab
    (snap-datos-lab-v1 y la copia en us-west-2)
    ⚠️  Cambiar región a us-west-2 y eliminar también esa copia

4.  EC2 → Volumes:
    Seleccionar "vol-datos-lab" → Actions → Detach volume (primero)
    Luego → Actions → Delete volume
    Hacer lo mismo con "vol-datos-lab-1b"

5.  KMS → Customer managed keys → "alias/lab-ebs-key"
    Actions → Schedule key deletion
    Waiting period: 7 days (mínimo)
    Confirm → "Schedule deletion"
    ⚠️  La clave no se elimina inmediatamente, entra en período de espera

6.  EC2 → Account attributes → EBS encryption
    Desactivar el cifrado por defecto si no lo quieres permanente
```

---

## 📌 COMANDOS CLI DE REFERENCIA RÁPIDA

```bash
# Listar volúmenes EBS con su estado
aws ec2 describe-volumes \
  --query 'Volumes[*].[VolumeId,State,Size,VolumeType,Encrypted,Tags[?Key==`Name`].Value|[0]]' \
  --output table

# Listar snapshots propios
aws ec2 describe-snapshots \
  --owner-ids self \
  --query 'Snapshots[*].[SnapshotId,State,VolumeSize,StartTime,Description]' \
  --output table

# Crear snapshot desde CLI
aws ec2 create-snapshot \
  --volume-id vol-xxxxxxxxxxxxxxxxx \
  --description "Snapshot manual desde CLI" \
  --tag-specifications 'ResourceType=snapshot,Tags=[{Key=Name,Value=snap-cli-test}]'

# Copiar snapshot a otra región
aws ec2 copy-snapshot \
  --source-region us-east-1 \
  --source-snapshot-id snap-xxxxxxxxxxxxxxxxx \
  --destination-region us-west-2 \
  --description "Copia DR en us-west-2"

# Ver políticas DLM activas
aws dlm get-lifecycle-policies \
  --query 'Policies[*].[PolicyId,Description,State]' \
  --output table
```
