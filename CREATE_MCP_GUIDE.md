# Urano MCP - Guía de Creación e Instalación

Esta guía técnica está orientada a desarrolladores que deseen crear e integrar **Paquetes MCP (Model Context Protocol)** externos en la arquitectura de Agentes de Urano. Aquí aprenderás desde el concepto básico hasta configuraciones avanzadas para exponer herramientas dinámicas o estáticas y definir contextos (`Skills`) especializados.

---

## Índice
1. [Quick Start](#1-quick-start)
2. [Estructura del Proyecto MCP](#2-estructura-del-proyecto-mcp)
3. [El Archivo `config.ts` (Core Manifest)](#3-el-archivo-configts-core-manifest)
4. [Añadiendo Lógica a los Plugins](#4-añadiendo-lógica-a-los-plugins)
5. [Inyectando Contexto con `SKILL.md`](#5-inyectando-contexto-con-skillmd)
6. [Empaquetado y Distribución (.zip)](#6-empaquetado-y-distribución-zip)

---

## 1. Quick Start

Para arrancar de inmediato, esto es lo único que necesitas hacer para crear e instalar tu primer MCP llamado "HelloWorld":

1. Crea una carpeta local llamada `HelloWorld`.
2. Dentro de esa carpeta, crea el archivo `config.ts`:
```typescript
export const HelloWorldConfig = {
    name: "HelloWorld",
    description: "Mi primer módulo MCP de prueba",
    icon: "Rocket",
    category: "Desarrollo",
    pluginSchemas: {
        Tests: {
            actions: {
                ping: { label: 'Saludar API', fields: [] }
            }
        }
    }
}
```
3. Crea la ruta de plugin `Plugins/Tests/TestsPlugin.ts`:
```typescript
export class TestsPlugin {
    async executeAction(action: string, data: any) {
        if (action === 'ping') return "Pong! Hola desde MCP";
        throw new Error("Action not found");
    }
}
```
4. Comprime el interior de la carpeta en disco como `HelloWorld.zip`.
5. Ve a **Urano (Desktop o UI) > Integraciones / MCP Manager**.
6. Haz clic en **Instalar MCP (.zip)**, sube tu archivo, y ¡listo! Los agentes con permisos podrán automáticamente ver y utilizar tu nueva herramienta `urano_helloworld_tests_ping`.

---

## 2. Estructura del Proyecto MCP

Un paquete MCP bien estructurado consta de configuraciones centrales, código de ejecución y documentación contextual. Cuando instalas un ZIP en Urano, el sistema extrae todo hacia la bóveda aislada `~/.urano/workspace/mcp/{ModuleName}/`.

```text
📁 MiModuloMCP/
├── 📄 config.ts                 <-- (Requerido) Manifiesto y esquemas de entorno
├── 📄 package.json              <-- (Opcional) Define dependencias propias si se requieren
├── 📄 SKILL.md                  <-- Sirve de System Prompt inyectable (`type: mcp`)
└── 📁 Plugins/
    └── 📁 {PluginName}/
        └── 📄 {PluginName}Plugin.ts  <-- Clase ejecutora que resuelve las acciones declaradas en config
```

---

## 3. El Archivo `config.ts` (Core Manifest)

Tu `config.ts` es el cerebro del entorno. Urano lee este archivo para renderizar las interfaces gráficas y configurar la Bóveda de contraseñas de forma transparente.

### Anatomía Avanzada

```typescript
export const SlackConfig = {
    name: "SlackIntegration",
    description: "Conecta a Slack directamente en el workspace comercial",
    icon: "MessageSquare",
    category: "Comunicaciones",

    // 1. Opciones de Entorno (Bóveda / UI visual)
    settings: [
        { name: 'SLACK_TOKEN', type: 'password', title: 'Bot OAuth Token' },
        { name: 'DEFAULT_CHANNEL', type: 'text', title: 'Canal Default (Ej: #general)' },
    ],

    // 2. Conectores Nativos MCP Server (Opcional)
    // Si tu módulo es un wrapper de un módulo MCP open source:
    mcpServer: {
        command: 'npx',
        args: ['-y', '@modelcontextprotocol/server-slack'],
        requiredEnv: ['SLACK_TOKEN']
    },

    // 3. Declaración Explicita de Esquemas de Herramientas
    pluginSchemas: {
        Chat: {
            actions: {
                sendMessage: {
                    label: 'Enviar Mensaje',
                    fields: [
                        { name: 'channel', type: 'required', label: 'ID Canal' },
                        { name: 'text', type: 'required', label: 'Cuerpo del mensaje' },
                    ]
                }
            }
        }
    }
};
```

> **NOTA SOBRE SEGURIDAD:** Todos los campos variables estipulados en `settings` alimentan el motor criptográfico local de Urano. Los agentes jamas ven los tokens estáticos en sus contextos.

---

## 4. Añadiendo Lógica a los Plugins

Las herramientas estructuradas en el `config.ts` bajo la clave `pluginSchemas` esperan que haya una clase homóloga bajo el árbol `Plugins/{Name}/{Name}Plugin.ts`. 

Para el schema anterior (`Chat`), necesitarías crear `Plugins/Chat/ChatPlugin.ts`:

```typescript
export class ChatPlugin {
    private configStore: any;

    constructor(moduleConfigRecopiladoPorUrano: any) {
        // En este paso, Urano inyectará en tiempo real desencriptado las keys de la boda
        this.configStore = moduleConfigRecopiladoPorUrano; 
    }

    async executeAction(action: string, payload: any) {
        if (action === 'sendMessage') {
            const { channel, text } = payload;
            // ... Aquí ejecuta fetch o lógica local ...
            return { sent: true, timestamp: Date.now() };
        }
        throw new Error(`Acción ${action} no soportada en el plugin de Chat`);
    }
}
```

Urano enlazará el puente dinámicamente y creará un schema oficial en formato OpenAI Functions llamado `urano_slackintegration_chat_sendmessage`.

---

## 5. Inyectando Contexto con `SKILL.md`

Las herramientas sueltas de API no son suficientes si deseas dotar al LLM de la inteligencia sobre cómo usar tu MCP, o si quieres enseñarle **políticas empresariales específicas** al utilizar el módulo.

Para eso, creamos un archivo en la raíz del plugin llamado `SKILL.md`.

```yaml
---
name: slack_operator
description: Skill maestro que sabe dar soporte avanzado en Slack
type: mcp
tools: [urano_slackintegration_chat_sendmessage]
---

# Funciones de Operador 

1. Nunca envíes mensajes usando `@channel` sin urgencia estricta empresarial.
2. Si un usuario local te pide reportar bugs, envíalos al `#debug_logs` utilizando formato Markdown resaltable.
3. El canal default de logs es el provisto en tu entorno criptográfico o predispuesto en la interfaz superior.

Tu objetivo principal es hacer reportes asíncronos rápidos y formales usando la herramienta.
```

### ¿Por qué `type: mcp`?
Gracias a esta etiqueta en el `frontmatter`, el **SkillRegistry** de Urano sabe que este skill **no debe listarse globalmente** en el panel universal del "Editor de Agentes", dado que de lo contrario, saturaría la experiencia de un editor humano listándole un sin fin de herramientas técnicas irrelevantes (Ej. *bases de datos postgres, conectores git*). El skill permanece escondido y disponible enteramente para rehidratarse de trasfondo siempre y cuando el Agente tenga los *Módulos MCP Autorizados* explícitos.

---

## 6. Empaquetado y Distribución (.zip)

Urano implementa un **Instalador de MCP Agnóstico** (`McpInstaller.ts`). 

1. **Dependencias Externas (`package.json`):** Si tu código en NodeJS requiere paquetes como `axios`, `zod` o conectores de Bases de Datos, asegúrate de crear un archivo `package.json` oficial. El administrador extraerá un nombre validado desde ahí.
2. **Uso de Librerías Externas (Bundling):** Instalar carpetas anidadas de `node_modules` directamente dentro de un `.zip` puede resultar pesado y generar problemas de límite de ruta en Windows. Si tu plugin MCP utiliza dependencias de terceros o librerías TS externas, la mejor práctica recomendada es hacer un bundle con **esbuild** o **Webpack** para compilar tu lógica `Plugin` a archivos más livianos y pre-resueltos:
    *   `npm install --save-dev esbuild`
    *   Ejemplo de comando de empaquetado: `npx esbuild Plugins/index.ts --bundle --platform=node --outdir=dist`
    *   Luego empacas solo los archivos destilados en el root de tu ZIP. Si esto se te dificulta puedes incluir `node_modules` siempre que no superen límites agresivos.
3. **Compress**: Selecciona únicamente el contenido raíz de toda la carpeta de tu MCP. (No importa si accidentalmente comprimes incluyendo el directorio papá de Github `Master-Branch/`, el instalador inteligente lo purgará automáticamente si percata un archivo solitario raíz).
4. **Instalación Manual via Archivo Libre**: Si el usuario no quiere usar el instalador de la UI o ZIP, basta con copiar los archivos a `~/.urano/workspace/mcp/{Nombre}` y el motor los compilará al vuelo en el próximo arranque o refresco de memoria. 

### Finalización del Ciclo (Agentes)

Una vez tu módulo MCP exista en la plataforma:
- Se lista bajo "Integraciones Externas" en el Gestor Visual.
- Para darle permiso a un Agente a invocar tu módulo secreto, debe ir a su panel de `Configuración de Agente > MCPs (Avanzado)` y marcar la casilla autoritativa permitiendo puentes de tu integración específica. Las Tools y el System Prompt (`SKILL.md` oculto) se vincularán in-memory.
