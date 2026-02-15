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

# Crear con las bases de datos de Amazon RDS

<h2 align="center">Crear con las bases de datos de Amazon RDS</h2>

<figure><img src="../../../../../.gitbook/assets/Crear con las bases de datos RDS.png" alt=""><figcaption></figcaption></figure>

#### Descripción de la imagen del laboratorio

El diagrama ilustra cómo fluye la comunicación entre los componentes de la arquitectura:

* **Usuario externo**: accede a la infraestructura a través de una **puerta de enlace de internet (Internet Gateway)**.
* **Instancia EC2 CommandHost**: ubicada en una **subred pública**, actúa como puente para administrar y probar la base de datos.
* **Base de datos RDS (MySQL)**: desplegada en una **subred privada** y accesible únicamente a través del puerto **3306** (MySQL).
* **Secrets Manager**: gestiona las credenciales de la base de datos y está integrado con la instancia RDS.
* **Función Lambda de rotación**: se encarga de actualizar periódicamente la contraseña en RDS y sincronizarla con el secreto en Secrets Manager.
* **Comunicación interna**: CommandHost utiliza credenciales dinámicas obtenidas de Secrets Manager para conectarse al motor de base de datos sin necesidad de almacenar contraseñas en texto plano.

***

* **Alta disponibilidad y resiliencia**
  * Empresas con aplicaciones críticas (banca, e-commerce, salud) no pueden permitirse interrupciones.
  * Multi-AZ garantiza continuidad automática en segundos si falla una instancia.
* **Seguridad de credenciales**
  * En la práctica, las contraseñas duras en código son un riesgo de fuga.
  * Secrets Manager centraliza, cifra y rota credenciales sin intervención manual.
* **Cumplimiento normativo**
  * Normas como GDPR, PCI DSS o HIPAA exigen rotación periódica de contraseñas y cifrado de datos en tránsito.
  * El laboratorio muestra cómo cumplir estos requisitos.
* **Escalabilidad operativa**
  * La arquitectura abstrae tareas pesadas (respaldos, parches, failover).
  * Equipos pequeños pueden operar entornos complejos sin sobrecarga de administración.
* **Integración con aplicaciones modernas**
  * Al usar APIs para recuperar secretos, las aplicaciones pueden desplegarse en **EC2, ECS, Lambda o Kubernetes** sin exponer credenciales.

<figure><img src="../../../../../.gitbook/assets/Barra.gif" alt=""><figcaption></figcaption></figure>

<details>

<summary>Configurar e implementar una base de datos de Amazon RDS</summary>

1. En la consola de AWS, en la barra de búsqueda, escriba **RDS** y seleccione el servicio.
2. Seleccione **Crear base de datos**.
3. En **Método de creación**, elija **Creación estándar**.
4. En **Opciones del motor**, seleccione **MySQL**.
5. En **Plantillas**, elija **Producción**.
6. En **Disponibilidad y durabilidad**, seleccione **Instancia de base de datos Multi-AZ**.
   * Esto garantiza una réplica síncrona en otra zona de disponibilidad, habilitando conmutación por error automática.
7. En **Configuración**:
   * **Identificador de la instancia**: `RDSLabDB`
   * **Usuario maestro**: `mydbAdminUser`
   * **Contraseña maestra**: `mydbAdminPassword` (copiar/pegar desde los valores dados en el laboratorio)
   * **Confirmar contraseña maestra**: repetir el mismo valor.
   * **Administración de credenciales**: Autoadministrado
8. En **Configuración de la instancia**:
   * Clase de instancia: `db.t3.micro` (categoría de clases ampliables).
9. En **Almacenamiento**:
   * Tipo de almacenamiento: `SSD gp3` (uso general).
   * Nota: para cargas más pesadas, se podrían usar IOPS provisionadas con mayor coste.
10. En **Conectividad**:
    * VPC: `RDSVPC`
    * Grupo de subredes: `mydbsubnetgroup`
    * **Acceso público**: No (la instancia no recibe IP pública).
    * Grupo de seguridad de VPC: `DBSecurityGroup`
    * Elimine cualquier otro grupo listado.
11. En **Supervisión**:
    * Desmarque **Habilitar motorización mejorada**.
12. En **Configuración adicional**:
    * **Nombre de base de datos inicial**: `MyRDSLab`
    * **Respaldos**: habilitados, retención de 10 días.
    * **Exportaciones de logs**: seleccione todos para enviarlos a **CloudWatch Logs**.
    * **Mantenimiento**: domingo, 23:00 UTC, duración de 1 hora.
13. Seleccione **Crear base de datos**.
14. Si aparece la ventana de complementos sugeridos, cierre sin aceptar.
15. Espere la creación (\~20 min). Para no detener el laboratorio, utilice la instancia ya aprovisionada llamada `mydb`.

#### Explicación técnica

* **Multi-AZ**\
  Amazon RDS crea una réplica secundaria en otra zona de disponibilidad y la mantiene en sincronía. Si ocurre una falla, realiza failover automático en segundos. Esto mejora la **alta disponibilidad (HA)** y la **resiliencia**.
* **Sin acceso público**\
  La base de datos no tiene IP pública. Se accede solo desde dentro de la VPC (p. ej. EC2 en subred privada o bastión). Esto minimiza la superficie de ataque.
* **Respaldos automatizados**\
  Se generan en la réplica secundaria cuando está disponible, reduciendo el impacto en el rendimiento de la instancia primaria.
* **Exportación de logs a CloudWatch**\
  Facilita monitoreo, alertas y auditoría centralizada.
* **Ventana de mantenimiento**\
  Garantiza que las actualizaciones controladas (parches, cambios de configuración) se realicen en horario definido, reduciendo impacto en producción.

#### Pregunta clave relacionada

* **¿Por qué RDS no usa una IP pública en este escenario?**\
  Porque la arquitectura se diseñó para que el acceso a la base de datos ocurra solo a través de recursos internos (EC2 CommandHost, aplicaciones privadas). Una IP pública expondría el motor de base de datos directamente a Internet, incrementando el riesgo de ataques.

</details>

<details>

<summary>Crear y verificar secretos mediante AWS Secrets Manager</summary>

Se crean credenciales seguras para la base de datos RDS y se prueban en consola, CLI y con rotación automática.

***

1. En la consola de AWS, busque **Secrets Manager** y seleccione **Almacenar un secreto nuevo**.
2. Tipo de secreto: **Credenciales para la base de datos de Amazon RDS**.
3. Ingrese:
   * Usuario: `mydbAdminUser`
   * Contraseña: `mydbAdminPassword`
4. Clave de cifrado: `aws/secretsmanager`.
5. Base de datos: `mydb`.
6. Nombre del secreto: `mydbsecret-xxxx` (sustituir xxxx por valores aleatorios).
7. No habilite la rotación automática por ahora.
8. Revise y seleccione **Almacenar**.

**Explicación técnica**\
Secrets Manager cifra y almacena credenciales de forma centralizada. Esto elimina la necesidad de incrustar contraseñas en código o archivos de configuración, reduciendo riesgos de fuga.

***

#### 2.2 Recuperar secretos desde la consola

1. En la consola de **Secrets Manager**, seleccione `mydbsecret-xxxx`.
2. En pestaña **Descripción general**, haga clic en **Recuperar valor del secreto**.
3. Visualice pares clave/valor y, en pestaña **Texto plano**, el JSON completo.

**Explicación técnica**\
Cuando una aplicación necesita acceder a la base de datos, invoca la API de Secrets Manager y obtiene las credenciales vigentes en formato JSON. No requiere almacenar secretos localmente.

***

#### 2.3 Recuperar secretos desde EC2 CommandHost (AWS CLI)

1. Conéctese a la instancia EC2 CommandHost usando el valor `CommandHostSessionUrl`.
2.  Ejecute:

    ```bash
    cd ~
    aws secretsmanager list-secret-version-ids --secret-id mydbsecret-xxxx
    ```

Resultado esperado: ID de versión (UUID), `VersionStages: AWSCURRENT`, ARN del secreto.\
3\. Obtenga el ARN directamente:

```bash
aws secretsmanager list-secret-version-ids --secret-id mydbsecret-xxxx --output text --query ARN
```

4.  Recupere el secreto actual:

    ```bash
    aws secretsmanager get-secret-value --secret-id SECRET_ARN --version-stage AWSCURRENT
    ```

    Resultado esperado: JSON con `username`, `password`, `host`, `dbname`, `port`.

**Explicación técnica**\
El ARN identifica unívocamente el secreto. `AWSCURRENT` es la etiqueta que marca la versión activa, lo cual es clave cuando hay rotaciones automáticas.

***

#### 2.4 Habilitar la rotación automática de secretos

1.  Desde CommandHost, configure variables de entorno con `jq`:

    ```bash
    secret=$(aws secretsmanager get-secret-value --secret-id SECRET_ARN | jq .SecretString | jq fromjson)
    user=$(echo $secret | jq -r .username)
    password=$(echo $secret | jq -r .password)
    endpoint=$(echo $secret | jq -r .host)
    port=$(echo $secret | jq -r .port)
    ```
2.  Conéctese a la base de datos:

    ```bash
    mysql -h $endpoint -u $user -P $port -p$password mydb
    ```

    Ejecute `STATUS;` para verificar la conexión.\
    **Nota:** el resultado inicial muestra `SSL: Not in use`, se corrige en la tarea de cifrado.
3. Cierre sesión con `exit`.
4. En la consola de AWS, abra el secreto `mydbsecret-xxxx` → pestaña **Rotación** → **Editar rotación**.
5. Configure:
   * **Habilitar rotación automática**
   * Intervalo: cada **30 días**
   * Función de rotación: `rotation-lambda`
6. Guarde la configuración.

**Explicación técnica**

* Secrets Manager rota automáticamente la contraseña en la base de datos y actualiza el secreto sin intervención humana.
* La función Lambda asociada se encarga de sincronizar el cambio.
* Aplicaciones que consultan `AWSCURRENT` no requieren modificaciones: siempre usan la credencial válida.

***

#### Preguntas clave relacionadas

1. **¿Qué ventaja tiene recuperar credenciales por API en vez de almacenarlas en un archivo de configuración?**\
   Evita exposición accidental en repositorios o sistemas de logging. Además, garantiza que siempre se use la versión actualizada tras una rotación.
2. **¿Por qué se utiliza `AWSCURRENT` en el secreto?**\
   Marca la versión activa. Durante la rotación puede haber varias versiones (pendiente, previa, actual), pero la etiqueta `AWSCURRENT` asegura que las aplicaciones usen la contraseña válida.
3. **¿Qué problema detectamos en la conexión inicial a MySQL?**\
   La conexión se estableció sin cifrado SSL. Esto permite interceptación de tráfico en tránsito, por lo que debe habilitarse SSL en producción.

</details>

