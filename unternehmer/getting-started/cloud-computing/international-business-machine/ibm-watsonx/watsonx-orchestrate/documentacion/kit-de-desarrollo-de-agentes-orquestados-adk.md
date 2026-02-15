---
description: >-
  Guía para instalar y usar el Agent Development Kit (ADK) de IBM watsonx
  Orchestrate. Incluye CLI, configuración de entorno, API key y un agente Hello
  World en YAML/JSON.
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

# Kit de desarrollo de agentes orquestados "ADK"

## Kit de desarrollo de agentes orquestados (ADK) para IBM watsonx Orchestrate

Instala y usa el **Agent Development Kit (ADK)** de **IBM watsonx Orchestrate (WXO)**.\
Cubre la **instalación del paquete `ibm-watsonx-orchestrate`**, la **CLI `orchestrate`**, la **conexión al SaaS con API key**, y el despliegue de un **agente “Hello World”** definido en **YAML/JSON**.

***

El ADK (Kit de Desarrollo de Agentes) es el set de herramientas para:

* Construir agentes de watsonx Orchestrate con un framework ligero.
* Definir agentes en **YAML o JSON** (specs claras y versionables).
* Crear **tools personalizadas en Python**.
* Gestionar el ciclo de vida del agente desde la **CLI `orchestrate`** (importar, probar y validar).

### Requisitos previos

Para seguir este tutorial, deberías tener:

* Manejo básico de terminal (bash / PowerShell).
* Conocimientos básicos de Python.

También necesitas acceso a IBM watsonx Orchestrate.\
Puedes registrarte en la [Prueba gratuita de 30 días](https://www.ibm.com/account/reg/us-en/signup?formid=urx-52753\&cm_sp=ibmdev-_-developer-_-trial\&utm_source=ibm_developer\&utm_content=in_content_link\&utm_id=tutorials_getting-started-with-watsonx-orchestrate).

### Crear un entorno virtual (venv)

Ejecuta:

```bash
python --version
pip --version
python -m venv venv
```

Activa el entorno:

* Windows: `venv\Scripts\activate`
* macOS/Linux: `source venv/bin/activate`

### Instalar el ADK (paquete `ibm-watsonx-orchestrate`) y validar la CLI

Con el entorno virtual activo:

```bash
pip install ibm-watsonx-orchestrate
orchestrate --help
```

Si la instalación fue exitosa, verás los comandos disponibles de la CLI.

<details>

<summary><strong>Conexión al SaaS de IBM watsonx Orchestrate (API key y Service instance URL)</strong></summary>

* Inicia sesión en tu instancia de IBM watsonx Orchestrate.
* Haz clic en tu **Icono de Perfil** (esquina superior derecha) > **Settings**.
* En la pestaña API details:
  * Copia el Service instance URL.
  * Haz clic en Generate API key y guarda la clave de forma segura (no se podrá volver a ver).

</details>

<details>

<summary><strong>Activación del entorno en CLI</strong></summary>

Ejecuta el siguiente comando reemplazando los marcadores de posición:

```bash
orchestrate env add -n <nombre_entorno> -u <instance_url> --type mcsp --activate
```

* Ejemplo: `orchestrate env add -n WXO-Lab -u https://api.dl.watson-orchestrate.ibm.com/instances/2025... --type mcsp --activate`
* Finalización: La CLI te solicitará la API Key. Pégala y presiona `Enter`.

</details>

<details>

<summary><strong>Definición y despliegue del primer agente</strong></summary>

Los agentes se definen mediante especificaciones YAML o JSON.

Crea un archivo llamado `hello-world-agent.yaml` con el siguiente contenido:

```yaml
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

```bash
orchestrate agents import -f hello-world-agent.yaml
```

</details>

<details>

<summary><strong>Pruebas y validación en la interfaz</strong></summary>

1. En la consola web de **IBM watsonx Orchestrate**, ve a **Build > Agent Builder**.
2. Selecciona el agente **Hello\_World\_Agent**.
3. En el panel de chat, escribe: “Who are you?”.
4. El agente debe responder según las instrucciones del YAML.

</details>

### Resumen de comandos rápidos (CLI `orchestrate`)

| **Acción**      | **Comando**                                |
| --------------- | ------------------------------------------ |
| Instalar ADK    | `pip install ibm-watsonx-orchestrate`      |
| Agregar Entorno | `orchestrate env add ...`                  |
| Importar Agente | `orchestrate agents import -f <file>.yaml` |
| Ayuda           | `orchestrate --help`                       |
