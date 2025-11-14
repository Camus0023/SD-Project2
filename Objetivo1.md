# README - OBJETIVO 1: Despliegue Monolítico con Dominio, SSL y NGINX

**Estudiantes: Juan Pablo Mejia Perez, Felipe Martinez y Santiago Palacio**

**Curso:** ST0263 - Tópicos Especiales en Telemática

**Período:** 2025-2

---

## 1. Descripción general del objetivo

**Objetivo 1 (20%)**

> Desplegar la aplicación BookStore Monolítica en dos (2) máquinas virtuales en AWS,
con un dominio propio, certificado SSL y proxy inverso en NGINX
(un servidor para la base de datos y otro servidor para la aplicación + NGINX).
> 

En este objetivo se implementa una arquitectura **cliente–servidor** simple pero realista en la nube:

- **1 VM de aplicación**: corre la app monolítica BookStore (Flask en Docker) + NGINX (reverse proxy).
- **1 VM de base de datos**: corre MySQL 8 en Docker.
- Se expone la aplicación en Internet a través de:
    - Un **dominio público** ([projectx2.duckdns.org](http://projectx2.duckdns.org/)).
    - Un **certificado SSL** emitido por **Let's Encrypt**.
    - NGINX como **terminador TLS** y proxy inverso hacia Flask.

---

## 1.1. Aspectos desarrollados

En el desarrollo del **Objetivo 1** se realizaron las siguientes actividades:

- Creación de dos instancias **EC2** en **AWS Academy**:
    - **BookStore-APP**: Servidor de aplicación + NGINX + Docker.
    - **BookStore-DB**: Servidor de base de datos MySQL (Docker).
- Instalación y configuración de:
    - **Docker Engine** y **Docker Compose Plugin** en ambas VMs.
    - **NGINX** en la VM de aplicación.
    - **Certbot** para la gestión de certificados SSL.
- Despliegue de la aplicación **BookStore-monolith** en un contenedor Docker (Flask).
- Configuración de la conexión de Flask hacia MySQL usando SQLAlchemy + PyMySQL.
- Registro y uso de un **dominio gratuito** en **DuckDNS**:
    - [projectx2.duckdns.org](http://projectx2.duckdns.org/) apuntando a la **Elastic IP** de la VM de aplicación.
- Emisión y configuración de un **certificado SSL** con **Let's Encrypt**.
- Redirección automática **HTTP → HTTPS** en NGINX.
- Configuración de **Security Groups** para:
    - Exponer solo puertos 22, 80 y 443 en la VM de aplicación.
    - Exponer MySQL (3306) únicamente hacia la VM de aplicación (no a Internet).
- Validación del funcionamiento mediante curl y navegador.

---

## 1.2. Aspectos no desarrollados en este objetivo

A propósito, en el **Objetivo 1** todavía **no se implementan**:

- Balanceadores de carga (ALB) ni Auto Scaling Group (ASG).
- Migración a servicios manejados como RDS.
- Pipelines CI/CD.
- Contenedorización de NGINX o DB en orquestadores (Kubernetes, ECS, etc.).

Estos aspectos quedan para objetivos posteriores del proyecto.

---

## 2. Diseño y arquitectura

### 2.1. Arquitectura lógica

```
                         Internet
                            │
                            ▼
                  projectx2.duckdns.org
                            │
                     (DNS → Elastic IP)
                            │
                            ▼
                 [ BookStore-APP (EC2) ]
             Ubuntu 24.04 + Docker + NGINX + Flask
              └─ Contenedor Flask (puerto 5000)
                            │
                (VPC / Red privada / SG)
                            │
                            ▼
                 [ BookStore-DB (EC2) ]
               Ubuntu 24.04 + Docker + MySQL 8
                 └─ Contenedor MySQL (3306)

```

### 2.2. Componentes

**BookStore-APP (VM de aplicación + NGINX)**

- SO: Ubuntu 24.04.3 LTS
- Servicios principales:
    - Docker Engine + Docker Compose.
    - Contenedor Flask (BookStore monolítica).
    - NGINX como reverse proxy.
    - Certbot (Let's Encrypt) para SSL.
- Puertos expuestos (Security Group SG-APP):
    - 22 (SSH) – desde IP del estudiante.
    - 80 (HTTP) – desde 0.0.0.0/0 (para redirección + Certbot).
    - 443 (HTTPS) – desde 0.0.0.0/0.

**BookStore-DB (VM de base de datos)**

- SO: Ubuntu 24.04.3 LTS
- Servicios:
    - Docker Engine.
    - Contenedor MySQL 8.4 usando imagen oficial mysql:8.
- Puertos (Security Group SG-DB):
    - 22 (SSH) – desde IP del estudiante.
    - 3306 (MySQL) – solo desde SG-APP (no accesible desde Internet).

**DNS y SSL**

- Dominio: [projectx2.duckdns.org](http://projectx2.duckdns.org/).
- Apunta a la Elastic IP de BookStore-APP (por ejemplo 18.233.65.44).
- Certificado: Let's Encrypt + Certbot configurado en NGINX.

---

## 3. Tecnologías utilizadas

- Infraestructura: AWS Academy – EC2 (IaaS).
- Sistemas Operativos: Ubuntu Server 24.04.3 LTS.
- Contenedores: Docker Engine + Docker Compose.
- Backend: Python 3.10, Flask, SQLAlchemy.
- Base de datos: MySQL 8 (imagen mysql:8).
- Cliente MySQL: PyMySQL.
- Capa web: NGINX (reverse proxy + SSL).
- Certificados SSL: Let's Encrypt (Certbot con plugin nginx).
- DNS dinámico: DuckDNS ([projectx2.duckdns.org](http://projectx2.duckdns.org/)).

---

## 4. Ambiente de ejecución

### 4.1. BookStore-APP (VM Aplicación + NGINX)

- Nombre: BookStore-APP.
- Tipo de instancia: t3.micro (según laboratorio).
- SO: Ubuntu 24.04.3 LTS.
- Elastic IP pública: 18.233.65.44 (ejemplo).
- Security Group SG-APP:
    - Inbound:
        - 22/tcp desde IP del estudiante.
        - 80/tcp desde 0.0.0.0/0.
        - 443/tcp desde 0.0.0.0/0.
    - Outbound: todo permitido (por defecto).

**Servicios activos:**

```bash
# NGINX
sudo systemctl status nginx

# Contenedores
docker ps

```

### 4.2. BookStore-DB (VM Base de Datos)

- Nombre: BookStore-DB.
- Tipo: t3.micro / t3.small (según lab).
- SO: Ubuntu 24.04.3 LTS.
- IP privada: 172.31.x.y (misma VPC).
- Security Group SG-DB:
    - 22/tcp desde IP del estudiante.
    - 3306/tcp solo desde SG-APP.

**Servicios:**

```bash
docker ps     # contenedor MySQL
docker logs bookstore-mysql --tail=50

```

---

## 5. Guía de instalación y configuración (paso a paso)

### 5.1. Paso 1 – Crear y configurar las VMs en AWS

1. Abrir el laboratorio de AWS Academy y la consola de EC2.
2. Crear BookStore-APP:
    - AMI: Ubuntu Server 24.04.
    - Security Group: SG-APP (22, 80, 443).
    - Asignar una Elastic IP.
3. Crear BookStore-DB:
    - AMI: Ubuntu Server 24.04.
    - Security Group: SG-DB (22, 3306 solo desde SG-APP).
4. Verificar conectividad básica entre instancias (ping IP privada, etc., si SG lo permite).

### 5.2. Paso 2 – Configurar MySQL en Docker (BookStore-DB)

En BookStore-DB:

```bash
# 1. Conexión SSH
ssh -i SD-Project.pem ubuntu@<IP_PUBLICA_DB>

# 2. Instalar Docker (repos oficial)
sudo apt update
sudo apt install -y ca-certificates curl gnupg
sudo install -m 0755 -d /etc/apt/keyrings
curl -fsSL <https://download.docker.com/linux/ubuntu/gpg> | sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg
echo \\
  "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] \\
  <https://download.docker.com/linux/ubuntu> $(. /etc/os-release; echo $VERSION_CODENAME) stable" \\
  | sudo tee /etc/apt/sources.list.d/docker.list >/dev/null
sudo apt update
sudo apt install -y docker-ce docker-ce-cli containerd.io docker-compose-plugin
sudo usermod -aG docker ubuntu
exit

# 3. Reingresar para tomar el grupo docker
ssh -i SD-Project.pem ubuntu@<IP_PUBLICA_DB>

# 4. Crear volumen y levantar MySQL
docker volume create bookstore_db_data

docker run -d --name bookstore-mysql \\
  -e MYSQL_DATABASE=bookstore \\
  -e MYSQL_USER=bookstore_user \\
  -e MYSQL_PASSWORD=bookstore_pass \\
  -e MYSQL_ROOT_PASSWORD=root_pass \\
  -p 3306:3306 \\
  -v bookstore_db_data:/var/lib/mysql \\
  mysql:8

# 5. Verificar estado
docker ps
docker logs bookstore-mysql --tail=50

```

### 5.3. Paso 3 – Configurar BookStore-APP (Flask + Docker + NGINX)

En BookStore-APP:

```bash
# 1. Conectarse por SSH
ssh -i SD-Project.pem ubuntu@18.233.65.44

# 2. Instalar Docker + NGINX + Certbot
sudo apt update
sudo apt install -y ca-certificates curl gnupg nginx certbot python3-certbot-nginx
sudo install -m 0755 -d /etc/apt/keyrings
curl -fsSL <https://download.docker.com/linux/ubuntu/gpg> | sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg
echo \\
  "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] \\
  <https://download.docker.com/linux/ubuntu> $(. /etc/os-release; echo $VERSION_CODENAME) stable" \\
  | sudo tee /etc/apt/sources.list.d/docker.list >/dev/null
sudo apt update
sudo apt install -y docker-ce docker-ce-cli containerd.io docker-compose-plugin
sudo usermod -aG docker ubuntu
sudo systemctl enable nginx
sudo systemctl start nginx
exit

# 3. Reingresar
ssh -i SD-Project.pem ubuntu@18.233.65.44

```

### 5.3.1. Clonar y preparar la aplicación BookStore

```bash
cd ~
git clone <https://github.com/><tu_usuario>/BookStore-monolith.git
cd BookStore-monolith

# Asegurar dependencias
grep -qi '^pymysql' requirements.txt || echo "pymysql" | sudo tee -a requirements.txt
grep -qi '^cryptography' requirements.txt || echo "cryptography" | sudo tee -a requirements.txt

```

Configurar la URL de base de datos (usando IP privada de BookStore-DB):

```python
# app.py (fragmento)
import os

app.config["SQLALCHEMY_DATABASE_URI"] = os.environ.get(
    "DATABASE_URL",
    "mysql+pymysql://bookstore_user:bookstore_pass@<IP_PRIVADA_DB>:3306/bookstore"
)

```

### 5.3.2. Docker Compose para producción

`docker-compose.prod.yml`:

```yaml
services:
  flaskapp:
    build: .
    container_name: flaskapp
    environment:
      - FLASK_ENV=production
      - DATABASE_URL=mysql+pymysql://bookstore_user:bookstore_pass@<IP_PRIVADA_DB>:3306/bookstore
    ports:
      - "127.0.0.1:5000:5000"
    restart: unless-stopped

```

Levantar la app:

```bash
docker compose -f docker-compose.prod.yml build --no-cache
docker compose -f docker-compose.prod.yml up -d

docker ps
curl -I <http://127.0.0.1:5000>

```

### 5.4. Paso 4 – Configurar dominio en DuckDNS

1. Ir a [https://www.duckdns.org](https://www.duckdns.org/).
2. Crear subdominio: [projectx2.duckdns.org](http://projectx2.duckdns.org/).
3. En IP, colocar la Elastic IP de BookStore-APP:
    - Por ejemplo: 18.233.65.44.
4. Guardar con "update ip".
5. Verificar:

```bash
dig +short projectx2.duckdns.org
# Debe devolver: 18.233.65.44

```

### 5.5. Paso 5 – Configurar NGINX como proxy inverso + SSL

Crear archivo de sitio NGINX:

```bash
sudo tee /etc/nginx/sites-available/bookstore >/dev/null <<'EOF'
# HTTP: redirige todo a HTTPS
server {
    listen 80;
    listen [::]:80;
    server_name projectx2.duckdns.org;

    location /.well-known/acme-challenge/ { root /var/www/html; }
    return 301 https://$host$request_uri;
}

# HTTPS: proxy a Flask
server {
    listen 443 ssl;
    listen [::]:443 ssl;
    server_name projectx2.duckdns.org;

    ssl_certificate     /etc/letsencrypt/live/projectx2.duckdns.org/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/projectx2.duckdns.org/privkey.pem;
    include /etc/letsencrypt/options-ssl-nginx.conf;
    ssl_dhparam /etc/letsencrypt/ssl-dhparams.pem;

    location / {
        proxy_pass         <http://127.0.0.1:5000>;
        proxy_http_version 1.1;
        proxy_set_header   Host $host;
        proxy_set_header   X-Real-IP $remote_addr;
        proxy_set_header   X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header   X-Forwarded-Proto $scheme;
    }
}
EOF

sudo rm -f /etc/nginx/sites-enabled/default
sudo ln -sf /etc/nginx/sites-available/bookstore /etc/nginx/sites-enabled/bookstore
sudo nginx -t
sudo systemctl reload nginx

```

Emitir certificado con Certbot:

```bash
sudo certbot --nginx -d projectx2.duckdns.org --agree-tos -m tu-correo@ejemplo.com --redirect

# Probar renovación
sudo certbot renew --dry-run

```

Validar:

```bash
curl -I <http://projectx2.duckdns.org>
# HTTP/1.1 301 Moved Permanently → Location: <https://projectx2.duckdns.org/>

curl -I <https://projectx2.duckdns.org>
# HTTP/2 200 o HTTP/1.1 200 OK

```

---

## 6. URL de acceso

**Aplicación desplegada (Objetivo 1):**

👉 [https://projectx2.duckdns.org](https://projectx2.duckdns.org/)

---

## 7. Pruebas y validación

- Conectividad App–DB verificada (la app puede crear tablas y consultar datos).
- Contenedores Up:

```bash
docker ps

```

- Acceso correcto en navegador a [https://projectx2.duckdns.org](https://projectx2.duckdns.org/) (candado verde).
- Redirección HTTP → HTTPS verificada con `curl -I <http://projectx2.duckdns.org`>.
- Puerto 3306 accesible solo desde SG-APP (no exponiendo la DB a Internet).

---

## 8. Problemas encontrados y soluciones

### 8.1. Docker instalado via snap no funcionaba con systemd

- **Problema:** Docker instalado desde snap no integraba bien con systemctl.
- **Solución:**
    - Eliminar Docker snap.
    - Instalar Docker Engine oficial desde repositorio de Docker (como se muestra en los pasos).

### 8.2. Permisos de la llave .pem

- **Problema:**`UNPROTECTED PRIVATE KEY FILE!`
- **Causa:** permisos 0664 en la llave.
- **Solución:**

```bash
chmod 600 SD-Project.pem

```

### 8.3. Error SQLALCHEMY_DATABASE_URI no configurado

- **Problema:**`RuntimeError: Either 'SQLALCHEMY_DATABASE_URI' or 'SQLALCHEMY_BINDS' must be set.`
- **Solución:**
Establecer DATABASE_URL como variable de entorno y usarla en `app.config["SQLALCHEMY_DATABASE_URI"]`.

### 8.4. Error de autenticación MySQL con PyMySQL

- **Problema:**`RuntimeError: 'cryptography' package is required for sha256_password or caching_sha2_password auth methods`
- **Causa:** MySQL 8 usa por defecto caching_sha2_password.
- **Soluciones posibles:**
    - Instalar cryptography en la imagen de Flask (recomendado).
    - O cambiar el plugin del usuario MySQL a mysql_native_password mediante:

```bash
docker exec -it bookstore-mysql \\
  mysql -uroot -proot_pass -e \\
"ALTER USER 'bookstore_user'@'%' IDENTIFIED WITH mysql_native_password BY 'bookstore_pass'; FLUSH PRIVILEGES;"

```

---

## 9. Conclusiones

Al finalizar el Objetivo 1, se logró:

- Desplegar la aplicación BookStore monolítica en dos máquinas virtuales en AWS (App + DB).
- Separar la capa de aplicación de la capa de datos, siguiendo buenas prácticas básicas de arquitectura.
- Exponer la aplicación a través de un dominio propio ([projectx2.duckdns.org](http://projectx2.duckdns.org/)).
- Proteger el acceso mediante HTTPS y un certificado válido de Let's Encrypt.
- Utilizar NGINX como proxy inverso, terminando TLS y redirigiendo tráfico hacia Flask.
- Configurar Security Groups para limitar correctamente el acceso a los servicios.

Este despliegue sirve como base sólida para los siguientes objetivos, donde se abordarán escalabilidad (ALB + ASG), mejoras de disponibilidad y otros patrones en la nube.

---

## 10. Referencias

- Docker: https://docs.docker.com/
- NGINX: https://nginx.org/en/docs/
- Let's Encrypt: https://letsencrypt.org/
- Certbot: https://certbot.eff.org/
- AWS EC2: https://docs.aws.amazon.com/ec2/
- Flask: https://flask.palletsprojects.com/
- MySQL: https://dev.mysql.com/doc/
- DuckDNS: https://www.duckdns.org/
