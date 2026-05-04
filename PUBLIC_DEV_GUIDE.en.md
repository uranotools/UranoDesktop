<div align="center">
  <p><b>Spanish Version: <a href="./PUBLIC_DEV_GUIDE.md">PUBLIC_DEV_GUIDE.md</a></b></p>
</div>

# 🚀 Publishing Guide: Urano MCP Marketplace

Welcome to the **Urano Desktop** developer community. This guide is designed to teach you how to package, distribute, and publish your MCP (Model Context Protocol) plugin in the official Urano catalog so that thousands of users can install it with a single click.

> [!NOTE]
> If you are looking for the guide on how to program the internal logic of an MCP plugin (API, Audio, MultiverseTabs), please refer to the official documentation within the application or in the [Urano Development Wiki](CREATE_MCP_GUIDE.en.md).

---

## 1. 📦 Preparing your Plugin (Bundling with esbuild)

Since users will install your plugin remotely, uploading heavy folders with full `node_modules` is not scalable and can cause path issues on Windows (MAX_PATH). 

The **mandatory official practice** is to package your code using `esbuild`.

1. In your project, install esbuild as a development dependency:
   ```bash
   npm install --save-dev esbuild
   ```

2. Run the bundling command, pointing to your `config.ts` and the plugins folder. For example:
   ```bash
   npx esbuild config.ts Plugins/**/*.ts --bundle --platform=node --outdir=dist --format=cjs
   ```

3. **Result**: You will have a `dist/` folder with your optimized code and no heavy external dependencies.

4. Copy your `SKILL.md` file (if you have one) and any static assets to the `dist/` folder.

5. **Compress**: Select **the contents** of the `dist/` folder and compress them into a ZIP file (e.g., `MyPlugin-1.0.0.zip`).
   *(Important: Compress the files inside, not the `dist` folder itself).*

---

## 2. 🌍 Uploading your Release to GitHub

You need to host your ZIP file publicly so that Urano can download it.

1. Create a public repository on GitHub for your plugin (e.g., `github.com/your-user/my-super-mcp`).
2. Go to the **Releases** section and create a new one.
3. Tag the release with a semantic version (e.g., `v1.0.0`).
4. In **Assets**, upload the `.zip` file you created in the previous step.
5. Copy the direct download link for the ZIP file.

---

## 3. 🚀 Publishing on the Urano Marketplace

The Urano Marketplace is decentralized and managed through a single file called `registry.json` hosted in this same repository.

To add your plugin to the store, you just need to make a **Pull Request (PR)** adding your data to the registry.

### JSON Structure

Open the `registry.json` file and add an object to the `"plugins"` array with this structure:

```json
{
  "id": "MySuperMcp",
  "name": "Super MCP for Tasks",
  "author": "Your Name/Organization",
  "description": "A brief description of what your plugin does. Try to be clear and concise.",
  "version": "1.0.0",
  "category": "Productivity",
  "icon": "Zap", 
  "tags": ["tasks", "organization", "ai"],
  "downloadUrl": "https://github.com/your-user/my-super-mcp/releases/download/v1.0.0/MyPlugin-1.0.0.zip",
  "verified": false
}
```

### Pull Request Rules
- Ensure your `id` is unique.
- Use strict semantic versioning (`MAJOR.MINOR.PATCH`).
- The `downloadUrl` field must be the direct link to your `.zip`.
- Leave `verified: false`. The Urano team will review the code and, if it meets security standards, will change it to `true`.

Once your PR is approved and merged, your plugin will appear instantly in the **Marketplace** tab for all Urano Desktop users.

---

## 🔄 How to Update my Plugin?

When you launch a new version:
1. Create a new Release in your repository with the new ZIP (`v1.0.1`).
2. Make a PR to `registry.json` modifying the `"version"` and the `"downloadUrl"` of your existing entry.
3. The Urano system will automatically detect the higher version and notify users to update with one click.
