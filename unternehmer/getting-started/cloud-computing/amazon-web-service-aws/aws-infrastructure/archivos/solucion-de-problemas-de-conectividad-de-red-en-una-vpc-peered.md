# Solución de problemas de conectividad de red en una VPC Peered

<h2 align="center">Solución de problemas de conectividad de red en una VPC Peered</h2>

{% embed url="https://soundcloud.com/kerbela/premiere-bicep-glue-acor-ht-rework-morc014?si=e1f468f1c2534f10b17134f81efc2a1f&utm_campaign=social_sharing&utm_medium=text&utm_source=clipboard" %}

El práctica cubre los conceptos de conectividad de Amazon Virtual Private Cloud (Amazon VPC). Muestra cómo configurar un emparejamiento de VPC entre dos VPC en la misma cuenta y permitir el tráfico HTTP cliente-servidor entre dos hosts en las VPC.

<figure><img src="../../../../../.gitbook/assets/Solucionar problemas de conectividad en VPC Peering.png" alt="Solucionar problemas de conectividad en VPC Peering"><figcaption><p>Solucionar problemas de conectividad en VPC Peering</p></figcaption></figure>

**`AnyCompany`** necesita permitir el tráfico HTTP entre un cliente y un servidor. La instancia de cliente HTTP reside en una VPC, mientras que la instancia de servidor HTTP reside en una VPC diferente en la misma cuenta de AWS. Un miembro junior de su equipo ya implementó la solución, pero parece que estaba mal configurada.

Como miembro del equipo de la nube de **`AnyCompany`**, se le ha asignado la tarea de comprobar la conectividad entre un cliente HTTP y un servidor. Su tarea es solucionar el problema y corregir la configuración para permitir el tráfico HTTP entre el cliente y el servidor a través del emparejamiento de VPC.

<figure><img src="../../../../../.gitbook/assets/Barra.gif" alt=""><figcaption></figcaption></figure>

<details>

<summary>Probar la conectividad cliente-servidor HTTP</summary>

Prueba la conectividad HTTP entre las instancias de cliente y servidor. Para probar la conectividad, inicie el tráfico HTTP desde la instancia de cliente HTTP y observe la respuesta.

Antes de comenzar a solucionar problemas, es útil resumir los requisitos para conectarse a un recurso en una VPC desde un recurso en una VPC del mismo nivel:

* Para cada recurso de cada VPC, verifique que la tabla de enrutamiento de su subred contenga una ruta que envíe tráfico destinado a la VPC del mismo nivel a través de la conexión de emparejamiento de la VPC.
* En el caso de las instancias EC2, verifique que los grupos de seguridad de las instancias EC2 permitan el tráfico de la VPC del mismo nivel.
* Para cada recurso de cada VPC, verifique que la ACL de red de su subred permita el tráfico de la VPC del mismo nivel.

<figure><img src="../../../../../.gitbook/assets/Problema de red VPC Peering.png" alt=""><figcaption></figcaption></figure>

Utilice <kbd>**Reachability Analyzer**</kbd> para identifica el componente de bloqueo. Busque <kbd>**Network Manager**</kbd> en el panel de navegación de la izquierda de la página, en la sección <kbd>**supervisión y solución de problemas**</kbd>, elija <kbd>**Reachability Analyzer**</kbd>.

<figure><img src="../../../../../.gitbook/assets/Evaluación con VPC Reachability Analyzer.gif" alt="Implementación de VPC Reachability Analyzer"><figcaption><p>Implementación de VPC Reachability Analyzer</p></figcaption></figure>

El fallo se ve en el grupo de seguridad, habilite el puerto 80 en el grupo <kbd>**sg-074e10fd4e2ac5cf4**</kbd> y vuelva a realizar el análisis.&#x20;

<figure><img src="../../../../../.gitbook/assets/Solucionar problemas de conectividad en VPC Peering - Target-Instance-SG.png" alt=""><figcaption></figcaption></figure>

Examine los saltos que bloquean el tráfico, debe tener en cuenta lo siguiente. La tabla de enrutamiento **`Target-Private-RT`** no tiene una ruta aplicable a la subred privada de origen a través del emparejamiento de VPC.

<figure><img src="../../../../../.gitbook/assets/Comprobar saltos VPC Reachability Analyzer.gif" alt="Implementación de VPC Reachability Analyzer inverso"><figcaption><p>Implementación de VPC Reachability Analyzer inverso</p></figcaption></figure>

* Revisa si las tablas de enrutamiento incluyen rutas hacia la VPC peering.
* La NACL de subred privada de origen no permite el tráfico entrante de la subred privada de destino.

<figure><img src="../../../../../.gitbook/assets/Solución completa VPC Reachability Analyzer.gif" alt=""><figcaption></figcaption></figure>

En una conexión de emparejamiento de VPC (VPC Peering), cada subred necesita una ruta explícita hacia la red CIDR de la otra VPC. Sin esta ruta, el tráfico no sabe cómo llegar a los recursos en la VPC emparejada. Vuelve a ejecutar la petición:

```bash
HTTP-Client$ wget http://10.2.2.200--2025-08-13 18:52:23--  http://10.2.2.200/
Connecting to 10.2.2.200:80... connected.
HTTP request sent, awaiting response... 200 OK
Length: 51 [text/html]
Saving to: ‘index.html.1’

100%[============================================================================================================================================================================================>] 51          --.-K/s   in 0s

2025-08-13 18:52:23 (8.02 MB/s) - ‘index.html.1’ saved [51/51]
```

</details>

