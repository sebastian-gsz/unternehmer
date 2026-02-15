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
    visible: false
  tags:
    visible: true
---

# Balanceador de cargas de aplicación con Cloud Armor

<h2 align="center">Balanceador de cargas de aplicación con Cloud Armor</h2>

{% embed url="https://soundcloud.com/klangkuenstler/cathedral-of-saw-filth-on-acid?si=43a002db58aa4201b8b42d2aef147f32&utm_campaign=social_sharing&utm_medium=text&utm_source=clipboard" %}

<figure><img src="../../../../../.gitbook/assets/Balanceador de cargas de aplicaciones con Cloud Armor.png" alt=""><figcaption></figcaption></figure>

> Despliegue y configura un **Balanceador de Cargas de Aplicaciones externo** en **Google Cloud**, utilizando dos grupos de instancias gestionadas como **backends** en distintas regiones. Además, implementar una política de seguridad con **Cloud Armor** para controlar el acceso, una habilidad crucial en **ciberseguridad**.
>
> El balanceo de cargas de aplicaciones de Google Cloud se implementa en el perímetro de la red de Google, en los puntos de presencia (PoP) de Google de todo el mundo. El tráfico de usuario dirigido a un balanceador de cargas de aplicaciones ingresa al PoP que se encuentra más cerca del usuario. Luego, su carga se balancea a través de la red global de Google al backend más cercano que cuente con la capacidad disponible suficiente.
>
> Con las listas de direcciones IP permitidas y las de direcciones IP bloqueadas de Cloud Armor, puedes restringir o permitir el acceso a tu balanceador de cargas de aplicaciones en el perímetro de Google Cloud, lo más cerca que se pueda del usuario y del tráfico malicioso. Esto evita que los usuarios o el tráfico maliciosos consuman recursos o ingresen a tus redes de nube privada virtual (VPC).

<figure><img src="../../../../../.gitbook/assets/Barra.gif" alt=""><figcaption></figcaption></figure>

<details>

<summary><strong>Configura de firewall y comprobación de estado</strong></summary>

**Crea la regla de firewall de HTTP:** Esta regla permitirá que el tráfico **HTTP** (puerto 80) desde cualquier origen de **Internet** (`0.0.0.0/0`) llegue a las instancias con la **etiqueta de red** `http-server`.&#x20;

* En la **consola de Cloud**, navega a **Menú de navegación** - > **Red de VPC** - > **Firewall**.
* Haz clic en **`Crear regla de firewall`**.
* Configura los siguientes valores:
  * **Nombre**: `default-allow-http`
  * **Red**: `default`
  * **Destinos**: **`Etiquetas de destino especificadas`**&#x20;
  * **Etiquetas de destino**: `http-server`&#x20;
  * **Filtro de origen**: **Rangos de IPv4**&#x20;
  * **Rangos de IPv4 de origen**: `0.0.0.0/0`&#x20;
  * **Protocolos y puertos**: **Protocolos y puertos especificados**; marca **TCP** y escribe `80`.
* Haz clic en **Crear**.
* **Creación con CLI**:&#x20;

```bash
gcloud compute --project= "Your project ID" firewall-rules create default-allow-http --description="Permite el trafico HTTP" --direction=INGRESS --priority=1000 --network=default --action=ALLOW --rules=tcp:80 --source-ranges=0.0.0.0/0 --target-tags=http-server
```

\
**Crea las reglas de firewall de verificación de estado**<br>

Las **verificaciones de estado** son esenciales para el **balanceo de cargas**. Los sondeos de estas verificaciones provienen de rangos **IP** específicos de **Google Cloud**, por lo que debes permitir explícitamente este tráfico.

* En la página de **`Políticas de firewall`**, haz clic en **`Crear regla de firewall`**.
* Configura los siguientes valores:
  * **Nombre**: `default-allow-health-check`
  * **Red**: `default`
  * **Destinos**: **`Etiquetas de destino especificadas`**
  * **Etiquetas de destino**: `http-server`&#x20;
  * **Filtro de origen**: **Rangos de IPv4**
  * **Rangos de IPv4 de origen**: `130.211.0.0/22`, `35.191.0.0/16` (asegúrate de ingresar ambos rangos, separados por un espacio).
  * **Protocolos y puertos**: **`Protocolos y puertos especificados`**; marca **`TCP`**.
* Haz clic en **Crear**.
* **Crear con CLI:**

```bash
gcloud compute --project= "Your project ID" firewall-rules create default-allow-health-check --direction=INGRESS --priority=1000 --network=default --action=ALLOW --rules=PROTOCOL:PORT,... --source-ranges=130.211.0.0/22,35.191.0.0/16 --target-tags=http-server
```

<figure><img src="../../../../../.gitbook/assets/Creación de politicas de firewall.gif" alt=""><figcaption></figcaption></figure>

</details>

<details>

<summary><strong>Configuración de plantillas compute engine y auto-scaling</strong></summary>

Crearás dos plantillas, una para cada región, que incluirán la **etiqueta de red** `http-server` para aplicar las reglas de firewall y un **script de inicio** para configurar el servidor web.

* En la **consola de Cloud**, ve a **Menú de navegación** > **Compute Engine** > **Plantillas de instancia**.
* Haz clic en **`Crear plantilla de instancias`**.
  * Configura la primera plantilla (**Region 1-template**):
  * **Nombre**: `europe-west1-template`
  * Ubicación: **`Global`**
  * Serie: **`E2`**
  * Tipo de máquina: **`e2-micro`**
  * Haz clic en Opciones avanzadas.
  * En Herramientas de redes, agrega Etiquetas de red: **`http-server`**.
* En Interfaces de red, haz clic en **`default`** y selecciona Subred: **`default europe-west1`**. Haz clic en **Listo**.
* Haz clic en la pestaña **Administración**.
* En **Metadatos**, haz clic en **Agregar elemento** y especifica:
  * Clave: startup-**`script-url`**
  * Valor: **`gs://cloud-training/gcpnet/httplb/startup.sh`**
  * Haz clic en **Crear**&#x20;

<figure><img src="../../../../../.gitbook/assets/Configura las plantillas de instancias Compute Engine.gif" alt=""><figcaption></figcaption></figure>

* Copia `europe-west1-template` para crear la segunda plantilla (**Region 2-template**):
  * Haz clic en **Region 1-template** y luego en **+Crear una similar**.
  * **Nombre**: `us-central1-template`
  * Asegúrate de que **Ubicación** sea **Global**.
  * En **Herramientas de redes**, asegúrate de que `http-server` esté como **etiqueta de red**.
  * En **Interfaces de red**, selecciona **Subred**: `default us-central1`. Haz clic en **Listo**.
  * Haz clic en **Crear**.

\
**Crea los grupos de instancias administrados**<br>

Los **grupos de instancias administradas (MIGs)** te permitirán escalar tus backends automáticamente según la carga, lo que es esencial para la **resiliencia** y la **eficiencia de costos**.

* En **Compute Engine**, haz clic en **Grupos de instancias**.
* Haz clic en **Crear grupo de instancias**.
* Configura el primer **MIG** (**Region 1-mig**):
  * **Nombre**: `europe-west1-mig`
  * **Plantilla de instancia**: `europe-west1-templatee`
  * **Ubicación**: **Varias zonas**
  * **Región**: `europe-west1`
  * **Número mínimo de instancias**: `1`
  * **Número máximo de instancias**: `2`
  * **Indicadores de ajuste de escala automático** > **`Tipo de indicador`**: **`Uso de CPU`**
  * **Uso de CPU objetivo**: `80`
  * **Período de inicialización**: `45`
  * Haz clic en **Crear**.
* Repite el procedimiento para **Region 2-mig**:
  * **Nombre**: `us-central1-mig`
  * **Plantilla de instancia**: `us-central1-template`
  * **Ubicación**: **Varias zonas**
  * **Región**: `us-central1`
  * **Número mínimo de instancias**: `1`
  * **Número máximo de instancias**: `2`
  * **Indicadores de ajuste de escala automático** > **`Tipo de indicador`**: **`Uso de CPU`**
  * **Uso de CPU objetivo**: `80`
  * **Período de inicialización**: `45`
  * Haz clic en **Crear**.

\
**Verifica los backends**<br>

* En **Compute Engine**, haz clic en **Instancias de VM**.
* Verifica que las instancias de **VM** (`us-central1-mig` y `europe-west1-mig`) se estén creando.
* Haz clic en la **IP externa** de una instancia de cada región para confirmar que la página web muestre la **IP de cliente**, el **Nombre de host** y la **Ubicación del servidor**. La **Ubicación del servidor** es el campo que identifica la región del backend.

<figure><img src="../../../../../.gitbook/assets/Crear grupo de instancias.gif" alt=""><figcaption></figcaption></figure>

</details>

<details>

<summary><strong>Configura el balanceador de cargas de aplicaciones</strong></summary>

* Navega a **Menú de navegación** > **Balanceo de cargas**.
* Haz clic en <kbd>**Crear balanceador de cargas**</kbd>.
* En **Balanceador de cargas de aplicaciones HTTP(S)**, haz clic en **Siguiente**.
* En <kbd>**Orientado al público o para uso interno**</kbd>, selecciona **Orientado al público (externo)** y haz clic en **Siguiente**.
* En **Implementación global o de una sola región**, selecciona <kbd>**Ideal para cargas de trabajo globales**</kbd> y haz clic en **Siguiente**.
* En <kbd>**Crear balanceador de cargas**</kbd>, haz clic en **Configurar**.
* Establece **Nombre del balanceador de cargas** como `http-lb`.

\
**Configura el frontend**<br>

* El **frontend** define cómo el **balanceador de cargas** recibe el tráfico de los clientes.
* Haz clic en **Configuración de frontend**.
* Especifica los siguientes valores para la **IP IPv4**:
  * **Protocolo**: `HTTP`
  * **Versión de IP**: `IPv4`
  * **Dirección IP**: **`Efímera`**
  * **Puerto**: `80`
  * Haz clic en **Listo**
* Haz clic en **Agregar IP y puerto de frontend**.
* Especifica los siguientes valores para la **IP IPv6**:
  * **Protocolo**: `HTTP`
  * **Versión de IP**: `IPv6`
  * **Dirección IP**: **`Asignación automática`**
  * **Puerto**: `80`
  * Haz clic en **Listo**.

\
**Configura el backend**<br>

El backend define los grupos de instancias a los que el balanceador de cargas enviará el tráfico, junto con las verificaciones de estado y las políticas de balanceo de carga.

* Haz clic en Configuración de backend.
* En Servicios y buckets de backend, haz clic en Crear un servicio de backend.
* Configura el servicio de backend:
  * Nombre: `http-backend`
  * Grupo de instancias: `Region 1-mig`
  * Números de puerto: `80`
  * Modo de balanceo: Tasa
  * Máximo de RPS: `50`
  * Capacidad: `100`
  * Haz clic en Listo.
* Haz clic en Agregar un backend.
* Configura el segundo backend:
  * Grupo de instancias: `Region 2-mig`
  * Números de puerto: `80`
  * Modo de balanceo: Utilización
  * Utilización máxima del backend: `80`
  * Capacidad: `100`
  * Haz clic en Listo.
* En Verificación de estado, selecciona Crear una verificación de estado.
* Configura la verificación de estado:
  * Nombre: `http-health-check`
  * Protocolo: `TCP`
  * Puerto: `80`
  * Haz clic en Guardar.
* Marca la casilla Habilitar registro.
* Configura la Tasa de muestreo como `1`.
* Haz clic en Crear para crear el servicio de backend.
* Haz clic en Aceptar.

\
**Revisa y crea el balanceador de cargas de aplicaciones**<br>

* Haz clic en <kbd>**Revisar y finalizar**</kbd>.
* Revisa la configuración del Backend y Frontend.
* Haz clic en <kbd>**Crear**</kbd>.
* Espera a que se cree el balanceador de cargas.
* Haz clic en el nombre del balanceador de cargas (`http-lb`).
* Anota las direcciones IPv4 e IPv6 del balanceador de cargas, que usarás en la siguiente tarea.

</details>

<details>

<summary>Prueba el balanceador de cargas de aplicaciones</summary>

Una vez que el balanceador de cargas esté activo, es crucial verificar que el tráfico se esté distribuyendo correctamente a tus backends.

\
**Accede al balanceador de cargas de aplicaciones**<br>

* Abre una nueva pestaña en tu navegador y ve a `http://[LB_IP_v4]` (reemplaza `[LB_IP_v4]` por la dirección IPv4 del balanceador de cargas).
  * El tráfico se desviará a la instancia de `Region 1-mig` o `Region 2-mig` según tu proximidad geográfica.

\
**Somete el balanceador de cargas de aplicaciones a prueba de esfuerzo**<br>

Crearás una VM adicional para simular una carga intensa y observar cómo el balanceador de cargas distribuye el tráfico.

* En la consola, ve a Menú de navegación > Compute Engine > Instancias de VM.
* Haz clic en Crear instancia.
* Configura la instancia `siege-vm`:
  * Nombre: `siege-vm`
  * Región: `Region 3` (la región más cercana a `Region 1`)
  * Zona: `Zone 3` (una zona dentro de `Region 3`)
  * Serie: `E2`
  * Haz clic en Crear.
* Espera a que se cree `siege-vm`.
* En `siege-vm`, haz clic en SSH para iniciar una terminal.
* Ejecuta los siguientes comandos para instalar `siege` y simular la carga (reemplaza `[LB_IP_v4]` con la IP de tu balanceador de cargas):

{% code overflow="wrap" %}
```bash
sudo apt-get -y install siege
export LB_IP=[LB_IP_v4]
siege -c 150 -t120s http://$LB_IP
```
{% endcode %}

* Mientras `siege` se ejecuta, ve a **Balanceo de cargas**.
* Haz clic en Backends.
* Haz clic en `http-backend`.
* Haz clic en la pestaña Monitoring.
* Observa el gráfico de Ubicación de frontend (tráfico total entrante). Deberías ver cómo el tráfico comienza a dirigirse a `Region 1-mig` (la más cercana) y, a medida que aumenta la carga, también se distribuye a `Region 2-mig`.
* Regresa a la terminal SSH de `siege-vm` y presiona CTRL+C para detener `siege`.

</details>

<details>

<summary><strong>Agrega siege-vm a la lista de bloqueo con Cloud Armor</strong></summary>

Ahora, pondrás a prueba la seguridad de tu balanceador de cargas utilizando Cloud Armor para bloquear el tráfico de la instancia `siege-vm`, simulando una mitigación de ataque.

\
**Crea la política de seguridad**<br>

* En la consola, ve a Menú de navegación > Compute Engine > Instancias de VM.
* Toma nota de la IP externa de `siege-vm`. La llamaremos `[SIEGE_IP]`.
* En la consola de Cloud, navega a Menú de navegación > Herramientas de redes - > Seguridad de red - > Políticas de Cloud Armor.
* Haz clic en Crear política.
* Configura los siguientes valores:
  * Nombre: `denylist-siege`
  * Acción de la regla predeterminada: Permitir
* Haz clic en Próximo paso.
* Haz clic en Agregar una regla.
* Configura la regla de lista de bloqueo:
  * Condición > Coincidencia: Ingresa la `[SIEGE_IP]`.
  * Acción: Rechazar
  * Código de respuesta: `403 (Prohibido)`
  * Prioridad: `1,000`
  * Haz clic en Listo.
* Haz clic en Próximo paso.
* Haz clic en Agregar destino.
* Configura el destino:
* Tipo: Servicio de backend (balanceador de cargas de aplicaciones externo)
* Destino: `http-backend`
* Haz clic en Crear política.
* Espera a que la política se cree completamente.

\
**Verifica la política de seguridad**<br>

* Regresa a la terminal SSH de `siege-vm`.
* Ejecuta el siguiente comando: <kbd>**curl http://$LB\_IP**</kbd>
  * El resultado debería ser un `403 Forbidden`. Si no lo es, espera unos minutos y vuelve a intentarlo, ya que la propagación de políticas de seguridad puede tardar.
* Abre una nueva pestaña en tu navegador y ve a `http://[LB_IP_v4]`.
  * Deberías poder acceder a la página web, ya que tu IP de cliente no está bloqueada por la política de Cloud Armor.
* En la terminal SSH de `siege-vm`, ejecuta de nuevo el comando para simular la carga: <kbd>**siege -c 150 -t120s http://$LB\_IP**</kbd>
  * Este comando no debería generar salida visible, ya que el tráfico está siendo bloqueado.
* Explora los registros de la política de seguridad para confirmar el bloqueo:
  * En la consola, ve a Menú de navegación > Seguridad de red > Políticas de Cloud Armor.
  * Haz clic en `denylist-siege`.
  * Haz clic en Registros.
  * Haz clic en Ver registros de políticas.
  * En la página de Logging, borra el texto en la Vista previa de la consulta.
  * Selecciona el recurso para el Balanceador de cargas de aplicaciones > `http-lb-forwarding-rule` > `http-lb`, y luego haz clic en Aplicar.
  * Haz clic en Ejecutar consulta.
  * Expande una entrada de registro en Resultados de la consulta.
  * Expande `httpRequest`. La solicitud debería ser de la dirección IP de `siege-vm`.
  * Expande `jsonPayload`.
  * Expande `enforcedSecurityPolicy`. Ten en cuenta que `configuredAction` se configuró como `DENY` para el `nombre` `denylist-siege`.

</details>
