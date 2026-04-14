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
├── 📄 SKILL.md                  <-- (Recomendado) Sirve de System Prompt inyectable (`type: mcp`)
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

> [!NOTE]
> **Resolución de Nombres (Urano >= 1.3.5):** El sistema ahora resuelve el nombre del módulo de forma **insensible a mayúsculas** (`case-insensitive`) escaneando el sistema de archivos. Esto significa que si tu carpeta es `SlackIntegration`, las llamadas dirigidas a `slackintegration` funcionarán correctamente.
### Retornando Contenido Multimodal (Imágenes, Archivos)
Gracias al Core Multimodal de Urano y el soporte de AI SDK v6, un plugin ya no está limitado a retornar solo texto `String`. Si devuelves un Array de `MessagePart`, el agente "verá" verdaderamente los componentes.

```typescript
    async executeAction(action: string, payload: any) {
        if (action === 'capture') {
            const rawBuffer = await miLibreriaDeCaptura();
            
            // Retorna un Array para saltarse el Stringify (Requiere Urano >= 1.2.5)
            return [
                { type: 'text', text: 'Aquí tienes la captura que solicitaste:' },
                { type: 'image', image: rawBuffer.toString('base64'), mimeType: 'image/png' }
            ];
        }
    }
```
> **¿Qué hace Urano con ese Array?** Para garantizar que APIs que no soportan visión en roles de herramientas (como OpenAI vía OpenRouter) no colapsen la memoria al tratar los Base64 como texto crudo, el **RuntimeLoop** interceptará tu componente `image`, dejará una mención en texto, y recreará un segundo mensaje invisible bajo el `role: "user"` justo abajo. Así el modelo lo analizará de forma 100% nativa.

---

## 5. Inyectando Badges desde de tu Plugin MCP (Opcional)

Si desarrollas una herramienta MCP que lanza procesos de fondo, abre sub-pestañas, u obtiene documentos extensos, tal vez desees ofrecer al usuario un clic de acceso rápido arriba de su barra de texto. Puedes inyectar un **SessionBadge** universal:

```typescript
// Desde tu archivo MyPlugin.ts
public async apiEjecutarTarea(payload: any) {
    const { CoreFactory } = await import('@core/CoreFactory');
    const manager = await CoreFactory.getSessionManager();
    const session = manager.get(payload.sessionId);
    
    if (session) {
        const badges = session.metadata.badges || [];
        // Evitar duplicados
        if (!badges.some(b => b.id === 'mi-reporte')) {
            badges.push({
                id: 'mi-reporte',
                label: 'Abrir Reporte Generado',
                icon: '📊',
                color: 'success', // 'default' | 'info' | 'success' | 'warning' | 'error'
                actionRoute: '/module/MyPlugin/plugins/Report/apiAbrirReporte',
                actionData: { reportId: '123' }
            });
            await manager.updateSessionMetadata(payload.sessionId, { badges });
        }
    }
    
    return { success: true };
}
```

Urano Front pintará tu badge usando la paleta de colores indicada, y desencadenará el IPC Request a tu endpoint proporcionado de manera transparente.

### Manejando la Acción en tu Plugin

Para que el badge sea funcional, tu clase de Plugin debe implementar el método que definiste en `actionRoute`. Si tu ruta es `/module/MyPlugin/plugins/Report/apiAbrirReporte`, tu plugin debe tener el método `apiAbrirReporte`:

```typescript
// En src/main/Modules/MyPlugin/Plugins/Report/ReportPlugin.ts
export class ReportPlugin extends PluginBase {
    
    // Este método se invoca cuando el usuario hace clic en el Badge
    async apiAbrirReporte(payload: { reportId: string }) {
        console.log("El usuario pidió abrir el reporte:", payload.reportId);
        
        // Aquí puedes lanzar procesos, abrir archivos locales, o
        // incluso inyectar un mensaje nuevo en el chat.
        return { success: true };
    }
}
```

> [!TIP]
> **Sobre los Imports**: Nota que usamos `await import(...)` para cargar `CoreFactory`. Esto es una **buena práctica mandatoria** en Urano Desktop para evitar "Circular Dependencies" (errores de importación cíclica) ya que los plugins se cargan al inicio del sistema antes de que el Core esté 100% inicializado.

---

## 6. Inyectando Contexto con `SKILL.md`

Las herramientas sueltas de API no son suficientes si deseas dotar al LLM de la inteligencia sobre cómo usar tu MCP, o si quieres enseñarle **políticas empresariales específicas** al utilizar el módulo.

---

### ¿Es mandatorio el `SKILL.md`? (Urano >= 1.3.5)
A diferencia de versiones anteriores, **ya no es obligatorio** tener un `SKILL.md` válido para que las herramientas técnicas del módulo se registren. Si el archivo falta o está mal formado, el agente aún podrá ver y ejecutar tus acciones definidas en `config.ts`, pero carecerá de las instrucciones narrativas sobre *cómo* y *cuándo* usarlas.

> [!TIP]
> **Uso recomendado:** Siempre incluye un `SKILL.md`. Sin él, el agente puede alucinar con los parámetros o ignorar las políticas de seguridad que definas.

# Funciones de Operador 

1. Nunca envíes mensajes usando `@channel` sin urgencia estricta empresarial.
2. Si un usuario local te pide reportar bugs, envíalos al `#debug_logs` utilizando formato Markdown resaltable.
3. El canal default de logs es el provisto en tu entorno criptográfico o predispuesto en la interfaz superior.

Tu objetivo principal es hacer reportes asíncronos rápidos y formales usando la herramienta.
```

### ¿Por qué `type: mcp`?
Gracias a esta etiqueta en el `frontmatter`, el **SkillRegistry** de Urano sabe que este skill **no debe listarse globalmente** en el panel universal del "Editor de Agentes", dado que de lo contrario, saturaría la experiencia de un editor humano listándole un sin fin de herramientas técnicas irrelevantes (Ej. *bases de datos postgres, conectores git*). El skill permanece escondido y disponible enteramente para rehidratarse de trasfondo siempre y cuando el Agente tenga los *Módulos MCP Autorizados* explícitos.

### 🧠 El Protocolo Skill-First (Mandatorio)
A partir de la versión 1.3.0, Urano Core impone un bloque de instrucciones `<mcp_protocol>` al Agente. Este bloque le prohíbe usar tus herramientas `urano_*` si no ha ejecutado primero `urano_read_skill`.

**¿Qué significa esto para ti?**
- Tu `SKILL.md` **siempre** será consultado antes de que el agente use tu API.
- Debes incluir reglas de validación y ejemplos claros de JSON en el `SKILL.md`.
- No necesitas saturar las descripciones de las herramientas en `config.ts`, ya que el "manual de instrucciones" completo reside en el skill y será leído bajo demanda.


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

### 🛡️ Desarrollo Responsable: Whitelisting
Si tu MCP tiene acceso a recursos sensibles (archivos, procesos, red), es **obligatorio** implementar una capa de validación en tu Plugin.
1. Define un campo en `settings` del `config.ts` (ej: `ALLOWED_APPS`).
2. En tu `executeAction`, lee ese valor desde el `configStore`.
3. Valida la petición contra ese valor. Si falla, retorna un error descriptivo: `"ATENCIÓN IA: El recurso solicitado está fuera de la lista blanca permitida por el usuario."`.
