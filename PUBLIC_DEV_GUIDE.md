# 🚀 Guía de Publicación: Urano MCP Marketplace

Bienvenido a la comunidad de desarrolladores de **Urano Desktop**. Esta guía está diseñada para enseñarte cómo empaquetar, distribuir y publicar tu plugin MCP (Model Context Protocol) en el catálogo oficial de Urano para que miles de usuarios puedan instalarlo con un solo clic.

> [!NOTE]
> Si buscas la guía de cómo programar la lógica interna de un plugin MCP (API, Audio, MultiverseTabs), consulta la documentación oficial dentro de la aplicación o en la [Wiki de Desarrollo de Urano]().

---

## 1. 📦 Preparando tu Plugin (Bundling con esbuild)

Dado que los usuarios instalarán tu plugin remotamente, subir carpetas pesadas con `node_modules` completos no es escalable y puede causar problemas de ruta en Windows (MAX_PATH). 

La **práctica oficial obligatoria** es empaquetar tu código usando `esbuild`.

1. En tu proyecto, instala esbuild como dependencia de desarrollo:
   ```bash
   npm install --save-dev esbuild
   ```

2. Ejecuta el comando de empaquetado, apuntando a tu `config.ts` y a la carpeta de plugins. Por ejemplo:
   ```bash
   npx esbuild config.ts Plugins/**/*.ts --bundle --platform=node --outdir=dist --format=cjs
   ```

3. **Resultado**: Tendrás una carpeta `dist/` con tu código optimizado y sin dependencias externas pesadas.

4. Copia tu archivo `SKILL.md` (si lo tienes) y cualquier asset estático hacia la carpeta `dist/`.

5. **Comprimir**: Selecciona **el contenido** de la carpeta `dist/` y comprímelo en un archivo ZIP (ej: `MiPlugin-1.0.0.zip`).
   *(Importante: Comprime los archivos de adentro, no la carpeta `dist` en sí).*

---

## 2. 🌍 Subiendo tu Release a GitHub

Necesitas alojar tu archivo ZIP de forma pública para que Urano pueda descargarlo.

1. Crea un repositorio público en GitHub para tu plugin (ej: `github.com/tu-usuario/mi-super-mcp`).
2. Ve a la sección **Releases** y crea uno nuevo.
3. Etiqueta el release con una versión semántica (ej: `v1.0.0`).
4. En **Assets**, sube el archivo `.zip` que creaste en el paso anterior.
5. Copia el enlace directo de descarga del archivo ZIP.

---

## 3. 🚀 Publicar en el Marketplace de Urano

El Marketplace de Urano está descentralizado y se gestiona a través de un único archivo llamado `registry.json` alojado en este mismo repositorio.

Para añadir tu plugin a la tienda, solo debes hacer un **Pull Request (PR)** agregando tus datos al registro.

### Estructura del JSON

Abre el archivo `registry.json` y añade un objeto al array `"plugins"` con esta estructura:

```json
{
  "id": "MiSuperMcp",
  "name": "Super MCP para Tareas",
  "author": "Tu Nombre/Organización",
  "description": "Una breve descripción de lo que hace tu plugin. Intenta ser claro y conciso.",
  "version": "1.0.0",
  "category": "Productividad",
  "icon": "Zap", 
  "tags": ["tareas", "organización", "ia"],
  "downloadUrl": "https://github.com/tu-usuario/mi-super-mcp/releases/download/v1.0.0/MiPlugin-1.0.0.zip",
  "verified": false
}
```

### Reglas para el Pull Request
- Asegúrate de que tu `id` sea único.
- Usa versión semántica estricta (`MAYOR.MENOR.PARCHE`).
- El campo `downloadUrl` debe ser el link directo a tu `.zip`.
- Deja `verified: false`. El equipo de Urano revisará el código y, si cumple con los estándares de seguridad, lo cambiará a `true`.

Una vez que tu PR sea aprobado y fusionado (`merged`), tu plugin aparecerá instantáneamente en la pestaña **Marketplace** de todos los usuarios de Urano Desktop.

---

## 🔄 ¿Cómo Actualizar mi Plugin?

Cuando lances una nueva versión:
1. Crea un nuevo Release en tu repositorio con el nuevo ZIP (`v1.0.1`).
2. Haz un PR a `registry.json` modificando la `"version"` y la `"downloadUrl"` de tu entrada existente.
3. El sistema de Urano detectará automáticamente la versión superior y notificará a los usuarios para que actualicen con un clic.
