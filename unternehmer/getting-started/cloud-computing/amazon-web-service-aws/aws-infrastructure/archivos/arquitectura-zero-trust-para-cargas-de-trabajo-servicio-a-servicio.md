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

# Arquitectura zero trust para cargas de trabajo servicio a servicio

<h2 align="center">Arquitectura zero trust para cargas de trabajo servicio a servicio</h2>

{% embed url="https://soundcloud.com/lilly_palmer/hype-boy?si=15f001c4b7d047d48d851b0562d1fa0d&utm_campaign=social_sharing&utm_medium=text&utm_source=clipboard" %}

<figure><img src="../../../../../.gitbook/assets/Arquitectura zero trust para cargas de trabajo servicio a servicio.png" alt=""><figcaption></figcaption></figure>

La imagen describe una arquitectura **servicio a servicio** con dos componentes:

{% stepper %}
{% step %}
**Servicios de llamada**

* Flota de instancias **EC2** y funciones **AWS Lambda** que hacen llamadas API.
* Estos nodos están distribuidos en **varias subredes y VPC**.
* Acceden a las APIs privadas del otro componente mediante **VPC Endpoints** usando **AWS PrivateLink**.
{% endstep %}

{% step %}
**Servicios de destino**

* Varias APIs REST privadas en **API Gateway**.
* Backend con **Lambda** y **DynamoDB**.
* No expuestas públicamente, solo accesibles desde VPCs autorizadas.
{% endstep %}
{% endstepper %}

**Relación con Zero Trust**:

* Se parte de un modelo que no asume que la red interna es segura.
* Se integran controles de identidad (credenciales, IAM) y controles de red (CIDR, VPC Endpoint) para cada solicitud.
* Cada llamada API debe ser autenticada y autorizada de forma individual (ej. **AWS Signature v4**).
* Se eliminan rutas innecesarias y se usan gateways como puntos de control y monitoreo.
* Se aplican niveles de seguridad según el valor de los recursos, no de forma uniforme y excesiva en un solo punto.

<img src="../../../../../.gitbook/assets/file.excalidraw (1).svg" alt="" class="gitbook-drawing">

**Situación real del laboratorio:**

* **Las APIs REST son privadas**
  * No tienen endpoint público en Internet.
  * Solo son accesibles **desde dentro de una VPC autorizada**.
* **Cómo se conectan los servicios de llamada a las APIs privadas**
  * Se crea un **VPC Endpoint de tipo interface** para API Gateway en las subredes donde están las instancias EC2 y Lambdas del _servicio de llamada_.
  * Ese endpoint usa **AWS PrivateLink** para dar una ruta interna y segura hacia API Gateway.
  * De este modo, la llamada no pasa por Internet, sino por la red interna de AWS.
* **Escenario simplificado en el laboratorio**
  * Aunque en la realidad podría haber **múltiples VPC, subredes y APIs**, aquí se asume que _servicio de llamada_ y _servicio de destino_ están en la **misma cuenta AWS** para simplificar la configuración.
  * Si estuvieran en **cuentas diferentes**, los conceptos de control de acceso, políticas de endpoint y autenticación serían los mismos, pero las condiciones IAM tendrían que incluir la **cuenta origen/destino**.

<figure><img src="../../../../../.gitbook/assets/Arquitectura zero trust para cargas de trabajo servicio a servicio - Realidad.png" alt=""><figcaption></figcaption></figure>

Esta parte describe el **escenario realista** que vas a analizar en el laboratorio y la dificultad principal de aplicar el modelo **Zero Trust**.

***

#### **Servicios involucrados**

1. **ServicioA** (Recomendaciones)
   * Genera recomendaciones basadas en datos históricos.
   * Nodo legítimo: **EC2 Intermediario esperado**.
   * Este EC2 obtiene los datos llamando a **ServicioB**.
2. **ServicioB** (Historial de pedidos)
   * Expone API privada en **API Gateway**.
   * Backend con **Lambda** + **DynamoDB** para almacenar pedidos.

***

#### **Otros actores en el entorno**

* **APIs desconocidas**: no forman parte del flujo esperado.
* **Intermediarios no deseados**: funciones Lambda o EC2 que, aunque están en la red, **no deberían acceder a ServicioB**.

***

#### **Controles de seguridad posibles**

Cada llamada API puede evaluarse con estas condiciones:

1. **Necesidad funcional**: ¿Ese intermediario realmente debe llamar al destino?
2. **Ruta de red válida**: ¿La llamada pasa por el VPC Endpoint correcto?
3. **Rango de IP válido (CIDR)**: ¿La IP de origen está autorizada?
4. **Permisos IAM correctos**: ¿El rol/tokens de IAM tienen privilegios para invocar la API?
5. **Firma SigV4**: ¿La llamada está firmada con credenciales IAM válidas?

***

#### **Tabla de intermediarios** <a href="#tabla-interactiva" id="tabla-interactiva"></a>

<table data-full-width="false"><thead><tr><th>Intermediario</th><th>Tipo</th><th>API destino</th><th>Ruta</th><th>CIDR</th><th>SigV4</th><th>IAM</th><th>Resultado deseado</th></tr></thead><tbody><tr><td>Esperado</td><td>EC2</td><td>ServicioB</td><td>VPCE SrvA</td><td>10.199.0.0/24</td><td>Sí</td><td>Sí</td><td>Permitir</td></tr><tr><td>Esperado</td><td>EC2</td><td>Desconocida</td><td>VPCE SrvA</td><td>10.199.0.0/24</td><td>Sí</td><td>Sí</td><td>Bloquear</td></tr><tr><td>No esperado #1</td><td>Lambda</td><td>ServicioB</td><td>VPCE SrvA</td><td>10.199.1.0/24</td><td>No</td><td>No</td><td>Bloquear</td></tr><tr><td>No esperado #2</td><td>Lambda</td><td>ServicioB</td><td>VPCE SrvA</td><td>10.199.0.0/24</td><td>No</td><td>No</td><td>Bloquear</td></tr><tr><td>No esperado #3</td><td>Lambda</td><td>ServicioB</td><td>VPCE SrvA</td><td>10.199.0.0/24</td><td>Sí</td><td>No</td><td>Bloquear</td></tr><tr><td>No esperado #4</td><td>EC2</td><td>ServicioB</td><td>Otro VPCE</td><td>10.199.0.0/24</td><td>Sí</td><td>Sí</td><td>Bloquear</td></tr></tbody></table>

***

#### **Problema central**

* **Confusión por coincidencias**:\
  Algunos intermediarios no deseados comparten **CIDR**, **ruta** o incluso **permisos** con el intermediario legítimo.
* **Reto**:\
  Diseñar un conjunto de controles (red + identidad) que permitan solo el tráfico legítimo **sin falsos positivos ni falsos negativos**.

***

Si quieres, puedo dibujarte **un diagrama de la arquitectura detallada** con cada intermediario y flechas de permitido/bloqueado para que sea más fácil entender **quién accede y quién no** antes de empezar a aplicar las tareas de endurecimiento. ¿Quieres que lo haga?

**En resumen**: el laboratorio simula un entorno de microservicios internos comunicándose solo por rutas privadas usando PrivateLink, sin exposición pública, y sobre ese escenario vas a aplicar las reglas de **Zero Trust** para filtrar quién realmente puede llamar a la API.





<details>

<summary>Revisar controles de seguridad existentes</summary>



</details>

**Componentes clave revisados:**

1. **API Gateway**
   * **Autorización**: configurada como `NONE` → permite cualquier llamada sin credenciales IAM ni autenticación SigV4.
   * **Política de recursos**: permite cualquier IP dentro de `10.199.0.0/24` → coincide con varias subredes, incluyendo intermediarios no deseados.
   * **Tipo de endpoint**: privado, accesible solo desde un VPC Endpoint.
2. **VPC Endpoint (API Gateway)**
   * **Política de recursos**: sin restricciones → permite tráfico a cualquier API privada en la misma región.
   * **Grupo de seguridad**: reglas amplias → acepta todo el tráfico HTTPS desde `10.199.0.0/16`.

**Explicación técnica:**\
Los controles actuales dependen de la red (IP y VPC Endpoint), lo que es contrario a los principios de Zero Trust. No hay validación de identidad ni control granular de recursos.

<details>

<summary><strong>Verificación de configuración de API Gateway</strong></summary>

1. Buscar el servicio **API Gateway**.
2. En la lista de API, seleccionar el enlace **`ServiceBAPI`**.
3. En el panel **Resources** (Recursos), seleccionar `/orders` y luego `GET`.
4. En la sección `/orders-GET-Method Execution`, seleccionar la pestaña **Method request** (Solicitud de método).
5. Verificar que la opción <kbd>**Authorization**</kbd> (Autorización) esté configurada como `NONE` (Ninguno).

{% hint style="danger" %}
En API Gateway, que la **autorización** de un método esté en `NONE` significa:

* **No se requiere autenticación**.
* La API no verificará credenciales de IAM.
* No se usará AWS Signature v4 (SigV4) para firmar la solicitud.
* Cualquier cliente con conectividad a ese endpoint podrá invocar el método, sin importar identidad o permisos.

En este laboratorio, eso implica:

* Mientras el cliente esté dentro del rango de IP permitido por la política de recursos, su llamada pasará.
* No hay validación de quién realiza la solicitud, solo de dónde se origina.

Esto es un control **puramente de red**, sin validación de identidad. En Zero Trust, es insuficiente porque:

* Un intermediario no autorizado que comparta el mismo rango IP podrá acceder.
* Si el endpoint es comprometido, no hay un segundo factor de filtrado basado en identidad o permisos.
{% endhint %}

6. En el panel de navegación izquierdo, seleccionar **Resource Policy** (Política de recursos).
7. Verificar la política de recursos configurada:

* Permite llamadas desde la dirección IP en el rango `10.199.0.0/24`, que corresponde a la Subred privada 1 en la VPC de <kbd>ServicioA</kbd> y la Subred privada 2 en otra VPC.

<figure><img src="../../../../../.gitbook/assets/Exposición de vulnerabilidad - API Gateway (Learning).png" alt="configuración de API Gateway - Solicitud de método &#x60;NONE&#x60;"><figcaption><p>Configuración de API Gateway mal configurada</p></figcaption></figure>

En el panel de navegación izquierdo, seleccionar <kbd>**Resource Policy**</kbd> (Política de recursos).

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "AWS": "*"
      },
      "Action": "execute-api:Invoke",
      "Resource": "arn:aws:execute-api:*:*:*",
      "Condition": {
        "IpAddress": {
          "aws:VpcSourceIp": "10.199.0.0/24"
        }
      }
    }
  ]
}
```

* `"Version": "2012-10-17"`: Esta es la versión de la sintaxis de la política de IAM, que es la estándar.
* `"Statement"`: Una política puede tener una o más declaraciones. En este caso, solo hay una.
* `"Effect": "Allow"`: La declaración tiene un efecto de permiso. Esto significa que si se cumplen todas las condiciones de la declaración, se concederá el acceso. La alternativa sería `"Deny"`, que denegaría el acceso explícitamente.
* `"Principal": { "AWS": "*" }`: El `Principal` es la entidad a la que se aplica la política. En este caso, `"AWS": "*"` indica que se aplica a cualquier entidad de AWS (usuarios, roles, etc.) que intente acceder al recurso. Sin embargo, este permiso tan amplio se restringe drásticamente por la condición que veremos a continuación.
* `"Action": "execute-api:Invoke"`: La acción que se permite es `execute-api:Invoke`, que específicamente se refiere a la capacidad de llamar o invocar una API de API Gateway.
* `"Resource": "arn:aws:execute-api:*:*:*"`: El recurso al que se aplica la política es `arn:aws:execute-api:*:*:*`. Este ARN (Amazon Resource Name) usa comodines (`*`) para indicar que la política aplica a todas las API Gateway en todas las regiones y para todas las etapas.
* `"Condition"`: Aquí es donde reside la restricción clave. La condición debe ser evaluada como verdadera para que la política se aplique.
  * `"IpAddress"`: La condición se basa en una dirección IP.
  * `"aws:VpcSourceIp": "10.199.0.0/24"`: La política solo permite la acción de invocar la API si la dirección IP de origen de la solicitud se encuentra dentro del rango CIDR `10.199.0.0/24`. El uso de `aws:VpcSourceIp` en lugar de `aws:SourceIp` significa que esta política está diseñada para tráfico interno de una VPC, no para tráfico proveniente de la Internet pública.

{% hint style="danger" %}
**Vulnerable a movimiento lateral:**

* Aunque la IP coincida, no se verifica que la solicitud esté firmada.
* Esto permite que cualquier proceso en la red interna llame a la API sin autorización real.
{% endhint %}

</details>

<details>

<summary>Verificación de controles de seguridad del punto de enlace de VPC</summary>

1. Buscar y seleccionar **VPC,** selecciona **Endpoints** (Puntos de conexión).
   1. Localizar el punto de conexión de VPC de API Gateway que reside en las subredes de la VPC de ServicioA.
   2. Seleccionar el punto de enlace con un **VPC endpoint ID** que coincide con `vpce-08369586a0130484e`.
   3. Agregar una etiqueta de nombre al punto de enlace para facilitar su identificación:
      1. Seleccionar la pestaña **Tags** (Etiquetas) y luego **Manage tags** (Administrar etiquetas).
      2. Agregar una etiqueta con:
         1. **Key**: `Name`
         2. **Value**: `ServiceA-APIGW-EP`
         3. Guardar los cambios.
2. Con el punto de enlace seleccionado, ir a la pestaña **Policy** (Política).

```json
{
	"Statement": [
		{
			"Action": "*",
			"Effect": "Allow",
			"Principal": "*",
			"Resource": "*"
		}
	]
}
```

{% hint style="danger" %}
Porque el **VPC Endpoint** tiene una política abierta que:

* Permite tráfico a **cualquier API privada en la región**.
* No limita **qué identidades** pueden invocar.
* No filtra por **ARN de API, método o etapa**.

Esto significa que **todos los intermediarios con acceso a la VPC** pueden usarlo, incluyendo los no autorizados, lo que rompe el principio de Zero Trust.
{% endhint %}

</details>

<details>

<summary>Ejecutar una evaluación de su posición de seguridad actual</summary>



<figure><img src="../../../../../.gitbook/assets/Ejecutar una evaluación de seguridad.png" alt=""><figcaption></figcaption></figure>

**Cómo funciona:**

* `runscanner` simula peticiones desde cada intermediario (esperado y no esperados) hacia:
  1. **API legítima** (<kbd>ServicioB</kbd>).
  2. **API desconocida**.
* Comprueba si la llamada llega o es bloqueada.
* Indica dónde se bloqueó (API Gateway, VPC Endpoint, etc.) y si es el resultado deseado (✔ o ✘).

**Resultados clave:**

* **Solo 2 de 6** patrones de llamadas se comportaron como deberían.
* Todas las llamadas que comparten el mismo rango de IP (`10.199.0.0/24`) pasan, aunque sean de intermediarios no deseados.
* No hay validación por identidad, solo por ubicación de red.

En este laboratorio, la **API desconocida** es otra API privada expuesta en API Gateway dentro del **Servicio de destino (**<kbd>**ServiceB**</kbd>**)**, pero **no forma parte del flujo legítimo**.

* No es la <kbd>**ServiceBAPI**</kbd> que debería recibir llamadas del Intermediario esperado.
* Está listada en la arquitectura como una **API adicional** que existe en el entorno pero no debería ser accesible desde ServicioA.
* En el script `runscanner`, se usa para probar si un intermediario puede acceder a **recursos que no necesita**, evaluando si los controles aíslan correctamente.

</details>

<details>

<summary>Mejorar la posición de seguridad mediante la autorización de IAM en la API Gateway</summary>

Actualmente, la seguridad de la API se basa únicamente en la red, permitiendo el acceso a cualquiera dentro de un rango de IP específico (`CIDR`). Esto es insuficiente para la Confianza Cero.

Para reforzar la seguridad, se habilita la autorización de IAM en la API Gateway. Esto obliga a que cada llamada a la API sea autenticada y autorizada individualmente usando el algoritmo AWS Signature Version 4 (SigV4). Con este cambio, ya no basta con estar en la red correcta; el solicitante también debe tener credenciales de IAM válidas y firmar la solicitud correctamente.

* Cambiar en API Gateway el método GET `/orders` de `NONE` a `AWS_IAM`.
* Implementar la API para aplicar cambios.

<figure><img src="../../../../../.gitbook/assets/Animation.gif" alt=""><figcaption></figcaption></figure>

{% file src="../../../../../.gitbook/assets/scanner.py" %}

{% file src="../../../../../.gitbook/assets/service_a_caller.py" %}

{% file src="../../../../../.gitbook/assets/service_a_caller_sigv4.py" %}

{% file src="../../../../../.gitbook/assets/service_a_unknownapi.py" %}

* Modificar el código Python en la Instancia-ServicioA para firmar las peticiones con **AWS Signature Version 4: `mv /tmp/lab/service_a_caller_sigv4.py /tmp/lab/service_a_caller.py`**
* **Realizamos una evaluación: `runscanner`**

<figure><img src="../../../../../.gitbook/assets/Evaluación número dos.gif" alt=""><figcaption></figcaption></figure>

En esta evaluación, ya se había configurado la **autenticación IAM con SigV4** en API Gateway.\
El filtrado se aplicaba así:

1. **Política de recursos de API Gateway** seguía permitiendo cualquier IP de `10.199.0.0/24`.
2. **Autorización IAM (SigV4)** exigía que las llamadas fueran firmadas con credenciales válidas.
3. **VPC Endpoint** proporcionaba la ruta de red.

**Explicación por caso:** [Broken link](/broken/pages/e61SB9TphOsLHJjpPwhT#tabla-de-intermediarios "mention")

* **Intermediario esperado → ServicioB**: permitido. Tiene IP dentro del rango, soporta SigV4 y accede vía VPC Endpoint válido.
* **Intermediario esperado → API desconocida**: permitido. La política de esa API desconocida también permite IP de `10.199.0.0/24` y no se modificó su autorización.
* **Intermediario no esperado #1 → ServicioB**: bloqueado. IP fuera del rango (`10.199.1.0/24`) y no soporta SigV4.
* **Intermediario no esperado #2 → ServicioB**: bloqueado. IP válida pero no soporta SigV4, y la API ahora exige firma.
* **Intermediario no esperado #3 → ServicioB**: permitido. IP válida, soporta SigV4 y usa ruta correcta.
* **Intermediario no esperado #4 → ServicioB**: permitido. IP válida, soporta SigV4 y usa ruta correcta.

**Clave:**\
La mejora con SigV4 permitió bloquear a intermediarios que no podían firmar solicitudes (ej. #2), pero aún quedaban brechas porque **otros no deseados con IP válida y SigV4 seguían autorizados** (#3 y #4).

</details>

<details>

<summary>Mejorar la posición de seguridad mediante una política de recursos de API Gateway</summary>

**Controles aplicados:**

* **`aws:PrincipalAccount`** → solo permite llamadas desde la cuenta de AWS autorizada (ServicioA).
* **`aws:SourceVpce`** → solo permite llamadas desde el VPC Endpoint autorizado de ServicioA.

**Efecto en cada llamada evaluada:**

1. **Intermediario esperado → ServicioB**\
   Permitido. Es de la cuenta correcta, usa el VPC Endpoint autorizado y soporta SigV4.
2. **Intermediario esperado → API desconocida**\
   Permitido. No se cambió la política de esa API, por eso sigue abierta.
3. **Intermediario no esperado #1 → ServicioB**\
   Bloqueado. No proviene del VPC Endpoint autorizado y no tiene cuenta válida.
4. **Intermediario no esperado #2 → ServicioB**\
   Bloqueado. No soporta SigV4 y tampoco pasa el control de identidad de la política.
5. **Intermediario no esperado #3 → ServicioB**\
   Permitido. Está en la misma cuenta, usa el VPC Endpoint autorizado y soporta SigV4.
6. **Intermediario no esperado #4 → ServicioB**\
   Bloqueado. Usa cuenta correcta pero desde un VPC Endpoint no autorizado.

#### Modificación políticas de API

* **Abrir API Gateway**
  * Seleccionar **ServiceBAPI**.
  * Ir a **Resource Policy**.
* **Editar política**
  * Eliminar la política existente.
  * Pegar la siguiente plantilla reemplazando valores:

<figure><img src="../../../../../.gitbook/assets/Actualizar politicas de API.png" alt=""><figcaption></figcaption></figure>

> Sustituir:
>
> * `AWS_ACCOUNT_ID_SERVICIOA` por el ID real de la cuenta de ServicioA.
> * `VPC_ENDPOINT_ID_SERVICIOA` por el ID real del VPC Endpoint de ServicioA.

3. **Guardar cambios**.

<figure><img src="../../../../../.gitbook/assets/Añadir controles combinados de identidad y red para que API Gateway.gif" alt=""><figcaption></figcaption></figure>

| Intermediario                  | Resultado | Justificación técnica                         |
| ------------------------------ | --------- | --------------------------------------------- |
| **Esperado → ServicioB**       | Permitido | Cuenta correcta + VPC Endpoint válido + SigV4 |
| **Esperado → API desconocida** | Permitido | API no modificada                             |
| **No esperado #1**             | Bloqueado | VPC Endpoint inválido                         |
| **No esperado #2**             | Bloqueado | No soporta SigV4 y no cumple política         |
| **No esperado #3**             | Permitido | Cuenta correcta + VPC Endpoint válido + SigV4 |
| **No esperado #4**             | Bloqueado | VPC Endpoint inválido                         |

</details>

<details>

<summary>Mejorar la posición de seguridad mediante la política de punto de enlace de VPC</summary>



**Objetivo**

1. **Identidad**:
   * Solo el **rol IAM** de la instancia autorizada de ServicioA puede usar ese VPC Endpoint para hacer llamadas.
   * Esto impide que otras instancias, Lambdas u otros recursos de la misma VPC (incluso de la misma cuenta) lo utilicen.
2. **Recurso específico**:
   * Se permite invocar únicamente el **método GET** sobre el recurso `/orders` de la API **ServiceBAPI** en la etapa `api`.
   * Se bloquea todo lo demás: otras APIs, otros métodos (`POST`, `PUT`, etc.) y otros recursos (`/clientes`, `/productos`, etc.).

**Contexto de aplicación**:

* En paso anterior, el propietario de **ServicioB** ya limitó el acceso a llamadas desde una cuenta específica y un VPC Endpoint autorizado.
* Aquí, como propietario de **ServicioA**, se asume responsabilidad de filtrar en **origen** para que desde tu infraestructura no salgan llamadas no previstas hacia ServicioB.
* Este control evita que una identidad interna comprometida o mal configurada use tu punto de salida para llegar a APIs que no están permitidas.

**Resultado esperado**:

* La API desconocida que antes estaba permitida queda bloqueada en el **nivel del VPC Endpoint**.
* Intermediarios no autorizados que antes pasaban el filtro de ServicioB también son bloqueados antes de salir de la red de ServicioA.

***

**1. Abrir VPC Endpoint de ServiceA**

* En consola AWS, buscar **VPC.** En menú lateral: **Endpoints**.
* Ubica y selecciona **ServiceA-APIGW-EP**.

**2. Editar política**

* En la pestaña **Policy**, seleccionar **Edit Policy**.
* Marcar **Custom (Personalizado)**.
* Eliminar la política actual.
* Pegar esta política ajustando valores:

```json
jsonCopiarEditar{
    "Statement": [
        {
            "Effect": "Allow",
            "Principal": {
                "AWS": "ARN_DEL_ROL_ServiceAInstanceRole"
            },
            "Action": "execute-api:Invoke",
            "Resource": "ARN_DEL_METODO_ServiceBAPI_GET_ORDERS"
        }
    ]
}
```

<figure><img src="../../../../../.gitbook/assets/Autorizar IAM para VPC Endpoint.png" alt=""><figcaption></figcaption></figure>

* `ARN_DEL_ROL_ServiceAInstanceRole` → valor de `ServiceAInstanceRoleARN`.
* `ARN_DEL_METODO_ServiceBAPI_GET_ORDERS` → valor de `APIMethodARN`.

5. Guardar cambios (**Save**).

***

#### **Efecto de la nueva política**

* Solo el **rol IAM** de la instancia de ServicioA puede invocar.
* Solo se permite el método GET `/orders` de ServiceBAPI en la etapa `api`.
* Se bloquea:
  * API desconocida.
  * Otros métodos o recursos de ServiceBAPI.
  * Cualquier otra identidad, incluso dentro de la misma cuenta.

***

#### **Verificación**

1. En **ServiceAInstance** (Session Manager) ejecutar:

```bash
bashCopiarEditarrunscanner
```

2. Validar que:
   * **Esperado → ServicioB**: permitido.
   * **Esperado → API desconocida**: bloqueado en nivel VPC Endpoint.
   * **No esperados #1, #2, #3**: bloqueados en nivel VPC Endpoint.
   * **No esperado #4**: bloqueado por API Gateway (política de la tarea 5).

***

#### **Resultados esperados**

<figure><img src="../../../../../.gitbook/assets/Evaluación número tres.gif" alt=""><figcaption></figcaption></figure>

***

#### **Clave técnica**

* Aquí se aplica **Zero Trust en doble capa**:
  * ServicioB valida cuenta y VPC Endpoint.
  * ServicioA valida identidad y recurso exacto.
* Esto elimina tráfico no autorizado **en origen**, reduciendo superficie de ataque y carga innecesaria.

</details>

<details>

<summary>Ajuste del grupo de seguridad del VPC Endpoint</summary>



</details>
