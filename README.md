# Índice
1. [Introducción](#Introducción)  
2. [Información sobre la tarea](#Información-sobre-la-tarea)  
    - [Objetivo](#Objetivo)  
    - [Requisitos previos](#Requisitos-previos)  
    - [Scripts de PowerShell](#Scripts-de-PowerShell)  
    - [Ejecución de los scripts](#Ejecución-de-los-scripts)  
3. [Problemas encontrados](#Problemas-encontrados)

# Introducción
Este repositorio ha sido creado para documentar y mostrar los scripts utilizados en la tarea "AWS: NextCloud y AWS CLI". 

Parte del módulo **Fundamentos de Hardware, correspondiente al curso ASIR_1 (2024 - 2025)**.
El material usado (scripts) ha sido entregado por nuestro profesor (Paco Cuadrado Fernandez)

## Información sobre la tarea
### Objetivo
El objetivo de esta tarea es desplegar una infraestructura en Amazon Web Services (AWS), formada por dos intancias **EC2 (Elastic Compute Cloud)** y una instancia **VPC (Virtual Private Cloud)**. 

Sobre las instancias EC2, se desplegaran los servicions (Apache y MariaDB) necesarios para la instalación y configuración de la plataforma **NextCloud**.

### Requisitos previos
Para la  Ejecución de los scripts de PowerShell, es necesario:
- Permitir la Ejecución de scripts de PowerShell en nuestro sistema operativo.
- Programa AWS CLI.
- Acceso desde AWS CLI a nuestra cuenta.
- Laboratorio.

### Scripts de PowerShell
Los scripts contenidos en este repositorio sirven para creación de **manera automática** de los servicios: (VPC) y (EC2) correspondientes a la plataforma de **Amazon Web Services (AWS)**.
- #### Creación de la VPC (awscli-crea-vpc_ACS.ps1)
Este script creará una red virtual, formada por una subred pública y una subred privada. En el script se debe definir el **CIDR** de cada subred. Para ello, es necesario modificar las siguientes variables:

````
	$bloque_cidr_vpc = "10.10.0.0/16"
	$bloque_subred_publica = "10.10.0.0/25"
	$bloque_subred_privada = "10.10.0.128/25"
````
Opcionalmente, se puede modificar las etiquetas que se asignará a la VPC y las dos subredes. Para ello, es necesario modificar el parámetro **--tag-specifications, el valor "Value" **.
````
$vpcId = (aws ec2 create-vpc `
    --cidr-block $bloque_cidr_vpc  `
    --region $region `
    --tag-specifications 'ResourceType=vpc,Tags=[{Key=Name,Value=ACSNextCloud}]' `
    --query 'Vpc.VpcId' --output text)

$publicSubnetId = (aws ec2 create-subnet `
    --vpc-id $vpcId `
    --cidr-block $bloque_subred_publica `
    --region $region --availability-zone "us-east-1a" `
    --tag-specifications 'ResourceType=subnet,Tags=[{Key=Name,Value=ACSNextCloud-publica}]' `
    --query 'Subnet.SubnetId' --output text)

$privateSubnetId = (aws ec2 create-subnet `
    --vpc-id $vpcId `
    --cidr-block $bloque_subred_privada `
    --region $region --availability-zone "us-east-1a" `
    --tag-specifications 'ResourceType=subnet,Tags=[{Key=Name,Value=ACSNextCloud-privada}]' `
    --query 'Subnet.SubnetId' --output text)
````
Al final la Ejecución de script, saldrá por pantalla un mensaje con los IDs correspondientes a la VPC y sus dos subredes:
````
===== ACS-NextCloud ===========
	VPC ID: vpc-0189b387969b6fe42
	Public Subnet ID: subnet-0d6e25b0675ee077e
	Private Subnet ID: subnet-0183577349b17581d
	Internet Gateway ID: igw-06c850d5460d15e6e
	NAT Gateway ID: nat-06ef5dc39b0c445d4
````
**Adicionalmente, se generará un fichero de texto con esta información.**
- #### Creación de las instancias EC2 (awscli-crea-ec2-v2-MariaDB.ps1)
Este script creará una instancia EC2 de forma automática. Para ello, es necesario tener en cuenta lo siguiente:
##### Parámetros requeridos por consola
	- **Nombre**: el nombre que tendrá la instancia EC2.
	- **VpcId**: la id de la VPC creada anteriormente. La instancia EC2 se creará dentro de esa red.
	- **SubnetId**: al igual que el anterior parámetro, se introducirá el Id de la subred, correspondiente al ID introducido en el anterior parámetro.

##### Grupos de seguridad
Para permitir conexiones entrantes a la instancia EC2, es necesario definir los grupos de seguridad. El bloque de código que se encarga de esto es el siguiente:

###### Creación del grupo de seguridad
Primero, se deberá de crear el grupo de seguridad, asignándole un nombre.
````
$securityGroupId = aws ec2 create-security-group `
    --group-name "EC2-MariaDB-ACS" `
    --description "Grupo de seguridad que abre los puertos 22 y 80" `
    --region $region `
    --vpc-id $vpcId `
    --query 'GroupId' `
    --output text
````
###### Reglas del grupo de seguridad
Una vez creado el grupo de seguridad, utilizaremos la variable ````$securityGroupID````, la cual almacenará el ID del grupo. A continuación, **definiremos las conexiones entrantes que permitiremos** a nuestra instancia EC2.

*Ejemplo: permitir conexiones entrantes en el puerto 22 (ssh)*
````
aws ec2 authorize-security-group-ingress `
    --group-id $securityGroupId `
    --region $region `
    --protocol tcp `
    --port 22 `
    --cidr 0.0.0.0/0 `
    --output text
````

##### User Data
Se pueden ejecutar instrucciones bash de forma automática, antes de conectarse a la instancia EC2. Para ello, he añadido lo siguiente al script:
````
$userData = @"
#!/bin/bash
sudo apt update -y && sudo apt dist-upgrade -y
sudo apt install -y apache2 php php-mbstring php-gd php-intl php-xml php-zip php-curl php-bz2 php-json php-cgi php-cli php-mysql unzip wget -y
sudo systemctl start apache2
sudo systemctl enable apache2
#sudo wget https://download.nextcloud.com/server/releases/latest.tar.bz2
#sudo bzip2 latest.tar.bz2
#sudo tar -xf latest.tar.bz2
#sudo mv nextcloud /var/www/
"@

$userDataBase64 = [Convert]::ToBase64String([Text.Encoding]::UTF8.GetBytes($userData))
````
La variable $userData almacenará las instrucciones bash. Es necesario que estás instrucciones estén en base 64.
Por último, a la hora de crear la instancia EC2, se añadirá el parámetro ````--user-data $userDataBase64````:
````
$instanceId = aws ec2 run-instances `
    --image-id $amiId `
    --instance-type $instanceType `
    --region $region `
    --key-name $keyName `
    --security-group-ids $securityGroupId `
    --subnet-id $subnetId `
    --tag-specifications "ResourceType=instance,Tags=[{Key=Name,Value=$Nombre}]" `
    --associate-public-ip-address `
    --count 1 `
    --user-data $userDataBase64 `
    --query 'Instances[0].InstanceId' `
    --output text
````
Una vez ejecutado cada script, se irá generando y actualizando un fichero de texto, con nombre **salida.txt**.
````
Fecha: 2025-03-22 15:55
VPC ID: vpc-0189b387969b6fe42
Public Subnet ID: subnet-0d6e25b0675ee077e
Private Subnet ID: subnet-0183577349b17581d
Internet Gateway ID: igw-06c850d5460d15e6e
NAT Gateway ID: nat-06ef5dc39b0c445d4
````
### Ejecución de los scripts
Para ejecutar los scripts, es necesario abrir una consola PowerShell. Accederemos a la ruta donde tengamos los scripts y, ejecutaremos lo siguiente:
````
.\awscli-crea-ec2-v2-Apache-Y-NextCloud.ps1
````
## Problemas encontrados
A la hora de intentar automatizar la descarga de los ficheros necesarios para desplegar NextCloud, la velocidad de descarga era demasiado lenta. En el script correspondiente a la instancia EC2 de Apache y NextCloud, las lineas que realizan esta acción están comentadas.
