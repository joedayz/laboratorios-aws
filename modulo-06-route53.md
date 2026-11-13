# Módulo 6 — DNS y enrutamiento Route 53
### Servicios: Route 53 · Health Checks · Routing Policies · Resolver

---

## TEORÍA

### DNS — Domain Name System
Sistema que traduce nombres legibles (ejemplo.com) a IPs (54.x.x.x). La resolución sigue una cadena: tu laptop → resolver recursivo del ISP → nameservers raíz → nameservers autoritativos (Route 53).

### Route 53 — Componentes clave
- **Hosted Zone**: contenedor de registros DNS para un dominio. Puede ser pública (internet) o privada (dentro de VPC).
- **Record Set**: un registro individual (A, CNAME, Alias, MX, TXT...)
- **Routing Policy**: cómo Route 53 responde cuando recibe una consulta DNS
- **Health Check**: monitoreo del endpoint (HTTP/HTTPS/TCP). Se integra con routing policies

### Routing Policies
| Política | Comportamiento | Caso de uso |
|----------|---------------|-------------|
| Simple | Siempre devuelve el mismo valor | Un solo servidor |
| Weighted | Distribuye según peso (0-255) | A/B testing, migraciones |
| Latency | Devuelve el servidor más cercano | Multi-región |
| Failover | Primary/Secondary automático | DR |
| Geolocation | Según país/continente del usuario | Contenido regional |
| Geoproximity | Según distancia + bias configurable | Ajuste fino de tráfico |
| Multivalue | Hasta 8 IPs + health check integrado | Simple balanceo |

---

## 🟢 LAB BÁSICO — Hosted Zone + Registros básicos + TTL

**Tiempo estimado: 35 min**
**Prerequisito:** Tener al menos una instancia EC2 con IP pública del Módulo 2

---

### Paso 1 — Verificar que tienes una Hosted Zone pública

> Si no tienes dominio propio, ve directo al **Paso 1b** con zona privada para practicar.

**Paso 1a — Con dominio propio:**
```
Route 53 → "Hosted zones"
Si ya existe una zona para tu dominio → usarla directamente
Si no existe → "Create hosted zone"
  Domain name:  tudominio.com
  Type:         Public hosted zone
Clic "Create hosted zone"
```

**Paso 1b — Sin dominio (zona privada para práctica):**
```
Route 53 → "Hosted zones" → "Create hosted zone"
  Domain name:  lab.interno
  Type:         Private hosted zone
  VPC Region:   us-east-1
  VPC ID:       lab-vpc-01
Clic "Create hosted zone"
(Ya creada en el Módulo 1 — usarla si existe)
```

---

### Paso 2 — Crear registro A (nombre → IP)

```
Route 53 → "Hosted zones" → abrir tu zona
Clic "Create record"

  Record name:  servidor01
  Record type:  A
  Value:        10.0.1.100   (IP privada) o IP pública de tu EC2
  TTL:          300

Clic "Create records"
✅ FQDN resultante: servidor01.lab.interno (zona privada)
                 o servidor01.tudominio.com (zona pública)
```

---

### Paso 3 — Crear registro CNAME

```
Route 53 → "Hosted zones" → "Create record"

  Record name:  www
  Record type:  CNAME
  Value:        servidor01.lab.interno  (apunta al registro A anterior)
  TTL:          300

Clic "Create records"
✅ www.lab.interno → servidor01.lab.interno → 10.0.1.100
```

---

### Paso 4 — Crear registro Alias apuntando a un ALB

```
⚠️  Alias NO puede apuntar a una IP arbitraria — solo a recursos AWS:
    ALB, NLB, CloudFront, S3 website, Elastic Beanstalk, otro Route 53

Route 53 → "Hosted zones" → "Create record"

  Record name:  app
  Record type:  A
  
  ✅ Alias  ← activar este toggle

  Route traffic to:
    Alias to Application and Classic Load Balancer
    Region: us-east-1
    Load balancer: (seleccionar "alb-web-lab" del Módulo 2)

  TTL: (no configurable en Alias — AWS gestiona el TTL)

Clic "Create records"
✅ Diferencia clave: Alias no cobra por queries y funciona en el apex del dominio
```

---

### Paso 5 — Verificar resolución DNS

```bash
# Desde instancia EC2 en lab-vpc-01 (para zona privada)
# O desde cualquier lugar (para zona pública)

# Instalar herramientas
sudo yum install bind-utils -y

# Consultar registro A
dig servidor01.lab.interno

# Salida esperada (sección ANSWER):
# servidor01.lab.interno. 300 IN A 10.0.1.100

# Consultar registro CNAME
dig www.lab.interno

# Salida esperada:
# www.lab.interno.    300 IN CNAME servidor01.lab.interno.
# servidor01.lab.interno. 300 IN A 10.0.1.100

# Ver TTL restante
dig +ttl servidor01.lab.interno

# nslookup alternativo
nslookup servidor01.lab.interno
```

---

### Paso 6 — Experimentar con TTL

```bash
# El TTL controla cuánto tiempo los resolvers cachean el registro

# Cambiar el TTL del registro a 60 segundos desde la consola:
# Route 53 → Hosted zone → seleccionar "servidor01" → Edit record
# TTL: 60 → Save

# Esperar que el TTL anterior expire (300 segundos)
# Cambiar el valor del registro a otra IP
# Observar que la propagación ahora tarda solo 60 segundos en vez de 300

# En producción:
# - Antes de un cambio planeado: bajar TTL a 60 (esperar que expire el TTL actual)
# - Hacer el cambio de IP
# - Después de confirmar que todo funciona: subir TTL de nuevo a 300 o 3600
```

---

## 🟡 LAB INTERMEDIO — Health Checks + Failover Routing + Weighted

**Tiempo estimado: 60 min**
**Prerequisito:** Tener dos instancias EC2 con Apache corriendo (una en us-east-1a, otra en us-east-1b)

---

### Paso 1 — Crear Health Check HTTP

```
Route 53 → Panel izquierdo → "Health checks" → "Create health check"

  Name:              hc-servidor-primario
  What to monitor:   Endpoint

  Specify endpoint by: IP address
  Protocol:          HTTP
  IP address:        54.xxx.xxx.xxx   (IP pública de tu EC2 primaria)
  Port:              80
  Path:              /   (ruta a verificar)

  Advanced configuration:
    Request interval:  Standard (30 seconds)
    Failure threshold: 3   (3 chequeos fallidos → unhealthy)
    String matching:   No
    
    Latency graphs:    ✅ Enable  (ver latencia en CloudWatch)
    Invert:            No
    Disabled:          No

Clic "Create health check"
⏳ Esperar ~1 minuto hasta que el estado sea "Healthy" (verde)
```

---

### Paso 2 — Configurar alarma en el Health Check

```
Route 53 → "Health checks" → seleccionar "hc-servidor-primario"
Pestaña "Alarms" → "Create alarm"

  Create alarm: Yes
  Send notification to new SNS topic
  Topic name:  alarma-rds-dns
  Recipient email: tu@email.com

Clic "Create alarm"
⚠️  Ve a tu email y confirma la suscripción SNS (clic en "Confirm subscription")
```

---

### Paso 3 — Failover Routing (Primary + Secondary)

**Escenario:** Si el servidor principal falla, Route 53 redirige al secundario automáticamente.

```
Necesitas dos registros con el MISMO nombre pero diferente destino.

REGISTRO PRIMARIO:
Route 53 → "Hosted zones" → abrir zona → "Create record"
  Record name:  api
  Record type:  A
  Value:        54.xxx.xxx.10   (IP del servidor primario)
  TTL:          60
  Routing policy: Failover
  Failover record type: Primary
  Health check:    hc-servidor-primario   ← CRÍTICO: asociar health check
  Record ID:       primary-us-east-1
Clic "Create records"
```

```
REGISTRO SECUNDARIO (crear de nuevo con misma configuración):
  Record name:  api
  Record type:  A
  Value:        [URL del S3 Static Website o segunda EC2]
              Para el lab usar: IP de segunda EC2 o endpoint S3
  TTL:          60
  Routing policy: Failover
  Failover record type: Secondary
  Health check:    (opcional para secondary)
  Record ID:       secondary-backup
Clic "Create records"
```

**Simular el failover:**
```bash
# En la instancia EC2 primaria — detener Apache
sudo systemctl stop httpd

# Route 53 realizará 3 health checks fallidos (90 segundos aprox.)
# Luego cambiará automáticamente a devolver la IP del Secundario
```

```
Monitorear en consola:
Route 53 → Health checks → "hc-servidor-primario"
Estado: "Healthy" → "Unhealthy" (rojo)

Esperar ~90 segundos y luego:
dig api.tudominio.com
# Ahora devuelve la IP del servidor secundario ✅

# Restaurar Apache para volver a Primary
sudo systemctl start httpd
# Después de 3 chequeos exitosos (~90 seg) → vuelve a Primary
```

---

### Paso 4 — Weighted Routing (A/B Testing)

```
Escenario: 80% tráfico a servidor v1, 20% a servidor v2

REGISTRO v1 (80%):
Route 53 → "Create record"
  Record name:  checkout
  Record type:  A
  Value:        54.100.100.10   (IP servidor v1)
  TTL:          60
  Routing policy: Weighted
  Weight:         80
  Record ID:      version-1-prod
Clic "Create records"

REGISTRO v2 (20%):
Route 53 → "Create record"
  Record name:  checkout
  Record type:  A
  Value:        54.200.200.20   (IP servidor v2)
  TTL:          60
  Routing policy: Weighted
  Weight:         20
  Record ID:      version-2-canary
Clic "Create records"
```

```bash
# Verificar distribución de tráfico
for i in {1..20}; do
  dig +short checkout.tudominio.com
done | sort | uniq -c | sort -rn
# Resultado esperado:
# 16 54.100.100.10  (v1 ~80%)
#  4 54.200.200.20  (v2 ~20%)
```

---

## 🔴 LAB AVANZADO — Private Hosted Zone + Resolver + DNSSEC

**Tiempo estimado: 70 min**

---

### Parte 1 — DNS Privado con múltiples VPCs

```
Escenario: una Hosted Zone privada compartida entre dos VPCs

Route 53 → "Hosted zones" → seleccionar "lab.interno"
Pestaña "Details" → "Manage associations"

Agregar asociación:
  Region: us-east-1
  VPC:    lab-vpc-02   (la segunda VPC del Módulo 1)

Clic "Associate"
✅ Ahora instancias en lab-vpc-02 también pueden resolver lab.interno
```

---

### Parte 2 — Resolver Outbound Endpoint (ya visto en Módulo 1, profundizamos)

```bash
# Desde instancia EC2 en lab-vpc-01:

# Verificar que el Resolver está funcionando
dig servidor01.lab.interno
# IP interna: 10.0.1.100 ✅

# Verificar resolución de dominios externos (via Resolver → internet)
dig google.com
# Debe resolver correctamente ✅

# Verificar que las reglas de forward aplican correctamente
dig server01.empresa.local
# Si configuraste la regla → se reenvía al DNS on-premise
# Si no hay regla → NXDOMAIN (no encontrado)
```

---

### Parte 3 — DNSSEC en Hosted Zone pública

DNSSEC añade firmas criptográficas a los registros DNS para prevenir ataques de envenenamiento de caché.

```
Route 53 → "Hosted zones" → seleccionar tu zona pública
Clic "DNSSEC signing" → "Enable DNSSEC signing"

  Key-signing key (KSK):
    Name:  lab-ksk-01
    Customer managed CMK:
      Crear nueva en KMS: alias/route53-lab-ksk
      (Route 53 crea la CMK automáticamente en KMS si eliges crear nueva)

Clic "Enable DNSSEC signing"
⏳ Esperar ~1 minuto
```

```
Después de habilitar:
  Route 53 → Hosted zone → pestaña "DNSSEC signing"
  Muestra: "Signing status: Signing"
  
  Puedes ver el DS record que debes publicar en el registrar del dominio:
  Copiar el DS record → publicarlo en tu registrador de dominio
  (Si registraste el dominio en Route 53: se hace automáticamente)
```

```bash
# Verificar DNSSEC desde terminal
dig +dnssec tudominio.com
# En la respuesta verás:
# ;; flags: qr rd ra ad;   ← "ad" significa "Authenticated Data" = DNSSEC válido

# Verificar que DNSSEC está configurado correctamente
dig DS tudominio.com @8.8.8.8
# Debe mostrar el DS record ✅
```

---

### Parte 4 — Geolocation Routing

```
Escenario: usuarios de Perú y Colombia ven contenido en español (servidor Lima),
           resto del mundo ve contenido en inglés (servidor us-east-1)

REGISTRO para Perú:
Route 53 → "Create record"
  Record name:  portal
  Record type:  A
  Value:        54.xxx.xxx.10  (servidor Lima)
  Routing policy: Geolocation
  Location:
    Continent: South America
    Country:   Peru
  Record ID:    geolocacion-peru
Clic "Create records"

REGISTRO para Colombia:
Route 53 → "Create record"
  Record name:  portal
  Record type:  A
  Value:        54.xxx.xxx.10  (mismo servidor Lima para el lab)
  Routing policy: Geolocation
  Location:
    Continent: South America
    Country:   Colombia
  Record ID:    geolocacion-colombia
Clic "Create records"

REGISTRO default (resto del mundo — OBLIGATORIO):
Route 53 → "Create record"
  Record name:  portal
  Record type:  A
  Value:        54.yyy.yyy.20  (servidor us-east-1)
  Routing policy: Geolocation
  Location: Default  ← este registro aplica cuando no hay match específico
  Record ID:    geolocacion-default
Clic "Create records"
```

```bash
# Verificar desde terminal usando un resolver DNS específico
# (simular consulta desde diferente ubicación)

# Desde DNS de Google (US)
dig portal.tudominio.com @8.8.8.8
# Debería retornar IP del servidor us-east-1

# Para simular desde Perú/Colombia → usar VPN o proxy de esas ubicaciones
# O usar: https://www.whatsmydns.net para verificar desde múltiples ubicaciones
```

---

### Parte 5 — Traffic Flow (políticas complejas)

```
Route 53 → Panel izquierdo → "Traffic policies" → "Create traffic policy"

  Policy name:    politica-global-lab
  Version desc:   Latency + Failover combinado

  En el editor visual de Traffic Flow:
  
  1. Seleccionar regla inicial: "Latency rule"
     Regions: us-east-1, us-west-2, sa-east-1
  
  2. Para cada región, agregar debajo:
     "Failover rule" → conectar a:
       Primary: endpoint EC2 de esa región (con health check)
       Secondary: endpoint S3 static de esa región
  
  3. Conectar todo y hacer clic "Create policy"

  Luego: "Create policy record"
    Hosted zone: tu zona
    DNS name:  portal
    Traffic policy: politica-global-lab
    TTL: 60
  Clic "Create policy record"
```

---

## 🧹 LIMPIEZA (en este orden)

```
1.  Route 53 → Health checks → eliminar "hc-servidor-primario"

2.  Route 53 → Traffic policies → eliminar policy records → eliminar políticas

3.  Route 53 → Hosted zones → abrir zona
    Seleccionar TODOS los records excepto NS y SOA (los de tipo NS y SOA son del sistema)
    Actions → Delete records

4.  Route 53 → Hosted zones → "lab.interno" → Delete hosted zone
    (Solo si la eliminaste del lab anterior también)

5.  Route 53 → DNSSEC → Disable DNSSEC signing (antes de eliminar la zona)

6.  KMS → Alias "alias/route53-lab-ksk" → Schedule key deletion

7.  SNS → Topics → eliminar "alarma-rds-dns"
    SNS → Subscriptions → eliminar la suscripción de email
```

---

## 📌 COMANDOS CLI DE REFERENCIA RÁPIDA

```bash
# Listar Hosted Zones
aws route53 list-hosted-zones \
  --query 'HostedZones[*].[Id,Name,Config.PrivateZone]' \
  --output table

# Listar registros de una Hosted Zone
# (reemplazar Z1234567890 con el ID de tu zona)
aws route53 list-resource-record-sets \
  --hosted-zone-id Z1234567890 \
  --query 'ResourceRecordSets[*].[Name,Type,TTL]' \
  --output table

# Ver health checks
aws route53 list-health-checks \
  --query 'HealthChecks[*].[Id,HealthCheckConfig.IPAddress,HealthCheckConfig.Type]' \
  --output table

# Ver estado actual de un health check
aws route53 get-health-check-status \
  --health-check-id HEALTH-CHECK-ID \
  --query 'CheckerIpRanges' \
  --output table

# Verificar DNS desde múltiples ubicaciones globales
# herramienta externa:
# https://www.whatsmydns.net/#A/tudominio.com

# Simular diferentes regiones con dig
dig @ns-1234.awsdns-12.co.uk tudominio.com   # nameserver UK
dig @ns-5678.awsdns-34.com tudominio.com     # nameserver US
```
