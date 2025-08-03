# MFE-With-webcomponent

You can absolutely **expose the same Angular module as both a Module Federation Remote and a Web Component** from a single Angular 16 project. Here's a detailed **step-by-step guide** to achieve this while supporting **Inputs (props)** and **Outputs (events)** for external communication:

---

# **1️⃣ Prerequisites**

* Angular 16 project already configured with **Module Federation** (via `@angular-architects/module-federation` or custom Webpack setup).
* A shared module/component you want to expose (e.g., `FormModule` containing `FormComponent`).

Goal:

* Continue exposing this module via **Module Federation**.
* Additionally, **bootstrap the same module as a Web Component** that can be used in any HTML page or non-Angular host.

---

# **2️⃣ Install Required Packages**

Angular Elements is needed to wrap components as Web Components:

```bash
npm install @angular/elements
```

---

# **3️⃣ Prepare Your Component for Inputs and Outputs**

Example: `form.component.ts`

```typescript
import { Component, EventEmitter, Input, Output } from '@angular/core';

@Component({
  selector: 'app-form',
  template: `
    <form (ngSubmit)="onSubmit()">
      <label>{{label}}</label>
      <input [(ngModel)]="formData" name="data"/>
      <button type="submit">Submit</button>
    </form>
  `
})
export class FormComponent {
  @Input() label: string = 'Default Label';      // Input property
  @Output() formSubmitted = new EventEmitter<string>(); // Output event

  formData: string = '';

  onSubmit() {
    this.formSubmitted.emit(this.formData);
  }
}
```

✅ This ensures:

* The component accepts **dynamic data (props)**.
* Emits an **event** on form submission.

---

# **4️⃣ Add a Web Component Bootstrap Module**

Create a new file: `app-web-component.module.ts`

This module will **bootstrap the component as a Web Component** only when building for web components.

```typescript
import { Injector, NgModule } from '@angular/core';
import { BrowserModule } from '@angular/platform-browser';
import { createCustomElement } from '@angular/elements';
import { FormComponent } from './form/form.component';
import { FormsModule } from '@angular/forms';

@NgModule({
  declarations: [FormComponent],
  imports: [BrowserModule, FormsModule],
  providers: [],
})
export class AppWebComponentModule {
  constructor(private injector: Injector) {}

  ngDoBootstrap() {
    const formElement = createCustomElement(FormComponent, { injector: this.injector });
    customElements.define('my-form-element', formElement);
  }
}
```

✅ This wraps your existing `FormComponent` and registers it as `<my-form-element>` in the browser.

---

# **5️⃣ Adjust `main.ts` for Dual Bootstrapping**

You need to **decide at runtime or build time** whether to bootstrap the Angular app (for MF) or register the web component.

Modify `main.ts`:

```typescript
import { platformBrowserDynamic } from '@angular/platform-browser-dynamic';
import { AppModule } from './app/app.module';
import { AppWebComponentModule } from './app/app-web-component.module';

const mode = (window as any)['WEB_COMPONENT_MODE'];

if (mode) {
  platformBrowserDynamic().bootstrapModule(AppWebComponentModule)
    .catch(err => console.error(err));
} else {
  platformBrowserDynamic().bootstrapModule(AppModule)
    .catch(err => console.error(err));
}
```

You can pass `window.WEB_COMPONENT_MODE = true;` before loading your JS in a non-Angular host.

---

# **6️⃣ Expose Module for Module Federation**

Your existing `webpack.config.js` (for MF) likely has:

```javascript
const ModuleFederationPlugin = require('webpack/lib/container/ModuleFederationPlugin');

module.exports = {
  plugins: [
    new ModuleFederationPlugin({
      name: "mfe",
      filename: "remoteEntry.js",
      exposes: {
        './FormModule': './src/app/form/form.module.ts',
      },
      shared: {
        "@angular/core": { singleton: true, strictVersion: true },
        "@angular/common": { singleton: true },
        "@angular/router": { singleton: true },
      },
    }),
  ],
};
```

No change needed here—you already expose `FormModule` for MF.

---

# **7️⃣ Build Targets**

You will have **two different build commands**:

### a) **Module Federation Build (unchanged)**

```bash
ng build --configuration production
```

Produces `remoteEntry.js` for MF host consumption.

---

### b) **Web Component Build**

Create a separate `angular.json` configuration:

```json
"web-component": {
  "builder": "@angular-devkit/build-angular:browser",
  "options": {
    "outputPath": "dist/web-component",
    "main": "src/main.ts",
    "index": "src/index.html",
    "polyfills": ["zone.js"],
    "scripts": [],
    "stylePreprocessorOptions": { "includePaths": ["src/styles"] }
  }
}
```

Then build:

```bash
ng build --configuration web-component
```

Include this `main.js` in any static HTML with:

```html
<script>
  window.WEB_COMPONENT_MODE = true;
</script>
<script src="main.js"></script>

<my-form-element label="Your Name"></my-form-element>

<script>
  document.querySelector('my-form-element')
    .addEventListener('formSubmitted', (e) => {
      console.log('Form submitted:', e.detail);
    });
</script>
```

---

# ✅ Final Outcome

* **Module Federation Remote:** Works as before via `remoteEntry.js`.
* **Web Component:** You can now drop `<my-form-element>` in any HTML or host (React, Vue, plain JS), with:

  * **Input:** `<my-form-element label="Enter Name"></my-form-element>`
  * **Output Event:** `formSubmitted` available as a custom DOM event.

---

Yes, you can absolutely **load your Angular Web Component dynamically at runtime** in another Angular app without bundling it upfront. This is a common approach to make your consuming app independent of the Web Component’s release cycle.

Here’s how you can do it:

---

## **1️⃣ Host your Web Component**

* Build the Web Component from App A:

```bash
ng build --configuration web-component
```

* Deploy the output (`main.js`, `polyfills.js`, `runtime.js`) to any **server/CDN** (e.g., `https://cdn.example.com/my-form-element/`).

---

## **2️⃣ Dynamically load scripts in the consuming app**

In your consuming Angular App (App B), you can write a **dynamic loader service** to append the script tags at runtime.

### `script-loader.service.ts`

```typescript
import { Injectable } from '@angular/core';

@Injectable({
  providedIn: 'root',
})
export class ScriptLoaderService {
  loadScript(src: string): Promise<void> {
    return new Promise((resolve, reject) => {
      if (document.querySelector(`script[src="${src}"]`)) {
        resolve(); // Already loaded
        return;
      }
      const script = document.createElement('script');
      script.type = 'text/javascript';
      script.src = src;
      script.onload = () => resolve();
      script.onerror = (err) => reject(err);
      document.body.appendChild(script);
    });
  }
}
```

---

## **3️⃣ Load the Web Component before usage**

In your consuming component:

```typescript
import { Component, AfterViewInit } from '@angular/core';
import { ScriptLoaderService } from './script-loader.service';

@Component({
  selector: 'app-root',
  template: `
    <h1>Host Angular App</h1>
    <div id="container"></div>
  `,
})
export class AppComponent implements AfterViewInit {
  constructor(private loader: ScriptLoaderService) {}

  async ngAfterViewInit() {
    await this.loader.loadScript('https://cdn.example.com/my-form-element/runtime.js');
    await this.loader.loadScript('https://cdn.example.com/my-form-element/polyfills.js');
    await this.loader.loadScript('https://cdn.example.com/my-form-element/main.js');

    // Now you can add the element dynamically
    const el = document.createElement('my-form-element');
    el.setAttribute('label', 'Dynamic Load Name');
    el.addEventListener('formSubmitted', (e: any) => {
      console.log('Form submitted dynamically:', e.detail);
    });
    document.getElementById('container')?.appendChild(el);
  }
}
```

---

## **4️⃣ Allow custom elements schema**

In `app.module.ts`:

```typescript
import { NgModule, CUSTOM_ELEMENTS_SCHEMA } from '@angular/core';
import { BrowserModule } from '@angular/platform-browser';
import { AppComponent } from './app.component';

@NgModule({
  declarations: [AppComponent],
  imports: [BrowserModule],
  bootstrap: [AppComponent],
  schemas: [CUSTOM_ELEMENTS_SCHEMA],
})
export class AppModule {}
```

---

# ✅ Result

* You can **update your Web Component independently** and just publish a new JS build to the CDN.
* The host Angular app **fetches and runs the latest version at runtime** without rebuilding.
* Inputs (`setAttribute`) and Outputs (`addEventListener`) still work exactly like when statically loaded.

---

------------------------------------------------------------------------------------------------------------------------------------------------------


To get a **single optimized and compressed `bundle.js`** for your Angular Web Component (Angular 16), you can tweak your build to:

* **Bundle everything** into one file.
* **Tree-shake unused code**.
* **Minify and compress the output** for minimal file size.

Here's the **final recommended configuration**:

---

## **1️⃣ Install Required Packages**

```bash
npm install @angular-builders/custom-webpack --save-dev
npm install terser-webpack-plugin --save-dev
```

---

## **2️⃣ Update `angular.json`**

Add a **custom webpack configuration** for your "web-component" build:

```json
"build": {
  "builder": "@angular-builders/custom-webpack:browser",
  "options": {
    "customWebpackConfig": {
      "path": "./webpack.single-bundle.js"
    },
    "outputPath": "dist/web-component",
    "index": "src/index.html",
    "main": "src/main.ts",
    "polyfills": ["zone.js"],
    "scripts": [],
    "aot": true,
    "optimization": true,
    "outputHashing": "none",
    "extractLicenses": true,
    "buildOptimizer": true,
    "vendorChunk": false,
    "namedChunks": false,
    "sourceMap": false
  }
}
```

---

## **3️⃣ Create `webpack.single-bundle.js`**

This merges all chunks, minifies, and outputs a single UMD bundle.

```javascript
const { merge } = require('webpack-merge');
const TerserPlugin = require('terser-webpack-plugin');

module.exports = (config) => {
  return merge(config, {
    optimization: {
      runtimeChunk: false,
      splitChunks: false,
      minimize: true,
      minimizer: [
        new TerserPlugin({
          extractComments: false,
          terserOptions: {
            compress: {
              drop_console: true,
              pure_funcs: ['console.log'] // remove console logs
            },
            output: {
              comments: false
            }
          }
        })
      ],
    },
    output: {
      filename: 'my-form-element.bundle.js',
      library: 'MyFormElementLib',
      libraryTarget: 'umd',
      clean: true
    },
    performance: {
      hints: false
    }
  });
};
```

---

## **4️⃣ Optimize Angular for minimal size**

In `main.ts`, **disable Angular dev mode** for production:

```typescript
import { enableProdMode } from '@angular/core';
enableProdMode();
```

And make sure you build with production mode:

```bash
ng build --configuration web-component
```

---

## **5️⃣ Result**

Your `dist/web-component` folder will now contain:

```
my-form-element.bundle.js  (Single compressed <~100–200KB gzipped, depending on your component)
```

This file:

* Contains all required Angular runtime, polyfills, and component code.
* Is tree-shaken, minified, and stripped of `console.log`.
* Can be dropped into any HTML page or Angular/React host:

```html
<script src="my-form-element.bundle.js"></script>

<my-form-element label="Your Name"></my-form-element>

<script>
  document.querySelector('my-form-element')
    .addEventListener('formSubmitted', e => console.log('Submitted:', e.detail));
</script>
```

---

### ✅ Tips for even smaller bundle:

1. **Use Angular Standalone Components** instead of large NgModules if possible (removes unused DI code).
2. **Avoid importing entire libraries** (e.g., only import what you need from `rxjs`).
3. **Enable build optimizer** and **disable source maps** as shown above.
4. If multiple web components are needed, consider **externalizing Angular** (`externals`) so multiple bundles can share one runtime.

---




