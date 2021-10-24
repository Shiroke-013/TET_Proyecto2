# TET_Proyecto2

## Producto Final: Aplicación escalable con recursos de aws e implementación a mano.

## Prerrequisitos:
- Cuenta de AWS con créditos.
- Dos o mas instancias de AWS con Wordpress instalado y configurado.

**NOTA:** Puede seguir el siguente tutorial proporcionado por AWS para la instalación de Worpress (https://aws.amazon.com/es/getting-started/hands-on/deploy-wordpress-with-amazon-rds/)

### Introducción
En esta entrega final se presentara el despliegue de una aplicación web de wordpress completamente escalable, se usarán servicio proporcionados por Aamazon Web Services (AWS) como lo son Amazon Relational Databases Service (RDS) y Amazon Elastic File System (EFS). Por otro lado se instalará a en dos instancias EC2 HAProxy como balanceador de cargas para las CMS de wordpress, además se usara keepalived para configurara una IP flotante que mejorará la disponiblidad de los CMS. Para la mejorar en rendimiento se usara CloudFlare como un Content Delivery Network (CDN).

### Arquitectura escalable
Se planteo la siguiente arquiectura escalable para el despliegue de la aplicación web en Worpress:
![ARQ](https://github.com/Shiroke-013/TET_Proyecto2/blob/main/Images/ARQ.jpg)

### Grupo de seguridad
Al crear las instancias EC2 para los CMS de Wordpress se debe tener encuenta que el grupo de seguridad debe tener las siguentes reglas de entrada:
![SG](https://github.com/Shiroke-013/TET_Proyecto2/blob/main/Images/SecurityGroups.jpeg)
**NOTA:** A cada una de las instancias CMS se les debe asociar una IP elastica.

### Configuración del RDS
En este caso se usara una una base de datos de Aurora compatible con MySQL, esto debido a que este tipo de base de datos genera a la vez una copia de la base de datos master que es de escritura y lecutra en otra zona de AWS. Si se llega a caer la base de datos de escritura la copia asiende a ser la base de datos master. La configuración del RDS es la siguiente:

**Paso 1:** Se debe acceder a la sección de RDS en la consola de AWS y seleccionar *"Create database"*

**Paso 2:** Se debe seleccionar *"standar create"* y posterior a esto el tipo de base de datos que se usara que es *"Amazon Aurora"*, así como se ve en la imagen.
![db1](https://github.com/Shiroke-013/TET_Proyecto2/blob/main/Images/CreateDB1.jpeg)

**Paso 3:** Se debe asegurar que la edición de aurora sea compatible con MySQL, en *Capacity type* debe ser *Provisioned* y se debe verificar que en *replicaction features* si este seleccionada la opción de *sigle-master*.
![db2](https://github.com/Shiroke-013/TET_Proyecto2/blob/main/Images/CreateDB2.jpeg)

**Paso 4:** Seleccionar en "Templates" que la base de datos sera usaba para prducción y en la sección de *DB cluster idefier* el nombre que se le dará al cluster este puede ser cualquier nombre.
![db3](https://github.com/Shiroke-013/TET_Proyecto2/blob/main/Images/CreateDB3.jpeg)

**Paso 5:** Se debe asignar un usuario master en este caso se llama "admin" y se le debe asignar una contraseña que se debe recordar para acceder a la base de datos luego.
![db4](https://github.com/Shiroke-013/TET_Proyecto2/blob/main/Images/CreateDB4.jpeg)

**Paso 6:** Para que la base de datos tenga lata disponibilidad se debe seleccionar la opción de *"Create an Aurora Replica or Reader node in a different AZ (recommended for scaled availability)"* esto creara una replica de lectura de la base de datos master en otra zona de AWS así si se llega a caer la zona donde esta la base de datos principal siempre estara la otra de respaldo.
![db5](https://github.com/Shiroke-013/TET_Proyecto2/blob/main/Images/CreateDB5.jpeg)

**Paso 7:** En conectividad se debe dejar las opciones de default en *Virtual Private Cloud (VPC)* y en *subnet group*. Por otro lado se le debe permitir acceso público para que los CMS puedan acceder a esta, por esto se cambia la opción de *public acces* a *Yes*.
![db6](https://github.com/Shiroke-013/TET_Proyecto2/blob/main/Images/CreateDB6.jpeg)

**Paso 8:** En la configuración de el grupo de seguridad se debe asegurar que sea el mismo de las CMS.
![db7](https://github.com/Shiroke-013/TET_Proyecto2/blob/main/Images/CreateDB7.jpeg)

**Paso 9:** Se debe abrir la configuración adicional y en esta agregar la base de datos inicial en este caso lleva el nombre de "wordpress".
![db8](https://github.com/Shiroke-013/TET_Proyecto2/blob/main/Images/CreateDB8.jpeg)

**Paso 10:** Seleccionar el boton de "Create database" y esperar a que esta suba por completo.

### Configuración el RDS en Wordpress

**Paso 1:** Se debe acceder a la instancia EC2 por ssh con el siguiente comando.
```javascript
chmod 600 <path/to/pem/file>
ssh -i <path/to/pem/file> ec2-user@<publicIpAddress>
```
**Paso 2:** Se debe acceder a la base de datos para crear los usuarios y por esto se debe instalar mysql así.
```javascript
sudo yum install -y mysql
```
luego los siguentes comandos para accede:
```javascript
export MYSQL_HOST=<your-endpoint>
mysql --user=<user> --password=<password> wordpress
```
**Paso 3:** Agregar el usuario de la base de datos.
```javascrip
CREATE USER 'wordpress' IDENTIFIED BY 'wordpress-pass';
GRANT ALL PRIVILEGES ON wordpress.* TO wordpress;
FLUSH PRIVILEGES;
Exit
```

### Configuración de EFS en las CMS

**Paso 1:** Se debe acceder a las sección de EFS en la consola de AWS y seleccionar en el menú al lado izquierdo *"File systems"*, después de esto se selecciona *"Create file system"*

**Paso 2:** Se le puede dar un nombre al *"File System"*, esto es opcional, en la configuración de la *"VPC"* se debe de dejar la que está por defecto y en la parte de *"Availability and Durability"* se pone *"Regional"*, finalmente se selecciona *"Create"*.
![EFS1](https://github.com/Shiroke-013/TET_Proyecto2/blob/main/Images/EFS1.png)

**Paso 3:** Se selecciona el *"File system ID"* del "File system" creado o se puede seleccionar y dar click en *"View details"*.

**Paso 4:** En el nuevo menú se selecciona "Attach", esto abrirá un menú, junto con un comando para ejecutar en las instancias, mantenga esto abierto ya que será de utilidad más adelante.
![EFS2](https://github.com/Shiroke-013/TET_Proyecto2/blob/main/Images/EFS2.png)
![EFS2](https://github.com/Shiroke-013/TET_Proyecto2/blob/main/Images/EFS2.png)

**Paso 5:** Ingresar a las instancias a las que se les va a montar el "File system" y ejecutar los siguientes comandos:
```javascript
sudo yum -y update  
sudo reboot  
sudo yum -y install nfs-utils
```

**Paso 6:** En las instancias que se quieren montar el *"File system"* se debe de crear una carpeta en el *"home"*, esta puede tener cualquier nombre, pero preferiblemente se pide que sea: *"efs"* ya que este está por defecto en el comando que aparece en el menú del paso anterior.

**Paso 7:** Ejecute el comando que aparece en el paso 4, puede ser tanto via "DNS" o "IP", luce así:
```javascript
 sudo mount -t nfs -o nfsvers=4.1,rsize=1048576,wsize=1048576,hard,timeo=600,retrans=2,noresvport <mount-target-DNS>:/   efs-mount-point
```

```javascript
sudo mount -t nfs -o nfsvers=4.1,rsize=1048576,wsize=1048576,hard,timeo=600,retrans=2,noresvport <mount-target-ip>:/  ~/efs-mount-point
```

### Configuración de HAProxy
Para la configuración del balanceador de cargas se debe crear primero dos instancias EC2 (una será el *"Master"* y la otra será *"BACKUP"*) con el mismo grupo de seguridad de las CMS, asociarles IPs elasticas a cada una y acceder a ellas por medio de ssh con el siguiente comando:
```javascript
chmod 600 <path/to/pem/file>
ssh -i <path/to/pem/file> ec2-user@<publicIpAddress>
```
****"NOTA: Se debe de tener en cuenta que los siguientes pasos se deben hacer en las dos instancias"****

Ahora se debe instalar HAProxy con el siguiente comando:
```javascript
sudo yum install -y haproxy
```
Despues de instalar se creara un archivo de configuración en la carpeta `/etc/haproxy/`accedemos a este con tu editor favorito así:
```javascript
sudo emacs haproxy.conf
```
En este archivo se deben editar las siguientes lineas para que queden así:
```javascript
frontend http_front
    mode http
    bind *:80
    default_backend app
```
```javascript
backend  app
    mode http
    balance roundrobin 
server cms1 <dirección IP privada de cms1>:80 check
server cms2 <dirección IP privada de cms2>:80 check
```
Se pueden agregar la cantidad de CMS que se desee y por ultimo se inicia el servicio:
```javascript
sudo systemctl enable haproxy
```
```javascript
sudo systemctl start haproxy
```

### Configuración IP flotante
Despues de haber configurado el *"HAProxy"* en cada una de las instancias se instalará *"Keepalived"*:
```javascript
sudo yum install -y keepalived
```

Despues de instalar se creara un archivo de configuración en la carpeta `/etc/keepalived/`. En este punto se tiene que decir cual de las dos instancias será el *"Master"* y cual será el *"Backup"* ya que aunque muy parecida, su configuración es diferente, accedemos al archivo de configuración con el editor de preferencia así:

```javascript
sudo emacs keepalived.conf
```

El archivo del *"Master"* Dee de lucir así:

```javascript
vrrp_script check_haproxy
{
    script "pidof haproxy"
    interval 5
    fall 2
    rise 2
}

vrrp_instance VI_1
{
    debug 2
    interface eth0
    state MASTER
    virtual_router_id 1
    priority 110
    unicast_src_ip <PRIVATE IP FROM MASTER INSTANCE>

    unicast_peer
    {
        <PRIVATE IP FROM BACKUP INSTANCE>
    }

    track_script
    {
        check_haproxy
    }

    notify_master /etc/keepalived/failover.sh
}
```

Y el archivo del *"Backup"* así:

```javascript
vrrp_script check_haproxy
{
    script "pidof haproxy"
    interval 5
    fall 2
    rise 2
}

vrrp_instance VI_1
{
    debug 2
    interface eth0
    state BACKUP
    virtual_router_id 1
    priority 100
    unicast_src_ip <PRIVATE IP FROM BACKUP INSTANCE>

    unicast_peer
    {
        <PRIVATE IP FROM MASTER INSTANCE>
    }

    track_script
    {
        check_haproxy
    }

    notify_master /etc/keepalived/failover.sh
}
```

Despues en la misma carpeta (`/etc/keepalived/`) tanto en el *"Master"* como en el *"Backup"* creamos un archivo llamado *"failover.sh"* y debería de lucir así:
```javascript
#!/bin/bash

EIP=<ELASTIC IP ADDRESS ASSOCIATED TO THE MASTER INSTANCE>
INSTANCE_ID=<INSTANCE ID> # Example: i-0bdd8a68eb573fd1a

/usr/bin/aws ec2 disassociate-address --public-ip $EIP
/usr/bin/aws ec2 associate-address --public-ip $EIP --instance-id $INSTANCE_ID
```

Nos aseguramos de que este archivo sea ejecutable con el siguiente comando:
```javascript
sudo chmod 700 failover.sh
```

Por último configuramos e iniciamos el servicio de *"Keepalived"*:
```javascript
sudo systemctl enable keepalived
```
```javascript
sudo systemctl start keepalived
```

****"NOTA: Para ver el estatus y parar los servicios puede utilizar los siguientes comandos:"****

```javascript
sudo systemctl status "servicio"
```

```javascript
sudo systemctl stop "servicio"
```

### Configuración CDN


### Referencias
- https://aws.amazon.com/es/getting-started/hands-on/deploy-wordpress-with-amazon-rds/
- https://docs.aws.amazon.com/efs/latest/ug/wt1-test.html
- https://www.peternijssen.nl/high-availability-haproxy-keepalived-aws/
- Para animos en el desarrollo: https://open.spotify.com/playlist/5fFFMGxmLZL48jlJd26WlA?si=0275f7890c154b4e
