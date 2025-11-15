# 

# README – OBJETIVO 2: Autoescalabilidad con ALB + ASG + RDS + EFS

**Estudiante:** Juan Pablo Mejia, Felipe Martinez y Santiago Palacio

**Curso:** ST0263 – Tópicos Especiales en Telemática

**Periodo:** 2025-2

---

## 1. Descripción general del objetivo

### Objetivo 2 (Autoescalabilidad)

Desplegar la app monolítica BookStore detrás de un Application Load Balancer (ALB) con un Auto Scaling Group (ASG) en subredes privadas, usando RDS MySQL como base de datos gestionada y EFS como almacenamiento compartido para archivos (uploads/logs). El tráfico público entra por el ALB en subredes públicas; las instancias se reemplazan automáticamente y escalan en al menos 2 AZ.

---

## 1.1. Alcance logrado

- VPC propia `p2-escala-vpc` con 2 públicas y 2 privadas (us-east-1a/b) + NAT Gateway (In 1 AZ).
- RDS MySQL (Single-AZ, sin extras de costo) con endpoint:
    - `database-1.crkmul8djqwa.us-east-1.rds.amazonaws.com` (DB `bookstore`, user `bookuser`).
- EFS `bookstore-EFS` (Regional, Bursting, sin backups) con mount targets en ambas privadas y SG `APP-SG`.
- Launch Template `bookstore-lt` (Amazon Linux 2023 + script user-data robusto).
- Target Group `bookstore-tg` (targets = Instances, HTTP:80, health check en `/`).
- Application Load Balancer `bookstore-alb` (público, 2 subredes públicas, SG `ALB-SG`, listener :80 → TG).
- Auto Scaling Group `bookstore-asg` (min 1 / desired 2 / max 3; subredes privadas; health check ELB).
- App desplegada desde GitHub:
    - Repo: `https://github.com/Camus0023/AWS---Final-Project.git`
    - Subcarpeta: `BookStore-monolith` (contiene `docker-compose.yml`).
- Health checks Healthy y ALB sirviendo contenido; diagnóstico de puertos realizado y corrección del compose para publicar :80.

---

## 1.2. Lo que no cubre este objetivo

- HTTPS en el ALB (ACM/443) y dominios Route53/DuckDNS al ALB.
- CI/CD.
- Contenedorización multi-servicio o migración a microservicios/orquestadores.
- Multi-AZ en RDS (por ahorro de créditos en Learner Lab).

---

## 2. Arquitectura

### 2.1. Diagrama lógico (alto nivel)

```
                         Internet
                            │
                            ▼
                 ┌──────────────────────┐
                 │  bookstore-alb (ALB) │
                 │   Security Group:    │
                 │      ALB-SG          │
                 │   Puerto 80 (HTTP)   │
                 └──────────┬───────────┘
                            │
              ┌─────────────┴─────────────┐
              │                           │
              ▼                           ▼
       ┌─────────────┐           ┌─────────────┐
       │  Subnet      │           │  Subnet     │
       │  Pública 1   │           │  Pública 2  │
       │  (us-east-1a)│           │  (us-east-1b)│
       └──────────────┘           └─────────────┘
              │                           │
              └─────────────┬─────────────┘
                            │
                 ┌──────────▼───────────┐
                 │   Target Group:      │
                 │   bookstore-tg       │
                 │   Health Check: /    │
                 └──────────┬───────────┘
                            │
              ┌─────────────┴─────────────┐
              │                           │
              ▼                           ▼
       ┌─────────────┐           ┌─────────────┐
       │  Subnet      │           │  Subnet     │
       │  Privada 1   │           │  Privada 2  │
       │  (us-east-1a)│           │  (us-east-1b)│
       │              │           │              │
       │ ┌─────────┐  │           │ ┌─────────┐ │
       │ │ EC2     │  │           │ │ EC2     │ │
       │ │Instance │  │           │ │Instance │ │
       │ │(ASG)    │  │           │ │(ASG)    │ │
       │ │Docker + │  │           │ │Docker + │ │
       │ │Flask    │  │           │ │Flask    │ │
       │ └────┬────┘  │           │ └────┬────┘ │
       │      │       │           │      │      │
       └──────┼───────┘           └──────┼──────┘
              │                          │
              └────────┬─────────────────┘
                       │
              ┌────────▼────────┐
              │                 │
         ┌────▼─────┐    ┌─────▼──────┐
         │   EFS    │    │  RDS MySQL │
         │bookstore-│    │ database-1 │
         │   EFS    │    │(Single-AZ) │
         │(Regional)│    │            │
         └──────────┘    └────────────┘
              │
      (Mount Targets en
       subredes privadas)

        ┌──────────────────┐
        │   NAT Gateway    │
        │  (Subnet Pública)│
        └────────┬─────────┘
                 │
         (Salida a Internet
          para subredes
           privadas)

```

---

## 2.2. Componentes principales

### VPC y Networking

- **VPC:** `p2-escala-vpc`
    - CIDR: 10.0.0.0/16 (ejemplo)
    - Subredes públicas: 2 (us-east-1a, us-east-1b)
    - Subredes privadas: 2 (us-east-1a, us-east-1b)
    - Internet Gateway (IGW) para subredes públicas
    - NAT Gateway en 1 AZ para salida de subredes privadas

### Application Load Balancer (ALB)

- **Nombre:** `bookstore-alb`
- **Tipo:** Application Load Balancer
- **Esquema:** Internet-facing
- **Subredes:** 2 públicas (us-east-1a/b)
- **Security Group:** `ALB-SG`
    - Inbound: Puerto 80 desde 0.0.0.0/0
- **Listener:** HTTP:80 → Target Group `bookstore-tg`

### Target Group

- **Nombre:** `bookstore-tg`
- **Tipo:** Instances
- **Protocolo:** HTTP
- **Puerto:** 80
- **Health Check:** `/` (ruta raíz)
- **Health Check Settings:**
    - Interval: 30s
    - Timeout: 5s
    - Healthy threshold: 2
    - Unhealthy threshold: 2

### Auto Scaling Group (ASG)

- **Nombre:** `bookstore-asg`
- **Launch Template:** `bookstore-lt`
- **Subredes:** 2 privadas (us-east-1a/b)
- **Capacidad:**
    - Mínimo: 1
    - Deseado: 2
    - Máximo: 3
- **Health Check Type:** ELB
- **Health Check Grace Period:** 300s

### Launch Template

- **Nombre:** `bookstore-lt`
- **AMI:** Amazon Linux 2023
- **Instance Type:** t3.micro / t2.micro
- **Security Group:** `APP-SG`
    - Inbound:
        - Puerto 80 desde ALB-SG
        - Puerto 22 desde IP administrativa (opcional)
        - Puerto 2049 (NFS) para EFS
- **User Data:** Script de inicialización que instala Docker, monta EFS y despliega la aplicación

### RDS MySQL

- **Identifier:** `database-1`
- **Endpoint:** `database-1.crkmul8djqwa.us-east-1.rds.amazonaws.com`
- **Engine:** MySQL 8.0
- **Deployment:** Single-AZ
- **Database:** `bookstore`
- **Usuario:** `bookuser`
- **Security Group:** `RDS-SG`
    - Inbound: Puerto 3306 desde APP-SG

### Elastic File System (EFS)

- **Nombre:** `bookstore-EFS`
- **Performance Mode:** General Purpose
- **Throughput Mode:** Bursting
- **Lifecycle Management:** Deshabilitado
- **Backups:** Deshabilitados
- **Mount Targets:** En ambas subredes privadas
- **Security Group:** `APP-SG` (puerto 2049/NFS)

---

## 3. Tecnologías utilizadas

- **Infraestructura:** AWS (VPC, EC2, ALB, ASG, RDS, EFS, NAT Gateway)
- **Sistema Operativo:** Amazon Linux 2023
- **Contenedores:** Docker + Docker Compose
- **Backend:** Python 3.10, Flask, SQLAlchemy
- **Base de datos:** RDS MySQL 8.0
- **Almacenamiento compartido:** EFS (NFS)
- **Balanceo de carga:** Application Load Balancer (ALB)
- **Autoescalado:** Auto Scaling Group (ASG)
- **Control de versiones:** GitHub

---

## 4. Guía de implementación (paso a paso)

### 4.1. Crear VPC y subredes

1. Crear VPC `p2-escala-vpc` con CIDR 10.0.0.0/16
2. Crear subredes:
    - Pública 1: 10.0.1.0/24 (us-east-1a)
    - Pública 2: 10.0.2.0/24 (us-east-1b)
    - Privada 1: 10.0.11.0/24 (us-east-1a)
    - Privada 2: 10.0.12.0/24 (us-east-1b)
3. Crear Internet Gateway y asociarlo a la VPC
4. Crear NAT Gateway en una subnet pública
5. Configurar tablas de rutas:
    - Tabla pública: 0.0.0.0/0 → IGW
    - Tabla privada: 0.0.0.0/0 → NAT Gateway

### 4.2. Crear Security Groups

**ALB-SG (ALB Security Group):**

```
Inbound:
- Type: HTTP, Protocol: TCP, Port: 80, Source: 0.0.0.0/0
Outbound:
- All traffic

```

**APP-SG (Application Security Group):**

```
Inbound:
- Type: HTTP, Protocol: TCP, Port: 80, Source: ALB-SG
- Type: SSH, Protocol: TCP, Port: 22, Source: My IP (opcional)
- Type: NFS, Protocol: TCP, Port: 2049, Source: APP-SG (self-reference)
Outbound:
- All traffic

```

**RDS-SG (Database Security Group):**

```
Inbound:
- Type: MySQL/Aurora, Protocol: TCP, Port: 3306, Source: APP-SG
Outbound:
- All traffic

```

### 4.3. Crear RDS MySQL

1. Ir a RDS Console → Create database
2. Configuración:
    - Engine: MySQL 8.0
    - Template: Free tier / Dev/Test
    - DB Instance Identifier: `database-1`
    - Master username: `admin`
    - Master password: (tu contraseña segura)
    - DB Instance Class: db.t3.micro / db.t2.micro
    - Storage: 20 GB gp2
    - **Multi-AZ:** No (Single-AZ para ahorrar)
    - VPC: `p2-escala-vpc`
    - Subnet group: (crear con subredes privadas)
    - Public access: No
    - VPC Security Group: `RDS-SG`
    - Database name: `bookstore`
3. Esperar a que el estado sea "Available"
4. Copiar el endpoint: `database-1.crkmul8djqwa.us-east-1.rds.amazonaws.com`

**Crear usuario de aplicación:**

```sql
-- Conectarse como admin
mysql -h database-1.crkmul8djqwa.us-east-1.rds.amazonaws.com -u admin -p

-- Crear usuario
CREATE USER 'bookuser'@'%' IDENTIFIED BY 'tu_password_segura';
GRANT ALL PRIVILEGES ON bookstore.* TO 'bookuser'@'%';
FLUSH PRIVILEGES;

```

### 4.4. Crear EFS

1. Ir a EFS Console → Create file system
2. Configuración:
    - Name: `bookstore-EFS`
    - VPC: `p2-escala-vpc`
    - Availability: Regional
    - Performance mode: General Purpose
    - Throughput mode: Bursting
    - Lifecycle management: None
    - Encryption: Enabled (opcional)
3. Mount targets:
    - Subnet privada 1 (us-east-1a): Security Group `APP-SG`
    - Subnet privada 2 (us-east-1b): Security Group `APP-SG`
4. Copiar el File System ID: `fs-xxxxxxxxx`

### 4.5. Crear Launch Template

**User Data Script:**

```bash
#!/bin/bash
# Actualizar sistema
yum update -y

# Instalar Docker
yum install -y docker
systemctl start docker
systemctl enable docker
usermod -aG docker ec2-user

# Instalar Docker Compose
curl -L "<https://github.com/docker/compose/releases/latest/download/docker-compose-$>(uname -s)-$(uname -m)" -o /usr/local/bin/docker-compose
chmod +x /usr/local/bin/docker-compose
ln -s /usr/local/bin/docker-compose /usr/bin/docker-compose

# Instalar amazon-efs-utils para montar EFS
yum install -y amazon-efs-utils

# Crear punto de montaje para EFS
mkdir -p /mnt/efs

# Montar EFS (reemplazar fs-xxxxxxxxx con tu File System ID)
echo "fs-xxxxxxxxx:/ /mnt/efs efs _netdev,tls 0 0" >> /etc/fstab
mount -a

# Esperar a que EFS esté montado
sleep 10

# Crear directorios de la aplicación
mkdir -p /opt/bookstore
cd /opt/bookstore

# Clonar repositorio
yum install -y git
git clone <https://github.com/Camus0023/AWS---Final-Project.git> .

# Navegar a la subcarpeta de la app
cd BookStore-monolith

# Configurar variables de entorno para la base de datos
cat > .env <<EOF
DATABASE_URL=mysql+pymysql://bookuser:tu_password_segura@database-1.crkmul8djqwa.us-east-1.rds.amazonaws.com:3306/bookstore
FLASK_ENV=production
EOF

# Modificar docker-compose.yml para exponer puerto 80
# Asegurar que el puerto mapeado sea 80:5000
cat > docker-compose.yml <<EOF
version: '3.8'
services:
  flaskapp:
    build: .
    container_name: flaskapp
    environment:
      - FLASK_ENV=production
      - DATABASE_URL=mysql+pymysql://bookuser:tu_password_segura@database-1.crkmul8djqwa.us-east-1.rds.amazonaws.com:3306/bookstore
    ports:
      - "80:5000"
    volumes:
      - /mnt/efs/uploads:/app/uploads
      - /mnt/efs/logs:/app/logs
    restart: unless-stopped
EOF

# Construir e iniciar la aplicación
docker-compose build --no-cache
docker-compose up -d

# Verificar estado
docker-compose ps

```

**Crear Launch Template en AWS Console:**

1. EC2 → Launch Templates → Create launch template
2. Configuración:
    - Name: `bookstore-lt`
    - AMI: Amazon Linux 2023
    - Instance type: t3.micro
    - Key pair: (tu key pair para SSH, opcional)
    - Network settings:
        - Security groups: `APP-SG`
    - Advanced details:
        - User data: (pegar el script anterior)
3. Create launch template

### 4.6. Crear Target Group

1. EC2 → Target Groups → Create target group
2. Configuración:
    - Target type: Instances
    - Target group name: `bookstore-tg`
    - Protocol: HTTP
    - Port: 80
    - VPC: `p2-escala-vpc`
    - Health check:
        - Protocol: HTTP
        - Path: `/`
        - Port: traffic port
        - Healthy threshold: 2
        - Unhealthy threshold: 2
        - Timeout: 5
        - Interval: 30
3. Create target group

### 4.7. Crear Application Load Balancer

1. EC2 → Load Balancers → Create load balancer
2. Seleccionar Application Load Balancer
3. Configuración:
    - Name: `bookstore-alb`
    - Scheme: Internet-facing
    - IP address type: IPv4
    - Network mapping:
        - VPC: `p2-escala-vpc`
        - Subnets: Seleccionar ambas subredes públicas
    - Security groups: `ALB-SG`
    - Listeners:
        - Protocol: HTTP
        - Port: 80
        - Default action: Forward to `bookstore-tg`
4. Create load balancer
5. Copiar el DNS name del ALB (ej: `bookstore-alb-xxxxxxxxx.us-east-1.elb.amazonaws.com`)

### 4.8. Crear Auto Scaling Group

1. EC2 → Auto Scaling Groups → Create Auto Scaling group
2. Configuración paso 1 (Launch template):
    - Name: `bookstore-asg`
    - Launch template: `bookstore-lt`
3. Paso 2 (Network):
    - VPC: `p2-escala-vpc`
    - Subnets: Seleccionar ambas subredes privadas
4. Paso 3 (Load balancing):
    - Attach to an existing load balancer
    - Choose from your load balancer target groups: `bookstore-tg`
    - Health checks:
        - EC2: Enabled
        - ELB: Enabled
        - Health check grace period: 300 seconds
5. Paso 4 (Group size and scaling):
    - Desired capacity: 2
    - Minimum capacity: 1
    - Maximum capacity: 3
    - Scaling policies: None (o Target tracking si deseas)
6. Create Auto Scaling group

### 4.9. Verificar despliegue

**Esperar que las instancias se lancen y pasen health checks:**

```bash
# Verificar estado del ASG
aws autoscaling describe-auto-scaling-groups --auto-scaling-group-names bookstore-asg

# Verificar targets en el Target Group
aws elbv2 describe-target-health --target-group-arn <ARN_del_TG>

```

**Acceder a la aplicación:**

```bash
# Obtener DNS del ALB
ALB_DNS=$(aws elbv2 describe-load-balancers --names bookstore-alb --query 'LoadBalancers[0].DNSName' --output text)

# Probar acceso
curl -I http://$ALB_DNS

# Acceder desde navegador
echo "http://$ALB_DNS"

```

---

## 5. URL de acceso

**Aplicación desplegada (Objetivo 2):**

👉 `http://bookstore-alb-xxxxxxxxx.us-east-1.elb.amazonaws.com`

*(Reemplazar con el DNS real del ALB)*

---

## 6. Pruebas y validación

### 6.1. Verificar health checks

```bash
# Desde AWS Console:
# EC2 → Target Groups → bookstore-tg → Targets tab
# Verificar que ambas instancias muestren estado "healthy"

# Desde CLI:
aws elbv2 describe-target-health --target-group-arn <TG_ARN>

```

### 6.2. Verificar balanceo de carga

```bash
# Hacer múltiples requests y verificar que se distribuyen entre instancias
for i in {1..10}; do
  curl -s http://$ALB_DNS | grep -o "Instance ID: .*" || echo "Request $i"
  sleep 1
done

```

### 6.3. Verificar autoescalado

```bash
# Verificar capacidad actual
aws autoscaling describe-auto-scaling-groups \\
  --auto-scaling-group-names bookstore-asg \\
  --query 'AutoScalingGroups[0].[MinSize,DesiredCapacity,MaxSize]'

# Terminar una instancia manualmente y verificar que ASG la reemplaza
aws ec2 terminate-instances --instance-ids <INSTANCE_ID>

# Esperar y verificar que se lanza una nueva instancia
aws autoscaling describe-auto-scaling-instances \\
  --query 'AutoScalingInstances[?AutoScalingGroupName==`bookstore-asg`]'

```

### 6.4. Verificar EFS

```bash
# Conectarse a una instancia vía Systems Manager Session Manager o SSH
ssh -i tu-key.pem ec2-user@<INSTANCE_PRIVATE_IP>

# Verificar montaje de EFS
df -h | grep efs
ls -la /mnt/efs

# Crear archivo de prueba
echo "Test from instance $(hostname)" > /mnt/efs/test.txt

# Desde otra instancia, verificar que el archivo está disponible
cat /mnt/efs/test.txt

```

### 6.5. Verificar conectividad a RDS

```bash
# Desde una instancia del ASG
docker exec -it flaskapp bash

# Instalar cliente MySQL (si no está)
apt-get update && apt-get install -y default-mysql-client

# Probar conexión
mysql -h database-1.crkmul8djqwa.us-east-1.rds.amazonaws.com -u bookuser -p

# Verificar base de datos
SHOW DATABASES;
USE bookstore;
SHOW TABLES;

```

---

## 7. Problemas encontrados y soluciones

### 7.1. Instancias fallan health checks

**Problema:** Las instancias en el Target Group permanecen en estado "unhealthy".

**Causas posibles:**

- La aplicación no está escuchando en el puerto 80
- El Security Group APP-SG no permite tráfico desde ALB-SG en puerto 80
- La aplicación tarda más en iniciar que el grace period

**Soluciones:**

```bash
# Verificar que docker-compose mapea correctamente el puerto
# En docker-compose.yml debe ser:
ports:
  - "80:5000"  # NO "127.0.0.1:5000:5000"

# Aumentar health check grace period
aws autoscaling update-auto-scaling-group \\
  --auto-scaling-group-name bookstore-asg \\
  --health-check-grace-period 600

# Verificar logs de la aplicación
docker logs flaskapp

```

### 7.2. EFS no monta correctamente

**Problema:** `/mnt/efs` está vacío o muestra errores al montar.

**Causas:**

- Security Group no permite NFS (puerto 2049)
- Mount targets no están en las subredes correctas
- amazon-efs-utils no está instalado

**Soluciones:**

```bash
# Verificar que APP-SG permite NFS desde sí mismo
# Inbound rule: Type NFS, Port 2049, Source APP-SG

# Verificar instalación de efs-utils
yum list installed | grep amazon-efs-utils

# Instalar si no está
yum install -y amazon-efs-utils

# Verificar montaje
mount | grep efs

# Montar manualmente para diagnosticar
mount -t efs -o tls fs-xxxxxxxxx:/ /mnt/efs

```

### 7.3. No hay conectividad a RDS

**Problema:** La aplicación no puede conectarse a RDS MySQL.

**Causas:**

- RDS-SG no permite conexiones desde APP-SG
- Credenciales incorrectas en la cadena de conexión
- RDS no está en estado "Available"

**Soluciones:**

```bash
# Verificar Security Group de RDS
# Debe permitir: Port 3306, Source APP-SG

# Probar conectividad desde instancia
telnet database-1.crkmul8djqwa.us-east-1.rds.amazonaws.com 3306

# Verificar credenciales en .env o docker-compose.yml
cat /opt/bookstore/BookStore-monolith/.env

```

### 7.4. User data no se ejecuta correctamente

**Problema:** La aplicación no se despliega automáticamente al lanzar instancias.

**Soluciones:**

```bash
# Verificar logs de cloud-init
sudo cat /var/log/cloud-init-output.log

# Verificar si Docker está corriendo
systemctl status docker

# Verificar si los contenedores están up
docker ps -a

# Re-ejecutar manualmente el script si es necesario
cd /opt/bookstore/BookStore-monolith
docker-compose down
docker-compose up -d --build

```

### 7.5. Aplicación expone puerto incorrecto

**Problema:** Se diagnosticó que el `docker-compose.yml` original mapeaba el puerto como `127.0.0.1:5000:5000`, haciéndolo inaccesible desde el ALB.

**Solución implementada:**

```yaml
# Cambio en docker-compose.yml
# ANTES:
ports:
  - "127.0.0.1:5000:5000"

# DESPUÉS:
ports:
  - "80:5000"

```

---

## 8. Optimizaciones y mejoras futuras

### 8.1. Implementar HTTPS

- Solicitar certificado SSL en AWS Certificate Manager (ACM)
- Configurar listener 443 en el ALB
- Redirigir HTTP (80) → HTTPS (443)

### 8.2. Configurar dominio personalizado

- Registrar dominio en Route53 o usar DuckDNS
- Crear A record apuntando al ALB
- Actualizar certificado SSL para incluir el dominio

### 8.3. Implementar políticas de escalado dinámico

```bash
# Target tracking scaling policy basada en CPU
aws autoscaling put-scaling-policy \\
  --auto-scaling-group-name bookstore-asg \\
  --policy-name cpu-target-tracking \\
  --policy-type TargetTrackingScaling \\
  --target-tracking-configuration file://scaling-policy.json

```

**scaling-policy.json:**

```json
{
  "PredefinedMetricSpecification": {
    "PredefinedMetricType": "ASGAverageCPUUtilization"
  },
  "TargetValue": 70.0
}

```

### 8.4. Migrar RDS a Multi-AZ

Para entornos de producción:

```bash
aws rds modify-db-instance \\
  --db-instance-identifier database-1 \\
  --multi-az \\
  --apply-immediately

```

### 8.5. Implementar CloudWatch Alarms

```bash
# Alarma para CPU alta
aws cloudwatch put-metric-alarm \\
  --alarm-name bookstore-high-cpu \\
  --comparison-operator GreaterThanThreshold \\
  --evaluation-periods 2 \\
  --metric-name CPUUtilization \\
  --namespace AWS/EC2 \\
  --period 300 \\
  --statistic Average \\
  --threshold 80 \\
  --alarm-actions <SNS_TOPIC_ARN>

```

### 8.6. Configurar backups automáticos

- Habilitar snapshots automáticos de RDS
- Configurar AWS Backup para EFS
- Implementar versionado de imágenes Docker

---

## 9. Conclusiones

Al finalizar el Objetivo 2, se logró:

- Implementar una arquitectura altamente disponible con ALB + ASG desplegada en múltiples zonas de disponibilidad
- Migrar la base de datos a RDS MySQL, eliminando la necesidad de gestionar la base de datos manualmente
- Implementar almacenamiento compartido con EFS para archivos estáticos y logs
- Configurar autoescalado automático con health checks de ELB
- Aislar las instancias de aplicación en subredes privadas, mejorando la seguridad
- Establecer una arquitectura de red robusta con VPC personalizada y NAT Gateway
- Lograr alta disponibilidad mediante despliegue multi-AZ y recuperación automática de instancias

**Comparación con Objetivo 1:**

| Aspecto | Objetivo 1 | Objetivo 2 |
| --- | --- | --- |
| Base de datos | MySQL en EC2 | RDS MySQL gestionado |
| Almacenamiento | Local en instancia | EFS compartido |
| Escalabilidad | Manual | Automática (ASG) |
| Disponibilidad | Single instance | Multi-AZ con ALB |
| Mantenimiento | Alto (manual) | Bajo (servicios gestionados) |
| Costo | Bajo | Moderado |

Esta arquitectura proporciona una base sólida para entornos de producción, con capacidad de escalar automáticamente según la demanda y recuperarse de fallos sin intervención manual.

---

## 10. Referencias

- **AWS VPC:** https://docs.aws.amazon.com/vpc/
- **Application Load Balancer:** https://docs.aws.amazon.com/elasticloadbalancing/latest/application/
- **Auto Scaling Groups:** https://docs.aws.amazon.com/autoscaling/ec2/userguide/
- **Amazon RDS:** https://docs.aws.amazon.com/rds/
- **Amazon EFS:** https://docs.aws.amazon.com/efs/
- **Launch Templates:** https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/ec2-launch-templates.html
- **Docker Compose:** https://docs.docker.com/compose/
- **Flask:** https://flask.palletsprojects.com/
- **AWS Security Groups:** https://docs.aws.amazon.com/vpc/latest/userguide/VPC_SecurityGroups.html
