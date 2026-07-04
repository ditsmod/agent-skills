---
name: ditsmod-rest-starter
description: Installs a starter REST application or configures/sets up a minimal REST application ("Hello, World!").
---

## Installation

```bash
git clone --depth 1 https://github.com/ditsmod/rest-starter.git my-app
cd my-app
rm -rf .git
git init
git add .
git commit -m 'Initial commit from rest-starter template'
npm i
```

Alternatively, you can use the starter monorepo:

```bash
git clone --depth 1 https://github.com/ditsmod/rest-monorepo-starter.git my-app
cd my-app
rm -rf .git
git init
git add .
git commit -m 'Initial commit from rest-monorepo-starter template'
npm i
```

## Minimal REST-application

Use this minimal setup instead of cloning a full starter repository when:
- The user requests a minimal, single-file, or lightweight demonstration/POC of Ditsmod.
- The project structure does not require a full starter template architecture.
- For quick debugging or unit testing of a simple Ditsmod REST application.

### Prerequisites

To run this minimal application, ensure you have:
1. Node.js >= v24.0.0.
2. The required Ditsmod and TypeScript packages installed:
   ```bash
   npm i @ditsmod/core @ditsmod/rest reflect-metadata
   npm i -D typescript @types/node
   ```
3. The following compiler options enabled in `tsconfig.json`:
   ```json
   {
     "compilerOptions": {
       "experimentalDecorators": true,
       "emitDecoratorMetadata": true
     }
   }
   ```

### Code Example

You can save this code in a file (e.g., `index.ts`) and run it:

```ts
import { controller, route, restRootModule, RestApplication } from '@ditsmod/rest';

@controller()
class ExampleController {
  @route('GET', 'hello')
  tellHello() {
    return 'Hello, World!';
  }
}

@restRootModule({ controllers: [ExampleController] })
class AppModule {}

const app = await RestApplication.create(AppModule);
app.server.listen(3000, '0.0.0.0');
```
