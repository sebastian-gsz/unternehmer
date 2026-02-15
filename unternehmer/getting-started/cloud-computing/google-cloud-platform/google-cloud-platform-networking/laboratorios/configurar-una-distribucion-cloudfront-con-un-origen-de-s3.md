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

# Configurar una distribución CloudFront con un origen de S3

<h2 align="center">Configurar una distribución CloudFront con un origen de S3</h2>

{% embed url="https://soundcloud.com/spectrae/rorganic-big-jet-plane-edit?si=2b40ea556d5a4c77a85a83bb1aa21da0&utm_campaign=social_sharing&utm_medium=text&utm_source=clipboard" %}

<figure><img src="../../../../../.gitbook/assets/Configurar una distribución CloudFront con un origen de S3.png" alt=""><figcaption></figcaption></figure>

Te guiare a través del proceso de configurar una distribución de **Amazon CloudFront** para entregar contenido de forma segura y eficiente desde un bucket de **Amazon S3**. El objetivo principal es aprender a integrar un bucket de S3 como origen de una Red de Entrega de Contenido (CDN) y, fundamentalmente, a protegerlo mediante el **Control de Acceso de Origen (OAC)**.

Esta configuración garantiza que el contenido del bucket solo sea accesible a través de CloudFront, bloqueando el acceso directo y mejorando tanto la seguridad como el rendimiento de la entrega de contenido.<br>

* **Crear un bucket de S3**: Creación de un bucket en Amazon S3 que servirá como origen para el contenido.
* **Configurar bucket para acceso público**: Modificación de la configuración de seguridad del bucket de S3 para permitir el acceso público a su contenido.
* **Añadir origen a CloudFront**: Integración del bucket de S3 como un nuevo origen en una distribución de CloudFront existente.
* **Asegurar bucket con OAC**: Implementación del Control de Acceso de Origen (OAC) para restringir el acceso al bucket de S3, permitiéndolo únicamente a través de la distribución de CloudFront.
* **Configurar políticas de recursos de S3**: Definición de políticas de bucket en S3 para gestionar el acceso, ya sea público o restringido a través de OAC.&#x20;

<figure><img src="../../../../../.gitbook/assets/Barra.gif" alt=""><figcaption></figcaption></figure>

<details>

<summary><strong>Explorar la distribución actual de CloudFront</strong></summary>

* En la barra de búsqueda de la consola de AWS, busca y selecciona **CloudFront**.
* Selecciona el enlace de ID para la única distribución disponible. Examina los contenidos de la pestaña **General**.
* Copia el valor del **ARN** y el **nombre del dominio de distribución** para usarlos más adelante.
  * **XXXH3EPXXX9J5**
  * **XXXXa5i594XXXq.cloudfront.net**
* Elige la pestaña **Security**. Contiene configuraciones para AWS WAF y restricciones geográficas, que no se usarán en este laboratorio.
* Selecciona la pestaña **Origins**. Muestra los orígenes de contenido de la distribución.
  * _Nota_: El origen actual es un Application Load Balancer (ALB).
  * Copia el valor del **Origin domain** (`LabELB-172999XXXX.us-west-X.elb.amazonaws.com`) y pégalo en una nueva pestaña del navegador. Verás la misma página web, pero servida directamente desde las instancias EC2 detrás del balanceador, sin pasar por la caché de CloudFront.

<figure><img src="../../../../../.gitbook/assets/Configurar distribución CloudFront con origen S3 - 1.gif" alt=""><figcaption></figcaption></figure>

* Selecciona la pestaña **Behaviors**. Define cómo CloudFront maneja las solicitudes de contenido. El comportamiento actual acepta solicitudes HTTP y HTTPS para el origen del ALB.
* Selecciona la pestaña e**rror pages**. Aquí se configuran las páginas de error personalizadas para códigos de estado HTTP 4xx y 5xx.
* Selecciona la pestaña **Invalidations**. Permite eliminar objetos de las cachés perimetrales de CloudFront.
* Elige la pestaña **Tags**. Muestra las etiquetas aplicadas a la distribución para su organización.

</details>

<details>

<summary><strong>Crear un bucket de S3</strong></summary>

* En la barra de búsqueda de la consola, busca y selecciona S3.
* En la sección **Buckets**, elige **Create bucket**.
* Configura lo siguiente:
  * **Bucket name**: Pega el valor de `lab-bucket-552447749` proporcionado en las instrucciones del laboratorio.
* Deja el resto de las configuraciones con sus valores predeterminados y selecciona **Create bucket**.

</details>

<details>

<summary><strong>Configura acceso público para el Bucket S3</strong></summary>

En este paso, revisarás la configuración de acceso predeterminada para los buckets de S3 y la modificarás para permitir el acceso público.

* Selecciona el enlace del **Bucket** recién creado.
* Ve a la pestaña **Permissions**.
* En la sección **Block public access (bucket settings)**, selecciona **Edit**.
* Desmarca la casilla **Block all public access**.
* Guarda los cambios, escribe `confirm` en el cuadro de diálogo de confirmación y selecciona **Confirm**.

\
**Configurar una política de lectura pública para el Bucket**<br>

* En la misma pestaña **Permissions**, localiza la sección **Bucket policy** y selecciona **Edit**.
* Copia el **Bucket ARN** que aparece encima del editor de políticas. Lo necesitarás para el siguiente paso.
*   Prepara la siguiente política JSON. Reemplaza `RESOURCE_ARN` con el ARN de tu bucket y añade `/*` al final para que la política se aplique a todos los objetos dentro del bucket.<br>

    ```
    json{
        "Version": "2012-10-17",
        "Id": "Policy1621958846486",
        "Statement": [
            {
                "Sid": "OriginalPublicReadPolicy",
                "Effect": "Allow",
                "Principal": "*",
                "Action": [
                    "s3:GetObject",
                    "s3:GetObjectVersion"
                ],
                "Resource": "RESOURCE_ARN/*"
            }
        ]
    }
    ```
* Pega la política JSON actualizada en el editor de políticas de S3.
* Selecciona **Save changes**.

\
**Cargar un objeto en el bucket y probar el acceso público**<br>

Ahora, cargarás un objeto al bucket para verificar que la configuración de acceso público funciona correctamente.

* **Crear una nueva carpeta en el bucket**
  1. Ve a la pestaña **Objects** de tu bucket.
  2. Selecciona **Create folder**.
  3. Nombra la carpeta `CachedObjects` y selecciona **Create folder**.
* **Cargar un objeto en el bucket**
  * Descarga el archivo `logo.png` proporcionado por el laboratorio en tu dispositivo.
  * Dentro de la carpeta **CachedObjects** en la consola de S3, selecciona **Upload**.
  * Elige **Add files**, selecciona el archivo `logo.png` que descargaste y haz clic en **Upload**.
* **Probar el acceso público a un objeto**
  * Una vez completada la carga, selecciona el archivo `logo.png` dentro de la carpeta.
  * En la página de detalles del objeto, copia la **Object URL**.
  * Pega la URL en una nueva pestaña del navegador. La imagen del logo debería mostrarse correctamente. Esto confirma que el objeto es públicamente accesible a través de su URL de S3 directa.

</details>

<details>

<summary><strong>Arquitectura Terraform o CloudFormation</strong></summary>



</details>

<figure><img src="../../../../../.gitbook/assets/Barra.gif" alt=""><figcaption></figcaption></figure>

{% stepper %}
{% step %}
**¿Qué es Amazon CloudFront y por qué se utiliza?**

Amazon CloudFront es una **Red de Entrega de Contenido (CDN)** global. Su función principal es acelerar la entrega de contenido web (estático y dinámico) a los usuarios finales.

* **Funcionamiento**: CloudFront almacena en caché copias de tu contenido en una red mundial de servidores perimetrales (Edge Locations). Cuando un usuario solicita tu contenido, la petición se dirige automáticamente a la ubicación perimetral más cercana, lo que reduce la latencia y mejora los tiempos de carga.
* **Origen**: El contenido original se mantiene en un "origen", que en este laboratorio es un bucket de S3 y un Application Load Balancer. CloudFront recupera el contenido del origen la primera vez que se solicita y lo guarda en caché por un tiempo determinado (TTL).
{% endstep %}

{% step %}
**¿Qué es un bucket de S3?**

Amazon S3 (Simple Storage Service) es un servicio de almacenamiento de objetos altamente escalable, duradero y seguro.

* **Almacenamiento de objetos**: A diferencia del almacenamiento en bloques (como los discos duros), S3 almacena datos como objetos, que consisten en el propio archivo y sus metadatos. Esto lo hace ideal para almacenar archivos web, copias de seguridad, datos de aplicaciones y big data.
* **Seguridad**: Por defecto, los buckets de S3 son privados. Para que el contenido sea accesible públicamente o a través de otros servicios de AWS como CloudFront, es necesario configurar políticas de bucket y permisos de acceso.
{% endstep %}

{% step %}
**¿Por qué asegurar el bucket de S3 con OAC?**

El **Control de Acceso de Origen (OAC)** es un mecanismo de seguridad mejorado de CloudFront que restringe el acceso a un bucket de S3 para que solo pueda ser accedido a través de una distribución de CloudFront específica.

* **Seguridad mejorada**: Al usar OAC, evitas que los usuarios accedan directamente a los archivos en tu bucket de S3 utilizando la URL del bucket. Todo el tráfico debe pasar por CloudFront, lo que te permite aplicar reglas de seguridad adicionales como AWS WAF, gestionar el acceso y registrar las solicitudes de manera centralizada.
* **Diferencia con OAI**: OAC es el sucesor recomendado de la Identidad de Acceso de Origen (OAI). OAC ofrece una seguridad más robusta y granular, soportando métodos HTTP adicionales y cifrado del lado del servidor con KMS.
{% endstep %}

{% step %}
**¿Por qué al principio se puede acceder al contenido tanto por la URL de CloudFront como por la del Application Load Balancer?**

Esto se debe a que el ALB está configurado como un origen público accesible desde Internet. CloudFront simplemente actúa como una capa de caché frente a él. La configuración inicial no restringe el acceso directo al ALB. El objetivo de usar CloudFront en este escenario es mejorar el rendimiento y, posteriormente con S3, centralizar y asegurar el acceso.
{% endstep %}

{% step %}
**¿Qué pasaría si no se configurara el OAC en el bucket de S3?**

Si el bucket se dejara con acceso público, cualquiera con la URL directa a un objeto podría acceder a él, saltándose la distribución de CloudFront. Esto no solo expone el contenido directamente, sino que también anula los beneficios de usar CloudFront, como el almacenamiento en caché, la reducción de costes de transferencia de datos desde S3 y las capas de seguridad adicionales que ofrece la CDN.
{% endstep %}

{% step %}
**¿Por qué es necesario desactivar "Block public access"?**

La configuración **"Block public access"** es una medida de seguridad a nivel de cuenta y de bucket que AWS habilita por defecto para prevenir la exposición accidental de datos. Al desactivarla, permites explícitamente la creación de políticas de bucket que pueden otorgar acceso público. Es un primer paso fundamental para cualquier escenario que requiera acceso público, como hospedar un sitio web estático o, en este caso, configurar un acceso público temporal antes de asegurarlo con CloudFront.
{% endstep %}

{% step %}
**¿Qué significa `Principal: "*"` en la política de bucket?**

El elemento `Principal` en una política de IAM define a quién se aplica el permiso.

* **`"Principal": "*"`**: Usar un asterisco (`*`) como principal significa "cualquier persona" o "anónimo". En el contexto de una política de bucket de S3, esto otorga permiso a cualquier usuario en Internet para realizar las acciones especificadas (`s3:GetObject` y `s3:GetObjectVersion`) sobre los recursos definidos (`Resource`).
* **`Resource: "arn:aws:s3:::lab-bucket-1234/*"`**: El `/*` al final del ARN del bucket es un comodín que hace que la política se aplique a todos los objetos (archivos) dentro del bucket, pero no al bucket en sí.

La combinación de un principal universal y un recurso que abarca todos los objetos es lo que hace que el contenido del bucket sea de lectura pública.
{% endstep %}

{% step %}
**¿Es seguro mantener un bucket de S3 con acceso público?**

Generalmente, **no es una práctica recomendada** mantener un bucket de S3 completamente público, especialmente si contiene datos sensibles. En este laboratorio, se hace de forma temporal para demostrar el funcionamiento del acceso directo y contrastarlo con el acceso restringido a través de CloudFront que se configurará más adelante. La mejor práctica es siempre seguir el principio de mínimo privilegio, otorgando acceso solo a las entidades que lo necesitan y a través de canales seguros como CloudFront con OAC.
{% endstep %}

{% step %}
**¿Cuál es la diferencia entre `s3:GetObject` y `s3:GetObjectVersion`?**

* **`s3:GetObject`**: Esta acción permite a un principal descargar un objeto de un bucket de S3.
* **`s3:GetObjectVersion`**: Si el versionado está habilitado en el bucket, esta acción permite descargar una versión específica de un objeto. Incluir ambas acciones asegura que el acceso de lectura funcione correctamente independientemente de si el versionado está activo o no.
{% endstep %}
{% endstepper %}
