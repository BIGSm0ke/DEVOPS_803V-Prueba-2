
# DEVOPS_803V-Prueba-2
============================================
  GUIA DE DESPLIEGUE - PROYECTO SEMESTRAL
============================================

ARQUITECTURA DEL PROYECTO
-------------------------
Frontend: React + Vite (Nginx) -> EC2 Frontend (10.0.1.203)
Backend:  Spring Boot + MySQL   -> EC2 Backend  (10.0.2.83)

Los dos EC2 estan en subredes distintas dentro de la misma VPC.
El frontend se comunica con los backends mediante IP privada (10.0.2.83).


============================================
1. REQUISITOS PREVIOS (en cada EC2)
============================================

Amazon Linux:
  sudo dnf install docker -y
  sudo systemctl enable docker
  sudo systemctl start docker
  sudo usermod -aG docker ec2-user

El grupo docker se activa cerrando sesion (exit) y conectandose de nuevo,
o ejecutando: newgrp docker


============================================
2. BACKEND EC2 (10.0.2.83)
============================================

2.1 Pull de imagenes
----------------------
  docker pull mysql:8.0
  docker pull bigsm0k3/back-ventas:latest
  docker pull bigsm0k3/back-despachos:latest

2.2 Crear volumen para persistencia de datos
-----------------------------------------------
  docker volume create mysql_data

2.3 Levantar MySQL
--------------------
  docker run -d --name mysql-db -p 3306:3306 \
    -e MYSQL_ROOT_PASSWORD=root123 \
    -e MYSQL_DATABASE=ventas_db \
    -v mysql_data:/var/lib/mysql \
    mysql:8.0 \
    --default-authentication-plugin=mysql_native_password

  Esperar ~15 segundos a que MySQL inicie.


2.4 Levantar back-ventas
---------------------------
  docker run -d --name back-ventas -p 8080:8080 \
    -e DB_ENDPOINT=10.0.2.83 \
    -e DB_PORT=3306 \
    -e DB_NAME=ventas_db \
    -e DB_USERNAME=root \
    -e DB_PASSWORD=root123 \
    bigsm0k3/back-ventas:latest

2.5 Levantar back-despachos
------------------------------
  docker run -d --name back-despachos -p 8081:8081 \
    -e DB_ENDPOINT=10.0.2.83 \
    -e DB_PORT=3306 \
    -e DB_NAME=ventas_db \
    -e DB_USERNAME=root \
    -e DB_PASSWORD=root123 \
    bigsm0k3/back-despachos:latest

2.6 Verificar estado
----------------------
  docker ps

============================================
3. FRONTEND EC2 (10.0.1.203)
============================================

3.1 Pull de imagen
--------------------
  docker pull bigsm0k3/front-despacho:latest

3.2 Levantar frontend
-----------------------
  docker run -d --name front-despacho -p 80:80 \
    bigsm0k3/front-despacho:latest

  El nginx.conf dentro de la imagen ya tiene la IP 10.0.2.83
  configurada para redirigir las llamadas /api/v1/ventas y
  /api/v1/despachos hacia el backend.

3.3 Verificar
---------------
  docker ps
  curl -I localhost:80


============================================
4. ACCEDER A LA APLICACION
============================================

  Desde el navegador:
    http://<IP-PUBLICA-FRONTEND>

  La pagina principal carga automaticamente los datos de
  ventas y despachos desde el backend.


============================================
5. EXPLICACION DEL WORKFLOW
============================================

El proyecto cuenta con 3 pipelines automatizados en GitHub Actions
(archivos en .github/workflows/):

  1. backend-ventas.yml
  2. backend-despachos.yml
  3. frontend.yml

Cada pipeline se activa con push a la rama "deploy" y solo cuando
hay cambios en su respectiva carpeta.

PASOS DE CADA PIPELINE:
  1. Checkout del codigo
  2. Setup del entorno (JDK 17 para backends, Node 20 para frontend)
  3. Ejecucion de tests (mvn verify / npm ci + lint + build)
  4. Login a Docker Hub usando secrets del repositorio
  5. Build y push de la imagen Docker a Docker Hub
  6. Conexion SSH al EC2 correspondiente
     - Git pull del repositorio
     - Docker login
     - docker compose pull
     - docker compose down && docker compose up -d

SECRETS REQUERIDOS EN GITHUB 
  DOCKER_USERNAME           
  DOCKER_PASSWORD           
  EC2_HOST                  
  EC2_USER                  
  EC2_SSH_KEY               
  MYSQL_VENTAS_ROOT_PASSWORD
  MYSQL_VENTAS_DATABASE
  MYSQL_VENTAS_USER
  MYSQL_VENTAS_PASSWORD
  MYSQL_DESPACHOS_ROOT_PASSWORD
  MYSQL_DESPACHOS_DATABASE
  MYSQL_DESPACHOS_USER
  MYSQL_DESPACHOS_PASSWORD

============================================
