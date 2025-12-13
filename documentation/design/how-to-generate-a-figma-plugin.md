---
status: draft
priority: medium
type: reference
tags: [design, figma, plugins, react, guides]
---

Creating a Figma plugin for your company's React component library is very doable—but it's important to understand that **Figma plugins cannot directly render React components**. They run in a sandboxed environment using HTML/CSS/JS, and what you *simulate* in Figma is usually:

* A UI mirroring your component library (for internal tooling)
* A way to insert Figma nodes that visually represent components
* A way to sync design tokens from your codebase
* A way to generate code based on Figma selections

Below is a clear, practical roadmap for building a plugin that uses your React component library as a **source of truth**.

---

# ✅ **High-level Approach**

To create a Figma plugin that aligns with your React component library, you’ll typically:

### **1. Extract your design tokens & component metadata from your React system**

Examples:

* Color tokens
* Typography tokens
* Elevation
* Spacing scales
* Component variants / props

You can export these from your React codebase using tools like:

* **Style Dictionary**
* **Token Studio format JSON**
* **Custom scripts that read your component config or theme**

---

### **2. Create a Figma plugin UI using React (with Vite or webpack)**

Even though the Figma **plugin runtime** is not React, the **plugin UI** (the modal or panel) *can* be built with React.

You structure it like this:

```
/plugin
  /ui
    index.html
    index.tsx  <-- React UI
  /code
    code.ts    <-- Figma plugin controller (no DOM)
  manifest.json
```

The React UI communicates with Figma’s plugin sandbox via `postMessage`.

---

### **3. Render “component representations” in Figma**

Your plugin cannot bring React components into Figma.
Instead, you create **Figma nodes** that visually match your components.

Example:
If your component library has a `<Button variant="primary"/>`, your plugin inserts a Figma frame styled with:

* background color = token.primary
* padding = token.spacing
* border radius = token.borderRadius
* text = "Button"
* configurable variants

Most teams generate these automatically using:

* A JSON map of component definitions
* A function that converts them into Figma nodes

---

### **4. (Optional) Sync code ↔ design**

Common advanced features:

#### **Insert component → generate JSX**

When a user selects a component in Figma, you show:

```jsx
<Button variant="primary" size="lg">Click me</Button>
```

#### **Take Figma design → suggest code**

Useful for handoff flows.

---

### **5. Package & publish privately**

Figma allows **private plugins** via your organization.

---

# 📦 Example Project Setup

### **manifest.json**

```json
{
  "name": "MyCompany Component Library",
  "id": "my-company-plugin",
  "api": "1.0.0",
  "main": "code/code.js",
  "ui": "ui/index.html"
}
```

---

### **plugin/code.ts (Figma runtime)**

```ts
figma.ui.onmessage = msg => {
  if (msg.type === 'insert-button') {
    const frame = figma.createFrame();
    frame.resize(120, 40);
    frame.name = "Button / Primary";

    const text = figma.createText();
    text.characters = "Button";
    frame.appendChild(text);

    figma.currentPage.appendChild(frame);
  }
};
```

---

### **ui/index.tsx (React UI)**

```tsx
function App() {
  return (
    <button
      onClick={() =>
        parent.postMessage({ pluginMessage: { type: 'insert-button' } }, '*')
      }
    >
      Insert Primary Button
    </button>
  );
}
```

---

# 📐 Recommended Architecture

```
/design-system
  tokens.json         <-- exported tokens from React library
  components.json     <-- metadata/prop variants (optional)

/figma-plugin
  manifest.json
  /ui (React)
  /code (Figma plugin runtime)
  /scripts
      sync-tokens.js  <-- copies tokens into plugin bundle
      sync-components.js
```

---

# 🛠 Tools That Make This Easier

### **For building plugins**

* Figma Plugin API
* `@figma/plugin-typings`
* Vite + React plugin boilerplates
* Figma Plugin Starter templates (TS + React)

### **For token-sync**

* Style Dictionary
* Token Studio sync
* Theo
* Custom token extraction scripts

---

# 🚀 Step-by-step Minimal Path

1. **Export your theme & tokens** into JSON
2. **Create a Figma plugin scaffold**

   ```
   npx degit figma/plugin-samples/javascript-react my-plugin
   ```
3. **Load tokens into the React UI**
4. **Create a UI that lists your components**
5. **On click → send message to Figma runtime**
6. **In runtime → create Figma nodes that look like your components**
7. Optional: add code generation or variant controls
8. Publish internally

---

# Want me to generate a starter project for you?

I can generate:

✅ full file structure
✅ manifest.json
✅ Vite + React config
✅ token syncing script
✅ component generator code

Or even:

📦 *“A complete working plugin based on your component library tokens”*

If you want that, just tell me:

* What React framework you use (MUI? Chakra? custom?)
* Whether you have tokens in JSON yet
* Whether you want code generation in the plugin

Happy to help build the whole thing!
