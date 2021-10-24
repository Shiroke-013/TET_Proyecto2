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


### Obtención de Certificado SSL
#### Creación de un "Instance Group"
- 1. Dentro de **Compute Engine** se dirige a la sección de **Instance Group**.
- 2. Se selecciona **Create Instance Group** y se selecciona: **New unmanaged instance group**.
- 3. Se pone el nombre de preferencia al grupo.
- 4. Ya que se tomó en cuenta la zona en la que se creo la instancia nos aseguramos que pongamos este grupo en la misma zona para poder agregar la instancia de WordPress.
- 5. Se agrega la máquina virtual donde se está corriendo el WordPress.
- 6. Se le da crear y queda lista.

### Obtención de Dominio
Para la realización de este paso se utilizó la plataforma Freenom la cuál nos permite conseguir un dominio de una manera bastante simple, para lograrlo se siguen los siguientes pasos:
 - 1. Se debe de ingresar a la página https://www.freenom.com/ e iniciar sesión (se puede utilizar una Google para ingresar).
 - 2. Después de haberse registrada, en la barra superior se sigue lo siguiente: **Services -> Registrarse**
 - 3. Se busca en la barra que dice: **Encontrar nuevo dominio gratis**, el dominio que se quiere, para este producto fue: **"adchaves"** y aparecen varias opciones como lo serían: "adchavesp.tk", "adchavesp.ml", "adchavesp.ga",...
 - 4. Se le da en **consigalo ahora -> finalizar compra**.
 - 5. Apareceran los dominios que se quisieron conseguir, no se modifica nada y se le da en **continuar**
 - 6. Se aceptan los **términos y condiciones** y se finaliza con el botón **completar pedido**
 - Despues de esto el dominio debería de aparecer en **Services -> Mis Dominios**

### Creación de un LoadBalancer

**NOTA: DURANTE LA CREACIÓN DE ESTEE GENERAREMOS UN CERTIFICADO SSL QUE PUEDE DEMORAR 30MIN EN ACTIVARSE**

- 1. Se navega en el menú izquierdo a **Network Services** -> **Load balancing** -> **Create Load Balancer**
- 2. Queremos que sea un **HTTP(S) Load Balancing** -> **Start Configuration**
- 3. Seleccionamos **From Internet to my VMs**
- 4. Hay varios Componentes que hay que configurar.
- 5. Se le asigna un nombre de preferencia al load balancer.
- 6. Empezamos configurando nuestro **BackEnd** (A donde nuestro tráfico Proxy irá) -> **Create Backend Services**.
- 7. Le asignamos un nombre a nuestro Backend.
- 8. Dejamos que el protocolo sea **HTTP** y seleccionamos en el **Instance group** el grupo que anteriormente creamos, agregamos en los puertos el puerto 80.
- 9. Además de esto tenemos que crear un **Health Check** el cual es una ruta en el backend que devolverá un **status** o un **body** que indica que el "backend" responde al tráfico y le permite al "load balancer" si este esta caido o activo.
- 10. Se le asigna un nombre y en la parte de **Protocol** le ponemos HTTP en el puerto 80 y le damos crear.
- 11. Ahora lo que tenemos que configurar es nuestro **FrontEnd**, le asignamos un nombre.
- 12. En la parte de **Protocol** seleccionamos **HTTPS(includes HTTPS/2)** y necesitamos reservar una IP estática, por lo cual en **ip address** creamos una nueva(solo la tenemos que nombrar).
- 13. El puerte debe apuntar al 443 y ahora en la parte de **Certificate** creamos un nuevo certificado(tenemos que nombrarlo) y en la configuración de este seleccionamos en **create mode** la opción que dice **"Create Google-managed certificate"**, esto hace que google se encarge del certificado, y se necesita especificar los dominios que se tienen en la sección de **Domains** el cuál fue previamente creado y creamos el certificado.
- 14. Después de esto todo debe de estar bien así que terminamos con el "FrontEnd" y solo nos falta darle en **Create** al load balancer.

### Creación de una DNS
- 1. Nos movemos a **Cloud DNS** -> **Create Zone**.
- 2. En la configuración la nombramos y ponemos el dominio que hemos creado, después solo se le da crear.
- 3. Esto crea varios registros y uno de ellos son nuestros **NAMESERVERS** y esto es lo que se va a usar para que nuestro dominio envie el tráfico a GCP.
- 4. Siendo así volveremos a Freenom -> Services -> My Domains, en el dominio con el que estamoso trabajando seleccionaremos **Manage Domain** -> **Management Tools** -> **Nameserver** y le daremos a la opción de **use custom nameserver**, los llenaremos con los 4 registros que creamos anteriormente.
- 5. Después de configurar lo anterior volvemos a la consola de GCP y le damos a **ADD RECORD SET** dentro de nuestra DNS.
- 6. Se puede crear un registro con cosas como: www, www1, www2, etc..., haremos que este sea de tipo **A** y el **TTL** lo pondremos en 1min.
- 7. En la parte de IP Address necesitamos poner la dirección de nuestro "Load Balancer" para ponerla allí(Solo se necesita la dirección IPv4, osea antes de: ":") y le daremos a crear.

### Configuración de WordPress
- 1. Ahora necesitamos configurar unas cuentas cosas de nuestra página de WordPress, vamos a acceder a la parte de administrador (wp-admin) con las credenciales que nos dieron al crear la instancia.
- 2. En la sección de **Plugins** -> **Add New**, buscaremos el siguiente puglin: SSL Insecure Content Fixer, lo instalaremos y lo activaremos.
- 3. Ahora en la sección de **Settings** tendremos una nueva opción **SSL Insecure Content**, dentro de ella iremos hasta abajo hasta la sección **HTTPS detection** seleccionaremos la opción que tenga load balancer en estet caso: HTTP_X_FORWARDED_PROTO(e.g. load balancer, reverse proxy, NginX) y guardaremos los cambios.
- 4. Ahora y ya para terminar iremos a **Settting** -> **General** y actualizaremos tanto el **WordPress Address (URL)** como **Site Address (URL)** en la parte que está la ip, la reemplazaremos por nuestro dominio para que quede algo como: http://adchavesp.tk, seguiremos dejando el http y guardaremos cambios.


# Para la próxima entrega
Se tiene en cuenta que para el siguiente producto se debe de crear la instancia de una manera diferente para poder conseguir lo que se va a pedir. También se tuvo en cuenta que durante la clase que se habló de este readme se mencionó que entregara el readme de esta manera de desarrollar este producto.
