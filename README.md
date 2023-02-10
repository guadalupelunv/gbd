## Gestión de Bases de Datos
Repositorio para prácticas de Base de Datos

## Práctica 4.3: Despliegue de una arquitectura EFS-EC2-MultiAZ


### Creación de Grupos de Seguridad

Para esta práctica primero en el servicio EC2, iremos a Grupos de Seguridad y creamos 2 grupos de seguridad, en uno lo llamaremos SGweb y abriremos el purto 80, HTTP desde cualquier IPv4 y el puerto 22 de SSH por si hay que modificarlo, el otro se llamará SGEfs con el puerto 2049 de NFS para cualquier IPv4. Quedando así:

![SGWeb](img/GS1.png)

![SGEfs](img/GS2.png)

### Creación de Instancias EC2

Seguimos en el servicio EC2 y ahora creamos una EC2 que se llamará Linux_01, con Amazon Linux, par de claves vockey, la VPC predeterminada pero elegimos la subred a, y permitimos que asigne una ip pública, se le asigna el grupo de seguridad que antes hemos creado con el nombre Sgweb. 

![Linux_01](img/linux1.png)

En configuración avanzada introducimos los datos de usuario que se muestran a continuación:

    #!/bin/bash
    yum update -y
    yum install httpd -y
    systemctl start httpd
    systemctl enable httpd
    yum -y install nfs-utils

![Datos de usuario](img/linux11.png)

Y lanzamos la instancia, mientras crearemos otra instancia llamada Linux_02, con la misma configuración Amazon Linux,par de claves vockey, VPC predeterminada pero con subred b y el mismo grupo de seguridad llamado SGweb y volvemos a configuración avanzada para pegar los datos de usuario, por último lanzamos la instancia.

![Linux_02](img/linux2.png)

Además creamos un par de ips elásticas para que al cerrar el laboratorio, esta no cambie, las creamos y las asociamos una por máquina.

### Creación del Sistema de Archivos

En el servicio EFS, vamos a crear un sistema de ficheros, llamado minfs y tendrá el VPC por defecto y elegimos la opción Estándar para que esté disponible en todas las zonas de disponibilidad.

![EFS](img/efs.png)

Entramos en nuestro sistema de ficheros llamado miefs y accederemos a los grupos de seguridad y en todas las subredes le asignaremos el grupo de seguridad SGEfs. 

![EFS](img/seguridad.png)

### Configuración de los servidores web

Cuando se han terminado de crear, conectaremos a ellas donde podemos verificar que se haya instalado correctamente Apache y después de verificar entraremos en /var/www/html y crearemos una carpeta llamada efs-mount con el comando “mkdir efs-mount”.

![Comprobación Apache](img/httpd1.png)

Ahora vamos a utilizar el comando para montar en un sistema nfs sobre la carpeta que hemos creado, y ejecutamos el comando cambiando el id por el nuestro.
“sudo mount -t nfs -o nfsvers=4.1,rsize=1048576,wsize=1048576,hard,timeo=600,retrans=2,noresvport fs-0b3af357a01238bb9.efs.us-east-1.amazonaws.com:/ efs-mount”.

Ahora con otro comando nos descargaremos la página web:
“wget https://s3.eu-west-1.amazonaws.com/www.profesantos.cloud/Netflix.zip”, lo descomprimimos con el comando “unzip Netflix.zip” y ya estaría la página web.

![Archivos Netflix](img/netflix.png)

Estos pasos también la hacemos en la segunda máquina EC2 para así que se vea la misma página web en los dos servidores. Si buscamos nuestra ip en internet y en la ruta accedemos al html de la página web se visualizará.

![Web1](img/web1.png)

Ahora vamos a modificar el archivo de Apache para simplemente acceder a la página con nuestra ip.

Así que modificaremos el archivo /etc/httpd/conf/httpd.conf con “sudo nano” o “vim” y modificaremos el DocumentRoot a DocumentRoot "/var/www/html/efs-mount", y después reiniciaremos el servicio httpd con “systemctl restart httpd”, esto se hará en los dos servidores.

![Configuración web](img/conf.png)

Con esto ya estaría los servidores web configurados y si ponemos simplemente la ip de los servidores nos mostrará la página web.

![Web](img/webfin.png)


### Creación del balanceador

Creamos otra EC2 más para hacer nuestro balanceador, se llamará Balanceador_Linux, con sistema Ubuntu, con par de claves vockey y le creamos un nuevo grupo de seguridad, donde abriremos los puertos SSH,HTTP y HTTPS.
También le asignaremos una ip elástica como a los nodos y se la asociaremos.

![Balanceador](img/balanceador.png)

Cuando se cree el balanceador, nos conectaremos a él e instalaremos Apache con "sudo apt install apache2", también debemos reiniciar el servicio con el comando "sudo systemctl restart apache2".

![Instalación Apache](img/apache.png)

A continuación, editamos el fichero /etc/apache2/sites-enabled/000-default.conf para configurar nuestro gestor de balanceo y pondremos lo siguiente:

    ProxyPass /balancer-manager !

    <Proxy balancer://Balanceador_Linux>
        # Server 1
        BalancerMember http://172.31.94.123

        # Server 2
        BalancerMember http://172.31.26.236
    </Proxy>
        ProxyPass / balancer:/Balanceador_Linux/
        ProxyPassReverse / balancer:/Balanceador_Linux/

    <Location /balancer-manager>
        SetHandler balancer-manager
        Order Deny,Allow
        Allow from all
    </Location>

![Configuración Balanceador Nodos](img/archivobalan.png)

Tras esto reiniciaremos Apache de nuevo con "sudo systemctl restart apache2" y al buscar la ip de nuestro balanceador saldrá la página web, si ponemos a continuacion de la ip "/balancer-manager" se nos mostrará el balanceador.

![Balanceador](img/balancer-manager.png)

Con esto el balanceador funciona y si en algún momento se cae un nodo, el otro seguiría mostrando la página web.