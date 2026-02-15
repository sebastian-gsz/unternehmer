---
description: >-
  Arquitectura multinivel en AWS: VPC 3 capas, subredes públicas/privadas, NAT
  Gateway, CloudFormation y Terraform para WordPress.
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

# Construya una arquitectura multinivel de AWS

## Construya una arquitectura multinivel de AWS

Diseñe una **arquitectura multinivel en AWS** para una aplicación web. Use una **VPC de 3 capas** con **subredes públicas y privadas**, **Internet Gateway**, **NAT Gateway**, **tablas de rutas** y **grupos de seguridad**.

Esta guía cubre dos enfoques de **Infrastructure as Code (IaC)**: **AWS CloudFormation** y **Terraform**. El objetivo es una base sólida para **WordPress de alta disponibilidad** en múltiples **Zonas de Disponibilidad (AZ)**.

* **Servicios clave**: Amazon VPC, Subnets, IGW, NAT Gateway, Route Tables, Security Groups.
* **Automatización**: CloudFormation (plantillas YAML/JSON) y Terraform (HCL).
* **Resultados**: red segmentada por capas, escalabilidad y mejor postura de seguridad.

<figure><img src="../../../../../.gitbook/assets/Construya una arquitectura multinivel de AWS.png" alt="Diagrama de arquitectura multinivel en AWS con VPC, subredes públicas y privadas, NAT Gateway e Internet Gateway"><figcaption></figcaption></figure>

La empresa SEO Consulting S. A. crea campañas de marketing para pequeñas y medianas empresas. Recientemente, la empresa te contrató para que trabajes con los equipos de ingeniería a fin de diseñar una prueba de concepto para su negocio. Hasta la fecha, ha alojado su aplicación orientada a los clientes en un centro de datos en las instalaciones, pero ha decidido trasladar sus operaciones a la nube para ahorrar dinero y transformar su negocio con un enfoque centrado en la nube. Algunos miembros del equipo de SEO Consulting, S.A. tienen experiencia en la nube y recomendaron utilizar los servicios de la nube de AWS para diseñar su solución.

Además, decidieron rediseñar su portal web. Los clientes utilizan el portal para acceder a sus cuentas, crear planes de marketing y poner en marcha análisis de datos en sus campañas de marketing. Les gustaría contar con un prototipo funcional en dos semanas. Debes diseñar una arquitectura que admita esta aplicación. La solución debe ser rápida, duradera, escalable y más rentable que su infraestructura en las instalaciones actual.

<figure><img src="../../../../../.gitbook/assets/Barra.gif" alt="Arquitectura de 3 capas en AWS para WordPress: capa pública, capa de aplicación privada y capa de base de datos privada"><figcaption></figcaption></figure>

### Arquitectura objetivo: VPC de 3 capas (multinivel)

Este diseño separa la carga en capas. Mejora la seguridad. Simplifica el enrutamiento.

* **Capa pública**: balanceador, bastion (opcional), NAT Gateway.
* **Capa de aplicación**: EC2/ECS/EKS privados.
* **Capa de datos**: RDS/ElastiCache/EFS privados.

{% hint style="info" %}
Si quieres más contexto, revisa [AWS Infrastructure](../).
{% endhint %}

<details>

<summary><strong>Revisar y poner en marcha una plantilla preconfigurada en CloudFormation</strong></summary>

<figure><img src="../../../../../.gitbook/assets/Construya una arquitectura multinivel de AWS - CloudFormation 1.png" alt="Despliegue de VPC multinivel en AWS con CloudFormation (plantilla preconfigurada)"><figcaption></figcaption></figure>

Se despliega infraestructura de red fundamental para la aplicación WordPress. Se utiliza una plantilla de AWS CloudFormation para automatizar la creación de una VPC segura y de alta disponibilidad que abarca múltiples Zonas de Disponibilidad (AZ).

1. **Situación del laboratorio**: El equipo de redes ha definido los requisitos para la infraestructura base. Se debe crear una **Virtual Private Cloud (VPC)** con un diseño multinivel que separe los recursos en diferentes capas de seguridad. Esta VPC debe incluir:
   * Dos **subredes públicas**.
   * Dos **subredes de aplicación privadas**.
   * Dos **subredes de base de datos privadas**.
   * Una **Gateway de Internet (IGW)** para permitir el acceso a internet desde las subredes públicas.
   * Una **Gateway NAT (NAT Gateway)** para permitir que los recursos en las subredes privadas inicien conexiones hacia internet (por ejemplo, para actualizaciones) sin ser accesibles directamente desde el exterior.
   * Tablas de enrutamiento asociadas correctamente a cada subred.
   * Dos **IPs Elásticas** (para las Gateways NAT, asegurando puntos de salida estáticos).
2. **Pasos de la tarea**:
   1. Descargar y revisar el archivo de plantilla `Plantilla lanzamiento CloudFormation - 1.yaml` para comprender los recursos de AWS que se definirán y crearán.
   2. Acceder a la consola de AWS y navegar al servicio **CloudFormation**.
   3. Iniciar el proceso para crear una nueva pila ("stack") utilizando una plantilla existente.
   4. Cargar o especificar la URL de Amazon S3 proporcionada para la plantilla `Plantilla lanzamiento CloudFormation - 1.yaml`.
   5. Configurar los parámetros de la pila:
      * **Nombre de la pila**: `VPCStack`
      * Dejar los demás parámetros con sus valores por defecto.
   6. Revisar y confirmar la creación de la pila. CloudFormation se encargará de aprovisionar todos los recursos definidos en la plantilla.
   7. Una vez que el estado de la pila sea `CREATE_COMPLETE`, navegar a la consola de VPC para verificar que todos los componentes de red (VPC, subredes, tablas de enrutamiento, gateways) han sido creados correctamente.

***

* **¿Qué es AWS CloudFormation y por qué se utiliza?**
  * **Funcionamiento**: AWS CloudFormation es un servicio de **Infraestructura como Código (IaC)**. Permite definir y aprovisionar la infraestructura de AWS de manera predecible y repetible mediante un archivo de plantilla (en formato YAML o JSON). En lugar de crear manualmente cada recurso (VPC, subred, etc.) a través de la consola, se describe el estado final deseado en la plantilla y CloudFormation se encarga de crearlo y configurarlo en el orden correcto.
  * **Ventajas**:
    * **Automatización y consistencia**: Elimina errores manuales y garantiza que el entorno se despliegue siempre de la misma manera, lo cual es crucial para entornos de desarrollo, pruebas y producción.
    * **Versionado y auditoría**: La plantilla puede ser versionada en un sistema de control de código (como Git), permitiendo rastrear cambios y revertir a versiones anteriores si es necesario.
    * **Eficiencia**: Acelera drásticamente el proceso de aprovisionamiento de infraestructuras complejas.
* **Arquitectura de VPC con subredes públicas y privadas**
  * **Subred pública**: Una subred se considera "pública" porque su tabla de enrutamiento tiene una ruta directa a la **Gateway de Internet (IGW)**. Esto permite que los recursos dentro de ella (como un Application Load Balancer) puedan comunicarse bidireccionalmente con internet. Son la "puerta de entrada" a la aplicación.
  * **Subred privada**: Una subred es "privada" porque no tiene una ruta directa a la IGW. Los recursos en ella (como servidores de aplicación o bases de datos) no son accesibles directamente desde internet, lo que constituye una capa fundamental de seguridad.
  * **Beneficio del diseño**: Esta separación crea una arquitectura de defensa en profundidad. Los servidores de aplicación y las bases de datos, que contienen la lógica de negocio y los datos sensibles, están protegidos de ataques directos. Solo el tráfico filtrado y distribuido por el balanceador de carga en la subred pública puede llegar a ellos.
* **El Rol de la Gateway NAT (NAT Gateway)**
  * **Problema a resolver**: Las instancias en subredes privadas (como los servidores de aplicación) a menudo necesitan acceder a internet para descargar parches de seguridad, actualizaciones de software o conectarse a APIs de terceros. Sin embargo, no deben tener una IP pública para no ser vulnerables.
  * **Solución**: La **NAT Gateway** se coloca en una subred pública y tiene una IP elástica (pública). Las tablas de enrutamiento de las subredes privadas se configuran para dirigir el tráfico destinado a internet hacia la NAT Gateway. De este modo, las instancias privadas pueden _iniciar_ conexiones hacia internet, y las respuestas regresan a través de la NAT Gateway, pero internet no puede _iniciar_ conexiones hacia las instancias privadas.

***

1. **¿Por qué es importante revisar la plantilla de CloudFormation antes de ejecutarla?**\
   Revisar la plantilla (`Plantilla lanzamiento CloudFormation - 1.yaml`) permite entender exactamente qué recursos se crearán, cómo se configurarán y cómo se conectarán entre sí. Esto es crucial para validar que la infraestructura cumple con los requisitos de seguridad y arquitectura (ej. verificar que las subredes de base de datos son privadas y no tienen rutas a la IGW) y para evitar costos inesperados o configuraciones incorrectas.
2. **¿Cuál es la diferencia fundamental entre una Gateway de Internet (IGW) y una Gateway NAT (NAT Gateway)?**
   * La **Gateway de Internet (IGW)** permite la comunicación bidireccional entre la VPC e internet. Se asocia a nivel de VPC y se usa en las tablas de enrutamiento de las subredes públicas. Es lo que permite que el tráfico _entre_ desde internet.
   * La **Gateway NAT (NAT Gateway)** es un servicio administrado que permite que instancias en una subred privada se conecten a internet u otros servicios de AWS, pero impide que internet inicie una conexión con esas instancias. Funciona como un "traductor" que permite el tráfico de salida, pero no de entrada.

***

Esta plantilla está escrita en formato YAML y su objetivo es crear una infraestructura de red completa y escalable en la nube de AWS.

{% code overflow="wrap" %}
```yaml
AWSTemplateFormatVersion: "2010-09-09"
Description: Lab7 Task 1 template which builds a VPC, supporting resources, a basic networking structure, and security groups for later tasks.
```
{% endcode %}

* **`AWSTemplateFormatVersion`**: Esta es la primera línea obligatoria en cualquier plantilla de CloudFormation. Define la versión del formato de la plantilla. `2010-09-09` es la versión más reciente y la única actualmente disponible.
* **`Description`**: Este campo opcional te permite añadir un comentario para describir el propósito de la plantilla. En este caso, indica que la plantilla creará una VPC (Virtual Private Cloud), recursos de soporte, una estructura de red básica y grupos de seguridad para ser utilizados en tareas posteriores.

**2. Parámetros (`Parameters`)**

La sección `Parameters` permite que la plantilla sea reutilizable y flexible, ya que puedes introducir valores personalizados al momento de crear la pila de CloudFormation. Si no se proporciona un valor, se utiliza el `Default`.

```
Parameters:
  VPCCIDR:
    Description: CIDR Block for VPC
    Type: String
    Default: 10.0.0.0/16
    AllowedValues:
      - 10.0.0.0/16

  PublicSubnet1Param:
    # ... y así sucesivamente para las demás subredes
```

En esta plantilla, se definen los siguientes parámetros:

* **`VPCCIDR`**: El rango de direcciones IP para la VPC principal. El valor por defecto es `10.0.0.0/16`, que proporciona 65,536 direcciones IP privadas.
* **`PublicSubnet1Param` y `PublicSubnet2Param`**: Los rangos de IP para dos subredes públicas. Estas subredes tendrán acceso directo a internet.
* **`AppSubnet1Param` y `AppSubnet2Param`**: Los rangos de IP para dos subredes privadas destinadas a las aplicaciones. Estas no tendrán acceso directo desde internet.
* **`DatabaseSubnet1Param` y `DatabaseSubnet2Param`**: Los rangos de IP para dos subredes privadas destinadas a las bases de datos, que es la capa más segura de la red.

**3. Recursos (`Resources`)**

Esta es la sección más importante de la plantilla, donde se definen todos los recursos de AWS que se van a crear.

**VPC y estructura de red**

```
  LabVPC:
    Type: 'AWS::EC2::VPC'
    Properties:
      CidrBlock: !Ref VPCCIDR
      # ...
  LabInternetGateway:
    Type: 'AWS::EC2::InternetGateway'

  AttachGateway:
    Type: 'AWS::EC2::VPCGatewayAttachment'
    Properties:
      VpcId: !Ref LabVPC
      InternetGatewayId: !Ref LabInternetGateway
```

* **`LabVPC`**: Crea la Virtual Private Cloud (VPC), que es una red virtual aislada. Utiliza el CIDR definido en los parámetros. Las propiedades `EnableDnsSupport` y `EnableDnsHostnames` permiten la resolución de nombres DNS dentro de la VPC.
* **`LabInternetGateway`**: Crea un Internet Gateway (IGW), que es el componente que permite la comunicación entre la VPC e internet.
* **`AttachGateway`**: Asocia el Internet Gateway recién creado con la `LabVPC`. Sin este paso, el IGW no funcionaría.

**Gateways NAT y direcciones IP elásticas**

```
  NATGateway1:
    Type: AWS::EC2::NatGateway
    Properties:
      AllocationId: !GetAtt ElasticIPAddress1.AllocationId
      SubnetId: !Ref PublicSubnet1
  ElasticIPAddress1:
    Type: AWS::EC2::EIP
    Properties:
      Domain: vpc
```

* **`ElasticIPAddress1` y `ElasticIPAddress2`**: Se reservan dos direcciones IP elásticas (EIP), que son direcciones IP públicas estáticas.
* **`NATGateway1` y `NATGateway2`**: Se crean dos Gateways NAT, uno en cada subred pública. Un Gateway NAT permite que las instancias en subredes privadas (como las de aplicación y base de datos) se conecten a internet para actualizaciones o para acceder a servicios externos, pero impide que internet inicie una conexión con esas instancias, lo que mejora la seguridad. Cada Gateway NAT utiliza una de las IP elásticas reservadas.

**Subredes (`Subnets`)**

```
  PublicSubnet1:
    Type: 'AWS::EC2::Subnet'
    Properties:
      VpcId: !Ref LabVPC
      CidrBlock: !Ref PublicSubnet1Param
      MapPublicIpOnLaunch: True
      AvailabilityZone: !Select [ '0', !GetAZs '' ]
      # ...
```

* **`PublicSubnet1` y `PublicSubnet2`**: Se crean dos subredes públicas. La propiedad `MapPublicIpOnLaunch: True` significa que cualquier instancia EC2 lanzada en estas subredes recibirá automáticamente una dirección IP pública. Cada subred se coloca en una Zona de Disponibilidad (AZ) diferente (`!Select [ '0', ...]` y `!Select [ '1', ...]`) para lograr alta disponibilidad.
* **`AppSubnet1`, `AppSubnet2`**, **`DatabaseSubnet1` y `DatabaseSubnet2`**: Se crean cuatro subredes privadas. `MapPublicIpOnLaunch: False` asegura que las instancias en estas subredes no sean accesibles directamente desde internet. También están distribuidas en dos Zonas de Disponibilidad para resiliencia.

**Enrutamiento (`Routing`)**

Esta sección configura cómo se dirige el tráfico dentro de la VPC.

```
  PublicRouteTable:
    Type: 'AWS::EC2::RouteTable'
    Properties:
      VpcId: !Ref LabVPC
  # ...
  PublicRoute:
    Type: 'AWS::EC2::Route'
    Properties:
      RouteTableId: !Ref PublicRouteTable
      DestinationCidrBlock: 0.0.0.0/0
      GatewayId: !Ref LabInternetGateway
  # ...
  PublicSubnet1RouteTableAssociation:
    Type: 'AWS::EC2::SubnetRouteTableAssociation'
    Properties:
      SubnetId: !Ref PublicSubnet1
      RouteTableId: !Ref PublicRouteTable
```

* **Tablas de rutas**: Se crean tres tablas de rutas: una pública (`PublicRouteTable`) y dos privadas (`PrivateRouteTableAZ1`, `PrivateRouteTableAZ2`), una para cada Zona de Disponibilidad.
* **Rutas**:
  * **`PublicRoute`**: Se añade una ruta a la tabla pública. `DestinationCidrBlock: 0.0.0.0/0` significa "todo el tráfico de internet". Esta ruta dirige ese tráfico al `LabInternetGateway`, permitiendo el acceso a internet.
  * **`PrivateRouteAZ1` y `PrivateRouteAZ2`**: Se añaden rutas a las tablas privadas. También dirigen el tráfico de internet (`0.0.0.0/0`), pero en este caso lo envían a los `NATGateway` correspondientes. Así es como las instancias en subredes privadas pueden acceder a internet.
* **Asociaciones de subredes**: Cada subred se asocia con su tabla de rutas correspondiente. Las subredes públicas se asocian con la tabla de rutas pública, y las subredes de aplicación y base de datos se asocian con las tablas de rutas privadas de su respectiva Zona de Disponibilidad.

**Grupos de seguridad (`Security Groups`)**

Los grupos de seguridad actúan como un firewall virtual para controlar el tráfico de entrada y salida de los recursos.

```
  AppInstanceSecurityGroup:
    Type: 'AWS::EC2::SecurityGroup'
    Properties:
      # ...
      SecurityGroupIngress:
        - IpProtocol: tcp
          FromPort: 80
          ToPort: 80
          CidrIp: 0.0.0.0/0
```

* **`AppInstanceSecurityGroup`**: Permite el tráfico entrante en el puerto 80 (HTTP) desde cualquier lugar (`0.0.0.0/0`). Este grupo sería para servidores web.
* **`RDSSecurityGroup`**: Creado para las instancias de RDS. Notablemente, no tiene reglas de entrada (`Ingress`) definidas aquí. Se espera que se añadan reglas más adelante, probablemente para permitir conexiones solo desde el grupo de seguridad de la aplicación.
* **`EFSMountTargetSecurityGroup`**: Permite el tráfico en el puerto 80 desde instancias que estén en `AppInstanceSecurityGroup`. Esto permitiría que las instancias de la aplicación se conecten a un sistema de archivos EFS (Elastic File System). Aunque el puerto 80 es para HTTP y EFS usa NFS (puerto 2049), el ejemplo usa el puerto 80. En un escenario real, se usaría el puerto correcto para el servicio.

**Salidas (`Outputs`)**

La sección `Outputs` declara valores que deseas que estén disponibles después de que la pila se haya creado. Estos pueden ser utilizados por otras pilas o para que los consultes fácilmente.[aws.amazon](https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/template-anatomy.html)

```
Outputs:
  VPCID:
    Description: "The VPC ID for the lab"
    Value: !Ref LabVPC
    Export:
      Name: "VPCID"
```

* Esta sección exporta los IDs de varios recursos clave como la VPC, las subredes de la base de datos y los grupos de seguridad.
* La palabra clave **`Export`** hace que estos valores estén disponibles para ser importados por otras plantillas de CloudFormation en la misma cuenta y región, lo cual es crucial para construir arquitecturas modulares. Por ejemplo, una plantilla diferente podría importar el `VPCID` para lanzar servidores dentro de esta VPC.

***

**Servicio Principal: Amazon EC2 (Elastic Compute Cloud)**

Aunque es conocido por las máquinas virtuales, EC2 es el servicio que gestiona la mayor parte de la infraestructura de red virtual.

* **Sub-servicio: Virtual Private Cloud (VPC)**
  * **Recurso en plantilla**: `AWS::EC2::VPC`
  * **Función**: Es el servicio fundamental que crea una **red virtual aislada** en la nube de AWS. Actúa como el contenedor principal para todos los demás recursos de red, como subredes e instancias. En la plantilla, se crea una VPC llamada `LabVPC`.
* **Sub-servicio: Internet Gateway (IGW)**
  * **Recurso en plantilla**: `AWS::EC2::InternetGateway`
  * **Función**: Componente que permite la comunicación bidireccional entre los recursos en tus subredes públicas y el internet. Sin un IGW, tu VPC estaría completamente aislada del exterior.
* **Sub-servicio: Elastic IP (EIP)**
  * **Recurso en plantilla**: `AWS::EC2::EIP`
  * **Función**: Proporciona una **dirección IPv4 pública y estática** (fija) que puedes asignar a un recurso. Tu intuición es correcta sobre que proporciona una IP única. Sin embargo, en esta plantilla su uso es muy específico: **no se asigna directamente a instancias, sino a los Gateways NAT**. Esto es crucial para que los Gateways NAT tengan un punto de salida a internet fijo y predecible.
* **Sub-servicio: NAT Gateway**
  * **Recurso en plantilla**: `AWS::EC2::NatGateway`
  * **Función**: Permite que las instancias en una **subred privada** (como las de aplicación o base de datos) inicien conexiones hacia internet (por ejemplo, para descargar actualizaciones), pero impide que internet inicie conexiones hacia esas instancias. Actúa como un intermediario seguro para el tráfico saliente.
* **Sub-servicio: Subnet**
  * **Recurso en plantilla**: `AWS::EC2::Subnet`
  * **Función**: Divide el rango de direcciones IP de tu VPC en segmentos más pequeños. Esto te permite organizar los recursos en grupos lógicos (ej. subredes públicas para servidores web, privadas para bases de datos) y aplicarles diferentes reglas de seguridad y enrutamiento.
* **Sub-servicio: Route Table (Tabla de Rutas)**
  * **Recurso en plantilla**: `AWS::EC2::RouteTable`
  * **Función**: Contiene un conjunto de reglas, llamadas **rutas**, que determinan a dónde se dirige el tráfico de red que sale de una subred. Cada subred debe estar asociada a una tabla de rutas.
* **Sub-servicio: Route (Ruta)**
  * **Recurso en plantilla**: `AWS::EC2::Route`
  * **Función**: Es una **regla específica dentro de una tabla de rutas**. Por ejemplo, la plantilla crea una ruta `0.0.0.0/0` (todo el tráfico de internet) que en las subredes públicas apunta al Internet Gateway y en las privadas apunta al NAT Gateway.
* **Sub-servicio: Security Group (Grupo de Seguridad)**
  * **Recurso en plantilla**: `AWS::EC2::SecurityGroup`
  * **Función**: Actúa como un **firewall virtual a nivel de instancia** que controla el tráfico de entrada y salida. Son "stateful", lo que significa que si permites una conexión de entrada, la respuesta de salida se permite automáticamente.
* **Acciones de Asociación (No son servicios, sino configuraciones de enlace)**
  * `AWS::EC2::VPCGatewayAttachment`: Conecta el Internet Gateway a la VPC.
  * `AWS::EC2::SubnetRouteTableAssociation`: Asocia una subred a una tabla de rutas.

**Servicio de Orquestación: AWS CloudFormation**

Este es el servicio que lee tu plantilla y se encarga de crear, configurar y conectar todos los recursos mencionados anteriormente de manera automática. No es un recurso _dentro_ de la plantilla, sino el motor que la ejecuta. Las funcionalidades clave que usa son:

* **Parameters**: Permite personalizar la plantilla en cada despliegue (ej. cambiando los rangos de IP).
* **Resources**: La sección principal donde se declaran todos los componentes a crear.
* **Outputs**: Permite exportar los identificadores (IDs) de los recursos creados para que puedan ser usados por otras plantillas o consultados fácilmente.
* **Funciones Intrínsecas**: Pequeñas piezas de código como `!Ref` (para referenciar otro recurso) o `!GetAtt` (para obtener un atributo de un recurso) que hacen que la plantilla sea dinámica y cohesiva.

</details>

<details>

<summary><strong>Revisar y poner en marcha una plantilla preconfigurada en Terraform</strong></summary>

<figure><img src="../../../../../.gitbook/assets/Construya una arquitectura multinivel de AWS - CloudFormation 1.png" alt="Despliegue de VPC multinivel en AWS con Terraform (infraestructura como código)"><figcaption></figcaption></figure>

**Despliegue de infraestructura WordPress con Terraform en AWS**

Se despliega una infraestructura de red robusta y escalable para aplicaciones WordPress utilizando **Terraform** como herramienta de Infraestructura como Código (IaC). Esta configuración crea automáticamente una VPC (Virtual Private Cloud) de alta disponibilidad que abarca múltiples Zonas de Disponibilidad para garantizar la resiliencia y el rendimiento óptimo de la aplicación.

**Situación del proyecto**

El equipo de DevOps ha definido los requisitos para una infraestructura base que soporte aplicaciones web de alto tráfico como WordPress. Se necesita crear una Virtual Private Cloud con un diseño multinivel que separe los recursos en diferentes capas de seguridad, siguiendo las mejores prácticas de arquitectura en la nube.

**Arquitectura de red requerida:**

* **Dos subredes públicas** distribuidas en diferentes Zonas de Disponibilidad
* **Dos subredes de aplicación privadas** para alojar los servidores web
* **Dos subredes de base de datos privadas** para máxima seguridad de los datos
* **Gateway de Internet (IGW)** para conectividad externa desde subredes públicas
* **Gateways NAT** para permitir actualizaciones seguras desde subredes privadas
* **Tablas de enrutamiento** configuradas específicamente para cada capa
* **Direcciones IP Elásticas** para puntos de salida estáticos y confiables

**Pasos para el despliegue con Terraform**

**1. Preparación del entorno**

* Descargar y revisar los archivos de configuración Terraform (`main.tf`, `outputs.tf`)
* Verificar que Terraform esté instalado y configurado
* Configurar las credenciales de AWS CLI o variables de entorno

**2. Inicialización y configuración**

```
bash# Inicializar el directorio de trabajo de Terraform
terraform init

# Configurar variables personalizadas (opcional)
# Los valores por defecto están optimizados para WordPress
terraform plan -var="region=us-west-2"
```

**3. Personalización de parámetros**

* **Región de AWS**: Seleccionar la región más cercana a los usuarios
* **Rangos de IP**: Los valores por defecto (`10.0.0.0/16`) son adecuados para la mayoría de casos
* **Nombre de recursos**: Mantener la nomenclatura estándar para facilitar la gestión

**4. Despliegue de la infraestructura**

```
bash# Aplicar la configuración
terraform apply

# Confirmar la creación cuando se solicite
# Terraform creará todos los recursos automáticamente
```

**5. Verificación del despliegue**

* Acceder a la consola de AWS VPC para verificar la creación de componentes
* Confirmar que todas las subredes estén en diferentes Zonas de Disponibilidad
* Validar las tablas de enrutamiento y sus asociaciones

***

**¿Qué es Terraform y por qué se utiliza para WordPress?**

**Terraform** es una herramienta de Infraestructura como Código (IaC) desarrollada por HashiCorp que permite definir y aprovisionar infraestructura de manera declarativa. En lugar de configurar manualmente cada componente en la consola de AWS, se describe el estado final deseado en archivos de configuración y Terraform se encarga de crear y gestionar todos los recursos de forma automática y ordenada.

**Ventajas específicas para WordPress:**

* **Reproducibilidad**: Garantiza que el entorno de WordPress se despliegue de manera idéntica en desarrollo, staging y producción.
* **Escalabilidad**: Facilita la expansión de la infraestructura cuando el tráfico de WordPress crece
* **Control de versiones**: Los archivos de configuración pueden almacenarse en Git, permitiendo rastrear cambios y colaborar en equipo
* **Automatización completa**: Elimina errores manuales y reduce el tiempo de aprovisionamiento de horas a minutos
* **Gestión de estado**: Terraform mantiene un registro del estado actual de la infraestructura, facilitando actualizaciones y modificaciones

**Arquitectura de VPC multinivel para WordPress**

**Capa pública: la puerta de entrada**

Las **subredes públicas** actúan como la primera línea de la arquitectura. Aquí se ubicarían componentes como:

* Application Load Balancers (ALB) para distribución de tráfico
* Instancias de bastión para acceso administrativo seguro
* Gateways NAT para conectividad de salida

**Beneficio clave**: Estas subredes tienen rutas directas al Internet Gateway, permitiendo comunicación bidireccional con internet mientras actúan como un filtro controlado para el tráfico entrante.

**Capa de aplicación: el núcleo de WordPress**

Las **subredes privadas de aplicación** alojan los servidores web que ejecutan WordPress. Esta separación proporciona múltiples beneficios de seguridad:

* Los servidores no son accesibles directamente desde internet
* Solo reciben tráfico filtrado a través del balanceador de carga
* Mantienen la capacidad de actualizarse a través de las Gateways NAT

**Capa de base de datos: máxima protección**

Las **subredes de base de datos** representan el nivel más seguro de la arquitectura, diseñadas específicamente para alojar:

* Instancias RDS MySQL/MariaDB para WordPress
* Sistemas de caché como ElastiCache
* Almacenamiento compartido EFS para archivos multimedia

**Principio de defensa en profundidad**: Esta arquitectura de tres capas crea múltiples barreras de seguridad. Los atacantes tendrían que comprometer varios niveles antes de acceder a los datos sensibles.

**El rol crítico de las gateways NAT**

**Problema a resolver**

Las instancias de WordPress en subredes privadas necesitan acceso a internet para:

* Descargar actualizaciones de seguridad del núcleo de WordPress
* Instalar y actualizar plugins desde el repositorio oficial
* Conectarse a APIs de terceros (servicios de pago, analytics, etc.)
* Sincronizar con CDNs como CloudFront para contenido estático

**Solución segura**

Las **NAT Gateways** se despliegan en las subredes públicas con direcciones IP elásticas. Las tablas de enrutamiento de las subredes privadas dirigen todo el tráfico hacia internet a través de estas gateways, creando un túnel seguro de salida.[dev](https://dev.to/francotel/despliega-un-wordpress-de-alta-disponibilidad-en-aws-con-terraform-en-30-minutos-guia-paso-a-paso-4dih)

**Ventaja de seguridad**: Las instancias privadas pueden iniciar conexiones hacia internet, pero internet no puede iniciar conexiones hacia ellas, protegiendo efectivamente contra ataques directos.

**Diferencias fundamentales: Internet Gateway vs NAT Gateway**

**Internet Gateway (IGW)**

* **Propósito**: Permite comunicación **bidireccional** entre la VPC e internet
* **Ubicación**: Se asocia a nivel de VPC completa
* **Uso**: Empleado en tablas de enrutamiento de subredes públicas
* **Función**: Portal principal para que el tráfico de internet llegue a los recursos públicos

**NAT Gateway**

* **Propósito**: Permite comunicación **unidireccional** (solo salida) desde subredes privadas
* **Ubicación**: Se despliega en subredes públicas específicas
* **Uso**: Referenciado en tablas de enrutamiento de subredes privadas
* **Función**: "Traductor" seguro que permite actualizaciones sin exponer recursos internos

**Ventajas de Terraform sobre métodos tradicionales**

**Gestión declarativa vs imperativa**

Con Terraform, describes **qué** quieres (el estado final), no **cómo** crearlo. Esto significa que puedes modificar la configuración y Terraform determinará automáticamente qué cambios aplicar para alcanzar el nuevo estado deseado.

**Plan de ejecución transparente**

Antes de aplicar cambios, `terraform plan` muestra exactamente qué recursos se crearán, modificarán o eliminarán, proporcionando completa visibilidad sobre el impacto de los cambios.

**Gestión de dependencias automática**

Terraform analiza las dependencias entre recursos y los crea en el orden correcto. Por ejemplo, creará las subredes antes de las NAT Gateways que dependen de ellas, sin requerir configuración manual de secuencias.

Esta infraestructura Terraform proporciona una base sólida y escalable para cualquier aplicación WordPress, desde blogs personales hasta sitios de comercio electrónico de alto tráfico, garantizando seguridad, disponibilidad y rendimiento desde el primer día.

</details>
