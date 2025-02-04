Project Structure

enhanced-dataviewjs/
├── node_modules/
├── src/
│   ├── api-extensions.ts
│   ├── enhanced-renderer.ts
│   ├── lib-manager.ts
│   ├── main.ts
│   └── types.ts
├── esbuild.config.mjs
├── main.js
├── manifest.json
├── package-lock.json
├── package.json
├── styles.css
└── tsconfig.json

Source Codes

-- src/api-extensions.ts --

```
import { LibraryManager } from './lib-manager';
import { ComponentConfig, RenderProps, IEnhancedDataviewAPI } from './types';

export class EnhancedDataviewAPI implements IEnhancedDataviewAPI {
    private libManager: LibraryManager;

    constructor() {
        this.libManager = new LibraryManager();
    }

    public async useLibrary(name: string): Promise<void> {
        await this.libManager.loadLibrary(name);
    }

    public async createComponent(config: ComponentConfig): Promise<void> {
        try {
            // Ensure all required libraries are loaded
            for (const lib of config.libraries) {
                await this.useLibrary(lib);
            }

            // Create render props
            const props: RenderProps = {
                MaterialUI: (window as any).MaterialUI,
                React: (window as any).React,
                ReactDOM: (window as any).ReactDOM
            };

            // Render the component
            return config.render(props);
        } catch (error) {
            console.error('Failed to create component:', error);
            throw error;
        }
    }

    public async createCard(title: string, content: string): Promise<void> {
        await this.useLibrary('react');
        await this.useLibrary('reactDOM');
        await this.useLibrary('materialUI');
        
        return this.createComponent({
            libraries: ['react', 'reactDOM', 'materialUI'],
            render: (props: RenderProps) => {
                const { Card, Typography, Box } = props.MaterialUI;
                const cardContainer = document.createElement('div');
                
                const element = props.React.createElement(Card, {
                    sx: { p: 2, m: 2 }
                }, 
                    props.React.createElement(Typography, {
                        variant: 'h6'
                    }, title),
                    props.React.createElement(Box, {
                        sx: { mt: 1 }
                    }, content)
                );

                props.ReactDOM.render(element, cardContainer);
                document.body.appendChild(cardContainer);
            }
        });
    }
}
```

-- src/enhanced-renderer.ts --

```
import { LibraryManager } from './lib-manager';
import { ComponentConfig } from './types';

export class EnhancedRenderer {
    constructor(private libManager: LibraryManager) {}

    async createComponent(config: ComponentConfig) {
        // Load required libraries
        await Promise.all(
            config.libraries.map(lib => this.libManager.loadLibrary(lib))
        );

        // Create dependencies object
        const deps = {
            MaterialUI: (window as any).MaterialUI,
            React: (window as any).React,
            ReactDOM: (window as any).ReactDOM
        };

        // Return rendered component
        return config.render(deps);
    }
}

```

-- src/lib-manager.ts --

```
import { Library } from './types';

export class LibraryManager {
    private libraries: { [key: string]: Library } = {
        react: {
            name: 'react',
            url: 'https://unpkg.com/react@17.0.2/umd/react.production.min.js',
            version: '17.0.2'
        },
        reactDOM: {
            name: 'reactDOM',
            url: 'https://unpkg.com/react-dom@17.0.2/umd/react-dom.production.min.js',
            version: '17.0.2'
        },
        materialUI: {
            name: 'materialUI',
            url: 'https://unpkg.com/@mui/material@5.14.11/umd/material-ui.production.min.js',
            version: '5.14.11'
        }
    };

    private loadedLibraries: Set<string> = new Set();
    private loadingPromises: Map<string, Promise<void>> = new Map();

    constructor() {
        // Only log once during initialization
        console.log('Initializing libraries...');
        Object.values(this.libraries).forEach(lib => {
            console.log(`Registered library: ${lib.name} v${lib.version}`);
        });
    }

    public async loadLibrary(name: string): Promise<void> {
        // If already loaded, return silently
        if (this.loadedLibraries.has(name)) {
            return;
        }

        // If currently loading, return the existing promise
        if (this.loadingPromises.has(name)) {
            return this.loadingPromises.get(name);
        }

        const library = this.libraries[name];
        if (!library) {
            throw new Error(`Library ${name} not found`);
        }

        // Create new loading promise
        const loadingPromise = this.loadScript(library.url).then(() => {
            this.loadedLibraries.add(name);
            this.loadingPromises.delete(name);
            console.log(`Loaded library: ${name}`);
        });

        // Store the loading promise
        this.loadingPromises.set(name, loadingPromise);

        return loadingPromise;
    }

    private loadScript(url: string): Promise<void> {
        return new Promise((resolve, reject) => {
            // Check if script is already loaded
            if (document.querySelector(`script[src="${url}"]`)) {
                resolve();
                return;
            }

            const script = document.createElement('script');
            script.src = url;
            script.onload = () => resolve();
            script.onerror = () => reject(new Error(`Failed to load script: ${url}`));
            document.head.appendChild(script);
        });
    }
}
```

-- src/main.ts --

```
import { Plugin, App } from 'obsidian';
import { EnhancedDataviewAPI } from './api-extensions';
import { GemHelpers, RenderProps } from './types';

const PLUGIN_VERSION = '1.0.0';

interface DataviewApi {
    index: any;
    page: any;
    pages: () => any;
    pagePaths: () => string[];
    settings: any;
}

declare module 'obsidian' {
    interface App {
        plugins: {
            getPlugin(id: string): any;
        };
    }
}

export default class EnhancedDataviewJS extends Plugin {
    private enhancedAPI: EnhancedDataviewAPI;
    private currentUser: string = 'farizessentia';

    private getFormattedUTCTime(): string {
        const now = new Date();
        return now.toISOString().slice(0, 19).replace('T', ' ');
    }

    async onload() {
        console.log(`Loading Enhanced DataviewJS v${PLUGIN_VERSION} - ${this.getFormattedUTCTime()}`);
        
        this.enhancedAPI = new EnhancedDataviewAPI();

        this.registerMarkdownCodeBlockProcessor('gemspace', async (source: string, el: HTMLElement, ctx: any) => {
            try {
                const dataviewPlugin = this.app.plugins.getPlugin('dataview');
                if (!dataviewPlugin) {
                    throw new Error('DataviewJS plugin is required');
                }

                const dataviewApi = dataviewPlugin.api;
                const container = el.createDiv('gemspace-container');

                const dv = {
                    // System info
                    version: PLUGIN_VERSION,
                    currentUser: this.currentUser,
                    getCurrentUTCTime: () => this.getFormattedUTCTime(),

                    // DataviewJS API methods
                    pages: () => dataviewApi.pages(),
                    page: (path: string) => dataviewApi.page(path),
                    pagePaths: () => dataviewApi.pagePaths(),
                    current: () => dataviewApi.current(),
                    filterPages: (pages: any, ...filters: any[]) => dataviewApi.filterPages(pages, ...filters),
                    isArray: (value: any) => Array.isArray(value),
                    date: (input: any) => dataviewApi.date(input),
                    
                    // UI Methods
                    header: (text: string) => {
                        const header = container.createEl('h1');
                        header.textContent = text;
                    },
                    paragraph: (text: string) => {
                        const p = container.createEl('p');
                        p.textContent = String(text);
                    },
                    list: (items: any[]) => {
                        const ul = container.createEl('ul');
                        items.forEach(item => {
                            const li = ul.createEl('li');
                            li.textContent = String(item);
                        });
                    },
                    table: (headers: string[], content: any[][]) => {
                        const table = container.createEl('table');
                        
                        // Create header row
                        const thead = table.createEl('thead');
                        const headerRow = thead.createEl('tr');
                        headers.forEach(header => {
                            const th = headerRow.createEl('th');
                            th.textContent = header;
                        });

                        // Create table body
                        const tbody = table.createEl('tbody');
                        content.forEach(row => {
                            const tr = tbody.createEl('tr');
                            row.forEach(cell => {
                                const td = tr.createEl('td');
                                td.textContent = String(cell);
                            });
                        });
                    },
                    
                    // Helper methods
                    formatDate: (date: any) => {
                        if (typeof date?.toFormat === 'function') {
                            return date.toFormat('yyyy-MM-dd HH:mm:ss');
                        }
                        return 'Invalid Date';
                    },
                    createContainer: () => {
                        return container.createDiv('gemspace-section');
                    },

                    // Enhanced features
                    enhanced: this.enhancedAPI,
                    container
                };

                // Add gem helpers
                const gemHelpers: GemHelpers = {
                    createButton: async (text: string, onClick: () => void) => {
                        await dv.enhanced.useLibrary('react');
                        await dv.enhanced.useLibrary('reactDOM');
                        await dv.enhanced.useLibrary('materialUI');
                        
                        return dv.enhanced.createComponent({
                            libraries: ['react', 'reactDOM', 'materialUI'],
                            render: (props: RenderProps) => {
                                const { Button } = props.MaterialUI;
                                const buttonContainer = container.createDiv('gemspace-button');
                                const element = props.React.createElement(Button, {
                                    variant: 'contained',
                                    color: 'primary',
                                    onClick,
                                    style: {
                                        margin: '8px 0'
                                    }
                                }, text);
                                
                                props.ReactDOM.render(element, buttonContainer);
                            }
                        });
                    },
                    createInfo: (message: string) => {
                        const info = container.createDiv('gemspace-info');
                        info.innerHTML = `ℹ️ ${message}`;
                        return info;
                    },
                    createWarning: (message: string) => {
                        const warning = container.createDiv('gemspace-warning');
                        warning.innerHTML = `⚠️ ${message}`;
                        return warning;
                    },
                    createSuccess: (message: string) => {
                        const success = container.createDiv('gemspace-success');
                        success.innerHTML = `✅ ${message}`;
                        return success;
                    },
                    createError: (message: string) => {
                        const error = container.createDiv('gemspace-error');
                        error.innerHTML = `❌ ${message}`;
                        return error;
                    }
                };

                Object.defineProperty(dv, 'gem', {
                    value: gemHelpers,
                    enumerable: true,
                    configurable: true,
                    writable: false
                });

                // Execute the code with error handling
                const wrappedCode = `(async () => {
                    try {
                        // Add version and time info
                        dv.gem.createInfo(\`GemSpace v\${dv.version} - \${dv.getCurrentUTCTime()}\`);
                        
                        ${source}
                    } catch (error) {
                        dv.gem.createError(\`Runtime Error: \${error.message}\`);
                        console.error('GemSpace Runtime Error:', error);
                    }
                })()`;

                const AsyncFunction = Object.getPrototypeOf(async function(){}).constructor;
                const fn = new AsyncFunction('dv', wrappedCode);
                await fn(dv);

            } catch (error) {
                const errorDiv = el.createDiv('gemspace-fatal-error');
                errorDiv.innerHTML = `
                    <div style="color: red; padding: 1rem; border: 1px solid red; border-radius: 4px; margin: 1rem 0;">
                        <h3>⚠️ GemSpace Error</h3>
                        <p>${error.message}</p>
                        <pre style="background: #f8f8f8; padding: 0.5rem; overflow-x: auto;">
                            ${error.stack || 'No stack trace available'}
                        </pre>
                    </div>
                `;
                console.error('GemSpace Fatal Error:', error);
            }
        });
    }

    onunload() {
        console.log(`Enhanced DataviewJS v${PLUGIN_VERSION} unloaded at ${this.getFormattedUTCTime()}`);
    }
}
```

-- src/types.ts --

```
export interface Library {
    name: string;
    url: string;
    version: string;
}

export interface RenderProps {
    MaterialUI: any;
    React: any;
    ReactDOM: any;
}

export interface ComponentConfig {
    libraries: string[];
    render: (props: RenderProps) => void;
}

export interface GemHelpers {
    createButton: (text: string, onClick: () => void) => Promise<void>;
    createInfo: (message: string) => HTMLDivElement;
    createWarning: (message: string) => HTMLDivElement;
    createSuccess: (message: string) => HTMLDivElement;
    createError: (message: string) => HTMLDivElement;
}

export interface IEnhancedDataviewAPI {
    useLibrary: (name: string) => Promise<void>;
    createComponent: (config: ComponentConfig) => Promise<void>;
    createCard: (title: string, content: string) => Promise<void>;
}
```

-- esbuild.config.mjs --

```
import esbuild from "esbuild";
import process from "process";
import builtins from "builtin-modules";

const banner =
`/*
THIS IS A GENERATED/BUNDLED FILE BY ESBUILD
if you want to view the source, please visit the github repository of this plugin
*/
`;

const prod = (process.argv[2] === 'production');

esbuild.build({
    banner: {
        js: banner,
    },
    entryPoints: ['src/main.ts'],
    bundle: true,
    external: [
        'obsidian',
        'electron',
        '@codemirror/autocomplete',
        '@codemirror/collab',
        '@codemirror/commands',
        '@codemirror/language',
        '@codemirror/lint',
        '@codemirror/search',
        '@codemirror/state',
        '@codemirror/view',
        ...builtins],
    format: 'cjs',
    watch: !prod,
    target: 'es2016',
    logLevel: "info",
    sourcemap: prod ? false : 'inline',
    treeShaking: true,
    outfile: 'main.js',
}).catch(() => process.exit(1));

```

-- main.js -

```
/*
THIS IS A GENERATED/BUNDLED FILE BY ESBUILD
if you want to view the source, please visit the github repository of this plugin
*/

var __create = Object.create;
var __defProp = Object.defineProperty;
var __getOwnPropDesc = Object.getOwnPropertyDescriptor;
var __getOwnPropNames = Object.getOwnPropertyNames;
var __getProtoOf = Object.getPrototypeOf;
var __hasOwnProp = Object.prototype.hasOwnProperty;
var __markAsModule = (target) => __defProp(target, "__esModule", { value: true });
var __export = (target, all) => {
  __markAsModule(target);
  for (var name in all)
    __defProp(target, name, { get: all[name], enumerable: true });
};
var __reExport = (target, module2, desc) => {
  if (module2 && typeof module2 === "object" || typeof module2 === "function") {
    for (let key of __getOwnPropNames(module2))
      if (!__hasOwnProp.call(target, key) && key !== "default")
        __defProp(target, key, { get: () => module2[key], enumerable: !(desc = __getOwnPropDesc(module2, key)) || desc.enumerable });
  }
  return target;
};
var __toModule = (module2) => {
  return __reExport(__markAsModule(__defProp(module2 != null ? __create(__getProtoOf(module2)) : {}, "default", module2 && module2.__esModule && "default" in module2 ? { get: () => module2.default, enumerable: true } : { value: module2, enumerable: true })), module2);
};
var __async = (__this, __arguments, generator) => {
  return new Promise((resolve, reject) => {
    var fulfilled = (value) => {
      try {
        step(generator.next(value));
      } catch (e) {
        reject(e);
      }
    };
    var rejected = (value) => {
      try {
        step(generator.throw(value));
      } catch (e) {
        reject(e);
      }
    };
    var step = (x) => x.done ? resolve(x.value) : Promise.resolve(x.value).then(fulfilled, rejected);
    step((generator = generator.apply(__this, __arguments)).next());
  });
};

```

// src/main.ts

```
__export(exports, {
  default: () => EnhancedDataviewJS
});
var import_obsidian = __toModule(require("obsidian"));

// src/lib-manager.ts
var LibraryManager = class {
  constructor() {
    this.libraries = {
      react: {
        name: "react",
        url: "https://unpkg.com/react@17.0.2/umd/react.production.min.js",
        version: "17.0.2"
      },
      reactDOM: {
        name: "reactDOM",
        url: "https://unpkg.com/react-dom@17.0.2/umd/react-dom.production.min.js",
        version: "17.0.2"
      },
      materialUI: {
        name: "materialUI",
        url: "https://unpkg.com/@mui/material@5.14.11/umd/material-ui.production.min.js",
        version: "5.14.11"
      }
    };
    this.loadedLibraries = new Set();
    this.loadingPromises = new Map();
    console.log("Initializing libraries...");
    Object.values(this.libraries).forEach((lib) => {
      console.log(`Registered library: ${lib.name} v${lib.version}`);
    });
  }
  loadLibrary(name) {
    return __async(this, null, function* () {
      if (this.loadedLibraries.has(name)) {
        return;
      }
      if (this.loadingPromises.has(name)) {
        return this.loadingPromises.get(name);
      }
      const library = this.libraries[name];
      if (!library) {
        throw new Error(`Library ${name} not found`);
      }
      const loadingPromise = this.loadScript(library.url).then(() => {
        this.loadedLibraries.add(name);
        this.loadingPromises.delete(name);
        console.log(`Loaded library: ${name}`);
      });
      this.loadingPromises.set(name, loadingPromise);
      return loadingPromise;
    });
  }
  loadScript(url) {
    return new Promise((resolve, reject) => {
      if (document.querySelector(`script[src="${url}"]`)) {
        resolve();
        return;
      }
      const script = document.createElement("script");
      script.src = url;
      script.onload = () => resolve();
      script.onerror = () => reject(new Error(`Failed to load script: ${url}`));
      document.head.appendChild(script);
    });
  }
};

// src/api-extensions.ts
var EnhancedDataviewAPI = class {
  constructor() {
    this.libManager = new LibraryManager();
  }
  useLibrary(name) {
    return __async(this, null, function* () {
      yield this.libManager.loadLibrary(name);
    });
  }
  createComponent(config) {
    return __async(this, null, function* () {
      try {
        for (const lib of config.libraries) {
          yield this.useLibrary(lib);
        }
        const props = {
          MaterialUI: window.MaterialUI,
          React: window.React,
          ReactDOM: window.ReactDOM
        };
        return config.render(props);
      } catch (error) {
        console.error("Failed to create component:", error);
        throw error;
      }
    });
  }
  createCard(title, content) {
    return __async(this, null, function* () {
      yield this.useLibrary("react");
      yield this.useLibrary("reactDOM");
      yield this.useLibrary("materialUI");
      return this.createComponent({
        libraries: ["react", "reactDOM", "materialUI"],
        render: (props) => {
          const { Card, Typography, Box } = props.MaterialUI;
          const cardContainer = document.createElement("div");
          const element = props.React.createElement(Card, {
            sx: { p: 2, m: 2 }
          }, props.React.createElement(Typography, {
            variant: "h6"
          }, title), props.React.createElement(Box, {
            sx: { mt: 1 }
          }, content));
          props.ReactDOM.render(element, cardContainer);
          document.body.appendChild(cardContainer);
        }
      });
    });
  }
};

// src/main.ts
var PLUGIN_VERSION = "1.0.0";
var EnhancedDataviewJS = class extends import_obsidian.Plugin {
  constructor() {
    super(...arguments);
    this.currentUser = "farizessentia";
  }
  getFormattedUTCTime() {
    const now = new Date();
    return now.toISOString().slice(0, 19).replace("T", " ");
  }
  onload() {
    return __async(this, null, function* () {
      console.log(`Loading Enhanced DataviewJS v${PLUGIN_VERSION} - ${this.getFormattedUTCTime()}`);
      this.enhancedAPI = new EnhancedDataviewAPI();
      this.registerMarkdownCodeBlockProcessor("gemspace", (source, el, ctx) => __async(this, null, function* () {
        try {
          const dataviewPlugin = this.app.plugins.getPlugin("dataview");
          if (!dataviewPlugin) {
            throw new Error("DataviewJS plugin is required");
          }
          const dataviewApi = dataviewPlugin.api;
          const container = el.createDiv("gemspace-container");
          const dv = {
            version: PLUGIN_VERSION,
            currentUser: this.currentUser,
            getCurrentUTCTime: () => this.getFormattedUTCTime(),
            pages: () => dataviewApi.pages(),
            page: (path) => dataviewApi.page(path),
            pagePaths: () => dataviewApi.pagePaths(),
            current: () => dataviewApi.current(),
            filterPages: (pages, ...filters) => dataviewApi.filterPages(pages, ...filters),
            isArray: (value) => Array.isArray(value),
            date: (input) => dataviewApi.date(input),
            header: (text) => {
              const header = container.createEl("h1");
              header.textContent = text;
            },
            paragraph: (text) => {
              const p = container.createEl("p");
              p.textContent = String(text);
            },
            list: (items) => {
              const ul = container.createEl("ul");
              items.forEach((item) => {
                const li = ul.createEl("li");
                li.textContent = String(item);
              });
            },
            table: (headers, content) => {
              const table = container.createEl("table");
              const thead = table.createEl("thead");
              const headerRow = thead.createEl("tr");
              headers.forEach((header) => {
                const th = headerRow.createEl("th");
                th.textContent = header;
              });
              const tbody = table.createEl("tbody");
              content.forEach((row) => {
                const tr = tbody.createEl("tr");
                row.forEach((cell) => {
                  const td = tr.createEl("td");
                  td.textContent = String(cell);
                });
              });
            },
            formatDate: (date) => {
              if (typeof (date == null ? void 0 : date.toFormat) === "function") {
                return date.toFormat("yyyy-MM-dd HH:mm:ss");
              }
              return "Invalid Date";
            },
            createContainer: () => {
              return container.createDiv("gemspace-section");
            },
            enhanced: this.enhancedAPI,
            container
          };
          const gemHelpers = {
            createButton: (text, onClick) => __async(this, null, function* () {
              yield dv.enhanced.useLibrary("react");
              yield dv.enhanced.useLibrary("reactDOM");
              yield dv.enhanced.useLibrary("materialUI");
              return dv.enhanced.createComponent({
                libraries: ["react", "reactDOM", "materialUI"],
                render: (props) => {
                  const { Button } = props.MaterialUI;
                  const buttonContainer = container.createDiv("gemspace-button");
                  const element = props.React.createElement(Button, {
                    variant: "contained",
                    color: "primary",
                    onClick,
                    style: {
                      margin: "8px 0"
                    }
                  }, text);
                  props.ReactDOM.render(element, buttonContainer);
                }
              });
            }),
            createInfo: (message) => {
              const info = container.createDiv("gemspace-info");
              info.innerHTML = `\u2139\uFE0F ${message}`;
              return info;
            },
            createWarning: (message) => {
              const warning = container.createDiv("gemspace-warning");
              warning.innerHTML = `\u26A0\uFE0F ${message}`;
              return warning;
            },
            createSuccess: (message) => {
              const success = container.createDiv("gemspace-success");
              success.innerHTML = `\u2705 ${message}`;
              return success;
            },
            createError: (message) => {
              const error = container.createDiv("gemspace-error");
              error.innerHTML = `\u274C ${message}`;
              return error;
            }
          };
          Object.defineProperty(dv, "gem", {
            value: gemHelpers,
            enumerable: true,
            configurable: true,
            writable: false
          });
          const wrappedCode = `(async () => {
                    try {
                        // Add version and time info
                        dv.gem.createInfo(\`GemSpace v\${dv.version} - \${dv.getCurrentUTCTime()}\`);
                        
                        ${source}
                    } catch (error) {
                        dv.gem.createError(\`Runtime Error: \${error.message}\`);
                        console.error('GemSpace Runtime Error:', error);
                    }
                })()`;
          const AsyncFunction = Object.getPrototypeOf(function() {
            return __async(this, null, function* () {
            });
          }).constructor;
          const fn = new AsyncFunction("dv", wrappedCode);
          yield fn(dv);
        } catch (error) {
          const errorDiv = el.createDiv("gemspace-fatal-error");
          errorDiv.innerHTML = `
                    <div style="color: red; padding: 1rem; border: 1px solid red; border-radius: 4px; margin: 1rem 0;">
                        <h3>\u26A0\uFE0F GemSpace Error</h3>
                        <p>${error.message}</p>
                        <pre style="background: #f8f8f8; padding: 0.5rem; overflow-x: auto;">
                            ${error.stack || "No stack trace available"}
                        </pre>
                    </div>
                `;
          console.error("GemSpace Fatal Error:", error);
        }
      }));
    });
  }
  onunload() {
    console.log(`Enhanced DataviewJS v${PLUGIN_VERSION} unloaded at ${this.getFormattedUTCTime()}`);
  }
};
```

-- manifest.json --

```
{
    "id": "enhanced-dataviewjs",
    "name": "Enhanced DataviewJS",
    "version": "1.0.0",
    "minAppVersion": "0.15.0",
    "description": "Enhances DataviewJS with support for external libraries and improved functionality",
    "author": "your-name",
    "authorUrl": "https://github.com/your-username",
    "isDesktopOnly": false
}

```


-- package.json --

```
{
    "name": "obsidian-enhanced-dataviewjs",
    "version": "1.0.0",
    "description": "Enhanced DataviewJS functionality for Obsidian",
    "main": "main.js",
    "scripts": {
        "dev": "node esbuild.config.mjs",
        "build": "tsc -noEmit && node esbuild.config.mjs production",
        "version": "node version-bump.mjs && git add manifest.json versions.json"
    },
    "keywords": ["obsidian", "dataviewjs", "enhancement"],
    "author": "your-name",
    "license": "MIT",
    "devDependencies": {
        "@types/node": "^16.11.6",
        "@typescript-eslint/eslint-plugin": "^5.2.0",
        "@typescript-eslint/parser": "^5.2.0",
        "builtin-modules": "^3.2.0",
        "esbuild": "0.13.12",
        "obsidian": "^1.4.0",
        "tslib": "2.3.1",
        "typescript": "^4.9.0"
    }
}

```

-- styles.css --

```
.gemspace-info {
    padding: 8px 16px;
    margin: 8px 0;
    background: var(--background-secondary);
    border-left: 4px solid #4a9eff;
    border-radius: 4px;
}

.gemspace-warning {
    padding: 8px 16px;
    margin: 8px 0;
    background: var(--background-secondary);
    border-left: 4px solid #ffb347;
    border-radius: 4px;
}

.gemspace-success {
    padding: 8px 16px;
    margin: 8px 0;
    background: var(--background-secondary);
    border-left: 4px solid #28a745;
    border-radius: 4px;
}

.gemspace-error {
    padding: 8px 16px;
    margin: 8px 0;
    background: var(--background-secondary);
    border-left: 4px solid #dc3545;
    border-radius: 4px;
}

.gemspace-button {
    margin: 8px 0;
}

.gemspace-fatal-error {
    padding: 16px;
    margin: 16px 0;
    background: #fff1f0;
    border: 1px solid #dc3545;
    border-radius: 4px;
}

.gemspace-fatal-error h3 {
    color: #dc3545;
    margin: 0 0 8px 0;
}

.gemspace-fatal-error pre {
    background: #f8f8f8;
    padding: 8px;
    overflow-x: auto;
    border-radius: 4px;
}
```

-- tsconfig.json --

```
{
    "compilerOptions": {
        "baseUrl": "src",
        "inlineSourceMap": true,
        "inlineSources": true,
        "module": "ESNext",
        "target": "ES6",
        "allowJs": true,
        "noImplicitAny": true,
        "moduleResolution": "node",
        "importHelpers": true,
        "isolatedModules": true,
        "strictNullChecks": true,
        "lib": [
            "DOM",
            "ES5",
            "ES6",
            "ES7",
            "ES2021"
        ],
        "skipLibCheck": true,
        "types": ["node"]
    },
    "include": [
        "**/*.ts"
    ]
}
```

----------------------------------------------------------------
Purpose and Overview:
This plugin is an enhancement for the DataviewJS functionality in Obsidian, providing additional features and improved capabilities for rendering and manipulating data. It's designed to work alongside the existing Dataview plugin while adding new functionality.

Key Features:

External Library Integration:

Supports React (v17.0.2)
ReactDOM (v17.0.2)
Material-UI (v5.14.11)
Manages library loading dynamically through the LibraryManager class
Enhanced UI Components:

Custom card creation with Material-UI integration
Styled notifications/messages:
Info messages (blue)
Warning messages (orange)
Success messages (green)
Error messages (red)
Material-UI button components
Core Functionality:

Extended DataviewJS API with additional features
Custom component creation system
Error handling and reporting
UTC time formatting
Table generation
List creation
Container management

