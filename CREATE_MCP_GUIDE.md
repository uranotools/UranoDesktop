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
7. [🎨 API Nativa de Renderizado UI y Badges](#7--api-nativa-de-renderizado-ui-y-badges)

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
    enabledPlugins: ['NombrePlugin'],
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

    // Delimitación de Entorno (Opcional)
    inCloud: true,      // Si se define como false, el módulo se ignora en la Nube (Desktop-Only)
    inDesktop: true,    // Si se define como false, el módulo se ignora en el Escritorio (Cloud-Only)
    // cloudOnly: false,// Alternativa: si es true, equivale a inDesktop: false

    // 1. Opciones de Entorno (Bóveda / UI visual)
    settings: [
        { name: 'SLACK_TOKEN', type: 'password', title: 'Bot OAuth Token' },
        // perAgent: true habilita que cada agente pueda sobreescribir este ajuste individualmente en su panel de configuración.
        { name: 'DEFAULT_CHANNEL', type: 'text', title: 'Canal Default (Ej: #general)', perAgent: true },
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
                        { name: 'attachmentFolder', type: 'dir', label: 'Carpeta de Adjuntos' },
                    ]
                }
            }
        }
    }
};
```

### Tipos de Campos en `fields`
Urano Front renderiza automáticamente los formularios basándose en el atributo `type`:
- `required`: Campo de texto obligatorio.
- `text`: Campo de texto opcional.
- `password`: Campo oculto (Bóveda).
- `dir`: **(Urano >= 1.3.5)** Selector de directorios nativo de Windows.
- `select`: Menú desplegable (requiere atributo `options: [{ label, value }]`).
- `prompt`: Área de texto multilínea para bloques de instrucciones.
- `iframe_picker`: **(Urano >= 1.5.5)** Selector interactivo de archivos/carpetas o vistas controladas (Google Drive, OneDrive, formularios propios, etc.). Soporta ejecución de SDKs en cliente (`onLaunch`), carga de IFrames (`iframeUrl`) y callbacks normalizadores (`callback`).

> **NOTA SOBRE SEGURIDAD:** Todos los campos variables estipulados en `settings` alimentan el motor criptográfico local de Urano. Los agentes jamas ven los tokens estáticos en sus contextos.

### Bóveda de Ajustes por Agente (Per-Agent Settings Vault)
**Urano >= 1.5.0** introduce la capacidad de sobreescribir valores de `settings` de manera individual por cada agente. 

Si deseas que un parámetro (como un modo de seguridad, un canal por defecto o una ruta local) sea configurable por cada agente, añade `perAgent: true` a su especificación de setting:
* **UI automática:** El panel de configuración de agente (`AgentConfigPanel` / `CloudAgentConfigPanel`) detectará el flag y mostrará un mini-formulario de override inline debajo de la tarjeta del MCP cuando el agente tenga activada esa integración.
* **Fallback global:** Si un agente no configura un valor personalizado en su panel, el motor utilizará automáticamente el valor global configurado en el Vault Manager como fallback.
* **Campos soportados:** El flag `perAgent: true` puede aplicarse a **cualquier tipo de campo** soportado en los formularios de Urano (`FieldRenderer.tsx`), incluyendo `text`, `number`, `money`, `textarea`, `email`, `checkbox`, `switch`, `select`, `date`, `color`, `directory`, `code`, `counter`, e `iframe_picker`. Por razones lógicas de seguridad, los únicos campos excluidos son los de tipo `password` (como API keys globales), los cuales **no** pueden ser sobreescritos individualmente por agente y siempre se resolverán globalmente desde la bóveda segura.

---

### Selectores Interactivos e IFrame Pickers (`iframe_picker`)
**Urano >= 1.5.5** introduce soporte para selectores interactivos universales. Esto permite a los desarrolladores de plugins MCP integrar de forma directa herramientas visuales avanzadas de selección (como exploradores de Google Drive, OneDrive, Dropbox o formularios de control propios) de forma 100% nativa.

El plugin puede definir este ajuste interactivo en su `config.ts` de dos formas distintas:

#### A. Integración Nativa de SDKs en Navegador (`onLaunch`)
Recomendado cuando la API/Servicio ofrece un SDK de javascript cargable en cliente (como Google Drive Picker). No necesitas hospedar ni programar ninguna página web: defines la función de inicialización directamente en el setting.

```typescript
{
    name: 'SYNC_FOLDER_DATA',
    type: 'iframe_picker',
    title: '📂 Carpeta de Google Drive',
    description: 'Selecciona la carpeta de Drive para lectura del Agente.',
    placeholder: 'Ninguna carpeta seleccionada',
    perAgent: true, // Habilita override específico por Agente

    // Se ejecuta al hacer clic en "Examinar" en la interfaz
    onLaunch: async (currentValue: any, resolve: (rawResult: any) => void, sdk: any) => {
        try {
            // 1. Recuperamos de forma segura credenciales guardadas en el Vault de este mismo plugin
            const clientId = await sdk.getSecret('GOOGLE_CLIENT_ID');
            
            // 2. Cargamos dinámicamente scripts oficiales evitando bloqueos de CORS
            await sdk.loadScript('https://apis.google.com/js/api.js');
            await sdk.loadScript('https://accounts.google.com/gsi/client');
            
            // 3. Inicializamos e invocamos el SDK interactivo
            const tokenClient = google.accounts.oauth2.initTokenClient({
                client_id: clientId,
                scope: 'https://www.googleapis.com/auth/drive.readonly',
                callback: (tokenRes) => {
                    const picker = new google.picker.PickerBuilder()
                        .addView(google.picker.ViewId.FOLDERS)
                        .setOAuthToken(tokenRes.access_token)
                        .setCallback((data) => {
                            if (data.action === google.picker.Action.PICKED) {
                                // 4. Retornamos el resultado de vuelta a Urano
                                resolve(data);
                            }
                        })
                        .build();
                    picker.setVisible(true);
                }
            });
            tokenClient.requestAccessToken();
        } catch (err) {
            sdk.showError("Error al iniciar el selector de Google: " + err.message);
        }
    },

    // Callback normalizador: toma la respuesta cruda y extrae la estructura a almacenar
    callback: (rawPayload: any) => {
        if (rawPayload && rawPayload.docs) {
            const normalized = rawPayload.docs.map((doc: any) => ({
                id: doc.id,
                name: doc.name,
                url: doc.url
            }));
            return { success: true, data: normalized };
        }
        return { success: false, error: 'Formato inválido' };
    }
}
```

#### B. Integración vía IFrame Web (`iframeUrl`)
Útil cuando ya tienes una página o portal interactivo administrado y deseas incrustarlo directamente en un iframe sandboxed dentro de la modal de Urano.

```typescript
{
    name: 'CUSTOM_SELECTOR',
    type: 'iframe_picker',
    title: '🌐 Selector de CRM',
    iframeUrl: '/module/MyCRM/settings/picker', // Ruta de tu vista
    iframeHeight: '380px',
    callback: (data: any) => {
        return { success: true, data };
    }
}
```
* **Bridge Contextual automático:** El iframe cargado en pantalla se cargará enriquecido con query params del entorno (`?token=...&theme=dark&platform=desktop`).
* **Protocolo de Selección:** Tu IFrame simplemente debe notificar al parent cuando el usuario elija un elemento usando `postMessage`:
  ```javascript
  window.parent.postMessage({ type: 'urano_config_select', value: rawSelectedData }, '*');
  ```

#### El Objeto `sdk` inyectado en `onLaunch`
* `sdk.allValues`: Objeto JSON con los valores en tiempo real del formulario actual.
* `sdk.loadScript(url: string): Promise<void>`: Carga y cachea de forma segura scripts en el navegador.
* `sdk.openPopup(url: string, title?: string, w?: number, h?: number): Window`: Abre ventanas emergentes de forma segura para autenticaciones OAuth.
* `sdk.getSecret(key: string): Promise<string | null>`: Recupera de forma **encriptada y aislada** secretos guardados en el Vault para este módulo (Bypass seguro local/cloud).
* `sdk.showError(message: string): void`: Despliega un banner rojo de error en los ajustes del formulario.

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

### 🪄 Variables de Contexto Inyectadas (La Magia del Core)

Al declarar tus herramientas en el `config.ts`, **NUNCA** debes pedirle al agente (LLM) que adivine el ID de la sesión (`sessionId`) o el ID del usuario. Esto genera alucinaciones y consume tokens innecesarios.

El motor de ejecución de Urano (`McpTool.ts`) intercepta secretamente todas las peticiones justo antes de ejecutar tu método, y le **inyecta parámetros adicionales** garantizados al objeto `payload`:

- `_sessionId`: El UUID exacto de la sesión de chat actual.
- `_parentSessionId`: Si el plugin fue invocado desde una pestaña secundaria o sub-agente, este es el ID de la sesión madre.
- `_userId`: El identificador del usuario local actual.

**Ejemplo de uso correcto:**
```typescript
async executeAction(action: string, payload: any) {
    // ❌ INCORRECTO: Obliga al LLM a inventar un ID (propenso a fallos)
    // const sid = payload.sessionId; 
    
    // ✅ CORRECTO: Extraído directamente del motor, garantizado 100%
    const sid = payload._sessionId; 
    
    if (action === 'sendMessage') {
        // ...
```

> [!NOTE]
> **Resolución de Nombres (Urano >= 1.3.5):** El sistema ahora resuelve el nombre del módulo de forma **insensible a mayúsculas** (`case-insensitive`) escaneando el sistema de archivos. Esto significa que si tu carpeta es `SlackIntegration`, las llamadas dirigidas a `slackintegration` funcionarán correctamente.
```typescript
    async executeAction(action: string, payload: any) {
        if (action === 'capture') {
            const rawBuffer = await miLibreriaDeCaptura();
            
            // Patrón A: Mensaje Multimodal Mixto (Urano >= 1.2.5)
            // Útil para retornar texto + imagen de forma explícita.
            // return [
            //    { type: 'text', text: 'Aquí tienes la captura:' },
            //    { type: 'image', image: rawBuffer.toString('base64'), mimeType: 'image/png' }
            // ];

            // Patrón B: Vision Interception (Recomendado Urano >= 1.4.0)
            // Al retornar un objeto con base64 y mimeType, el RuntimeLoop activará 
            // la intercepción automática, moviendo la imagen a un rol de usuario 
            // para evitar bloat en el contexto.
            return {
                path: payload.targetPath,
                base64: rawBuffer.toString('base64'),
                mimeType: 'image/png'
            };
        }
    }
```
> **¿Qué hace Urano con estos objetos?** Para garantizar que APIs que no soportan visión en roles de herramientas (como OpenAI vía OpenRouter) no colapsen la memoria al tratar los Base64 como texto crudo, el **RuntimeLoop** interceptará tu respuesta si contiene `base64` y `mimeType`. 
> 1. Dejará una mención en texto en el log de la herramienta.
> 2. Recreará un mensaje invisible bajo el `role: "user"` justo abajo con la imagen real. 
> De esta forma, el modelo recibe la imagen de forma 100% nativa y segura.

---

## 5. Inyectando Badges desde de tu Plugin MCP (Opcional)

Si desarrollas una herramienta MCP que lanza procesos de fondo, abre sub-pestañas, u obtiene documentos extensos, tal vez desees ofrecer al usuario un clic de acceso rápido arriba de su barra de texto. Puedes inyectar un **SessionBadge** universal:

```typescript
// Desde tu archivo MyPlugin.ts
public async apiEjecutarTarea(payload: any) {
    // [IMPORTANTE] Evita importar @core directamente en plugins externos/workspace
    // ya que los alias de ruta no se resuelven fuera del entorno compilado de Urano.
    // En su lugar, usa la capacidad inyectada por el sistema en this.config._callPlugin.

    await this.config._callPlugin(
        'Core',              // Módulo destino
        'SessionManager',    // O la abstracción que necesites
        'updateBadge',       // Acción específica
        {
            sessionId: payload.sessionId,
            badge: {
                id: 'mi-reporte',
                label: 'Abrir Reporte Generado',
                icon: '📊',
                color: 'success',
                actionRoute: '/module/MyPlugin/plugins/Report/apiAbrirReporte',
                actionData: { reportId: '123' }
            }
        }
    );
    
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

Aquí tienes un ejemplo completo y real de un archivo `SKILL.md` (basado en el módulo MiniChat de Urano):

```markdown
---
name: MiniChat
description: Capacidades de visión contextual, captura de pantalla y control del floating chat para asistencia en escritorio.
tools: [urano_minichat_capture_get_consent, urano_minichat_capture_set_consent, urano_minichat_capture_capture_screen_with_consent, urano_minichat_tools_open]
type: mcp
---

# Skill: MiniChat Desktop Intelligence

Este módulo otorga al agente "ojos" sobre el escritorio del usuario y la capacidad de controlar la ventana de chat flotante para ofrecer una experiencia de asistencia proactiva y contextual.

## Herramientas de Visión (Capture)

### `urano_minichat_capture_capture_screen_with_consent`
Captura la pantalla actual del usuario, respetando su configuración de privacidad.
- **Uso**: Fundamental cuando el usuario dice "Mira esto", "¿Qué opinas de lo que tengo abierto?", o cuando necesitas contexto visual para resolver una duda técnica.
- **Resultado**: Devuelve una confirmación textual y una imagen en base64 que se inyecta automáticamente en tu contexto de visión (si tu modelo soporta visión).

## Protocolo de Uso

1.  **Visión Contextual**: Siempre que el usuario haga referencias deícticas ("esto", "aquí", "esta ventana"), utiliza `capture_screen_with_consent` para obtener la verdad visual.
2.  **Privacidad**: Si el usuario deniega el consentimiento, no insistas excesivamente. Explica por qué la visión te ayudaría a ser más útil.

> [!IMPORTANT]
> Cuando realices una captura de pantalla exitosa, la imagen NO aparecerá en el historial de texto como una URL, sino que se enviará directamente a tu arquitectura multimodal. Podrás "verla" en el siguiente paso del loop de razonamiento.
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

## 6. Desarrollo, Empaquetado y Distribución (Marketplace)

Urano implementa un robusto **Ecosistema de Plugins (Marketplace)** y un **Modo Desarrollador (Live Test)**. Atrás quedaron los días de compilar y subir ZIPs manualmente durante la fase de desarrollo.

### 🛠️ Modo Desarrollador (Dev Mode / Live Test)

Para probar tu plugin en tiempo real mientras escribes código, utiliza la pestaña **Desarrollador** en el **MCP Manager** de Urano:

1.  **Vincular Carpeta (Symlink):** En la interfaz, haz clic en "Vincular Carpeta Local" y selecciona el directorio raíz de tu proyecto MCP (donde está tu `config.ts`).
2.  **Edición en Caliente (Hot Reload):** Urano creará un enlace simbólico (Junction en Windows) hacia tu bóveda. Un servicio en segundo plano (`fs.watch`) monitorizará tus archivos TypeScript/JavaScript.
3.  **Recarga Automática:** Cuando guardas un archivo, Urano invalidará el caché de Node (`require.cache`) y recargará tus esquemas y clases en milisegundos. Verás el evento `HOT_RELOAD` en el panel de actividad.
4.  **Soporte TypeScript Nativo:** Los desarrolladores no necesitan pre-compilar a JavaScript durante el desarrollo. Urano transpilara tus archivos `.ts` al vuelo.
5.  **No se requiere código fuente:** Los desarrolladores no necesitan compilar Urano Desktop ni tener acceso a su código fuente para probar sus plugins nativamente.
6.  **Librerías Externas (NPM):** Si tu plugin necesita paquetes externos (ej. `axios`), simplemente abre una terminal en tu carpeta, ejecuta `npm init -y` e instala tus librerías con `npm install`. Durante el Modo Dev, Urano resolverá tu carpeta `node_modules` local automáticamente. Al finalizar, el comando `--bundle` empaquetará estas librerías para el Marketplace.

### 📦 Construcción y Bundling (esbuild)

Instalar carpetas enteras de `node_modules` directamente dentro de un archivo final no es escalable, protege poco tu propiedad intelectual y genera problemas de límites de ruta (MAX_PATH en Windows).

La **práctica mandatoria** para distribuir tu MCP es crear un bundle con **esbuild**:

1.  Instala las dependencias: `npm install --save-dev esbuild`
2.  Ejecuta esbuild apuntando a tus puntos de entrada agregando el flag para empaquetar dependencias y excluir los módulos nativos:
    ```bash
    npx esbuild config.ts Plugins/**/*.ts --bundle --platform=node --outdir=dist --format=cjs --external:@core/*
    ```
    * **¿Por qué `--bundle`?** El ejecutable de Urano `.exe` no trae dependencias externas de terceros (como `axios`, `ethers`, etc.). Este flag empaqueta tus librerías npm dentro del plugin para que funcione sin instalación (Plug & Play).
    * **¿Por qué `--external:@core/*`?** Los módulos internos de Urano (`@core`, `@models`) ya existen en memoria. Si los empaquetas, causarás errores fatales de duplicación y fallos de `Cannot find module`.
3.  **Resultado:** Obtendrás un código minificado y unificado en la carpeta `dist`.
4.  Copia cualquier archivo estático necesario (como `SKILL.md` o imágenes) a la carpeta `dist`.
5.  Comprime **el contenido** de la carpeta `dist` en un archivo `.zip`. *(No comprimas la carpeta dist en sí, sino los archivos que están dentro).*

> [!TIP]
> **¿Tu editor (VSCode) te da error al importar `@core/...`?**
> Para solucionar el subrayado rojo, crea un archivo `tsconfig.json` en la raíz de tu plugin:
> ```json
> {
>   "compilerOptions": {
>     "baseUrl": ".",
>     "paths": {
>       "@core/*": ["*"],
>       "@models/*": ["*"],
>       "@modules/*": ["*"]
>     }
>   }
> }
> ```
> *(Nota: Esto silencia el error visual. En ejecución, Urano inyectará los módulos reales).*

### 🚀 Publicación en el Marketplace (GitHub Registry)

Urano gestiona su Marketplace a través de un archivo remoto `registry.json` alojado en un repositorio público de GitHub.

1.  **Sube tu Release:** Crea un Release en tu repositorio de GitHub o en cualquier servidor accesible públicamente, y sube tu archivo `.zip` compilado como un asset.
2.  **Solicita Inclusión (Pull Request):** Modifica el archivo `registry.json` del repositorio oficial del ecosistema Urano añadiendo la entrada de tu plugin:

```json
{
  "id": "MiSuperMcp",
  "name": "Super MCP",
  "author": "Tu Nombre",
  "description": "Breve descripción de qué hace el plugin.",
  "version": "1.0.0",
  "category": "Productividad",
  "icon": "Zap", 
  "tags": ["herramienta", "ia"],
  "downloadUrl": "https://github.com/tu-usuario/tu-repo/releases/download/v1.0.0/misupermcp.zip",
  "verified": false
}
```

### 🔄 Sistema de Versionamiento y Actualizaciones

El backend de Urano Desktop se encargará del resto:
-   **Instalación Remota:** El usuario hace clic en "Instalar" y Urano descarga el ZIP, valida la integridad (Zip Slip protection) y lo extrae en la bóveda aislada de producción.
-   **Archivo de Control:** Urano genera automáticamente un archivo `_installed.json` dentro de la carpeta del plugin para rastrear la `version` actual instalada y el `source` (registry o dev-link).
-   **Actualizaciones Automáticas:** Un servicio en segundo plano revisa el `registry.json` periódicamente (cada 6 horas). Si detecta una versión superior (SemVer), notifica al usuario en la pestaña de Marketplace para actualizar a 1-clic. Antes de actualizar, Urano crea un backup automático de la versión anterior en `.mcp_backups`.

### 🛡️ Desarrollo Responsable: Whitelisting
Si tu MCP tiene acceso a recursos sensibles (archivos, procesos, red), es **obligatorio** implementar una capa de validación en tu Plugin.
1. Define un campo en `settings` del `config.ts` (ej: `ALLOWED_APPS`).
2. En tu `executeAction`, lee ese valor desde el `configStore`.
3. Valida la petición contra ese valor. Si falla, retorna un error descriptivo: `"ATENCIÓN IA: El recurso solicitado está fuera de la lista blanca permitida por el usuario."`.

### 🛡️ Arquitectura de Seguridad (Vault & Scanner)
Urano implementa un modelo de confianza cero (Zero Trust) para los plugins instalados:
1.  **Aislamiento de Secretos (Module-Level Isolation)**: El `Vault` de Urano utiliza `AsyncLocalStorage` para restringir el acceso criptográfico. Un plugin en ejecución **solo puede solicitar secretos que pertenezcan a su propio módulo**. Si un plugin malicioso intenta acceder a las llaves de AWS de otro módulo (ej. `Vault.getSecret('AWS', 'ACCESS_KEY')`), Urano bloqueará la solicitud y devolverá `null`.
2.  **Escáner de Seguridad Estático**: Cuando un usuario vincula una carpeta de desarrollo o instala un `.zip`, el sistema analiza estáticamente el código fuente del plugin en busca de vulnerabilidades o API peligrosas (`child_process`, `fs`, `eval`, `net`).
3.  **Transparencia al Usuario**: Si el escáner detecta capacidades críticas, Urano inyectará una alerta visual de seguridad (🛡️) en el Marketplace y en el Dashboard, para que el usuario o el desarrollador estén informados sobre qué partes del sistema está solicitando controlar el plugin antes de que decida encenderlo.

---

## 7. Creando MCPs con Conciencia de Audio (Audio-Aware Plugins)

Urano incluye un **Audio Engine nativo** (`useAudioEngine`) que permite capturar voz del usuario sin que tu MCP tenga que manejar micrófonos directamente. Los chunks de voz ya transcritos llegan a tu plugin como texto plano.

### Patrón: Plugin que procesa chunks de voz en tiempo real

Ideal para MCPs como presentaciones en vivo, asistentes de dictado, o comandos de voz.

```typescript
// Plugins/Presentation/PresentationPlugin.ts
export class PresentationPlugin {
    private currentStageIndex = 0
    private stages: { keywords: string[]; uiSpec: any }[] = []

    // Acción estándar: carga el guión
    async apiLoadpresentation(payload: { stages: any[] }) {
        this.stages = payload.stages
        this.currentStageIndex = 0
        return { loaded: true, totalStages: this.stages.length }
    }

    // Acción de alta frecuencia: recibe chunks de voz directamente del AudioEngine
    // Esta acción NO invoca el LLM en cada llamada — es pura lógica de Plugin.
    async apiProcessvoicechunk(payload: { chunk: string }) {
        const current = this.stages[this.currentStageIndex]
        if (!current) return { action: 'none' }

        // Verificar si el chunk contiene palabras clave de la etapa actual
        const match = current.keywords.some(kw =>
            payload.chunk.toLowerCase().includes(kw.toLowerCase())
        )

        if (match) {
            this.currentStageIndex++
            return { action: 'advance', newStage: this.currentStageIndex }
        }

        return { action: 'none' }
    }
}
```

### Cómo el Frontend llama tu plugin directamente (sin agente)

Para latencia ultra-baja (< 600ms), el frontend puede bypasear el LLM y llamar tu plugin directo:

```typescript
// En el componente React que controla la sesión
const handleVoiceChunk = async (text: string) => {
    const res = await window.electronAPI.customIpcCall(
        '/module/LivePresenter/plugins/Presentation/processvoicechunk',
        { chunk: text }
    )
    if (res.action === 'advance') {
        // Disparar generateDynamicUI en MultiverseTabs para la nueva etapa
    }
}

// Pasarlo al AudioEngine
audioEngine.start({
    mode: 'chunk',
    onChunk: handleVoiceChunk
})
```

### Flujo Completo: Voz → Plugin → Visual

```
Micrófono
   ↓  (Web Speech API, ~150ms)
AudioEngine.onChunk(text)
   ↓  (IPC directo, <10ms)
PresentationPlugin.processVoiceChunk()
   ↓  (lógica interna, <5ms)
generateDynamicUI() → MultiverseTab
   ↓  (React render, ~100ms)
Visual actualizado
━━━━━━━━━━━━━━━━━━━━━━━━━━━━
Total: ~265ms (sin LLM) ✅
```

> [!TIP]
> Reserva el LLM para generar el **contenido** de los stages al inicio (cuando cargas el guión), no para procesar cada fragmento de voz. Así obtienes la máxima inteligencia con la mínima latencia operativa.

### Selección de Dispositivo de Audio por el Usuario

Desde el **AgentsDashboard > Configuración de Audio**, el usuario puede elegir el micrófono que se usará. No necesitas gestionar esto en tu plugin.


## 8. Renderizado Dinámico y el Patrón LivePresenter

A partir de la versión 1.3.0, los módulos MCP pueden renderizar interfaces React complejas en el frontend utilizando el módulo **MultiverseTabs**. Esto permite que un agente no solo "hable", sino que "construya" aplicaciones en tiempo real.

### Invocando un Renderizado desde un Plugin

Si tu plugin necesita mostrar un dashboard, una gráfica o una presentación, debe invocar la acción `generatedynamicui` de MultiverseTabs.

```typescript
// Plugins/Visualization/VizPlugin.ts
public async apiRenderDashboard(payload: { data: any, sessionId: string }) {
    // Invocamos el plugin de MultiverseTabs utilizando el helper inyectado
    // Esto garantiza compatibilidad tanto en modo Dev como en Producción.
    await this.config._callPlugin('MultiverseTabs', 'Tabs', 'generatedynamicui', {
        sessionId: payload.sessionId,
        purpose: 'dashboard-analytics',
        spec: {
            title: "Análisis de Datos Pro",
            component: "DynamicGrid", // Componente React registrado en UranoFront
            props: {
                dataset: payload.data,
                refreshInterval: 5000
            }
        }
    });

    return { status: 'Rendering started in Multiverse' };
}
```

### El Patrón de "Ejecución Híbrida" (Recomendado)

Para una experiencia de usuario fluida, sigue este patrón en tus plugins:

1.  **Respuesta Estática Inmediata**: Retorna un resumen en texto o un `MessagePart[]` multimodal para que el usuario sepa que la acción comenzó.
2.  **Mejora Dinámica Asíncrona**: Dispara el `generatedynamicui` de trasfondo para abrir la pestaña interactiva.

```typescript
async executeAction(action: string, payload: any) {
    if (action === 'analyze') {
        // 1. Disparar UI pesada asíncronamente
        this.apiRenderDashboard(payload); 

        // 2. Retornar confirmación instantánea al Agent context
        return "He iniciado el análisis visual en una nueva pestaña del Multiverso. Mientras se carga, puedo confirmarte que los datos preliminares muestran un crecimiento del 20%.";
    }
}
```

---

## 9. Lista Blanca y Seguridad (FileSystem)

Si tu módulo maneja archivos locales, es vital que el agente sepa qué rutas tiene permitidas de antemano.

1.  **Acción `listAllowed`**: Implementa siempre una acción que devuelva el contenido de tus `settings` de rutas autorizadas.
2.  **Protocolo en `SKILL.md`**: Instruye al agente para que **siempre** ejecute `listAllowed` antes de intentar leer o escribir archivos. Esto evita errores de permisos (`ENOENT`) y mejora la confianza del modelo.

```markdown
# Instrucciones de FileSystem
Antes de realizar cualquier operación de archivos, DEBES llamar a `urano_filesystem_localfiles_listallowed` para conocer las rutas seguras autorizadas por el usuario. No intentes acceder a rutas fuera de esta lista.
```


## 🛠️ Protocolo de Resiliencia y Estabilidad (AI SDK v6)

Urano Desktop implementa capas de protección automática en el **RuntimeLoop** para garantizar que los agentes MCP no corrompan las sesiones:

### 1. Limpieza de Herramientas Huérfanas
Si un turno es interrumpido (cancelación manual, crash de red), el motor detecta automáticamente los mensajes de tipo `assistant` que contienen llamadas a herramientas sin resultado. Antes de iniciar un nuevo turno, el sistema limpia estas llamadas para evitar el error `AI_MissingToolResultsError`.

### 2. Validación Nativa de Visión (Hybrid Pattern)
Antes de inyectar imágenes en el contexto de un plugin MCP (ej. capturas de pantalla), el sistema valida si el modelo configurado soporta visión (GPT-4o, Claude 3.5, etc.). 

**Lógica de Intercepción**: Para asegurar la estabilidad, el **RuntimeLoop** intercepta automáticamente cualquier respuesta de herramienta (ya sea un objeto plano o dentro de un Array) que contenga las propiedades `base64` o `image`. En lugar de dejar el binario pesado en la respuesta de la herramienta, lo extrae y lo mueve a un mensaje inyectado adyacente, dejando un texto placeholder ligero en su lugar.

**Estándar de Retorno para Desarrolladores:**
Para que tus herramientas nativas o servidores MCP envíen archivos multimodales (imágenes) al LLM de forma segura sin provocar errores de validación de tokens, simplemente retorna la data usando una de estas estructuras:

```javascript
// Opción 1: Objeto Directo (Ej. LocalFilesPlugin)
return {
    base64: "iVBORw0KGgo...", 
    mimeType: "image/png" 
};

// Opción 2: Partes Nativas de AI SDK (Ej. DesktopPlugin / SystemEye)
return [
    { type: "text", text: "Imagen procesada" },
    { type: "image", image: "iVBORw0KGgo...", mimeType: "image/png" }
];
```
Al cumplir este estándar, no importa si tu tool es MCP o Nativo: el motor de Urano manejará la imagen de la misma manera universal, asegurando que llegue limpia a los modelos multimodales.

---

## 🎨 Extensión de UI: Session Badges

Para módulos MCP que generan documentos o abren contextos externos, se recomienda utilizar el **API de Session Badges**. Esto permite inyectar botones nativos interactivos de acceso rápido directamente en la conversación del usuario. 

Sin embargo, dado que los plugins externos no deben importar `@core`, deben realizar esta actualización a través de `_callPlugin` (ver sección siguiente).

---

## 🔌 Limitaciones de Imports y el Patrón Inyectado

Los módulos MCP en Urano Desktop no están limitados a responder al LLM; tienen acceso a las capacidades del motor de Urano. Sin embargo, existe una distinción crítica entre módulos **Nativos** y módulos **Externos (Workspace)**.

### 🔌 Accediendo a `@core` en Plugins Externos

A diferencia de versiones anteriores, **SÍ puedes utilizar importaciones directas de `@core`** en tus plugins externos. Dado que Urano registra `module-alias` a nivel global en la aplicación principal, cualquier llamado a `require('@core/...')` funcionará nativamente sin importar en qué directorio de tu computadora resida tu plugin.

Sin embargo, para que esto funcione correctamente en tu entorno de desarrollo (sin que TypeScript lance errores por no encontrar el código fuente de Urano), debes realizar dos configuraciones clave:

1. **Silenciar los Errores de TypeScript (Opcional pero recomendado)**: Como instalaste Urano vía `.exe` y no tienes el repositorio fuente localmente, VSCode subrayará `@core` en rojo. Para solucionarlo, crea un archivo `urano.d.ts` en la raíz de tu proyecto MCP con este contenido básico:
   ```typescript
   declare module '@core/PluginBase' {
       export class PluginBase { constructor(config: any); }
   }
   declare module '@core/Security/Vault' {
       export class Vault { static getSecret(module: string, key: string): string; }
   }
   declare module '@core/Router' {
       export const Router: any;
   }
   ```
   *(También puedes simplemente usar `// @ts-ignore` arriba de tus imports).*

2. **Excluir `@core` en esbuild (OBLIGATORIO)**: Es crítico indicarle a `esbuild` que **NO** intente empaquetar `@core` en tu ZIP, ya que estas clases serán provistas por el ejecutable de Urano en tiempo de ejecución. Añade el flag `--external:@core/*`:
   ```bash
   npx esbuild config.ts Plugins/**/*.ts --bundle --platform=node --outdir=dist --format=cjs --external:@core/*
   ```

Gracias a este patrón, puedes importar clases nativas de forma limpia:
`import { PluginBase } from '@core/PluginBase';`
`import { Vault } from '@core/Security/Vault';`

### 🔀 Comunicación entre Plugins Externos (`_callPlugin`)

```typescript
constructor(moduleConfig: any) {
    this.config = moduleConfig;
    // this.config._callPlugin ahora está disponible
}

private async callOtherPlugin(targetModule, targetPlugin, action, data) {
    return await this.config._callPlugin(
        targetModule, 
        targetPlugin, 
        action, 
        data
    );
}
```

---

## 🎙️ Integración con el Audio Engine Nativo

Urano cuenta con un sistema nativo de captura de voz (`useAudioEngine`) disponible en el frontend. Los módulos MCP pueden recibir audio como entrada de los siguientes modos:

### Cómo llegan los chunks de voz a tu MCP

El `AudioEngine` transcribe el audio y lo envía al agente **como texto plano** a través del canal IPC normal (`agentSessionSend`). Tu MCP no necesita escuchar el micrófono directamente: el agente recibirá el texto ya procesado.

Para un MCP como `LivePresenter` que necesita reaccionar a cada chunk sin invocar el LLM, se puede registrar un handler directo:

```typescript
// En tu Plugin, expón una acción para recibir chunks de voz del frontend
async apiProcessvoicechunk(payload: { chunk: string, sessionId: string }) {
    // Lógica interna: comparar chunk con stage actual, decidir si avanzar
    // Sin invocar el LLM si no es necesario
    return { action: 'none' | 'advance' | 'update' }
}
```

El frontend llama directamente al IPC:
```typescript
// Desde AudioEngine en modo LivePresenter (Phase 2)
window.electronAPI.customIpcCall('/module/LivePresenter/plugins/Presentation/processvoicechunk', {
    chunk: transcribedText,
    sessionId: activeSessionId,
})
```

> [!TIP]
> Este patrón (Audio → Plugin directo, sin LLM por chunk) es la clave para lograr latencia **< 600ms** de voz a visual en tiempo real.

### Selección de Dispositivo de Audio

Los usuarios pueden seleccionar su micrófono preferido desde el **AgentsDashboard > Configuración de Audio**. El `deviceId` seleccionado se pasa al `AudioEngine.start()` y está disponible para Whisper en implementaciones futuras.

---

## ✋ Protocolo de Aprobación Interactiva (Tool Approval)

A partir de la versión reciente, el motor de Urano soporta pausar la ejecución de herramientas que realizan acciones destructivas o críticas (como ejecución de comandos de terminal, envío de correos masivos, etc.) para solicitar confirmación interactiva al usuario.

### ¿Cómo implementarlo en un Plugin Nativo?

Al definir la herramienta en el método `apiList`, simplemente añade la propiedad `requiresApproval: true`:

```typescript
export default class SystemTerminal extends PluginBase {
    async apiList() {
        return [{
            name: 'execute_command',
            description: 'Ejecuta comandos de terminal. El motor pausará y pedirá permiso al usuario.',
            parametersSchema: z.object({
                command: z.string()
            }),
            requiresApproval: true, // <-- Activa la pausa en el motor y la UI
            execute: async (args) => {
                // Solo se ejecutará si el usuario hace clic en "Aprobar" en el frontend
                return await miEjecutor(args.command);
            }
        }];
    }
}
```

### Ciclo de Vida del Evento:
1. El Agente decide usar la herramienta y el motor intercepta `requiresApproval: true`.
2. El **RuntimeLoop** pausa su hilo asíncrono y emite el evento IPC `agent-tool-approval-requested`.
3. El **Dashboard (React)** transforma la burbuja de estado en un diálogo amarillo interactivo de "Aprobación Requerida" mostrando los argumentos (ej. el comando a ejecutar) junto a dos botones: Aprobar o Rechazar.
4. Al hacer clic, se emite el IPC de vuelta `agent-tool-approval-response`.
5. Si es aprobado, el **RuntimeLoop** resume y ejecuta la función `execute()`. Si es rechazado, se aborta y se devuelve un `ToolError` diciendo "User rejected the action", permitiendo que el Agente entienda que se le denegó el permiso.

---

## ⚡ Patrón: Rate Limiter para APIs de Terceros (Serial Queue)

### El Problema

El **RuntimeLoop** ejecuta todas las herramientas generadas en un mismo turno usando `Promise.all()`. Esto significa que si el agente emite tres tool calls en un mismo paso:

```
user_feed()  ─┐
user_stats() ─┼─ Promise.all → disparo simultáneo → 429 Rate Limit
search()      ─┘
```

Si las tres tools llaman a la misma API externa con límite de 1 req/seg (como apiexternalexample), las tres requests HTTP llegan concurrentemente y las últimas dos fallan con `429 Too Many Requests`.

### Por Qué NO Añadir una Bandera `parallel: false` al Core

Podría parecer tentador añadir `parallel: false` al schema de la tool en `config.ts` para que el `RuntimeLoop` las ejecute en serie. Sin embargo, esto no es la solución correcta porque:

- Requeriría cambiar `ModuleConfig.ts`, `SkillRegistry.ts`, `McpTool.ts` y `RuntimeLoop.ts` — alto riesgo por un caso muy específico.
- Haría que **todas** las herramientas del turno esperasen, incluso las que no usan la API limitada.
- Violaría el principio de responsabilidad única: el RuntimeLoop no debe conocer las políticas de rate-limiting de APIs externas.

### La Solución: Cola Serial a Nivel HTTP (Serial Queue)

La solución correcta es implementar un **semáforo de promesas** que serialice únicamente las llamadas HTTP a la API limitada, **completamente invisible** para el agente y el RuntimeLoop.

**Implementación de referencia** (ver `PluginNombrePlugin.ts`):

```typescript
// ── Constantes de rate limiting ────────────────────────────────────────────
const MI_API_MIN_INTERVAL_MS = 1100; // 1.1s entre requests (margen sobre 1/s)
let miApiLastCall = 0;
let miApiQueue: Promise<any> = Promise.resolve();

// Encola la función 'fn' para ejecución serial con delay mínimo entre calls.
function miApiEnqueue<T>(fn: () => Promise<T>): Promise<T> {
    const result = miApiQueue.then(async () => {
        const now = Date.now();
        const wait = MI_API_MIN_INTERVAL_MS - (now - miApiLastCall);
        if (wait > 0) await new Promise(r => setTimeout(r, wait));
        miApiLastCall = Date.now();
        return fn();
    });
    // Encadenar para que la siguiente llamada espere ESTA, no solo la anterior.
    miApiQueue = result.catch(() => {});
    return result;
}

// Wrapper de fetch que usa la cola
async function miFetch(path: string, apiKey: string): Promise<any> {
    return miApiEnqueue(async () => {
        const res = await fetch(`https://api.mi-servicio.com${path}`, {
            headers: { 'Authorization': `Bearer ${apiKey}` }
        });
        // Manejo de 429 con retry automático usando Retry-After
        if (res.status === 429) {
            const retryAfter = parseInt(res.headers.get('Retry-After') || '1', 10);
            await new Promise(r => setTimeout(r, retryAfter * 1000 + 100));
            const retry = await fetch(`https://api.mi-servicio.com${path}`, {
                headers: { 'Authorization': `Bearer ${apiKey}` }
            });
            if (!retry.ok) throw new Error(`API error ${retry.status}`);
            return retry.json();
        }
        if (!res.ok) throw new Error(`API error ${res.status}: ${await res.text()}`);
        return res.json();
    });
}
```

### Cómo Funciona la Cola

```
Promise.all dispara:       user_feed()   user_stats()   search()
                                │               │              │
                           llaman a:        miFetch()     miFetch()     miFetch()
                                │               │              │
                         miApiEnqueue()  miApiEnqueue() miApiEnqueue()
                                │               │              │
                         ┌──────▼───────────────▼──────────────▼──────┐
                         │          COLA SERIAL (miApiQueue)           │
                         │   Req 1 → [espera 1.1s] → Req 2 → [1.1s]  │
                         │   → Req 3                                   │
                         └────────────────────────────────────────────┘
```

El `Promise.all` del RuntimeLoop termina cuando **las tres promesas resuelven**, lo cual sucede secuencialmente con 1.1s de separación. Desde el punto de vista del agente y del RuntimeLoop, la diferencia es solo de latencia, no de comportamiento.

### Cuándo Usar Este Patrón

Usa una cola serial cuando tu plugin integra una API externa que:

| Condición | Ejemplo |
|-----------|---------|
| Tiene rate limit de escritura O lectura | apiexternalexample: ~1 req/s, Twitter API: 300 req/15min |
| El agente puede llamar múltiples herramientas del mismo proveedor en un turno | `user_feed` + `user_stats` + `search` → todas usan apiexternalexample |
| El error de rate limit no es recuperable en esa misma request | `429` sin Retry-After header |

> [!TIP]
> La cola es un **singleton a nivel de módulo** (declarada fuera de la clase del plugin). Esto garantiza que incluso si múltiples sesiones del mismo agente están activas simultáneamente, todas las llamadas a la API comparten la misma cola y respetan el rate limit global.

> [!NOTE]
> Si la API no retorna el header `Retry-After`, usa un valor conservador de 1–2 segundos como fallback. El retry es opcional pero muy recomendado para evitar que un pico momentáneo de tráfico derribe toda la funcionalidad.

> [!CAUTION]
> **Timeout Crítico**: Cuando uses una cola serial, es obligatorio que el `fetch` dentro de la cola tenga un timeout (usando `AbortController`). Si una petición se queda colgada sin timeout, **bloqueará toda la cola** para el resto de herramientas y usuarios, dejando el módulo inutilizable hasta reiniciar.
> También se recomienda un timeout para el tiempo de espera *en* la cola (ej. abortar si la petición lleva >15s esperando su turno).

---

## 💎 Patrones Avanzados MCP (Urano Standard)

Para asegurar la consistencia entre módulos complejos (como UranoMaps o SystemEye), se han estandarizado los siguientes patrones de desarrollo:

### 1. Gestión de Ventanas por Sesión (`label:sessionId`)

Cuando un plugin abre pestañas en `MultiverseTabs`, debe evitar duplicar ventanas innecesariamente y permitir que el agente recupere contextos específicos.

**Patrón de Registro:**
```typescript
// Registro dinámico: "nombre_amigable:sessionId" -> tabId
const dashboardRegistry = new Map<string, string>();
const dashboardParams = new Map<string, any>();

async apiLaunch(params: { label: string, sessionId: string, ... }) {
    const key = `${params.label}:${params.sessionId}`;
    let tabId = dashboardRegistry.get(key);
    
    if (tabId && this.tabExists(tabId)) {
        // Si ya existe, solo la enfocamos y actualizamos
        return this.apiUpdate({ tabId, ... });
    }
    
    // Si no existe, creamos una nueva y registramos
    tabId = await this.createNewTab();
    dashboardRegistry.set(key, tabId);
    return { tabId, status: 'launched' };
}
```

### 2. Persistencia de Estado de Ventanas (`state.json`)

Los plugins que manejan datos en memoria (alertas, órdenes, posiciones) deben persistir su estado en el `userData` del usuario para sobrevivir a reinicios.

**Ubicación Recomendada:**
- **Windows**: `%APPDATA%/Urano Desktop/[plugin]_state.json`
- **Fallback**: `~/.urano/[plugin]_state.json`

**Implementación de Referencia:**
```typescript
function saveState() {
    const p = path.join(os.homedir(), '.urano', 'mi_plugin_state.json');
    const data = { registry: Object.fromEntries(dashboardRegistry) };
    fs.writeFileSync(p, JSON.stringify(data, null, 2));
}
```

### 3. Lógica de Captura y Caché de Archivos

Para que un agente pueda "ver" el contenido de una ventana o mapa, el plugin debe implementar una herramienta de captura que genere URLs locales.

**Estándares de Captura:**
1. **Directorio de Caché**: Siempre usar `path.join(os.homedir(), '.urano', 'cache', 'screenshots')`.
2. **Retorno Multimodal**: La herramienta debe devolver un array con el path local y el base64 para que el `RuntimeLoop` lo procese.

**Ejemplo de retorno:**
```typescript
return [
    { type: 'text', text: `Captura guardada en: ${filePath}` },
    { type: 'image', image: buffer.toString('base64'), mimeType: 'image/png' }
];
```

### 4. Estándares de Renderizado Dinámico (`JsonLive`)

Al crear nuevos componentes UI (como el widget `Map`), se deben seguir estas reglas para asegurar compatibilidad con `JsonLiveRenderer`:

- **Atomicidad**: Cada componente debe ser autónomo y usar `extractStyle()` para sus propiedades CSS.
- **Patching Fluido**: El plugin debe preferir `patchDynamicUI` sobre `generateDynamicUI` para actualizaciones de tiempo real (evita parpadeos).
- **Estructura de Nodo**:
    ```json
    {
      "type": "NuevoComponente",
      "props": { "glass": true, "glow": "#ff0000", ... },
      "children": [ ... ]
    }
    ```

> [!TIP]
> Si tu componente requiere librerías pesadas (como Leaflet o ECharts), impleméntalo usando **Lazy Loading** en el frontend para no degradar el tiempo de carga inicial de la aplicación.

---

## 🔒 Sistema de Permisos OS Genérico

Urano implementa un mecanismo centralizado para que cualquier MCP pueda solicitar permisos a nivel de sistema operativo de manera nativa, con **verificación real** (no solo registro) antes de guardar el estado.

### Tipos de Permiso Soportados

| `permissionType` | Descripción | macOS | Windows | Linux |
|---|---|---|---|---|
| `screen` | Captura de pantalla | Dialog del sistema / Ajustes | Configuración de privacidad | Ninguno requerido |
| `microphone` | Acceso al micrófono | `askForMediaAccess` nativo | `ms-settings:privacy-microphone` | Ninguno requerido |
| `camera` | Acceso a la cámara | `askForMediaAccess` nativo | `ms-settings:privacy-webcam` | Ninguno requerido |
| `desktop-audio` | Audio del escritorio (loopback) | ⚠️ Requiere dispositivo virtual | WASAPI loopback vía renderer | PulseAudio/PipeWire |

> [!WARNING]
> En **macOS**, la captura de audio del escritorio (`desktop-audio`) no está disponible de forma nativa. Se requiere instalar **BlackHole** (gratuito) o **Loopback** y configurarlo como salida de audio del sistema.

### 1. Definición en `config.ts`
Agrega un campo de tipo `button` en el array de `settings` de tu módulo. El `name` servirá como slug en la bovéda (`Vault`) para persistir si el usuario otorgó el permiso.

```typescript
// config.ts de tu módulo
settings: [
    {
        name: 'capture-permission',          // Slug del permiso guardado en Vault
        title: 'Captura de Pantalla',
        type: 'button',
        buttonText: 'Solicitar Permiso OS',
        description: 'Permite al agente ver el contexto visual de tu pantalla',
        actionRoute: 'mcp-request-os-permission',
        actionPayload: { permissionType: 'screen' },
    },
    {
        name: 'microphone-access',
        title: 'Acceso al Micrófono',
        type: 'button',
        buttonText: 'Solicitar Permiso OS',
        description: 'Permite al agente escuchar audio del micrófono',
        actionRoute: 'mcp-request-os-permission',
        actionPayload: { permissionType: 'microphone' },
    },
    {
        name: 'desktop-audio-access',
        title: 'Audio del Escritorio',
        type: 'button',
        buttonText: 'Solicitar Permiso OS',
        description: 'Permite al agente escuchar el audio del sistema (loopback)',
        actionRoute: 'mcp-request-os-permission',
        actionPayload: { permissionType: 'desktop-audio' },
    },
],
```

### 2. Comportamiento Automático
- El frontend inyecta automáticamente `moduleName` y `permissionSlug` (= `name` del campo) en el payload.
- El backend **verifica el permiso real** antes de guardar:
  - `screen`: Captura un thumbnail real; si está vacío, retorna error y abre los Ajustes del Sistema en macOS.
  - `microphone`/`camera`: Usa `systemPreferences.getMediaAccessStatus()` en macOS para verificar el estado antes de pedir; en caso de `'denied'` abre directamente la pantalla de privacidad.
  - `desktop-audio`: Informa al usuario sobre limitaciones de plataforma.
- El estado `'granted'` se guarda en el Vault **solo si el permiso es real**.
- Si la captura posterior detecta que el thumbnail está vacío, **revoca automáticamente** el estado `'granted'` y resetea el badge a ámbar.

### 3. Leer el permiso desde un Plugin

Verifica si el permiso fue otorgado consultando directamente el `Vault` desde tu plugin:

```typescript
import { Vault } from '../../../../core/Security/Vault';

async apiMyAction() {
    const isGranted = Vault.getSecret(this.moduleName, 'capture-permission') === 'granted';
    if (!isGranted) {
        return { success: false, consentRequired: true, message: 'Permiso no otorgado' };
    }
    // ... usar el permiso
}
```

> [!TIP]
> El `McpManager` muestra automáticamente la sección "Permisos del Sistema" con badges de estado para todos los campos de tipo `button`. Los badges en ámbar indican que se requiere acción; los verdes que el permiso está otorgado y verificado.

---

### 4. Usar los Permisos Dentro de un Plugin

Una vez que el usuario otorgó el permiso, aquí tienes la implementación de referencia para cada tipo.

#### 📸 `screen` — Capturar la pantalla

Usa `desktopCapturer` de Electron directamente desde el proceso **main**. No requiere el renderer.

```typescript
// En tu Plugin (Proceso Main — Node.js / Electron)
import { desktopCapturer } from 'electron';
import { Vault } from '../../../../core/Security/Vault';

export default class MiPlugin extends PluginBase {

    async apiCaptureScreen() {
        // 1. Verificar permiso
        const isGranted = Vault.getSecret(this.moduleName, 'capture-permission') === 'granted';
        if (!isGranted) {
            return { success: false, consentRequired: true, message: 'Permiso de captura no otorgado. Ve a la configuración del módulo.' };
        }

        // 2. Capturar
        const sources = await desktopCapturer.getSources({
            types: ['screen'],
            thumbnailSize: { width: 1920, height: 1080 },
        });

        const primary = sources[0];
        const thumb = primary?.thumbnail;

        if (!thumb || thumb.isEmpty()) {
            // Revocar si el OS ya no lo permite
            Vault.deleteSecret(this.moduleName, 'capture-permission');
            return { success: false, message: 'El SO denegó la captura. El permiso fue revocado.' };
        }

        const base64 = thumb.toPNG().toString('base64');

        // 3. Retornar imagen — el RuntimeLoop la inyectará como parte multimodal
        return [
            { type: 'text', text: 'Captura de pantalla tomada exitosamente.' },
            { type: 'image', image: base64, mimeType: 'image/png' },
        ];
    }
}
```

---

#### 🎙️ `microphone` — Grabar audio del micrófono

La captura de micrófono ocurre en el **proceso renderer** (React/Web Audio API), no en el main. Tu plugin expone una herramienta que el agente llama; el frontend reacciona y graba.

**Paso 1 — Plugin expone la herramienta (Main Process):**

```typescript
// En tu Plugin
export default class MiPlugin extends PluginBase {

    async apiStartMicCapture(payload: { durationMs?: number }) {
        const isGranted = Vault.getSecret(this.moduleName, 'microphone-access') === 'granted';
        if (!isGranted) {
            return { success: false, consentRequired: true, message: 'Permiso de micrófono no otorgado.' };
        }

        // Emitir evento al renderer para que inicie la grabación
        // El renderer escucha 'plugin-mic-start' y retorna el audio vía IPC
        this.emitToRenderer('plugin-mic-start', {
            durationMs: payload.durationMs || 5000,
            sessionId: this.sessionId,
        });

        return { success: true, message: 'Grabación de micrófono iniciada.' };
    }
}
```

**Paso 2 — Renderer captura y devuelve el audio:**

```typescript
// En tu componente React (Renderer)
api.on('plugin-mic-start', async ({ durationMs, sessionId }) => {
    const stream = await navigator.mediaDevices.getUserMedia({ audio: true, video: false });
    const recorder = new MediaRecorder(stream);
    const chunks: BlobPart[] = [];

    recorder.ondataavailable = (e) => chunks.push(e.data);
    recorder.onstop = async () => {
        const blob = new Blob(chunks, { type: 'audio/webm' });
        const buffer = await blob.arrayBuffer();
        const base64 = btoa(String.fromCharCode(...new Uint8Array(buffer)));
        // Enviar de regreso al main process
        api.customIpcCall('plugin-mic-result', { sessionId, base64, mimeType: 'audio/webm' });
        stream.getTracks().forEach(t => t.stop());
    };

    recorder.start();
    setTimeout(() => recorder.stop(), durationMs);
});
```

> [!NOTE]
> Si ya tienes `useAudioEngine` activo en el frontend, puedes reciclarlo directamente en lugar de crear un nuevo `MediaRecorder`. Consulta la sección **Integración con el Audio Engine Nativo** de esta guía.

---

#### 🔊 `desktop-audio` — Capturar audio del escritorio (loopback)

Solo disponible en **Windows** (y Linux). Captura lo que está sonando en el sistema en tiempo real.

**Paso 1 — Plugin emite la señal (Main Process):**

```typescript
export default class MiPlugin extends PluginBase {

    async apiStartDesktopAudio(payload: { durationMs?: number }) {
        const isGranted = Vault.getSecret(this.moduleName, 'desktop-audio-access') === 'granted';
        if (!isGranted) {
            return { success: false, consentRequired: true, message: 'Permiso de audio del escritorio no otorgado.' };
        }
        if (process.platform === 'darwin') {
            return { success: false, message: 'macOS no soporta captura de audio del escritorio nativamente. Instala BlackHole.' };
        }

        this.emitToRenderer('plugin-desktop-audio-start', {
            durationMs: payload.durationMs || 5000,
            sessionId: this.sessionId,
        });

        return { success: true, message: 'Captura de audio del escritorio iniciada.' };
    }
}
```

**Paso 2 — Renderer captura con WASAPI loopback:**

```typescript
// En el renderer (React) — solo funciona en Windows/Linux
api.on('plugin-desktop-audio-start', async ({ durationMs, sessionId }) => {
    // Obtener fuentes de escritorio con audio habilitado
    const sources = await (window as any).electronAPI.customIpcCall(
        'desktop-capturer-get-sources', { types: ['screen'], fetchWindowIcons: false }
    );

    const stream = await navigator.mediaDevices.getUserMedia({
        audio: {
            mandatory: {
                chromeMediaSource: 'desktop',
                chromeMediaSourceId: sources[0]?.id,  // Primera pantalla
            }
        } as any,
        video: false,
    });

    const recorder = new MediaRecorder(stream);
    const chunks: BlobPart[] = [];

    recorder.ondataavailable = (e) => chunks.push(e.data);
    recorder.onstop = async () => {
        const blob = new Blob(chunks, { type: 'audio/webm' });
        const buffer = await blob.arrayBuffer();
        const base64 = btoa(String.fromCharCode(...new Uint8Array(buffer)));
        api.customIpcCall('plugin-desktop-audio-result', { sessionId, base64, mimeType: 'audio/webm' });
        stream.getTracks().forEach(t => t.stop());
    };

    recorder.start();
    setTimeout(() => recorder.stop(), durationMs);
});
```

> [!IMPORTANT]
> Para que `getUserMedia` con `chromeMediaSource: 'desktop'` funcione en el renderer de Electron, la ventana debe tener habilitado `contextIsolation: false` o el preload debe exponer los IDs de las fuentes de captura. En Urano, esto se gestiona mediante el IPC `desktop-capturer-get-sources`.

> [!WARNING]
> En **macOS**, este bloque de código fallará silenciosamente porque el sistema no expone el audio del sistema a través de `getUserMedia`. Verifica con `process.platform === 'darwin'` antes de ejecutar.

---

## 10. Control de Hardware y Sistema Operativo (OS-Level)

A diferencia de los entornos en la nube (como ChatGPT), **los plugins de Urano Desktop se ejecutan localmente en la máquina del usuario bajo Node.js**. Esto significa que tus plugins tienen acceso directo al Sistema Operativo, hardware y periféricos, permitiéndote crear agentes verdaderamente autónomos en el mundo real.

### 🔌 Acceso Nativo a Node.js

No estás limitado a hacer peticiones web (HTTP). Puedes importar cualquier módulo nativo de Node.js o instalar dependencias pesadas en tu plugin:

*   **`child_process`**: Para ejecutar scripts de Python, Bash o PowerShell.
*   **`robotjs` / `@nut-tree/nut-js`**: Para simular teclado y ratón y controlar aplicaciones de terceros.
*   **`serialport`**: Para conectar Urano con hardware externo como Arduino, Raspberry Pi o domótica.

### Ejemplo: Plugin de Automatización de Escritorio (Macro)
Este ejemplo muestra cómo un Agente puede tomar control físico del teclado y ratón del usuario para realizar tareas aburridas o interactuar con videojuegos.

1. Abre la terminal en tu carpeta de plugin e instala la librería:
   ```bash
   npm init -y
   npm install @nut-tree/nut-js
   ```

2. Implementa la acción en tu clase de Plugin:

```typescript
import { keyboard, mouse, Key, Point } from "@nut-tree/nut-js";

export class DesktopMacroPlugin {
    async executeAction(action: string, payload: any) {
        if (action === 'press_shortcut') {
            // Ejemplo: El agente decide presionar Ctrl+S para guardar el trabajo del usuario
            await keyboard.pressKey(Key.LeftControl, Key.S);
            await keyboard.releaseKey(Key.LeftControl, Key.S);
            return "Atajo de teclado ejecutado con éxito.";
        }
        
        if (action === 'move_mouse') {
            // El agente mueve el ratón físicamente a una coordenada
            await mouse.setPosition(new Point(payload.x, payload.y));
            return `Ratón movido a las coordenadas ${payload.x}, ${payload.y}.`;
        }
    }
}
```

### ⚠️ Reglas de Seguridad para Control de OS

Dado que el control del sistema operativo es crítico y podría ser peligroso si el Agente comete un error, **Urano Desktop impone reglas estrictas**:

1.  **Aprobación Obligatoria (`requiresApproval: true`)**: Cualquier herramienta que ejecute comandos destructivos o macros impredecibles **DEBE** tener esta bandera activada en su schema (`config.ts`). El RuntimeLoop de Urano pausará el agente y le mostrará un botón amarillo al usuario pidiendo permiso explícito antes de ejecutar el comando. (Ver la sección "Protocolo de Aprobación Interactiva" más arriba).
2.  **Advertencia del Escáner Estático**: Cuando el usuario instale tu plugin ZIP, el Escáner de Seguridad de Urano detectará que usas librerías nativas como `child_process` o `serialport` y le mostrará un icono de advertencia 🛡️ en el Marketplace: *"Este plugin tiene acceso total al Sistema Operativo"*.

### 🌉 Acceso a la API Core Avanzada
Si desarrollaste el plugin usando la técnica de inyección de alias explicada anteriormente, tienes acceso directo a los subsistemas del motor local de Urano. Puedes usar el inyector dinámico para pedirle al Core que haga cosas nativas del SO por ti, sin tener que reprogramarlas:

```typescript
// En tu plugin, pidiendo al Core de Urano que lance una notificación nativa de Windows/macOS
await this.config._callPlugin('Core', 'NotificationManager', 'show', {
    title: "Agente de Hardware",
    body: "He terminado de imprimir la pieza 3D. ¿Qué hacemos ahora?"
});
```

---

## 7. 🎨 API Nativa de Renderizado UI y Badges

> **Novedad v2.0** — Esta API permite que cualquier plugin externo lance interfaces visuales ricas directamente en el chat, sin código de frontend.

### ¿Qué es PluginCore?

`PluginCore` es un módulo nativo de Urano (disponible como `@core/PluginCore`) que ofrece helpers de alto nivel para:

| Método | Descripción |
|---|---|
| `PluginCore.launchUI()` | Lanza una pestaña visual (JsonLiveRenderer) en MultiverseTabs |
| `PluginCore.addBadge()` | Inyecta un badge de acceso rápido **encima del input del chat** |
| `PluginCore.removeBadge()` | Elimina un badge por su ID |
| `PluginCore.updateUI()` | Actualiza en tiempo real una pestaña ya abierta |
| `PluginCore.focusTab()` | Trae al frente una pestaña existente |

### Configuración del empaquetado

Como todos los módulos `@core/*`, debes excluirlo del bundle al empaquetar:

```bash
# esbuild — marca @core como externo
esbuild src/index.ts --bundle --external:@core/* --outfile=dist/index.js
```

### Ejemplo completo: Lanzar un Dashboard + Badge

```typescript
import { PluginBase } from '@core/PluginBase';
import { PluginCore } from '@core/PluginCore';

export class MyDashboardPlugin extends PluginBase {

    /**
     * Herramienta MCP: el agente llama a esto cuando el usuario pide ver su dashboard.
     * El agente solo necesita saber el sessionId (viene automático como _sessionId).
     */
    async apiLaunchdashboard(params: { label?: string; _sessionId?: string }): Promise<any> {
        const sessionId = params._sessionId || '';
        const label = params.label || 'Mi Dashboard';

        // 1. Construir la especificación UI (JsonLiveRenderer)
        const uiSpec = {
            root: {
                type: 'Column',
                props: { padding: 24, gap: 16 },
                children: [
                    {
                        type: 'Hero',
                        props: {
                            title: `📊 ${label}`,
                            subtitle: 'Panel de control de tu plugin',
                            badge: 'LIVE'
                        }
                    },
                    {
                        type: 'Grid',
                        props: { columns: 3, gap: 16 },
                        children: [
                            {
                                type: 'Stat',
                                props: { label: 'Total', value: '42', color: 'info', icon: '📈' }
                            },
                            {
                                type: 'Stat',
                                props: { label: 'Activos', value: '8', color: 'success', icon: '✅' }
                            },
                            {
                                type: 'Stat',
                                props: { label: 'Errores', value: '0', color: 'error', icon: '⚠️' }
                            }
                        ]
                    },
                    {
                        type: 'Table',
                        props: {
                            title: 'Últimos eventos',
                            columns: ['Evento', 'Fecha', 'Estado'],
                            rows: [
                                { Evento: 'Sync completado', Fecha: '2025-05-17', Estado: 'OK' },
                                { Evento: 'Webhook recibido', Fecha: '2025-05-16', Estado: 'OK' }
                            ]
                        }
                    }
                ]
            }
        };

        // 2. Lanzar la UI (crea una pestaña nueva o hace focus en una existente)
        const result = await PluginCore.launchUI({
            sessionId,
            label,
            uiSpec,
            systemPrompt: `Eres el asistente del panel "${label}". Ayuda al usuario con sus datos.`,
            badgeIcon: 'LayoutDashboard',
            badgeColor: 'info'
        });

        if (result.success) {
            return {
                success: true,
                tabId: result.tabId,
                message: result.isNew
                    ? `Panel "${label}" lanzado correctamente.`
                    : `Panel "${label}" ya estaba abierto — traído al frente.`
            };
        }

        return { success: false, message: 'Error al lanzar el panel.' };
    }
}
```

### Flujo visual resultante

Cuando el agente llama a `apiLaunchdashboard`:

1. Aparece una nueva pestaña `🧩 Mi Dashboard` en el área MultiverseTabs.
2. **Inmediatamente** aparece un badge `📊 Mi Dashboard` sobre el input del chat.
3. Al hacer click en el badge, se lleva al usuario a la pestaña.
4. Si el usuario cierra la pestaña y vuelve a pedirla, el badge la reabre automáticamente.

### Actualización en tiempo real

Para modificar la UI sin destruirla (por ejemplo, actualizar una tabla de datos):

```typescript
async apiUpdatedashboard(params: { tabId: string; newData: any[] }): Promise<any> {
    await PluginCore.updateUI({
        tabId: params.tabId,
        patch: {
            root: {
                props: {
                    // Actualizar solo los datos de la tabla
                    rows: params.newData
                }
            }
        },
        purpose: 'Actualización de datos'
    });

    return { success: true, message: 'Panel actualizado.' };
}
```

### Catálogo de Widgets (JsonLiveRenderer)

La especificación `uiSpec` soporta más de 40 widgets nativos:

#### Layout
| Widget | Descripción | Props clave |
|---|---|---|
| `Column` | Flex vertical | `gap`, `padding`, `alignItems` |
| `Row` | Flex horizontal | `gap`, `justifyContent` |
| `Grid` | CSS Grid | `columns`, `gap` |
| `Box` | Contenedor libre | `backgroundColor`, `borderRadius` |
| `GlassPanel` | Panel glassmorphism | `backdropBlur`, `padding` |
| `Stack` | Posicionamiento absoluto | — |
| `Center` | Centrado ambos ejes | — |
| `ScrollView` | Scroll interno | `horizontal` |

#### Datos y contenido
| Widget | Descripción | Props clave |
|---|---|---|
| `Table` | Tabla con paginación | `columns`, `rows`, `pageSize` |
| `Chart` | ECharts (bar, line, pie) | `chartType`, `series`, `xAxis` |
| `Stat` | Métrica con icono | `label`, `value`, `color`, `delta` |
| `MetricGroup` | Grupo de métricas | `items[]` |
| `Markdown` | Texto con formato | `content` |
| `Code` | Bloque de código | `content`, `language` |

#### Interactividad
| Widget | Descripción | Props clave |
|---|---|---|
| `Button` | Botón con acción IPC | `label`, `variant`, `onClickAction` |
| `Form` | Formulario con inputs | `fields[]`, `submitRoute` |
| `Chip` | Chip clickeable | `label`, `color`, `onClickAction` |

#### Visual
| Widget | Descripción | Props clave |
|---|---|---|
| `Hero` | Banner superior | `title`, `subtitle`, `badge`, `imageUrl` |
| `Card` | Tarjeta contenedor | `title`, `content` |
| `Badge` | Etiqueta | `label`, `level` |
| `Avatar` | Imagen con iniciales | `src`, `name`, `size` |
| `Skeleton` | Cargador | `lines` |
| `Spinner` | Spinner animado | `size`, `color` |
| `Map` | Mapa TomTom (UranoMaps) | `center`, `zoom`, `markers` |

> 💡 **Tip**: Para ver todos los widgets con ejemplos interactivos, pide al agente de Urano: `"muéstrame el catálogo de widgets de JsonLiveRenderer"`.

### Seguridad y permisos

`PluginCore` usa los mismos canales IPC que los plugins nativos. No requiere permisos especiales adicionales. Los badges se limpian automáticamente cuando la sesión termina.

### Declarar la herramienta en `config.ts`

```typescript
export const MyDashboardConfig = {
    name: "MyDashboard",
    // ...
    pluginSchemas: {
        Dashboard: {
            actions: {
                launchdashboard: {
                    label: 'Abrir Dashboard',
                    description: 'Lanza el panel visual del plugin con estadísticas en tiempo real.',
                    fields: [
                        {
                            name: 'label',
                            label: 'Nombre del panel',
                            type: 'text',
                            required: false,
                            helpText: 'Nombre personalizado para la pestaña. Por defecto: "Mi Dashboard".'
                        }
                    ]
                }
            }
        }
    }
};
```

---

### Uso de Marcadores Multimodales Inteligentes (`[last_image]`)

Al construir plugins para Urano (tanto nativos como empaquetados externos), es común querer procesar archivos de imágenes enviados en el chat. En lugar de obligar al agente a extraer, descargar, o codificar las imágenes en base64 para pasarlas como argumentos pesados (lo cual degrada el rendimiento de la GPU/CPU y agota el límite de tokens), el core de Urano soporta **Marcadores Multimodales Inteligentes**.

#### ¿Cómo funcionan?
Si tu herramienta espera recibir un archivo de imagen en un parámetro (por ejemplo, `photo` o `imagePath`), puedes indicarle al agente en la descripción del parámetro que soporta el marcador `[last_image]`, `[last_photo]`, o `@last_image`.

Cuando el agente coloca uno de estos marcadores en la llamada, `McpTool.ts` intercepta la solicitud antes de que llegue a tu plugin:
1. Extrae la última imagen del historial de la sesión activa de forma asíncrona.
2. Crea un archivo temporal físico en el disco (`~/.urano/temp/last_image_{timestamp}_{random}.png`).
3. Sustituye dinámicamente el marcador por la **ruta absoluta de este archivo temporal**.

#### Ejemplo de Declaración en `config.ts`

```typescript
export const ImageProcessorConfig = {
    name: "ImageProcessor",
    settings: [],
    pluginSchemas: {
        Processor: {
            actions: {
                process_image: {
                    label: 'Procesar Imagen',
                    description: 'Analiza o transforma una imagen local.',
                    fields: [
                        {
                            name: 'photo',
                            label: 'Ruta de la imagen',
                            type: 'text',
                            required: true,
                            helpText: 'Ruta local absoluta de la imagen. Puedes pasar "[last_image]" para procesar la última foto del chat.'
                        }
                    ]
                }
            }
        }
    }
};
```

#### Ejemplo de Implementación en tu Plugin (`ProcessorPlugin.ts`)

Dado que el motor reemplaza el marcador con una ruta de archivo local absoluta antes de invocar la acción, tu código de plugin puede consumirla directamente usando el módulo estándar `fs` de NodeJS sin código adicional:

```typescript
import { PluginBase } from '@core/PluginBase';
import fs from 'fs';
import path from 'path';

export class ProcessorPlugin extends PluginBase {

    async apiProcess_image(data: { photo: string, _sessionId?: string }): Promise<any> {
        const imagePath = data.photo;

        // Comprobación estándar de archivo local (el core ya resolvió [last_image] a una ruta real)
        if (!fs.existsSync(imagePath)) {
            throw new Error(`No se encontró el archivo de imagen en la ruta: ${imagePath}`);
        }

        const fileStats = fs.statSync(imagePath);
        const fileName = path.basename(imagePath);

        // Procesar la imagen (ej: OCR, subida, compresión, etc.)
        // ...

        return {
            content: [
                {
                    type: 'text',
                    text: `Imagen procesada con éxito: ${fileName} (${(fileStats.size / 1024).toFixed(1)} KB)`
                }
            ]
        };
    }
}

---

## 8. Módulos Híbridos: Combinando MCP Tools y Engine Plugins

Una técnica de diseño sumamente potente y avanzada en Urano es la creación de **Módulos Híbridos**. Un solo paquete MCP puede actuar simultáneamente como un plugin MCP normal (exponiendo herramientas declarativas al LLM) y como un **Engine Plugin (Session Middleware)** (que intercepta silenciosamente el ciclo de vida de la sesión en segundo plano).

Esto permite, por ejemplo, que un módulo de `Gmail` o `Slack` provea herramientas al agente (ej: `gmail_send_email`) y, al mismo tiempo, maneje de manera desatendida badges visuales en la interfaz de chat (como `✉️ Correo Enviado`) o modifique el prompt de sistema dinámicamente según las acciones que realice la herramienta.

### Configuración Híbrida en `config.ts`

Para declarar un módulo híbrido, simplemente incluye tanto el objeto `pluginSchemas` como los campos de `enginePlugin` y `engineHooks` en tu manifiesto:

```typescript
export const GmailConfig = {
    name: "GmailIntegration",
    description: "Conexión con Gmail y middleware de auditoría de correos.",
    icon: "Mail",
    category: "Productividad",

    // 1. Declaración de Engine Plugin (Middleware de Sesión)
    enginePlugin: true,
    engineHooks: ['onSessionStart', 'preMessageProcess'],

    // 2. Declaración de MCP normal (Herramientas expuestas al LLM)
    pluginSchemas: {
        Inbox: {
            actions: {
                send_email: {
                    label: 'Enviar Correo',
                    description: 'Envía un email usando Gmail. Soporta adjuntos.',
                    fields: [
                        { name: 'to', label: 'Destinatario', type: 'text', required: true },
                        { name: 'subject', label: 'Asunto', type: 'text', required: true },
                        { name: 'body', label: 'Mensaje', type: 'textarea', required: true }
                    ]
                }
            }
        }
    }
};
```

### Estructura de Archivos del Módulo Híbrido

```text
📁 GmailIntegration/
├── 📄 config.ts
└── 📁 Plugins/
    ├── 📁 Inbox/
    │   └── 📄 InboxPlugin.ts                  <-- Resuelve la herramienta "send_email"
    └── 📁 Engine/
        └── 📄 GmailIntegrationEnginePlugin.ts <-- Middleware en segundo plano
```

### Comunicación entre MCP Tools y el Engine (Uso de Session Metadata)

Dado que las herramientas y el Engine Plugin se ejecutan para la misma sesión en el mismo proceso de Node/Bun, pueden comunicarse compartiendo información en la metadata de la sesión de forma completamente segura:

1. **La herramienta MCP guarda un estado temporal en la sesión**:
   En la acción `apiSend_email` de tu clase `InboxPlugin.ts`, puedes acceder al `_sessionId` (inyectado automáticamente en los datos de la llamada) y guardar valores en la sesión activa:
   ```typescript
   import { SessionManager } from '@core/runtime/SessionManager';

   export class InboxPlugin {
       async apiSend_email(data: any): Promise<any> {
           // 1. Enviar el correo electrónico mediante APIs...
           const sendResult = await gmailService.send(data);

           // 2. Almacenar información de la transacción en la sesión activa
           if (data._sessionId) {
               const session = SessionManager.getSession(data._sessionId);
               if (session) {
                   session.metadata['_engine_GmailIntegration_lastSubject'] = data.subject;
                   session.metadata['_engine_GmailIntegration_needsUpdate'] = true;
               }
           }

           return { status: 'success', id: sendResult.id };
       }
   }
   ```

2. **El Engine Plugin lee la metadata y actúa**:
   En el gancho `preMessageProcess` de `GmailIntegrationEnginePlugin.ts`, consumes el estado guardado y aplicas cambios en la UI o inyectas instrucciones en caliente:
   ```typescript
   import { EnginePluginBase } from '@core/EnginePluginBase';
   import type { SessionContext } from '@core/runtime/SessionContext';

   export class GmailIntegrationEnginePlugin extends EnginePluginBase {
       async preMessageProcess(ctx: SessionContext, message: any): Promise<any> {
           const lastSubject = ctx.getPluginData('lastSubject');
           const needsUpdate = ctx.getPluginData('needsUpdate');

           if (needsUpdate && lastSubject) {
               // Inyectamos un badge dinámico en la cabecera del chat en la UI
               ctx.addBadge({
                   id: 'gmail-sent',
                   label: `✉️ Enviado: "${lastSubject}"`,
                   color: 'success'
               });

               // Inyectamos instrucciones contextuales en caliente en el system prompt
               ctx.appendCustomInstructions(
                   `[SISTEMA]: Acabas de enviar con éxito un correo con asunto "${lastSubject}". Confírmaselo amablemente al usuario si te lo pregunta.`
               );

               // Reseteamos el flag de control
               ctx.setMetadata('needsUpdate', false);
           }

           return { status: 'continue' };
       }
   }
   ```

```

Esta solución garantiza compatibilidad transparente tanto para el entorno de Escritorio (Electron) como en la Nube (Docker/Hono), ya que las rutas temporales se adaptan dinámicamente al sistema de archivos local (`os.homedir()`).
```