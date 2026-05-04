<div align="center">
  <p><b>Spanish Version: <a href="./CREATE_MCP_GUIDE.md">CREATE_MCP_GUIDE.md</a></b></p>
</div>

# Urano MCP - Creation and Installation Guide

This technical guide is aimed at developers who wish to create and integrate external **MCP (Model Context Protocol) Packages** into the Urano Agent architecture. Here you will learn from basic concepts to advanced configurations for exposing dynamic or static tools and defining specialized contexts (`Skills`).

---

## Index
1. [Quick Start](#1-quick-start)
2. [MCP Project Structure](#2-mcp-project-structure)
3. [The `config.ts` File (Core Manifest)](#3-the-configts-file-core-manifest)
4. [Adding Logic to Plugins](#4-adding-logic-to-plugins)
5. [Injecting Context with `SKILL.md`](#5-injecting-context-with-skillmd)
6. [Packaging and Distribution (.zip)](#6-packaging-and-distribution-zip)

---

## 1. Quick Start

To get started immediately, here is all you need to do to create and install your first MCP called "HelloWorld":

1. Create a local folder called `HelloWorld`.
2. Inside that folder, create the `config.ts` file:
```typescript
export const HelloWorldConfig = {
    name: "HelloWorld",
    description: "My first trial MCP module",
    icon: "Rocket",
    category: "Development",
    pluginSchemas: {
        Tests: {
            actions: {
                ping: { label: 'Greet API', fields: [] }
            }
        }
    }
}
```
3. Create the plugin path `Plugins/Tests/TestsPlugin.ts`:
```typescript
export class TestsPlugin {
    async executeAction(action: string, data: any) {
        if (action === 'ping') return "Pong! Hello from MCP";
        throw new Error("Action not found");
    }
}
```
4. Compress the interior of the folder on disk as `HelloWorld.zip`.
5. Go to **Urano (Desktop or UI) > Integrations / MCP Manager**.
6. Click on **Install MCP (.zip)**, upload your file, and that's it! Agents with permissions will automatically be able to see and use your new tool `urano_helloworld_tests_ping`.

---

## 2. MCP Project Structure

A well-structured MCP package consists of central configurations, execution code, and contextual documentation. When you install a ZIP in Urano, the system extracts everything into the isolated vault `~/.urano/workspace/mcp/{ModuleName}/`.

```text
📁 MyMCPModule/
├── 📄 config.ts                 <-- (Required) Manifest and environment schemas
├── 📄 package.json              <-- (Optional) Defines own dependencies if required
├── 📄 SKILL.md                  <-- (Recommended) Serves as an injectable System Prompt (`type: mcp`)
└── 📁 Plugins/
    └── 📁 {PluginName}/
        └── 📄 {PluginName}Plugin.ts  <-- Execution class that resolves actions declared in config
```

---

## 3. The `config.ts` File (Core Manifest)

Your `config.ts` is the brain of the environment. Urano reads this file to render graphical interfaces and configure the password Vault transparently.

### Advanced Anatomy

```typescript
export const SlackConfig = {
    name: "SlackIntegration",
    description: "Connect to Slack directly in the business workspace",
    icon: "MessageSquare",
    category: "Communications",

    // 1. Environment Options (Vault / Visual UI)
    settings: [
        { name: 'SLACK_TOKEN', type: 'password', title: 'Bot OAuth Token' },
        { name: 'DEFAULT_CHANNEL', type: 'text', title: 'Default Channel (e.g., #general)' },
    ],

    // 2. Native MCP Server Connectors (Optional)
    // If your module is a wrapper for an open-source MCP module:
    mcpServer: {
        command: 'npx',
        args: ['-y', '@modelcontextprotocol/server-slack'],
        requiredEnv: ['SLACK_TOKEN']
    },

    // 3. Explicit Declaration of Tool Schemas
    pluginSchemas: {
        Chat: {
            actions: {
                sendMessage: {
                    label: 'Send Message',
                    fields: [
                        { name: 'channel', type: 'required', label: 'Channel ID' },
                        { name: 'text', type: 'required', label: 'Message body' },
                        { name: 'attachmentFolder', type: 'dir', label: 'Attachments Folder' },
                    ]
                }
            }
        }
    }
};
```

### Field Types in `fields`
Urano Front automatically renders forms based on the `type` attribute:
- `required`: Mandatory text field.
- `text`: Optional text field.
- `password`: Hidden field (Vault).
- `dir`: **(Urano >= 1.3.5)** Native Windows directory selector.
- `select`: Dropdown menu (requires `options: [{ label, value }]` attribute).
- `prompt`: Multi-line text area for instruction blocks.

> **SECURITY NOTE:** All variable fields stipulated in `settings` feed Urano's local cryptographic engine. Agents never see static tokens in their contexts.

---

## 4. Adding Logic to Plugins

The tools structured in `config.ts` under the `pluginSchemas` key expect a counterpart class under the tree `Plugins/{Name}/{Name}Plugin.ts`. 

For the previous schema (`Chat`), you would need to create `Plugins/Chat/ChatPlugin.ts`:

```typescript
export class ChatPlugin {
    private configStore: any;

    constructor(moduleConfigCompiledByUrano: any) {
        // In this step, Urano will inject decrypted vault keys in real-time
        this.configStore = moduleConfigCompiledByUrano; 
    }

    async executeAction(action: string, payload: any) {
        if (action === 'sendMessage') {
            const { channel, text } = payload;
            // ... Execute fetch or local logic here ...
            return { sent: true, timestamp: Date.now() };
        }
        throw new Error(`Action ${action} not supported in the Chat plugin`);
    }
}
```

Urano will dynamically link the bridge and create an official schema in OpenAI Functions format called `urano_slackintegration_chat_sendmessage`.

> [!NOTE]
> **Name Resolution (Urano >= 1.3.5):** The system now resolves the module name in a **case-insensitive** manner by scanning the file system. This means that if your folder is `SlackIntegration`, calls directed to `slackintegration` will work correctly.
```typescript
    async executeAction(action: string, payload: any) {
        if (action === 'capture') {
            const rawBuffer = await myCaptureLibrary();
            
            // Pattern A: Mixed Multimodal Message (Urano >= 1.2.5)
            // Useful for explicitly returning text + image.
            // return [
            //    { type: 'text', text: 'Here is the capture:' },
            //    { type: 'image', image: rawBuffer.toString('base64'), mimeType: 'image/png' }
            // ];

            // Pattern B: Vision Interception (Recommended Urano >= 1.4.0)
            // By returning an object with base64 and mimeType, the RuntimeLoop will activate 
            // automatic interception, moving the image to a user role 
            // to avoid context bloat.
            return {
                path: payload.targetPath,
                base64: rawBuffer.toString('base64'),
                mimeType: 'image/png'
            };
        }
    }
```
> **What does Urano do with these objects?** To ensure that APIs that do not support vision in tool roles (like OpenAI via OpenRouter) do not collapse memory by treating Base64 as raw text, the **RuntimeLoop** will intercept your response if it contains `base64` and `mimeType`. 
> 1. It will leave a text mention in the tool log.
> 2. It will recreate an invisible message under `role: "user"` right below with the actual image. 
> In this way, the model receives the image 100% natively and securely.

---

## 5. Injecting Badges from your MCP Plugin (Optional)

If you develop an MCP tool that launches background processes, opens sub-tabs, or retrieves extensive documents, you may want to offer the user a quick-access click above their text bar. You can inject a universal **SessionBadge**:

```typescript
// From your MyPlugin.ts file
public async apiExecuteTask(payload: any) {
    // [IMPORTANT] Avoid importing @core directly in external/workspace plugins
    // as path aliases do not resolve outside the compiled Urano environment.
    // Instead, use the capability injected by the system in this.config._callPlugin.

    await this.config._callPlugin(
        'Core',              // Target module
        'SessionManager',    // Or the abstraction you need
        'updateBadge',       // Specific action
        {
            sessionId: payload.sessionId,
            badge: {
                id: 'my-report',
                label: 'Open Generated Report',
                icon: '📊',
                color: 'success',
                actionRoute: '/module/MyPlugin/plugins/Report/apiOpenReport',
                actionData: { reportId: '123' }
            }
        }
    );
    
    return { success: true };
}
```

Urano Front will paint your badge using the indicated color palette and trigger the IPC Request to your provided endpoint transparently.

### Handling the Action in your Plugin

For the badge to be functional, your Plugin class must implement the method you defined in `actionRoute`. If your route is `/module/MyPlugin/plugins/Report/apiOpenReport`, your plugin must have the method `apiOpenReport`:

```typescript
// In src/main/Modules/MyPlugin/Plugins/Report/ReportPlugin.ts
export class ReportPlugin extends PluginBase {
    
    // This method is invoked when the user clicks on the Badge
    async apiOpenReport(payload: { reportId: string }) {
        console.log("The user requested to open the report:", payload.reportId);
        
        // Here you can launch processes, open local files, or
        // even inject a new message into the chat.
        return { success: true };
    }
}
```

> [!TIP]
> **About Imports**: Note that we use `await import(...)` to load `CoreFactory`. This is a **mandatory best practice** in Urano Desktop to avoid "Circular Dependencies" (cyclic import errors) since plugins are loaded at system startup before the Core is 100% initialized.

---

## 6. Injecting Context with `SKILL.md`

Loose API tools are not enough if you want to provide the LLM with intelligence on how to use your MCP, or if you want to teach it **specific business policies** when using the module.

---

### Is `SKILL.md` mandatory? (Urano >= 1.3.5)
Unlike previous versions, it is **no longer mandatory** to have a valid `SKILL.md` for the module's technical tools to register. If the file is missing or malformed, the agent will still be able to see and execute your actions defined in `config.ts`, but it will lack narrative instructions on *how* and *when* to use them.

> [!TIP]
> **Recommended Use:** Always include a `SKILL.md`. Without it, the agent may hallucinate with parameters or ignore security policies you define.

Here is a complete and real example of a `SKILL.md` file (based on Urano's MiniChat module):

```markdown
---
name: MiniChat
description: Contextual vision capabilities, screen capture, and floating chat control for desktop assistance.
tools: [urano_minichat_capture_get_consent, urano_minichat_capture_set_consent, urano_minichat_capture_capture_screen_with_consent, urano_minichat_tools_open]
type: mcp
---

# Skill: MiniChat Desktop Intelligence

This module gives the agent "eyes" over the user's desktop and the ability to control the floating chat window to offer a proactive and contextual assistance experience.

## Vision Tools (Capture)

### `urano_minichat_capture_capture_screen_with_consent`
Captures the user's current screen, respecting their privacy settings.
- **Use**: Fundamental when the user says "Look at this", "What do you think of what I have open?", or when you need visual context to resolve a technical doubt.
- **Result**: Returns a textual confirmation and a base64 image that is automatically injected into your vision context (if your model supports vision).

## Usage Protocol

1.  **Contextual Vision**: Whenever the user makes deictic references ("this", "here", "this window"), use `capture_screen_with_consent` to get the visual truth.
2.  **Privacy**: If the user denies consent, do not insist excessively. Explain why vision would help you be more useful.

> [!IMPORTANT]
> When you perform a successful screen capture, the image will NOT appear in the text history as a URL, but will be sent directly to your multimodal architecture. You will be able to "see" it in the next step of the reasoning loop.
```

### Why `type: mcp`?
Thanks to this tag in the `frontmatter`, Urano's **SkillRegistry** knows that this skill **should not be listed globally** in the universal "Agent Editor" panel, since otherwise, it would saturate the experience of a human editor listing endless irrelevant technical tools (e.g., *postgres databases, git connectors*). The skill remains hidden and available entirely to rehydrate in the background as long as the Agent has explicit *Authorized MCP Modules*.

### 🧠 The Skill-First Protocol (Mandatory)
Starting with version 1.3.0, Urano Core imposes a `<mcp_protocol>` block of instructions on the Agent. This block prohibits it from using your `urano_*` tools if it has not first executed `urano_read_skill`.

**What does this mean for you?**
- Your `SKILL.md` will **always** be consulted before the agent uses your API.
- You must include validation rules and clear JSON examples in the `SKILL.md`.
- You do not need to clutter tool descriptions in `config.ts`, as the full "instruction manual" resides in the skill and will be read on demand.


---

## 6. Development, Packaging, and Distribution (Marketplace)

Urano implements a robust **Plugin Ecosystem (Marketplace)** and a **Developer Mode (Live Test)**. Gone are the days of manually compiling and uploading ZIPs during the development phase.

### 🛠️ Developer Mode (Dev Mode / Live Test)

To test your plugin in real-time while writing code, use the **Developer** tab in Urano's **MCP Manager**:

1.  **Link Folder (Symlink):** In the interface, click on "Link Local Folder" and select the root directory of your MCP project (where your `config.ts` is).
2.  **Hot Reloading:** Urano will create a symbolic link (Junction in Windows) to your vault. A background service (`fs.watch`) will monitor your TypeScript/JavaScript files.
3.  **Automatic Reload:** When you save a file, Urano will invalidate the Node cache (`require.cache`) and reload your schemas and classes in milliseconds. You will see the `HOT_RELOAD` event in the activity panel.
4.  **No source code required:** Developers do not need to compile Urano Desktop or have access to its source code to test their plugins natively.

### 📦 Construction and Bundling (esbuild)

Installing entire `node_modules` folders directly inside a final file is not scalable, provides little protection for your intellectual property, and generates path limit issues (MAX_PATH on Windows).

The **mandatory practice** for distributing your MCP is to create a bundle with **esbuild**:

1.  Install dependencies: `npm install --save-dev esbuild`
2.  Run esbuild pointing to your entry points (example):
    `npx esbuild config.ts Plugins/**/*.ts --bundle --platform=node --outdir=dist --format=cjs`
3.  **Result:** You will get minified and unified code in the `dist` folder.
4.  Copy any necessary static files (such as `SKILL.md` or images) to the `dist` folder.
5.  Compress **the content** of the `dist` folder into a `.zip` file. *(Do not compress the dist folder itself, but the files inside).*

### 🚀 Publishing in the Marketplace (GitHub Registry)

Urano manages its Marketplace through a remote `registry.json` file hosted on a public GitHub repository.

1.  **Upload your Release:** Create a Release in your GitHub repository or any publicly accessible server, and upload your compiled `.zip` file as an asset.
2.  **Request Inclusion (Pull Request):** Modify the `registry.json` file of the official Urano ecosystem repository by adding your plugin entry:

```json
{
  "id": "MySuperMCP",
  "name": "Super MCP",
  "author": "Your Name",
  "description": "Brief description of what the plugin does.",
  "version": "1.0.0",
  "category": "Productivity",
  "icon": "Zap", 
  "tags": ["tool", "ai"],
  "downloadUrl": "https://github.com/your-user/your-repo/releases/download/v1.0.0/mysupermcp.zip",
  "verified": false
}
```

### 🔄 Versioning and Updates System

Urano Desktop's backend will handle the rest:
-   **Remote Installation:** The user clicks "Install" and Urano downloads the ZIP, validates integrity (Zip Slip protection), and extracts it into the isolated production vault.
-   **Control File:** Urano automatically generates an `_installed.json` file within the plugin folder to track the current `version` installed and the `source` (registry or dev-link).
-   **Automatic Updates:** A background service checks the `registry.json` periodically (every 6 hours). If it detects a higher version (SemVer), it notifies the user in the Marketplace tab to update with 1-click. Before updating, Urano creates an automatic backup of the previous version in `.mcp_backups`.

### 🛡️ Responsible Development: Whitelisting
If your MCP has access to sensitive resources (files, processes, network), it is **mandatory** to implement a validation layer in your Plugin.
1. Define a field in `settings` of `config.ts` (e.g., `ALLOWED_APPS`).
2. In your `executeAction`, read that value from the `configStore`.
3. Validate the request against that value. If it fails, return a descriptive error: `"AI ATTENTION: The requested resource is outside the white list allowed by the user."`.

---

## 7. Creating Audio-Aware MCPs

Urano includes a **native Audio Engine** (`useAudioEngine`) that allows capturing user voice without your MCP having to handle microphones directly. Already transcribed voice chunks arrive at your plugin as plain text.

### Pattern: Plugin that processes voice chunks in real-time

Ideal for MCPs such as live presentations, dictation assistants, or voice commands.

```typescript
// Plugins/Presentation/PresentationPlugin.ts
export class PresentationPlugin {
    private currentStageIndex = 0
    private stages: { keywords: string[]; uiSpec: any }[] = []

    // Standard action: loads the script
    async apiLoadpresentation(payload: { stages: any[] }) {
        this.stages = payload.stages
        this.currentStageIndex = 0
        return { loaded: true, totalStages: this.stages.length }
    }

    // High-frequency action: receives voice chunks directly from the AudioEngine
    // This action DOES NOT invoke the LLM in each call — it is pure Plugin logic.
    async apiProcessvoicechunk(payload: { chunk: string }) {
        const current = this.stages[this.currentStageIndex]
        if (!current) return { action: 'none' }

        // Check if the chunk contains keywords from the current stage
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

### How the Frontend calls your plugin directly (without agent)

For ultra-low latency (< 600ms), the frontend can bypass the LLM and call your plugin directly:

```typescript
// In the React component that controls the session
const handleVoiceChunk = async (text: string) => {
    const res = await window.electronAPI.customIpcCall(
        '/module/LivePresenter/plugins/Presentation/processvoicechunk',
        { chunk: text }
    )
    if (res.action === 'advance') {
        // Trigger generateDynamicUI in MultiverseTabs for the new stage
    }
}

// Pass it to the AudioEngine
audioEngine.start({
    mode: 'chunk',
    onChunk: handleVoiceChunk
})
```

### Full Flow: Voice → Plugin → Visual

```
Microphone
   ↓  (Web Speech API, ~150ms)
AudioEngine.onChunk(text)
   ↓  (Direct IPC, <10ms)
PresentationPlugin.processVoiceChunk()
   ↓  (internal logic, <5ms)
generateDynamicUI() → MultiverseTab
   ↓  (React render, ~100ms)
Updated Visual
━━━━━━━━━━━━━━━━━━━━━━━━━━━━
Total: ~265ms (without LLM) ✅
```

> [!TIP]
> Reserve the LLM to generate the **content** of the stages at the beginning (when you load the script), not to process each voice fragment. This way you get maximum intelligence with minimum operational latency.

### User Audio Device Selection

From **AgentsDashboard > Audio Configuration**, the user can choose the microphone to be used. You do not need to manage this in your plugin.


## 8. Dynamic Rendering and the LivePresenter Pattern

Starting with version 1.3.0, MCP modules can render complex React interfaces in the frontend using the **MultiverseTabs** module. This allows an agent to not just "talk", but to "build" applications in real-time.

### Invoking a Rendering from a Plugin

If your plugin needs to show a dashboard, a chart, or a presentation, it must invoke the `generatedynamicui` action of MultiverseTabs.

```typescript
// Plugins/Visualization/VizPlugin.ts
public async apiRenderDashboard(payload: { data: any, sessionId: string }) {
    // We invoke the MultiverseTabs plugin using the injected helper
    // This ensures compatibility in both Dev and Production modes.
    await this.config._callPlugin('MultiverseTabs', 'Tabs', 'generatedynamicui', {
        sessionId: payload.sessionId,
        purpose: 'dashboard-analytics',
        spec: {
            title: "Pro Data Analysis",
            component: "DynamicGrid", // React component registered in UranoFront
            props: {
                dataset: payload.data,
                refreshInterval: 5000
            }
        }
    });

    return { status: 'Rendering started in Multiverse' };
}
```

### The "Hybrid Execution" Pattern (Recommended)

For a smooth user experience, follow this pattern in your plugins:

1.  **Immediate Static Response**: Return a text summary or a multimodal `MessagePart[]` so the user knows the action has started.
2.  **Asynchronous Dynamic Improvement**: Trigger the background `generatedynamicui` to open the interactive tab.

```typescript
async executeAction(action: string, payload: any) {
    if (action === 'analyze') {
        // 1. Trigger heavy UI asynchronously
        this.apiRenderDashboard(payload); 

        // 2. Return instant confirmation to the Agent context
        return "I have started the visual analysis in a new Multiverse tab. While it loads, I can confirm that preliminary data shows a growth of 20%.";
    }
}
```

---

## 9. White List and Security (FileSystem)

If your module handles local files, it is vital that the agent knows which paths are allowed beforehand.

1.  **`listAllowed` Action**: Always implement an action that returns the content of your authorized path `settings`.
2.  **`SKILL.md` Protocol**: Instruct the agent to **always** execute `listAllowed` before attempting to read or write files. This avoids permission errors (`ENOENT`) and improves model confidence.

```markdown
# FileSystem Instructions
Before performing any file operations, you MUST call `urano_filesystem_localfiles_listallowed` to know the secure paths authorized by the user. Do not attempt to access paths outside this list.
```


## 🛠️ Resilience and Stability Protocol (AI SDK v6)

Urano Desktop implements automatic protection layers in the **RuntimeLoop** to ensure MCP agents do not corrupt sessions:

### 1. Cleanup of Orphaned Tools
If a turn is interrupted (manual cancellation, network crash), the engine automatically detects `assistant` type messages that contain tool calls without results. Before starting a new turn, the system cleans up these calls to avoid the `AI_MissingToolResultsError`.

### 2. Native Vision Validation (Hybrid Pattern)
Before injecting images into the context of an MCP plugin (e.g., screen captures), the system validates if the configured model supports vision (GPT-4o, Claude 3.5, etc.). 

**Interception Logic**: To ensure stability, the **RuntimeLoop** automatically intercepts any tool response (whether a plain object or within an Array) that contains the `base64` or `image` properties. Instead of leaving the heavy binary in the tool response, it extracts it and moves it to an adjacent injected message, leaving a light placeholder text in its place.

**Return Standard for Developers:**
For your native tools or MCP servers to securely send multimodal files (images) to the LLM without causing token validation errors, simply return the data using one of these structures:

```javascript
// Option 1: Direct Object (e.g., LocalFilesPlugin)
return {
    base64: "iVBORw0KGgo...", 
    mimeType: "image/png" 
};

// Option 2: AI SDK Native Parts (e.g., DesktopPlugin / SystemEye)
return [
    { type: "text", text: "Processed image" },
    { type: "image", image: "iVBORw0KGgo...", mimeType: "image/png" }
];
```
By following this standard, it doesn't matter if your tool is MCP or Native: the Urano engine will handle the image in the same universal way, ensuring it arrives clean to the multimodal models.

---

## 🎨 UI Extension: Session Badges

For MCP modules that generate documents or open external contexts, it is recommended to use the **Session Badges API**. This allows injecting interactive native quick-access buttons directly into the user's conversation. 

However, since external plugins should not import `@core`, they must perform this update via `_callPlugin` (see next section).

---

## 🔌 Import Limitations and the Injected Pattern

MCP modules in Urano Desktop are not limited to responding to the LLM; they have access to the capabilities of the Urano engine. However, there is a critical distinction between **Native** modules and **External (Workspace)** modules.

### ⚠️ The Alias Problem (`@core`)
Urano Desktop uses TypeScript aliases (such as `@core/*`). When you develop an external plugin in a separate folder:
1.  **In Development**: The on-the-fly transpiler cannot resolve `@core` because your folder is outside the project "root".
2.  **In Production**: The bundled code could lose the reference to the global Node context.

> [!CAUTION]
> **Direct Imports Prohibited**: Do not use `import { ... } from '@core'` in plugins intended for the Workspace. Your plugin will fail with a `MODULE_NOT_FOUND` error upon execution.

### ✅ The Solution: Injected Dependency (`_callPlugin`)
The system automatically injects a bridge function into the configuration object that your constructor receives.

```typescript
constructor(moduleConfig: any) {
    this.config = moduleConfig;
    // this.config._callPlugin is now available
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

### When to use `@core`?
You can only use `@core` imports if you are developing a **native** module that will be compiled along with Urano Desktop's main source code (within `src/main/Modules`).

---

## 🎙️ Integration with the Native Audio Engine

Urano has a native voice capture system (`useAudioEngine`) available in the frontend. MCP modules can receive audio as input in the following ways:

### How voice chunks arrive at your MCP

The `AudioEngine` transcribes the audio and sends it to the agent **as plain text** through the normal IPC channel (`agentSessionSend`). Your MCP does not need to listen to the microphone directly: the agent will receive the already processed text.

For an MCP like `LivePresenter` that needs to react to each chunk without invoking the LLM, a direct handler can be registered:

```typescript
// In your Plugin, expose an action to receive voice chunks from the frontend
async apiProcessvoicechunk(payload: { chunk: string, sessionId: string }) {
    // Internal logic: compare chunk with current stage, decide whether to advance
    // Without invoking the LLM if not necessary
    return { action: 'none' | 'advance' | 'update' }
}
```

The frontend calls the IPC directly:
```typescript
// From AudioEngine in LivePresenter mode (Phase 2)
window.electronAPI.customIpcCall('/module/LivePresenter/plugins/Presentation/processvoicechunk', {
    chunk: transcribedText,
    sessionId: activeSessionId,
})
```

> [!TIP]
> This pattern (Audio → Direct Plugin, without LLM per chunk) is the key to achieving voice-to-visual latency of **< 600ms** in real-time.

### Audio Device Selection

Users can select their preferred microphone from **AgentsDashboard > Audio Configuration**. The selected `deviceId` is passed to `AudioEngine.start()` and is available for Whisper in future implementations.

---

## ✋ Interactive Approval Protocol (Tool Approval)

As of the recent version, the Urano engine supports pausing the execution of tools that perform destructive or critical actions (such as terminal command execution, mass email sending, etc.) to request interactive confirmation from the user.

### How to implement it in a Native Plugin?

When defining the tool in the `apiList` method, simply add the `requiresApproval: true` property:

```typescript
export default class SystemTerminal extends PluginBase {
    async apiList() {
        return [{
            name: 'execute_command',
            description: 'Executes terminal commands. The engine will pause and ask the user for permission.',
            parametersSchema: z.object({
                command: z.string()
            }),
            requiresApproval: true, // <-- Activates pause in the engine and UI
            execute: async (args) => {
                // This will only execute if the user clicks "Approve" in the frontend
                return await myExecutor(args.command);
            }
        }];
    }
}
```

### Event Lifecycle:
1. The Agent decides to use the tool and the engine intercepts `requiresApproval: true`.
2. The **RuntimeLoop** pauses its asynchronous thread and emits the IPC event `agent-tool-approval-requested`.
3. The **Dashboard (React)** transforms the status bubble into an interactive yellow "Approval Required" dialog showing the arguments (e.g., the command to be executed) along with two buttons: Approve or Reject.
4. Upon clicking, the IPC response `agent-tool-approval-response` is emitted back.
5. If approved, the **RuntimeLoop** resumes and executes the `execute()` function. If rejected, it aborts and returns a `ToolError` saying "User rejected the action", allowing the Agent to understand that permission was denied.

---

## ⚡ Pattern: Rate Limiter for Third-Party APIs (Serial Queue)

### The Problem

The **RuntimeLoop** executes all tools generated in a single turn using `Promise.all()`. This means that if the agent emits three tool calls in a single step:

```
user_feed()  ─┐
user_stats() ─┼─ Promise.all → simultaneous trigger → 429 Rate Limit
search()      ─┘
```

If all three tools call the same external API with a limit of 1 req/sec (such as apiexternalexample), the three HTTP requests arrive concurrently and the last two fail with `429 Too Many Requests`.

### Why NOT Add a `parallel: false` Flag to Core

It might seem tempting to add `parallel: false` to the tool schema in `config.ts` so that the `RuntimeLoop` executes them serially. However, this is not the correct solution because:

- It would require changing `ModuleConfig.ts`, `SkillRegistry.ts`, `McpTool.ts`, and `RuntimeLoop.ts` — high risk for a very specific case.
- It would make **all** tools in the turn wait, even those that do not use the limited API.
- It would violate the single responsibility principle: the RuntimeLoop should not know about external API rate-limiting policies.

### The Solution: Serial Queue at HTTP Level (Serial Queue)

The correct solution is to implement a **promise semaphore** that serializes only the HTTP calls to the limited API, **completely invisible** to the agent and the RuntimeLoop.

**Reference Implementation** (see `MarketsPlugin.ts`):

```typescript
// ── Rate limiting constants ────────────────────────────────────────────
const MY_API_MIN_INTERVAL_MS = 1100; // 1.1s between requests (margin over 1/s)
let myApiLastCall = 0;
let myApiQueue: Promise<any> = Promise.resolve();

// Enqueues the 'fn' function for serial execution with minimum delay between calls.
function myApiEnqueue<T>(fn: () => Promise<T>): Promise<T> {
    const result = myApiQueue.then(async () => {
        const now = Date.now();
        const wait = MY_API_MIN_INTERVAL_MS - (now - myApiLastCall);
        if (wait > 0) await new Promise(r => setTimeout(r, wait));
        myApiLastCall = Date.now();
        return fn();
    });
    // Chain so that the next call waits for THIS one, not just the previous one.
    myApiQueue = result.catch(() => {});
    return result;
}

// Fetch wrapper that uses the queue
async function myFetch(path: string, apiKey: string): Promise<any> {
    return myApiEnqueue(async () => {
        const res = await fetch(`https://api.my-service.com${path}`, {
            headers: { 'Authorization': `Bearer ${apiKey}` }
        });
        // Handle 429 with automatic retry using Retry-After
        if (res.status === 429) {
            const retryAfter = parseInt(res.headers.get('Retry-After') || '1', 10);
            await new Promise(r => setTimeout(r, retryAfter * 1000 + 100));
            const retry = await fetch(`https://api.my-service.com${path}`, {
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

### How the Queue Works

```
Promise.all triggers:       user_feed()   user_stats()   search()
                                 │               │              │
                            call:           myFetch()     myFetch()     myFetch()
                                 │               │              │
                          myApiEnqueue()  myApiEnqueue() myApiEnqueue()
                                 │               │              │
                          ┌──────▼───────────────▼──────────────▼──────┐
                          │          SERIAL QUEUE (myApiQueue)         │
                          │   Req 1 → [waits 1.1s] → Req 2 → [1.1s]   │
                          │   → Req 3                                   │
                          └────────────────────────────────────────────┘
```

The RuntimeLoop's `Promise.all` ends when **all three promises resolve**, which happens sequentially with a 1.1s separation. From the point of view of the agent and the RuntimeLoop, the difference is only in latency, not in behavior.

### When to Use This Pattern

Use a serial queue when your plugin integrates an external API that:

| Condition | Example |
|-----------|---------|
| Has write OR read rate limit | apiexternalexample: ~1 req/s, Twitter API: 300 req/15min |
| The agent can call multiple tools from the same provider in one turn | `user_feed` + `user_stats` + `search` → all use apiexternalexample |
| The rate limit error is not recoverable in that same request | `429` without Retry-After header |

> [!TIP]
> The queue is a **module-level singleton** (declared outside the plugin class). This ensures that even if multiple sessions of the same agent are active simultaneously, all API calls share the same queue and respect the global rate limit.

> [!NOTE]
> If the API does not return the `Retry-After` header, use a conservative value of 1–2 seconds as a fallback. Retry is optional but highly recommended to prevent a momentary traffic spike from bringing down all functionality.

> [!CAUTION]
> **Critical Timeout**: When using a serial queue, it is mandatory that the `fetch` inside the queue has a timeout (using `AbortController`). If a request gets hung without a timeout, it will **block the entire queue** for the rest of the tools and users, leaving the module unusable until restart.
> A timeout for the wait time *in* the queue is also recommended (e.g., abort if the request has been waiting >15s for its turn).

---

## 💎 Advanced MCP Patterns (Urano Standard)

To ensure consistency between complex modules (such as UranoMaps or SystemEye), the following development patterns have been standardized:

### 1. Window Management by Session (`label:sessionId`)

When a plugin opens tabs in `MultiverseTabs`, it should avoid unnecessarily duplicating windows and allow the agent to recover specific contexts.

**Registration Pattern:**
```typescript
// Dynamic registration: "friendly_name:sessionId" -> tabId
const dashboardRegistry = new Map<string, string>();
const dashboardParams = new Map<string, any>();

async apiLaunch(params: { label: string, sessionId: string, ... }) {
    const key = `${params.label}:${params.sessionId}`;
    let tabId = dashboardRegistry.get(key);
    
    if (tabId && this.tabExists(tabId)) {
        // If it already exists, we just focus and update it
        return this.apiUpdate({ tabId, ... });
    }
    
    // If it doesn't exist, we create a new one and register it
    tabId = await this.createNewTab();
    dashboardRegistry.set(key, tabId);
    return { tabId, status: 'launched' };
}
```

### 2. Window State Persistence (`state.json`)

Plugins that handle data in memory (alerts, orders, positions) must persist their state in the user's `userData` to survive restarts.

**Recommended Location:**
- **Windows**: `%APPDATA%/Urano Desktop/[plugin]_state.json`
- **Fallback**: `~/.urano/[plugin]_state.json`

**Reference Implementation:**
```typescript
function saveState() {
    const p = path.join(os.homedir(), '.urano', 'my_plugin_state.json');
    const data = { registry: Object.fromEntries(dashboardRegistry) };
    fs.writeFileSync(p, JSON.stringify(data, null, 2));
}
```

### 3. Capture Logic and File Caching

For an agent to "see" the content of a window or map, the plugin must implement a capture tool that generates local URLs.

**Capture Standards:**
1. **Cache Directory**: Always use `path.join(os.homedir(), '.urano', 'cache', 'screenshots')`.
2. **Multimodal Return**: The tool must return an array with the local path and base64 for the `RuntimeLoop` to process it.

**Return example:**
```typescript
return [
    { type: 'text', text: `Capture saved in: ${filePath}` },
    { type: 'image', image: buffer.toString('base64'), mimeType: 'image/png' }
];
```

### 4. Dynamic Rendering Standards (`JsonLive`)

When creating new UI components (such as the `Map` widget), these rules must be followed to ensure compatibility with `JsonLiveRenderer`:

- **Atomicity**: Each component must be autonomous and use `extractStyle()` for its CSS properties.
- **Fluid Patching**: The plugin should prefer `patchDynamicUI` over `generateDynamicUI` for real-time updates (prevents flickering).
- **Node Structure**:
    ```json
    {
      "type": "NewComponent",
      "props": { "glass": true, "glow": "#ff0000", ... },
      "children": [ ... ]
    }
    ```

> [!TIP]
> If your component requires heavy libraries (such as Leaflet or ECharts), implement it using **Lazy Loading** in the frontend so as not to degrade the initial loading time of the application.

---

## 🔒 Generic OS Permission System

Urano implements a centralized mechanism so that any MCP can request permissions at the operating system level natively, with **real verification** (not just registration) before saving the state.

### Supported Permission Types

| `permissionType` | Description | macOS | Windows | Linux |
|---|---|---|---|---|
| `screen` | Screen capture | System dialog / Settings | Privacy settings | None required |
| `microphone` | Microphone access | Native `askForMediaAccess` | `ms-settings:privacy-microphone` | None required |
| `camera` | Camera access | Native `askForMediaAccess` | `ms-settings:privacy-webcam` | None required |
| `desktop-audio` | Desktop audio (loopback) | ⚠️ Requires virtual device | WASAPI loopback via renderer | PulseAudio/PipeWire |

> [!WARNING]
> On **macOS**, desktop audio capture (`desktop-audio`) is not available natively. It requires installing **BlackHole** (free) or **Loopback** and configuring it as system audio output.

### 1. Definition in `config.ts`
Add a field of type `button` in the `settings` array of your module. The `name` will serve as a slug in the Vault (`Vault`) to persist whether the user granted the permission.

```typescript
// config.ts of your module
settings: [
    {
        name: 'capture-permission',          // Slug of the permission saved in Vault
        title: 'Screen Capture',
        type: 'button',
        buttonText: 'Request OS Permission',
        description: 'Allows the agent to see the visual context of your screen',
        actionRoute: 'mcp-request-os-permission',
        actionPayload: { permissionType: 'screen' },
    },
    {
        name: 'microphone-access',
        title: 'Microphone Access',
        type: 'button',
        buttonText: 'Request OS Permission',
        description: 'Allows the agent to listen to microphone audio',
        actionRoute: 'mcp-request-os-permission',
        actionPayload: { permissionType: 'microphone' },
    },
    {
        name: 'desktop-audio-access',
        title: 'Desktop Audio',
        type: 'button',
        buttonText: 'Request OS Permission',
        description: 'Allows the agent to listen to system audio (loopback)',
        actionRoute: 'mcp-request-os-permission',
        actionPayload: { permissionType: 'desktop-audio' },
    },
],
```

### 2. Automatic Behavior
- The frontend automatically injects `moduleName` and `permissionSlug` (= `name` of the field) into the payload.
- The backend **verifies the real permission** before saving:
  - `screen`: Captures a real thumbnail; if empty, returns error and opens System Settings on macOS.
  - `microphone`/`camera`: Uses `systemPreferences.getMediaAccessStatus()` on macOS to verify status before requesting; if `'denied'`, it opens the privacy screen directly.
  - `desktop-audio`: Informs the user about platform limitations.
- The `'granted'` status is saved in the Vault **only if the permission is real**.
- If subsequent capture detects the thumbnail is empty, it **automatically revokes** the `'granted'` status and resets the badge to amber.

### 3. Read Permission from a Plugin

Check if permission was granted by directly querying the `Vault` from your plugin:

```typescript
import { Vault } from '../../../../core/Security/Vault';

async apiMyAction() {
    const isGranted = Vault.getSecret(this.moduleName, 'capture-permission') === 'granted';
    if (!isGranted) {
        return { success: false, consentRequired: true, message: 'Permission not granted' };
    }
    // ... use the permission
}
```

> [!TIP]
> The `McpManager` automatically shows the "System Permissions" section with status badges for all `button` type fields. Amber badges indicate action is required; green that permission is granted and verified.

---

### 4. Using Permissions Within a Plugin

Once the user has granted permission, here is the reference implementation for each type.

#### 📸 `screen` — Capture screen

Use Electron's `desktopCapturer` directly from the **main** process. It does not require the renderer.

```typescript
// In your Plugin (Main Process — Node.js / Electron)
import { desktopCapturer } from 'electron';
import { Vault } from '../../../../core/Security/Vault';

export default class MyPlugin extends PluginBase {

    async apiCaptureScreen() {
        // 1. Verify permission
        const isGranted = Vault.getSecret(this.moduleName, 'capture-permission') === 'granted';
        if (!isGranted) {
            return { success: false, consentRequired: true, message: 'Capture permission not granted. Go to module settings.' };
        }

        // 2. Capture
        const sources = await desktopCapturer.getSources({
            types: ['screen'],
            thumbnailSize: { width: 1920, height: 1080 },
        });

        const primary = sources[0];
        const thumb = primary?.thumbnail;

        if (!thumb || thumb.isEmpty()) {
            // Revoke if the OS no longer allows it
            Vault.deleteSecret(this.moduleName, 'capture-permission');
            return { success: false, message: 'The OS denied capture. Permission was revoked.' };
        }

        const base64 = thumb.toPNG().toString('base64');

        // 3. Return image — RuntimeLoop will inject it as multimodal part
        return [
            { type: 'text', text: 'Screen capture taken successfully.' },
            { type: 'image', image: base64, mimeType: 'image/png' },
        ];
    }
}
```

---

#### 🎙️ `microphone` — Record microphone audio

Microphone capture occurs in the **renderer process** (React/Web Audio API), not in main. Your plugin exposes a tool the agent calls; the frontend reacts and records.

**Step 1 — Plugin exposes tool (Main Process):**

```typescript
// In your Plugin
export default class MyPlugin extends PluginBase {

    async apiStartMicCapture(payload: { durationMs?: number }) {
        const isGranted = Vault.getSecret(this.moduleName, 'microphone-access') === 'granted';
        if (!isGranted) {
            return { success: false, consentRequired: true, message: 'Microphone permission not granted.' };
        }

        // Emit event to renderer to start recording
        // Renderer listens for 'plugin-mic-start' and returns audio via IPC
        this.emitToRenderer('plugin-mic-start', {
            durationMs: payload.durationMs || 5000,
            sessionId: this.sessionId,
        });

        return { success: true, message: 'Microphone recording started.' };
    }
}
```

**Step 2 — Renderer captures and returns audio:**

```typescript
// In your React component (Renderer)
api.on('plugin-mic-start', async ({ durationMs, sessionId }) => {
    const stream = await navigator.mediaDevices.getUserMedia({ audio: true, video: false });
    const recorder = new MediaRecorder(stream);
    const chunks: BlobPart[] = [];

    recorder.ondataavailable = (e) => chunks.push(e.data);
    recorder.onstop = async () => {
        const blob = new Blob(chunks, { type: 'audio/webm' });
        const buffer = await blob.arrayBuffer();
        const base64 = btoa(String.fromCharCode(...new Uint8Array(buffer)));
        // Send back to main process
        api.customIpcCall('plugin-mic-result', { sessionId, base64, mimeType: 'audio/webm' });
        stream.getTracks().forEach(t => t.stop());
    };

    recorder.start();
    setTimeout(() => recorder.stop(), durationMs);
});
```

> [!NOTE]
> If you already have `useAudioEngine` active in the frontend, you can recycle it directly instead of creating a new `MediaRecorder`. See the **Integration with the Native Audio Engine** section of this guide.

---

#### 🔊 `desktop-audio` — Capture desktop audio (loopback)

Only available on **Windows** (and Linux). Captures what is playing on the system in real-time.

**Step 1 — Plugin emits signal (Main Process):**

```typescript
export default class MyPlugin extends PluginBase {

    async apiStartDesktopAudio(payload: { durationMs?: number }) {
        const isGranted = Vault.getSecret(this.moduleName, 'desktop-audio-access') === 'granted';
        if (!isGranted) {
            return { success: false, consentRequired: true, message: 'Desktop audio permission not granted.' };
        }
        if (process.platform === 'darwin') {
            return { success: false, message: 'macOS does not support native desktop audio capture. Install BlackHole.' };
        }

        this.emitToRenderer('plugin-desktop-audio-start', {
            durationMs: payload.durationMs || 5000,
            sessionId: this.sessionId,
        });

        return { success: true, message: 'Desktop audio capture started.' };
    }
}
```

**Step 2 — Renderer captures with WASAPI loopback:**

```typescript
// In the renderer (React) — only works on Windows/Linux
api.on('plugin-desktop-audio-start', async ({ durationMs, sessionId }) => {
    // Get desktop sources with audio enabled
    const sources = await (window as any).electronAPI.customIpcCall(
        'desktop-capturer-get-sources', { types: ['screen'], fetchWindowIcons: false }
    );

    const stream = await navigator.mediaDevices.getUserMedia({
        audio: {
            mandatory: {
                chromeMediaSource: 'desktop',
                chromeMediaSourceId: sources[0]?.id,  // First screen
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
> For `getUserMedia` with `chromeMediaSource: 'desktop'` to work in the Electron renderer, the window must have `contextIsolation: false` enabled or the preload must expose the capture source IDs. In Urano, this is managed via the `desktop-capturer-get-sources` IPC.

> [!WARNING]
> On **macOS**, this code block will fail silently because the system does not expose system audio via `getUserMedia`. Verify with `process.platform === 'darwin'` before executing.
