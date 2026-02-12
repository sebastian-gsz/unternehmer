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

# Kit de desarrollo de agentes orquestados "ADK"

<h2 align="center">Kit de desarrollo de agentes orquestados "ADK"</h2>

***

IBM Watsonx Orchestrate incluye el Kit de Desarrollo de Agentes (ADK), que es un conjunto de herramientas orientadas al desarrollador para construir, probar y gestionar agentes.

Con el ADK, los desarrolladores tienen la libertad y el control para diseñar agentes potentes usando un framework ligero y una CLI sencilla. Puedes definir agentes en archivos YAML o JSON claros, crear herramientas personalizadas en Python y gestionar todo el ciclo de vida del agente con solo unos pocos comandos.

***

**Requisitos previos**

Esta documentación detalla el proceso de instalación del Agent Development Kit (ADK) de watsonx Orchestrate, la configuración del entorno y el despliegue de un agente "Hello World".

Para sacar el máximo partido a este tutorial, deberías haber:

* Familiaridad básica con los comandos terminal y bash
* Conocimientos básicos de Python y programación general

Además, necesitarás acceso a Watsonx Orchestrate, al que puedes inscribirte [Prueba gratuita de 30 días](https://www.ibm.com/account/reg/us-en/signup?formid=urx-52753\&cm_sp=ibmdev-_-developer-_-trial\&utm_source=ibm_developer\&utm_content=in_content_link\&utm_id=tutorials_getting-started-with-watsonx-orchestrate). Ejecuta los siguientes comandos:

```
python --version
pip --version
python -m venv venv
```

Activar el entorno:

* Windows: `venv\Scripts\activate`
* macOS/Linux: `source venv/bin/activate`

Instalación del ADK:

Con el entorno virtual activo, instala el paquete oficial de IBM:

```
pip install ibm-watsonx-orchestrate
orchestrate --help
```

Si la instalación fue exitosa, se desplegará la lista de comandos disponibles de la CLI.

***

<details>

<summary><strong>Conexión al SaaS de Watsonx Orchestrate</strong></summary>

* Inicia sesión en tu instancia de Watsonx Orchestrate.
* Haz clic en tu **Icono de Perfil** (esquina superior derecha) > **Settings**.
* En la pestaña API details:
  * Copia el Service instance URL.
  * Haz clic en Generate API key y guarda la clave de forma segura (no se podrá volver a ver).

</details>

<details>

<summary><strong>Activación del entorno en CLI</strong></summary>

Ejecuta el siguiente comando reemplazando los marcadores de posición:

```
orchestrate env add -n <nombre_entorno> -u <instance_url> --type mcsp --activate
```

* Ejemplo: `orchestrate env add -n WXO-Lab -u https://api.dl.watson-orchestrate.ibm.com/instances/2025... --type mcsp --activate`
* Finalización: La CLI te solicitará la API Key. Pégala y presiona `Enter`.

</details>

<details>

<summary><strong>Definición y despliegue del primer agente</strong></summary>

Los agentes se definen mediante especificaciones YAML o JSON.

Crea un archivo llamado `hello-world-agent.yaml` con el siguiente contenido:

```
spec_version: v1
kind: native
name: Hello_World_Agent
description: A simple Hello World agent
instructions: >
  You are a test agent created for a tutorial on how to get started with watsonx Orchestrate ADK. 
  When the user asks "who are you", respond with: I'm the Hello World Agent. 
  Congratulations on completing the Getting Started with watsonx Orchestrate ADK tutorial!
llm: watsonx/meta-llama/llama-3-2-90b-vision-instruct
style: default
collaborators: []
tools: []
```

Navega en tu terminal hasta la carpeta del archivo y ejecuta:

```
orchestrate agents import -f hello-world-agent.yaml
```

</details>

<details>

<summary><strong>Pruebas y validación en la interfaz</strong></summary>

Abrir Agent BuilderEn la consola web de watsonx Orchestrate, ve al menú principal > Build > Agent Builder.Seleccionar el agenteSelecciona Hello\_World\_Agent.ValidaciónEn el panel de chat de la derecha, escribe: “Who are you?”. El agente debe responder según las instrucciones definidas en el YAML.

</details>

**Resumen de comandos rápidos**

| **Acción**      | **Comando**                                |
| --------------- | ------------------------------------------ |
| Instalar ADK    | `pip install ibm-watsonx-orchestrate`      |
| Agregar Entorno | `orchestrate env add ...`                  |
| Importar Agente | `orchestrate agents import -f <file>.yaml` |
| Ayuda           | `orchestrate --help`                       |
