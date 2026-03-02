# DSpace

Proyecto de configuracion y despliegue de DSpace 9 desarrollado por **Isaac Esteban Haro Torres**.

---

## Descripcion

Guia paso a paso para instalar y levantar **DSpace 9.1** en **Ubuntu 24.04 LTS** (backend + frontend).

Versiones validadas al 2 de marzo de 2026.

---

## Descargas Oficiales

- DSpace backend (releases): https://github.com/DSpace/DSpace/releases
- DSpace backend (ultima release): https://github.com/DSpace/DSpace/releases/latest
- DSpace Angular frontend (releases): https://github.com/DSpace/dspace-angular/releases
- DSpace Angular frontend (ultima release): https://github.com/DSpace/dspace-angular/releases/latest
- Documentacion oficial de instalacion DSpace 9.x: https://wiki.lyrasis.org/display/DSDOC9x/Installing+DSpace
- Solr (descargas oficiales): https://solr.apache.org/downloads.html

---

## Prerrequisitos

- Java JDK 17
- Apache Maven
- Apache Ant
- Apache Tomcat 10
- PostgreSQL 16
- Solr 8.11.4
- DSpace 9.1 (backend)
- DSpace Angular 9.1 (frontend)
- Node.js + npm
- NVM
- Yarn
- gedit (editor)

Nota: Se recomienda usar un servidor limpio para evitar conflictos de versiones.

---

## 1) Preparar sistema y usuario

```bash
sudo apt update && sudo apt upgrade -y
sudo apt install wget curl git build-essential gedit zip unzip -y

sudo useradd -m dspace
sudo passwd dspace
sudo usermod -aG sudo dspace

sudo mkdir -p /dspace
sudo chown dspace:dspace /dspace
```

---

## 2) Instalar Java, Maven y Ant

```bash
sudo apt install openjdk-17-jdk maven ant -y
```

Configurar variables:

```bash
sudo gedit /etc/environment
```

Agregar al final:

```bash
JAVA_HOME="/usr/lib/jvm/java-17-openjdk-amd64"
JAVA_OPTS="-Xmx512M -Xms64M -Dfile.encoding=UTF-8"
```

Aplicar y validar:

```bash
source /etc/environment
echo $JAVA_HOME
echo $JAVA_OPTS
java -version
mvn -version
ant -version
```

---

## 3) Instalar y configurar PostgreSQL 16

```bash
sudo apt install postgresql postgresql-client postgresql-contrib libpostgresql-jdbc-java -y
psql -V
sudo pg_ctlcluster 16 main start
sudo systemctl status postgresql
```

Configurar acceso:

```bash
sudo gedit /etc/postgresql/16/main/postgresql.conf
```

Asegurar:

```ini
listen_addresses = 'localhost'
```

Editar autenticacion:

```bash
sudo gedit /etc/postgresql/16/main/pg_hba.conf
```

Agregar esta linea arriba de `Database administrative login by Unix domain socket`:

```ini
# DSpace configuration
host    dspace    dspace    127.0.0.1/32    md5
```

Reiniciar:

```bash
sudo systemctl restart postgresql
```

Crear usuario y base de datos:

```bash
sudo -u postgres createuser --no-superuser --pwprompt dspace
sudo -u postgres createdb --owner=dspace --encoding=UNICODE dspace
sudo -u postgres psql dspace -c "CREATE EXTENSION IF NOT EXISTS pgcrypto;"
```

---

## 4) Instalar Solr 8.11.4

```bash
cd /opt
sudo wget https://downloads.apache.org/lucene/solr/8.11.4/solr-8.11.4.zip
sudo unzip solr-8.11.4.zip
sudo chown -R dspace:dspace /opt/solr-8.11.4
```

Levantar Solr con usuario dspace:

```bash
su - dspace
cd /opt/solr-8.11.4/bin
./solr start
./solr status
exit
```

Auto-inicio al reiniciar:

```bash
crontab -u dspace -e
```

Agregar:

```bash
@reboot /opt/solr-8.11.4/bin/solr start
```

Verificar en navegador:

`http://localhost:8983/solr`

---

## 5) Descargar DSpace 9.1 (backend)

```bash
sudo mkdir -p /build
cd /build
sudo wget https://github.com/DSpace/DSpace/archive/refs/tags/dspace-9.1.tar.gz
sudo tar zxvf dspace-9.1.tar.gz
sudo chmod -R 777 /build
```

---

## 6) Instalar y ajustar Tomcat 10

```bash
sudo apt install tomcat10 -y
```

Permitir escritura a /dspace:

```bash
sudo gedit /lib/systemd/system/tomcat10.service
```

Agregar bajo `[Service]`:

```ini
ReadWritePaths=/dspace/
```

Ajustar UTF-8 en conector HTTP:

```bash
sudo gedit /etc/tomcat10/server.xml
```

Comentar conector default y agregar:

```xml
<Connector port="8080" protocol="HTTP/1.1"
           minSpareThreads="25"
           enableLookups="false"
           redirectPort="8443"
           connectionTimeout="20000"
           disableUploadTimeout="true"
           URIEncoding="UTF-8" />
```

Recargar y reiniciar:

```bash
sudo systemctl daemon-reload
sudo systemctl restart tomcat10
```

---

## 7) Configurar backend de DSpace

```bash
cd /build/DSpace-dspace-9.1/dspace/config
sudo cp local.cfg.EXAMPLE local.cfg
sudo gedit local.cfg
```

Ajustar al menos:

```ini
dspace.server.url = http://localhost:8080/server
dspace.ui.url = http://localhost:4000
dspace.name = DSpace at My University

db.username = dspace
db.password = dspace

solr.server = http://localhost:8983/solr
```

Compilar e instalar:

```bash
cd /build/DSpace-dspace-9.1
mvn package

cd dspace/target/dspace-installer
ant fresh_install
```

Desplegar webapps a Tomcat:

```bash
sudo cp -R /dspace/webapps/* /var/lib/tomcat10/webapps/ROOT
sudo cp -R /dspace/webapps/* /var/lib/tomcat10/webapps/
```

Copiar configsets a Solr:

```bash
sudo cp -R /dspace/solr/* /opt/solr-8.11.4/server/solr/configsets
sudo chown -R dspace:dspace /opt/solr-8.11.4/server/solr/configsets
```

Reiniciar Solr:

```bash
su - dspace
cd /opt/solr-8.11.4/bin
./solr restart
exit
```

Migrar BD y crear admin:

```bash
cd /dspace/bin
sudo ./dspace database migrate
sudo ./dspace create-administrator
```

Ajustar permisos y reiniciar Tomcat:

```bash
sudo chown -R tomcat:tomcat /dspace/
sudo systemctl restart tomcat10
```

Pruebas backend:

- `http://localhost:8080/server`
- `http://localhost:8080/server/oai/request?verb=Identify`

---

## 8) Instalar frontend DSpace Angular 9.1

Instalar Node y npm base:

```bash
sudo apt install nodejs npm -y
```

Instalar NVM:

```bash
sudo su
curl -o- https://raw.githubusercontent.com/creationix/nvm/master/install.sh | bash
export NVM_DIR="$HOME/.nvm"
[ -s "$NVM_DIR/nvm.sh" ] && . "$NVM_DIR/nvm.sh"
[ -s "$NVM_DIR/bash_completion" ] && . "$NVM_DIR/bash_completion"
```

Instalar Node (ejemplo LTS):

```bash
nvm install 20
npm install --global yarn pm2
exit
```

Descargar frontend:

```bash
cd /home/dspace
sudo wget https://github.com/DSpace/dspace-angular/archive/refs/tags/dspace-9.1.tar.gz
tar zxvf dspace-9.1.tar.gz
rm dspace-9.1.tar.gz
cd /home/dspace/dspace-angular-dspace-9.1
yarn install
```

Configurar ambiente:

```bash
cd config
cp config.example.yml config.dev.yml
cp config.example.yml config.prod.yml
gedit config.prod.yml
```

Ajustar bloque REST:

```yaml
rest:
  ssl: false
  host: localhost
  port: 8080
```

### Modo desarrollo

```bash
cd /home/dspace/dspace-angular-dspace-9.1
yarn run start:dev
```

Si falla por memoria/TLS:

```bash
export NODE_TLS_REJECT_UNAUTHORIZED='0'
export NODE_OPTIONS="--max-old-space-size=8192"
yarn run start:dev
```

### Modo produccion + PM2

```bash
cd /home/dspace/dspace-angular-dspace-9.1
yarn run build:prod
```

Crear archivo PM2:

```bash
sudo gedit /home/dspace/dspace-angular-dspace-9.1/dspace-ui.json
```

Contenido:

```json
{
  "apps": [
    {
      "name": "dspace-ui",
      "cwd": "/home/dspace/dspace-angular-dspace-9.1/",
      "script": "dist/server/main.js",
      "instances": "max",
      "exec_mode": "cluster",
      "env": {
        "NODE_ENV": "production"
      }
    }
  ]
}
```

Iniciar y detener:

```bash
pm2 start /home/dspace/dspace-angular-dspace-9.1/dspace-ui.json
pm2 stop /home/dspace/dspace-angular-dspace-9.1/dspace-ui.json
```

Prueba frontend:

- `http://localhost:4000`

---

## Comandos rapidos de operacion

```bash
# Backend/Tomcat
sudo systemctl restart tomcat10
sudo systemctl status tomcat10

# PostgreSQL
sudo systemctl restart postgresql
sudo systemctl status postgresql

# Solr
su - dspace -c "/opt/solr-8.11.4/bin/solr status"

# Frontend PM2
pm2 list
pm2 restart dspace-ui
```

---

## Fuente base usada

- http://dspacegeek.blogspot.com/2022/05/install-dspace7-ubuntu-debian.html
- https://wiki.lyrasis.org/display/DSDOC7x/Installing+DSpace

Adaptado y actualizado para Ubuntu 24.04 LTS y DSpace 9.1.

---

## Desarrollado por Isaac Esteban Haro Torres

**Ingeniero en Sistemas | Full Stack | Automatizacion | Data**

- Email: zackharo1@gmail.com
- WhatsApp: 098805517
- GitHub: https://github.com/ieharo1
- Portafolio: https://ieharo1.github.io/portafolio-isaac.haro/

---

Copyright 2026 Isaac Esteban Haro Torres - Todos los derechos reservados.





