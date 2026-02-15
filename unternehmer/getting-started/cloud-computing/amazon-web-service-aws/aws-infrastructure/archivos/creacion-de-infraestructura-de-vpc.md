---
layout:
  width: default
  title:
    visible: false
  description:
    visible: false
  tableOfContents:
    visible: true
  outline:
    visible: true
  pagination:
    visible: true
  metadata:
    visible: true
  tags:
    visible: true
---

# Creación de infraestructura de VPC

<h2 align="center">Creación de infraestructura de VPC</h2>



{% embed url="https://soundcloud.com/999999999music/999999999-x-flkn-acid-blood?si=fe0c213a17a14f598c472c69529e4891&utm_campaign=social_sharing&utm_medium=text&utm_source=clipboard" %}

<figure><img src="../../../../../.gitbook/assets/Creación de la infraestructura de VPC.png" alt=""><figcaption></figcaption></figure>

Este laboratorio es un ejercicio fundamental de networking en AWS. Su propósito principal es guiarte en la **construcción manual de una arquitectura de red segura y común de dos niveles (two-tier architecture)**.

1. **Una capa pública**: Contiene recursos que deben ser accesibles desde Internet, como un servidor web frontal.
2. **Una capa privada**: Contiene recursos de backend (como servidores de aplicaciones o bases de datos) que deben estar completamente aislados de Internet por seguridad, pero que aún necesitan poder iniciar conexiones hacia el exterior (por ejemplo, para descargar actualizaciones de software).

Para lograr esto te llevare a través de la configuración de los componentes esenciales de una VPC:

* **VPC**: La red virtual privada que aislará todos tus recursos.
* **Subredes**: Dividirás la VPC en una `Public Subnet` y una `Private Subnet`.
* **Gateways**:
  * **Internet Gateway (IGW)**: Para dar a la subred pública una ruta hacia y desde Internet.
  * **NAT Gateway (NGW)**: Para permitir que los recursos en la subred privada accedan a Internet sin exponerlos a conexiones entrantes.
* **Tablas de enrutamiento**: Para controlar el flujo de tráfico, dirigiendo el tráfico de la subred pública al IGW y el de la privada a la NGW.
* **Grupos de seguridad**: Para actuar como firewalls a nivel de instancia, definiendo reglas de tráfico específicas (por ejemplo, permitir tráfico HTTP desde Internet al servidor público, y luego desde el servidor público al servidor privado).

Al finalizar, no solo habrás lanzado instancias EC2, sino que habrás construido y comprendido la arquitectura de red subyacente que las protege y conecta de manera segura, un patrón fundamental para casi cualquier despliegue en la nube.

### Creación  de arquitectura con terraform (Opcional)

```hcl
# 1. Configuración del Proveedor de AWS
# Especifica que vamos a trabajar con AWS y en qué región.
# Cambia "us-east-1" a la región que estés utilizando.
provider "aws" {
  region = "us-east-1"
}

# 2. Creación de la VPC (Corresponde a la Tarea 1)
# Define la red virtual aislada principal.
resource "aws_vpc" "lab_vpc" {
  cidr_block           = "10.0.0.0/16"
  enable_dns_support   = true
  enable_dns_hostnames = true
  tags = {
    Name = "Lab VPC (Terraform)"
  }
}

# 3. Creación de las Subredes (Corresponde a la Tarea 2)
# Subred pública, con IPs públicas asignadas automáticamente a las instancias.
resource "aws_subnet" "public_subnet" {
  vpc_id                  = aws_vpc.lab_vpc.id
  cidr_block              = "10.0.0.0/24"
  map_public_ip_on_launch = true
  availability_zone       = "${data.aws_region.current.name}a" # Usa la primera AZ de la región
  tags = {
    Name = "Public Subnet (Terraform)"
  }
}

# Subred privada, sin IPs públicas.
resource "aws_subnet" "private_subnet" {
  vpc_id                  = aws_vpc.lab_vpc.id
  cidr_block              = "10.0.2.0/23"
  availability_zone       = "${data.aws_region.current.name}a" # Usa la misma AZ para consistencia
  tags = {
    Name = "Private Subnet (Terraform)"
  }
}

# 4. Creación de Gateways (Corresponde a las Tareas 3 y 9)
# Internet Gateway para dar acceso a internet a la subred pública.
resource "aws_internet_gateway" "lab_igw" {
  vpc_id = aws_vpc.lab_vpc.id
  tags = {
    Name = "Lab IGW (Terraform)"
  }
}

# IP Elástica (EIP) estática para la NAT Gateway.
resource "aws_eip" "nat_eip" {
  domain = "vpc"
  depends_on = [aws_internet_gateway.lab_igw]
}

# NAT Gateway para dar acceso a internet a la subred privada.
# Se despliega en la subred pública.
resource "aws_nat_gateway" "lab_ngw" {
  allocation_id = aws_eip.nat_eip.id
  subnet_id     = aws_subnet.public_subnet.id
  tags = {
    Name = "Lab NGW (Terraform)"
  }
}

# 5. Configuración de Tablas de Enrutamiento (Corresponde a las Tareas 4 y 9)
# Tabla de rutas para la subred pública.
resource "aws_route_table" "public_rt" {
  vpc_id = aws_vpc.lab_vpc.id
  route {
    cidr_block = "0.0.0.0/0"
    gateway_id = aws_internet_gateway.lab_igw.id
  }
  tags = {
    Name = "Public Route Table (Terraform)"
  }
}

# Asociación de la tabla de rutas pública con la subred pública.
resource "aws_route_table_association" "public_assoc" {
  subnet_id      = aws_subnet.public_subnet.id
  route_table_id = aws_route_table.public_rt.id
}

# Tabla de rutas para la subred privada.
resource "aws_route_table" "private_rt" {
  vpc_id = aws_vpc.lab_vpc.id
  route {
    cidr_block     = "0.0.0.0/0"
    nat_gateway_id = aws_nat_gateway.lab_ngw.id
  }
  tags = {
    Name = "Private Route Table (Terraform)"
  }
}

# Asociación de la tabla de rutas privada con la subred privada.
resource "aws_route_table_association" "private_assoc" {
  subnet_id      = aws_subnet.private_subnet.id
  route_table_id = aws_route_table.private_rt.id
}

# 6. Creación de Grupos de Seguridad (Corresponde a las Tareas 5 y 10)
# Grupo de seguridad para la instancia pública.
resource "aws_security_group" "public_sg" {
  name        = "public_sg_tf"
  description = "Allow HTTP inbound traffic"
  vpc_id      = aws_vpc.lab_vpc.id

  ingress {
    from_port   = 80
    to_port     = 80
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }

  egress {
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }

  tags = {
    Name = "Public SG (Terraform)"
  }
}

# Grupo de seguridad para la instancia privada.
resource "aws_security_group" "private_sg" {
  name        = "private_sg_tf"
  description = "Allow HTTP from public security group"
  vpc_id      = aws_vpc.lab_vpc.id

  ingress {
    from_port       = 80
    to_port         = 80
    protocol        = "tcp"
    security_groups = [aws_security_group.public_sg.id] # ¡Clave! Permite tráfico desde el SG público.
  }

  egress {
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }

  tags = {
    Name = "Private SG (Terraform)"
  }
}

# 7. Búsqueda de AMI y Creación de Instancias EC2 (Corresponde a Tareas 6 y 11)
# Obtiene la región actual de forma dinámica.
data "aws_region" "current" {}

# Busca dinámicamente la última AMI de Amazon Linux 2023.
# Esto hace la plantilla reutilizable y no depende de un ID de AMI fijo.
data "aws_ami" "amazon_linux_2023" {
  most_recent = true
  owners      = ["amazon"]
  filter {
    name   = "name"
    values = ["al2023-ami-2023.*-kernel-6.1-x86_64"]
  }
  filter {
    name   = "architecture"
    values = ["x86_64"]
  }
}

# Script de User Data que se ejecutará en ambas instancias.
locals {
  user_data = <<-EOF
              #!/bin/bash
              yum update -y
              yum install -y httpd php
              systemctl enable httpd.service
              systemctl start httpd
              cd /var/www/html
              wget https://us-west-2-tcprod.s3.amazonaws.com/courses/ILT-TF-200-ARCHIT/v7.9.11.prod-60bc4f16/lab-2-VPC/scripts/instanceData.zip
              unzip instanceData.zip
              EOF
}

# Instancia en la subred pública.
resource "aws_instance" "public_instance" {
  ami                    = data.aws_ami.amazon_linux_2023.id
  instance_type          = "t3.micro"
  subnet_id              = aws_subnet.public_subnet.id
  vpc_security_group_ids = [aws_security_group.public_sg.id]
  user_data              = local.user_data
  # Nota: El rol de IAM para Session Manager se debe crear y adjuntar aquí.
  # iam_instance_profile = "EC2InstProfile" 

  tags = {
    Name = "Public Instance (Terraform)"
  }
}

# Instancia en la subred privada.
resource "aws_instance" "private_instance" {
  ami                    = data.aws_ami.amazon_linux_2023.id
  instance_type          = "t3.micro"
  subnet_id              = aws_subnet.private_subnet.id
  vpc_security_group_ids = [aws_security_group.private_sg.id]
  user_data              = local.user_data
  # Nota: El rol de IAM para Session Manager se debe crear y adjuntar aquí.
  # iam_instance_profile = "EC2InstProfile" 

  tags = {
    Name = "Private Instance (Terraform)"
  }
}

# 8. Salidas (Outputs)
# Muestra la IP pública de la instancia pública al finalizar.
output "public_instance_ip" {
  description = "Public IP address of the public EC2 instance"
  value       = aws_instance.public_instance.public_ip
}

output "public_instance_dns" {
  description = "Public DNS of the public EC2 instance"
  value       = aws_instance.public_instance.public_dns
}

```

<figure><img src="../../../../../.gitbook/assets/Barra.gif" alt=""><figcaption></figcaption></figure>

<details>

<summary><strong>Crear una Virtual Private Cloud en una región</strong></summary>

Crearás una nueva Virtual Private Cloud (VPC) en la nube de AWS.

* En la barra de búsqueda de la consola de administración de AWS, busca y selecciona **VPC**.
* En el panel de navegación izquierdo, elige **Your VPCs** (Sus VPCs). Verás una VPC predeterminada que AWS crea en cada región para facilitar el inicio rápido de recursos.
* Elige **Create VPC** (Crear VPC).
* Configura los siguientes ajustes:
  * **Resources to create** (Recursos a crear): `VPC only` (Solo VPC).
  * **Name Tag** (Etiqueta de nombre): `FirstProof`
  * **IPv4 CIDR**: `10.0.0.0/16`
* Elige **Create VPC** (Crear VPC).

<figure><img src="../../../../../.gitbook/assets/Creación de Virtual Private Machine.gif" alt="Creación de Virtual Private Cloud en AWS"><figcaption><p>Creación de Virtual Private Cloud en AWS</p></figcaption></figure>

{% hint style="warning" %}
**Opcional: creación a través de CLI**
{% endhint %}

{% code overflow="wrap" %}
```bash
aws ec2 create-vpc --instance-tenancy "default" --cidr-block "10.0.0.0/16" --tag-specifications '{"resourceType":"vpc","tags":[{"key":"Name","value":"FirstProof"}]}' 
```
{% endcode %}

* Verifica que el estado de la `FirstProof` sea **Available** (Disponible).
* En la misma página, con la `FirstProof` seleccionada, elige **Actions** (Acciones) y luego **Edit VPC settings** (Editar la configuración de VPC).
* En la sección **DNS settings** (Configuración de DNS), selecciona la casilla **Enable DNS hostnames** (Habilitar nombres de host del DNS).
* Selecciona **Save** (Guardar).

<figure><img src="../../../../../.gitbook/assets/Activar sistema de nombres de dominio (DNS) en VPC.gif" alt="Habilitar DNS en Virtual Private Cloud"><figcaption><p>Habilitar DNS en Virtual Private Cloud</p></figcaption></figure>

***

#### **¿Qué es una Amazon VPC (Virtual Private Cloud)?**

Una VPC es fundamentalmente tu propio centro de datos virtual y privado dentro de la nube de AWS. Es una sección de la nube de AWS lógicamente aislada donde puedes lanzar recursos en una red virtual que tú defines.

* **Aislamiento y seguridad**: La principal ventaja de una VPC es el aislamiento. Por defecto, nada puede entrar ni salir de una VPC que tú creas. Actúa como un contenedor seguro para tus recursos (como instancias EC2, bases de datos, etc.). Tienes control total sobre quién y qué puede acceder a estos recursos.
* **Control de red**: Te otorga control absoluto sobre tu entorno de red. Puedes:
  * **Seleccionar tu propio rango de direcciones IP**: Como se hizo en el laboratorio con `10.0.0.0/16`.
  * **Crear subredes**: Dividir tu rango de IP en segmentos más pequeños para organizar los recursos (por ejemplo, una subred para servidores web públicos y otra para bases de datos privadas).
  * **Configurar tablas de enrutamiento**: Definir cómo se dirige el tráfico dentro de tu VPC y hacia el exterior.
  * **Configurar puertas de enlace**: Conectar tu VPC a Internet (Internet Gateway) o a tu red corporativa (Virtual Private Gateway).

#### **¿Qué es un bloque CIDR y por qué se usó `10.0.0.0/16`?**

CIDR (Classless Inter-Domain Routing) es la notación estándar para especificar un rango de direcciones IP.

* **`10.0.0.0`**: Esta es la dirección IP inicial del rango. Pertenece a un bloque de direcciones reservado para redes privadas, lo que significa que no es enrutable en la Internet pública y es ideal para uso interno.
* **`/16`**: Esta es la máscara de red. Indica cuántos bits iniciales de la dirección IP son fijos e identifican la red. Una dirección IPv4 tiene 32 bits.
  * Al fijar los primeros 16 bits (`10.0`), los 16 bits restantes (32 - 16 = 16) están disponibles para asignar a los hosts (recursos como instancias EC2) dentro de la red.
  * El número de direcciones disponibles se calcula como 2^(bits de host), es decir, 2^16, lo que resulta en **65,536** direcciones IP.

Elegir un bloque `/16` es una práctica común al crear una VPC porque proporciona un espacio de direcciones muy amplio. Esto ofrece la flexibilidad de dividir la VPC en múltiples subredes de diferentes tamaños (por ejemplo, subredes `/24` o `/23` como se hará más adelante) para distintos propósitos (web, aplicaciones, bases de datos, etc.) sin riesgo de quedarse sin direcciones IP.

#### **¿Por qué habilitar los "nombres de host DNS"?**

Habilitar esta opción instruye a la VPC para que, a cualquier instancia EC2 que se lance con una dirección IP pública, se le asigne automáticamente un nombre de host DNS público. Este nombre sigue un formato como `ec2-52-42-133-255.us-west-2.compute.amazonaws.com`.

* **Facilidad de acceso**: Es mucho más fácil recordar o usar un nombre DNS que una dirección IP numérica, especialmente si la IP pública es dinámica y puede cambiar (por ejemplo, si la instancia se detiene y se reinicia).
* **Requisito para la resolución de nombres**: Este nombre DNS público se resuelve a la dirección IP pública de la instancia desde fuera de la VPC (desde Internet) y a la dirección IP privada de la instancia desde dentro de la VPC. Esto simplifica la comunicación tanto interna como externa.

Es importante destacar que habilitar esta opción **no concede** acceso a Internet. Simplemente proporciona un nombre. El acceso real depende de otros componentes como el Internet Gateway, las tablas de enrutamiento y los grupos de seguridad.

</details>

<details>

<summary><strong>Crear subredes públicas y privadas</strong></summary>

1. En el panel de navegación izquierdo de la consola de VPC, elige **Subnets** (Subredes).
2. Elige **Create subnet** (Crear subred).
3. Configura los siguientes ajustes:
   * **VPC ID**: Selecciona `FirstProof`.
   * **Subnet name** (Nombre de subred): `Public Subnet`.
   * **Availability Zone** (Zona de disponibilidad): Selecciona la **primera** zona de disponibilidad de la lista (no elijas "Sin preferencia").
   * **IPv4 subnet CIDR block** (Bloque de CIDR de subred IPv4): `10.0.0.0/24`.
4. Elige **Create Subnet** (Crear subred) y verifica que su estado sea **Available** (Disponible).
5. Con la `Public Subnet` recién creada y seleccionada, elige **Actions** (Acciones) y luego **Edit subnet settings** (Editar configuración de subred).
6. En la sección **Auto-assign IP settings** (Configuración de asignar IP de forma automática), marca la casilla **Enable auto-assign public IPv4 address** (Habilitar asignación automática de la dirección IPv4 pública).
7. Selecciona **Save** (Guardar).

**Crear una subred privada**

1. Elige nuevamente **Create subnet** (Crear subred).
2. Configura los siguientes ajustes:
   * **VPC ID**: Selecciona `FirstProof`.
   * **Subnet name** (Nombre de subred): `Private Subnet`.
   * **Availability Zone** (Zona de disponibilidad): Selecciona la **misma** zona de disponibilidad que usaste para la subred pública.
   * **IPv4 subnet CIDR block** (Bloque de CIDR de subred IPv4): `10.0.2.0/23`.
3. Elige **Create Subnet** (Crear subred) y verifica que su estado sea **Available** (Disponible).

<figure><img src="../../../../../.gitbook/assets/Creación de subredes publica con asignación automatica IPv4.gif" alt="Creación de subredes en Virtual Private Cloud"><figcaption><p>Creación de subredes en Virtual Private Cloud</p></figcaption></figure>

***

**¿Qué es una subred y por qué las usamos?**

Una subred (o _subnet_) es una subdivisión o un rango más pequeño de direcciones IP extraído del bloque CIDR principal de tu VPC. La creación de subredes es una práctica de red fundamental por varias razones:

* **Organización y aislamiento**: Permiten agrupar recursos con funciones y requisitos de seguridad similares. El patrón más común es el que se implementa en este laboratorio:
  * **Subredes públicas**: Para recursos que necesitan acceso directo a Internet (por ejemplo, servidores web, balanceadores de carga públicos).
  * **Subredes privadas**: Para recursos que no deben ser accesibles desde Internet, pero que sí pueden necesitar acceder a él (por ejemplo, servidores de aplicaciones, bases de datos, tareas de backend).
* **Control de tráfico**: A cada subred se le asocia una tabla de rutas, lo que te permite controlar con precisión cómo fluye el tráfico desde y hacia ese segmento de red.
* **Alta disponibilidad**: Las subredes están vinculadas a una única Zona de Disponibilidad (AZ). Para construir aplicaciones resilientes, se despliegan recursos en subredes distribuidas en múltiples AZs. Si una AZ falla, los recursos en las otras AZs pueden seguir operando.&#x20;

{% hint style="info" %}
Este laboratorio no incluye la implementación de infraestructura multizona.
{% endhint %}

**Comparación de CIDR: `/24` vs. `/23`**

La máscara de red (el número después de la `/`) determina el tamaño de la subred.

* **Public subnet (`10.0.0.0/24`)**:
  * La máscara `/24` fija los primeros 24 bits (`10.0.0`).
  * Quedan 8 bits para los hosts (32 - 24 = 8).
  * Esto proporciona 2^8 = **256 direcciones IP**.
  * **Nota importante**: AWS reserva las primeras cuatro y la última dirección IP de cada subred para sus propias funciones de red (router de la VPC, DNS, uso futuro, etc.). Por lo tanto, una subred `/24` tiene en realidad **251** direcciones IP utilizables.
* **Private subnet (`10.0.2.0/23`)**:
  * La máscara `/23` fija los primeros 23 bits. Esto abarca los rangos `10.0.2.x` y `10.0.3.x`.
  * Quedan 9 bits para los hosts (32 - 23 = 9).
  * Esto proporciona 2^9 = **512 direcciones IP** (o 507 utilizables).

La decisión de hacer la subred privada más grande es un patrón de diseño común. La "superficie de ataque" pública se mantiene pequeña y controlada, mientras que la mayor parte de la infraestructura (servidores de aplicación, microservicios, bases de datos) reside en la capa privada más grande, permitiendo un mayor escalado interno.

#### **¿Qué hace realmente la opción "Habilitar asignación automática de la dirección IPv4 pública"?**

Esta es una configuración de conveniencia a nivel de subred. Cuando está habilitada, cualquier instancia EC2 que se lance **dentro de esta subred específica** solicitará y recibirá automáticamente una dirección IP pública del pool de direcciones de Amazon.

* **Funcionamiento**: La instancia sigue teniendo su IP privada fija del rango CIDR de la subred (ej. `10.0.0.51`). La IP pública que recibe es un recurso separado que el Internet Gateway utilizará para realizar una traducción de direcciones de red (NAT) 1 a 1, permitiendo que la instancia se comunique con Internet.
* **Importancia**: Sin una IP pública, incluso con una ruta a un Internet Gateway, una instancia no puede ser alcanzada desde el exterior. Esta opción automatiza la asignación de dicha IP. Se desactiva explícitamente en subredes privadas para garantizar que las instancias allí no puedan ser contactadas directamente desde Internet.

***

#### **Preguntas clave**

#### **1. ¿Por qué la subred llamada `Public Subnet` todavía no es realmente "pública"?**

Nombrar una subred como "pública" es solo una etiqueta para la organización humana. Para que una subred sea funcionalmente pública, debe cumplir dos condiciones técnicas que aún no se han configurado en este punto:

1. **Tener una ruta a Internet**: Su tabla de rutas asociada debe tener una regla que dirija el tráfico destinado a Internet (representado como `0.0.0.0/0`) hacia un **Internet Gateway (IGW)**.
2. **Recursos con IP Públicas**: Las instancias dentro de ella deben tener direcciones IP públicas.

En este momento del laboratorio, hemos creado la subred y configurado la asignación automática de IP, pero nos falta el componente crucial del enrutamiento. Esto se abordará en las Tareas 3 y 4.

</details>

<details>

<summary><strong>Crear una gateway de Internet</strong></summary>

Crearás y adjuntarás el componente que actúa como la puerta de tu VPC hacia Internet.

1. En el panel de navegación izquierdo de la consola de VPC, elige **Internet gateways** (Puertas de enlace de internet).
2. Elige **Create Internet gateway** (Crear puerta de enlace de internet).
3. Configura los siguientes ajustes:
   * **Name tag** (Etiqueta de nombre): `FirstProof IGW`
4. Elige **Create internet gateway**.
5. Una vez creada, con la `FirstProof IGW` todavía seleccionada, elige **Actions** (Acciones) y luego **Attach to VPC** (Asociar a VPC).
6. En el menú desplegable, selecciona la `FirstProof`.
7. Elige **Attach Internet gateway** (Adjuntar puerta de enlace de internet).
8. Verifica que el estado de la `FirstProof IGW` cambie a **Attached** (Adjunto).

<figure><img src="../../../../../.gitbook/assets/Creación de Internet Gateway; enlazado VPC.gif" alt="Creación de Internet Gateway y asociación con Virtual Private Cloud AWS"><figcaption><p>Creación de Internet Gateway y asociación con Virtual Private Cloud AWS</p></figcaption></figure>

#### **Enrutar el tráfico de Internet en la subred pública a la gateway de Internet**

Ahora que la puerta existe, debes indicar a la subred pública cómo usarla creando y asociando una tabla de enrutamiento.

1. En el panel de navegación izquierdo, elige **Route Tables** (Tablas de enrutamiento).
2. Elige **Create route table** (Crear tabla de enrutamiento).
3. Configura los siguientes ajustes:
   * **Name** (Nombre): `Public Route Table`
   * **VPC**: Selecciona `FirstProof`.
4. Selecciona **Create route table**.
5. Con la `Public Route Table` recién creada y seleccionada, ve a la pestaña **Routes** (Rutas) en el panel inferior.
6. Selecciona **Edit routes** (Editar rutas).
7. Elige **Add route** (Agregar ruta) y configura lo siguiente:
   * **Destination** (Destino): `0.0.0.0/0`
   * **Target** (Objetivo): Elige `Internet Gateway` y selecciona la `FirstProof` que creaste.
8. Selecciona **Save changes** (Guardar cambios).

<figure><img src="../../../../../.gitbook/assets/Creación de route table y asociación VPC.gif" alt="Asociación de route table a Virtual Private Cloud AWS"><figcaption><p>Asociación de route table a Virtual Private Cloud AWS</p></figcaption></figure>

* Ahora, ve a la pestaña **Subnet associations** (Asociaciones de subredes).
* Elige **Edit subnet associations** (Editar asociaciones de subredes).
* Marca la casilla correspondiente a la **Public Subnet**.
* Selecciona **Save associations** (Guardar asociaciones).

<figure><img src="../../../../../.gitbook/assets/Asociación subnet a route table.png" alt="Asociación de subnet a route table AWS"><figcaption><p>Asociación de subnet a route table AWS</p></figcaption></figure>

#### **Crear un grupo de seguridad público**

Finalmente, crearás un firewall a nivel de instancia para permitir explícitamente el tráfico web entrante.

1. En el panel de navegación izquierdo, selecciona **Security Groups** (Grupos de seguridad).
2. Elige **Create security group** (Crear grupo de seguridad).
3. Configura los siguientes ajustes:
   * **Security group name** (Nombre del grupo de seguridad): `Public SG`
   * **Description** (Descripción): `Allows incoming traffic to public instance.`
   * **VPC**: Selecciona `FirstProof`.
4. En la sección **Inbound rules** (Reglas de entrada), elige **Add rule** (Agregar regla):
   * **Type** (Tipo): `HTTP`
   * **Source** (Fuente): `Anywhere-IPv4`
5. En la sección de **Tags** (Etiquetas), agrega una nueva etiqueta:
   * **Key** (Clave): `Name`
   * **Value** (Valor): `Public SG`
6. Elige **Create security group** (Crear grupo de seguridad).

<figure><img src="../../../../../.gitbook/assets/Creación de security group acceso HTTP.gif" alt="Creación de un grupo de seguridad con acceso HTTP"><figcaption><p>Creación de un grupo de seguridad con acceso HTTP</p></figcaption></figure>

***

#### **¿Qué es un Internet Gateway (IGW)?**

Un Internet Gateway es un componente de VPC **redundante, de alta disponibilidad y escalado horizontalmente** que permite la comunicación entre tu VPC e Internet. Tiene dos funciones principales:

1. **Proporcionar un destino en la tabla de rutas**: Actúa como el "objetivo" (`Target`) para todo el tráfico destinado a Internet (la ruta `0.0.0.0/0`). Sin un IGW, no hay a dónde enviar el tráfico que sale de la VPC.
2. **Realizar NAT 1:1**: Para las instancias con direcciones IPv4 públicas, el IGW realiza la traducción de direcciones de red (NAT). Cuando el tráfico sale de la instancia, el IGW traduce la dirección IP de origen (la IP privada de la instancia) a la IP pública de la instancia. Cuando el tráfico de respuesta regresa, el IGW realiza la traducción inversa.

Adjuntar un IGW a una VPC es como instalar la puerta principal de un edificio. Sin embargo, solo instalar la puerta no es suficiente; necesitas indicar a la gente (el tráfico de la subred) cómo llegar a ella.

#### **¿Cómo funcionan las Tablas de Enrutamiento?**

Una tabla de enrutamiento es como el **GPS de tu subred**. Contiene un conjunto de reglas, llamadas **rutas**, que determinan hacia dónde se dirige el tráfico de red que se origina en la subred.

* **Ruta local**: Cada tabla de enrutamiento incluye, por defecto, una ruta `local` (por ejemplo, `10.0.0.0/16` -> `local`). Esta ruta no se puede eliminar y permite que todos los recursos dentro de la VPC se comuniquen entre sí utilizando sus direcciones IP privadas.
* **La ruta a internet (`0.0.0.0/0`)**: Esta es la "ruta por defecto". El CIDR `0.0.0.0/0` es una notación que significa "cualquier dirección IP que no coincida con otra ruta más específica en esta tabla". Al dirigir este tráfico al IGW, estás diciendo: "Si el destino no está dentro de mi VPC, envíalo a Internet a través del Internet Gateway".
* **Asociación de subred**: Una tabla de enrutamiento es inútil hasta que se asocia con una o más subredes. Al asociar la `Public Route Table` con la `Public Subnet`, hiciste que la subred se volviera _verdaderamente_ pública, ya que ahora tiene un camino definido hacia Internet.

#### **¿Qué es un Grupo de Seguridad (Security Group)?**

Un Grupo de Seguridad (SG) es un **firewall virtual con estado** que se aplica a nivel de la interfaz de red de una instancia (ENI), no de la subred.

* **Firewall con estado (Stateful)**: Este es su atributo más importante. Si permites una conexión entrante (por ejemplo, HTTP en el puerto 80), el Grupo de Seguridad **automáticamente permite el tráfico de respuesta saliente** en un puerto efímero, sin necesidad de una regla de salida explícita para ello. Esto simplifica enormemente la configuración.
* **Permitir por defecto**: Los Grupos de Seguridad funcionan con una política de "denegar todo". No permiten ningún tráfico entrante a menos que se agregue una regla explícita de `Allow`. Por defecto, todo el tráfico saliente está permitido.
* **La Regla creada (`HTTP` desde `Anywhere-IPv4`)**: Esta regla se traduce como: "Permitir que cualquier dispositivo en Internet (`Source: 0.0.0.0/0`) inicie una conexión con los recursos asociados a este SG en el puerto `80` (el puerto estándar para el protocolo `HTTP`)". Esta es la regla que permitirá a los usuarios acceder al servidor web que se lanzará más adelante.

</details>

<details>

<summary><strong>Iniciar una instancia de Amazon EC2 en una subred pública</strong></summary>

Ahora que toda la infraestructura de red está lista, lanzarás un servidor web en la subred pública.

1. En la barra de búsqueda de la consola de AWS, busca y selecciona **EC2**.
2. En el panel de la izquierda, elige **EC2 Dashboard** (Panel de EC2) y luego haz clic en **Launch instances** (Lanzar instancias).
3. **Nombre y etiquetas**:
   * En el campo **Name**, ingresa `Public Instance`.
4. **Imágenes de aplicaciones y SO (Amazon Machine Image)**:
   * Asegúrate de que **Amazon Linux** esté seleccionado.
   * Verifica que la AMI sea **Amazon Linux 2023 AMI**.
5. **Tipo de instancia**:
   * En el menú desplegable, elige `t3.micro`.
6. **Par de claves (inicio de sesión)**:
   * Selecciona **Proceed without a key pair (Not recommended)** (Continuar sin par de claves \[No recomendado]).
7. **Configuración de red**:
   * Haz clic en **Edit** (Editar).
   * **VPC**: Selecciona `FirstProof`.
   * **Subnet** (Subred): Selecciona `Public Subnet`.
   * **Auto-assign public IP** (Asignar IP pública automáticamente): Asegúrate de que esté en **Enable** (Habilitar).
   * **Firewall (security groups)**: Elige **Select existing security group** (Seleccionar grupo de seguridad existente).
   * En **Common security groups**, selecciona `Public SG`.
8. **Detalles avanzados**:
   * Expande la sección **Advanced details**.
   * **IAM instance profile** (Perfil de instancias de IAM): Selecciona el rol disponible (debería tener un nombre como `EC2InstProfile`).
   *   **User Data** (Datos de usuario): Copia y pega el siguiente script en el campo de texto:<br>

       ```bash
       bash#!/bin/bash
       # Instala y configura un servidor web Apache
       yum update -y
       yum install -y httpd php8.1
       systemctl enable httpd.service
       systemctl start httpd
       cd /var/www/html
       wget https://us-west-2-tcprod.s3.amazonaws.com/courses/ILT-TF-200-ARCHIT/v7.9.11.prod-60bc4f16/lab-2-VPC/scripts/instanceData.zip
       unzip instanceData.zip
       ```
9. Revisa el resumen y elige **Launch instance** (Lanzar instancia).
10. Ve a **View all instances** (Ver todas las instancias) y espera a que el **Instance state** (Estado de la instancia) sea `Running` y la **Status check** (Comprobación de estado) muestre `2/2 checks passed`.

#### **Conectarse a la instancia pública a través de HTTP**

Verificarás que el servidor web es accesible desde Internet.

1. En la lista de instancias, selecciona la `Public Instance`.
2. En el panel de detalles inferior, busca el valor de **Public IPv4 DNS** y cópialo.
3.  Abre una nueva pestaña en tu navegador, pega el DNS en la barra de direcciones y presiona Enter.<br>

    > **Nota**: No uses la opción "Open address", ya que intenta conectar por HTTPS, que no está configurado. La conexión debe ser por HTTP.
4. Deberías ver una página web que muestra el ID de la instancia y su Zona de Disponibilidad.

#### **Conectarse a la instancia mediante Session Manager**

Utilizarás AWS Systems Manager (SSM) para obtener acceso de línea de comandos a la instancia de forma segura.

1. Con la `Public Instance` aún seleccionada, haz clic en el botón **Connect** (Conectar).
2. Ve a la pestaña **Session Manager**.
3. Haz clic en **Connect** (Conectar). Se abrirá una nueva pestaña del navegador con una terminal de shell.
4.  Para probar la conectividad saliente de la instancia, ejecuta el siguiente comando:

    ```
    bashcurl -I https://aws.amazon.com/training/
    ```
5. Deberías recibir una respuesta `HTTP/2 200`, confirmando que la instancia puede alcanzar servicios externos en Internet.

***

#### **¿Qué es el Script de "User Data"?**

User Data es un script que se ejecuta **una sola vez** durante el primer arranque de una instancia EC2. Es la herramienta principal para el _bootstrapping_, es decir, la automatización de la configuración inicial. El script de este laboratorio realiza las siguientes acciones:

1. `yum update -y`: Actualiza todos los paquetes del sistema operativo a sus últimas versiones.
2. `yum install -y httpd php8.1`: Instala el software del servidor web Apache (`httpd`) y PHP.
3. `systemctl enable/start httpd`: Habilita el servicio Apache para que se inicie automáticamente en futuros reinicios y lo inicia en ese momento.
4. `wget ...` y `unzip ...`: Navega al directorio raíz del servidor web, descarga un archivo ZIP desde un bucket público de S3 y lo descomprime. Este ZIP contiene la página web de ejemplo que viste en la Tarea 7.

#### **¿Por qué procedimos sin un par de claves y cómo funciona Session Manager?**

Un par de claves (Key Pair) se usa tradicionalmente para la autenticación en una conexión SSH (que requiere abrir el puerto 22). En este laboratorio, optamos por un método más moderno y seguro: **AWS Systems Manager Session Manager**.

**Funcionamiento de Session Manager**:

1. **Agente SSM**: Las AMIs de Amazon Linux vienen con el agente de Systems Manager (SSM Agent) preinstalado.
2. **Rol de IAM**: Al adjuntar el **Perfil de Instancia de IAM** (`EC2InstProfile`), le diste a la instancia EC2 el permiso para comunicarse con la API del servicio Systems Manager.
3. **Conexión saliente segura**: El agente SSM en la instancia inicia una **conexión saliente** segura hacia los puntos de enlace del servicio SSM en la red de AWS.
4. **Túnel seguro**: Cuando solicitas una conexión a través de la consola, AWS establece un túnel seguro a través de esa conexión existente, dándote acceso de shell a la instancia.

La gran ventaja de seguridad es que **no necesitas abrir ningún puerto de entrada** (como el puerto 22 para SSH) en tu grupo de seguridad, reduciendo drásticamente la superficie de ataque de la instancia. La autenticación y autorización se gestionan completamente a través de IAM, no de claves SSH.

</details>

<details>

<summary><strong>Crear de Gateway NAT y configurar el enrutamiento de la subred privada</strong></summary>

Crearás una NAT Gateway. Este componente permitirá que las instancias en la subred privada inicien conexiones salientes a Internet (por ejemplo, para descargar actualizaciones) sin permitir que Internet inicie conexiones entrantes hacia ellas, manteniendo así su aislamiento y seguridad.

1. Regresa a la consola de **VPC**.
2. En el panel de navegación izquierdo, elige **NAT gateways** (Puertas de enlace NAT).
3. Elige **Create NAT gateway** (Crear puerta de enlace NAT).
4. Configura los siguientes ajustes:
   * **Name** (Nombre): `FirstProof NGW`
   * **Subnet** (Subred): Selecciona la `Public Subnet`.
   * **Elastic IP allocation ID** (ID de asignación de IP elástica): Elige **Allocate Elastic IP** (Asignar IP elástica).
5. Selecciona **Create NAT gateway**. El aprovisionamiento puede tardar unos minutos.
6. Mientras esperas, crearás la tabla de enrutamiento para la subred privada. En el panel izquierdo, elige **Route Tables** (Tablas de enrutamiento).
7. Elige **Create route table** (Crear tabla de enrutamiento).
8. Configura los siguientes ajustes:
   * **Name** (Nombre): `Private Route Table`
   * **VPC**: Selecciona `FirstProof`.
9. Selecciona **Create route table**.
10. Con la `Private Route Table` recién creada y seleccionada, ve a la pestaña **Routes** (Rutas) en el panel inferior.
11. Selecciona **Edit routes** (Editar rutas).
12. Elige **Add route** (Agregar ruta) y configura lo siguiente:
    * **Destination** (Destino): `0.0.0.0/0`
    * **Target** (Objetivo): Elige `NAT Gateway` y selecciona la `FirstProof NGW` que creaste.
13. Selecciona **Save changes** (Guardar cambios).
14. Ahora, ve a la pestaña **Subnet associations** (Asociaciones de subredes).
15. Elige **Edit subnet associations** (Editar asociaciones de subredes).
16. Marca la casilla correspondiente a la **Private Subnet**.
17. Selecciona **Save associations** (Guardar asociaciones).

***

#### **¿Qué es una NAT Gateway y por qué se coloca en una subred pública?**

Una **NAT (Network Address Translation) Gateway** es un servicio administrado por AWS que resuelve un problema fundamental de la arquitectura de dos niveles: ¿cómo pueden los recursos privados (como una base de datos o un servidor de aplicaciones) acceder a Internet para descargar parches de seguridad, actualizaciones de software o comunicarse con APIs externas, sin exponerse a ser atacados desde Internet?

* **Funcionamiento**:
  1. Una instancia en la subred privada envía tráfico destinado a Internet.
  2. La tabla de rutas de la subred privada dirige este tráfico a la NAT Gateway.
  3. La NAT Gateway, que vive en la **subred pública**, reemplaza la dirección IP privada de origen de la instancia con su propia dirección IP pública (la Elastic IP).
  4. Luego, reenvía el tráfico al **Internet Gateway** (usando la tabla de rutas de la subred pública).
  5. Para el mundo exterior, todo el tráfico parece originarse desde la IP de la NAT Gateway. El tráfico de respuesta regresa a la NAT Gateway, que lo traduce de nuevo y lo envía a la instancia privada correcta.
* **Ubicación en Subred Pública**: La NAT Gateway **debe** estar en una subred pública porque necesita una ruta directa al Internet Gateway para poder enviar el tráfico hacia y desde Internet. Si estuviera en una subred privada, quedaría aislada y no podría cumplir su función.

#### **¿Qué es una Elastic IP y por qué es necesaria para la NAT Gateway?**

Una **Elastic IP (EIP)** es una dirección IPv4 pública **estática** que puedes asignar a tu cuenta de AWS. A diferencia de las IPs públicas estándar que se asignan y liberan cuando una instancia EC2 se detiene y se inicia, una EIP permanece bajo tu control hasta que la liberes explícitamente.

* **Necesidad para la NAT Gateway**: La NAT Gateway necesita una EIP para proporcionar un **punto de salida a Internet fijo y predecible** para todas las instancias en la subred privada. Esto es crucial por varias razones:
  * **Whitelistings**: Si necesitas que tus servicios privados accedan a una API de un tercero que requiere que tu dirección IP de origen esté en una lista de permitidos (whitelist), puedes proporcionarles la EIP de la NAT Gateway.
  * **Consistencia**: Asegura que la IP pública de salida no cambie, garantizando una conectividad estable.

#### **Flujo del tráfico: ¿Por qué la `Private Route Table` apunta a la NAT Gateway?**

Al configurar la ruta por defecto (`0.0.0.0/0`) en la `Private Route Table` para que apunte a la `Lab NGW`, has definido explícitamente el flujo de tráfico saliente para la subred privada:

1. **Origen**: Una instancia en `Private Subnet` intenta conectarse a `google.com`.
2. **Enrutamiento**: El router de la VPC consulta la `Private Route Table` asociada.
3. **Coincidencia**: La solicitud coincide con la ruta `0.0.0.0/0`.
4. **Destino Interno**: El `Target` es la NAT Gateway. El tráfico se envía a la NAT Gateway dentro de la VPC.
5. **Traducción y Salida**: La NAT Gateway realiza la traducción de direcciones (privada -> pública) y, como reside en la `Public Subnet`, utiliza la `Public Route Table` para enviar el tráfico al `Internet Gateway` y finalmente a Internet.

Esto completa el aislamiento de la subred privada: puede "hablar" con el mundo, pero el mundo no puede iniciar una "conversación" con ella.

</details>

<details>

<summary><strong>Crear un grupo de seguridad para recursos privados</strong></summary>

Crearás un firewall para la instancia privada. La regla clave aquí no permitirá tráfico desde Internet, sino específicamente desde los recursos que estén en el grupo de seguridad público (`Public SG`).

1. En la consola de **VPC**, en el panel de navegación izquierdo, selecciona **Security Groups** (Grupos de seguridad).
2. Elige **Create security group** (Crear grupo de seguridad).
3. Configura los siguientes ajustes:
   * **Security group name** (Nombre del grupo de seguridad): `Private SG`
   * **Description** (Descripción): `Allows incoming traffic to private instance from public security group.`
   * **VPC**: Selecciona `FirstProof`.
4. En la sección **Inbound rules** (Reglas de entrada), elige **Add rule** (Agregar regla):
   * **Type** (Tipo): `HTTP`
   * **Source** (Fuente): Elige `Custom` (Personalizado) y en el campo de texto que aparece, comienza a escribir `sg-`. Selecciona el grupo de seguridad `Public SG` de la lista que se despliega.
5. En la sección de **Tags** (Etiquetas), agrega una nueva etiqueta:
   * **Key** (Clave): `Name`
   * **Value** (Valor): `Private SG`
6. Elige **Create security group** (Crear grupo de seguridad).

#### **Iniciar una instancia de Amazon EC2 en una subred privada**

Lanzarás un servidor en la subred privada. Este servidor no tendrá IP pública y estará protegido por el nuevo grupo de seguridad.

1. Navega a la consola de **EC2** y haz clic en **Launch instances** (Lanzar instancias).
2. **Nombre y etiquetas**:
   * En el campo **Name**, ingresa `Private Instance`.
3. **Imágenes de aplicaciones y SO (Amazon Machine Image)**:
   * Asegúrate de que la AMI sea **Amazon Linux 2023 AMI**.
4. **Tipo de instancia**:
   * Elige `t3.micro`.
5. **Par de claves (inicio de sesión)**:
   * Selecciona **Proceed without a key pair (Not recommended)**.
6. **Configuración de red**:
   * Haz clic en **Edit** (Editar).
   * **VPC**: Selecciona `FirstProof`.
   * **Subnet** (Subred): Selecciona `Private Subnet`.
   * **Auto-assign public IP** (Asignar IP pública automáticamente): Asegúrate de que esté en **Disable** (Deshabilitar).
   * **Firewall (security groups)**: Elige **Select existing security group** (Seleccionar grupo de seguridad existente).
   * En **Common security groups**, selecciona `Private SG`.
7. **Detalles avanzados**:
   * Expande la sección **Advanced details**.
   * **IAM instance profile**: Selecciona el rol disponible (`EC2InstProfile`).
   *   **User Data**: Copia y pega el mismo script que usaste para la instancia pública:<br>

       ```
       bash#!/bin/bash
       yum update -y
       yum install -y httpd php8.1
       systemctl enable httpd.service
       systemctl start httpd
       cd /var/www/html
       wget https://us-west-2-tcprod.s3.amazonaws.com/courses/ILT-TF-200-ARCHIT/v7.9.11.prod-60bc4f16/lab-2-VPC/scripts/instanceData.zip
       unzip instanceData.zip
       ```
8. Revisa el resumen y elige **Launch instance**.
9. Espera a que el estado de la `Private Instance` sea `Running` y la comprobación de estado `2/2 checks passed`.

#### **Conectarse a la instancia de Amazon EC2 en la subred privada**

Al igual que con la instancia pública, usarás Session Manager para acceder y verificar que puede conectarse a Internet a través de la NAT Gateway.

1. En la lista de instancias, selecciona la `Private Instance` y haz clic en **Connect** (Conectar).
2. Ve a la pestaña **Session Manager** y haz clic en **Connect**.
3.  En la terminal de shell que se abre, prueba la conectividad saliente:<br>

    ```
    bashcurl -I https://aws.amazon.com/training/
    ```
4. Deberías ver una respuesta `HTTP/2 200`, lo que confirma que la instancia privada, a pesar de no tener una IP pública, puede acceder a Internet a través de la NAT Gateway.

***

#### **La magia de referenciar un grupo de seguridad como fuente**

Esta es una de las características más potentes y fundamentales de los Grupos de Seguridad de AWS para construir arquitecturas seguras. Cuando configuras la regla de entrada del `Private SG` con la fuente `Public SG`, estás diciendo:

> "Permite el tráfico entrante en el puerto 80 **únicamente si se origina desde una interfaz de red (ENI) que tenga el grupo de seguridad `Public SG` adjunto**."

**¿Por qué es tan importante?**

* **No depende de IPs**: No necesitas saber las direcciones IP de tus instancias públicas, que pueden cambiar si se terminan y se reemplazan (por ejemplo, en un grupo de Auto Scaling). La regla es dinámica y se basa en la pertenencia al grupo, no en IPs estáticas.
* **Micro-segmentación**: Permite un control de tráfico extremadamente granular. Puedes crear una cadena de confianza entre las capas de tu aplicación (por ejemplo, Web -> App -> Base de datos) simplemente referenciando los grupos de seguridad de cada capa.
* **Seguridad mejorada**: Evita la necesidad de abrir rangos de CIDR amplios entre tus subredes. En lugar de permitir el tráfico desde toda la `Public Subnet` (`10.0.0.0/24`), solo lo permites desde los recursos específicos dentro de esa subred que cumplen la función de capa pública. AWS gestiona la traducción de la referencia `sg-xxxxx` a las IPs privadas de las instancias correspondientes en tiempo real.

#### **¿Por qué la instancia privada necesita el script "User Data"?**

Aunque la instancia privada no servirá tráfico directamente desde Internet, se le instaló un servidor web por dos razones en el contexto de este laboratorio:

1. **Simular un servidor de aplicaciones**: En una arquitectura real, esta instancia podría ser un servidor de aplicaciones que expone una API en el puerto 80 (o cualquier otro puerto). La instancia pública actuaría como un proxy inverso, recibiendo las solicitudes de los usuarios y pasándolas a este servidor de aplicaciones privado.
2. **Validación de conectividad**: Tener un servicio escuchando en el puerto 80 en la instancia privada es crucial para la tarea opcional de _troubleshooting_, donde verificarás que la instancia pública puede conectarse a la privada usando `curl`.

#### **Flujo de Conexión de la Instancia Privada a Internet**

El éxito del comando `curl` demuestra la arquitectura completa en acción:

1. **Origen**: La `Private Instance` (sin IP pública) inicia la conexión.
2. **Ruta Privada**: La `Private Route Table` dirige el tráfico `0.0.0.0/0` a la **NAT Gateway**.
3. **Traducción NAT**: La NAT Gateway en la `Public Subnet` recibe el tráfico, cambia la IP de origen por su propia Elastic IP pública.
4. **Ruta Pública**: La NAT Gateway usa la `Public Route Table` para enviar el tráfico al **Internet Gateway**.
5. **Salida**: El IGW envía la solicitud a Internet.
6. **Respuesta**: La respuesta regresa al IGW, luego a la NAT Gateway, que la traduce de nuevo y la envía a la IP privada original de la `Private Instance`. Todo esto sucede de forma transparente para la instancia.

</details>
