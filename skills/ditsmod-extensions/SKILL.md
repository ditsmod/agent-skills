---
name: ditsmod-extensions
description: Practical guidance for creating, implementing, registering, configuring, ordering, exporting, grouping, and overriding Ditsmod extensions. Use when you need to write custom extensions with stage1/stage2/stage3 hooks, inject ExtensionManager to retrieve same-module or app-wide data (handling delay), dynamically register providers, or configure module extensions metadata.
---

# Ditsmod Extensions

Use this skill when writing custom Ditsmod extensions or configuring metadata for existing extensions.

## Application Mental Model

Extensions run after Ditsmod has collected static metadata from decorators, module imports/exports, and providers. They usually run before request handlers are created.

Extensions are like "infrastructure providers" (similar to a cloud provider):

- Business logic is written in services, not in extensions.
- Extensions prepare the infrastructure (e.g., dynamically adding route interceptors, setting up OpenAPI docs, or initializing database connections).

Extensions can be asynchronous and depend on each other.

---

## Implementing An Extension

An extension is a class decorated with `@injectable()` that implements the `Extension` interface:

```ts
import { injectable, Extension, Logger } from "@ditsmod/core";

@injectable()
export class MyExtension implements Extension<any> {
  constructor(private logger: Logger) {}

  async stage1(isLastModule: boolean) {
    // Stage 1: Called when providers are dynamically collected.
  }

  async stage2(injectorPerMod: Injector) {
    // Stage 2: Called after stage1 runs for all modules.
  }

  async stage3() {
    // Stage 3: Called after stage2 runs for all modules.
  }
}
```

### Constructor Injection Constraints

- **CRITICAL**: The injector for the extension constructor is initialized **before any extensions run**.
- You can inject statically registered module-level or app-level providers.
- You **cannot** inject providers in the constructor that are dynamically added by other extensions.

---

## Registering An Extension

Register extensions in the module metadata `extensions` array using decorators like `@featureModule`, `@restModule`, etc.

### Direct Class Registration

For local extensions that only need to run in the current module:

```ts
@restModule({
  extensions: [SimpleExtension],
})
export class AppModule {}
```

### Config Object Registration

Use the object form for advanced configuration:

```ts
@restModule({
  extensions: [
    {
      extension: SimpleExtension,
      beforeExtensions: [RestRouteExtension],
      afterExtensions: [],
      export: true,
    },
  ],
})
export class FeatureModule {}
```

---

## Configuration Properties

### Ordering

- `beforeExtensions`: Array of extensions that must run **after** this extension.
- `afterExtensions`: Array of extensions that must run **before** this extension.

### Exporting

- `export: true`: The extension runs in the host module and in any module that imports this module.
- `exportOnly: true`: The extension is made available to importing modules, but **does not run** in the host module where it is declared.

### Overriding

- `overrideExtension`: Replaces an imported external extension with a custom one:
  ```ts
  extensions: [
    { extension: MyExtension, overrideExtension: ExternalExtension },
  ];
  ```

---

## Extension Groups

Groups allow extensions to be abstracted as types of work. Extensions in a group should return metadata sharing a common base interface.

- Use `groups` to register an extension into one or more groups:
  ```ts
  extensions: [
    {
      extension: OpenapiRouteExtension,
      groups: [RestRouteExtension],
      export: true,
    },
  ];
  ```
- The group token is the extension class (e.g., `RestRouteExtension`).
- When another extension requests data for `RestRouteExtension`, it receives the outputs of both `RestRouteExtension` and `OpenapiRouteExtension`.

---

## Using ExtensionManager

`ExtensionManager` manages dependencies, caches stage1/group results, and detects cyclic dependency chains.

Inject it in the constructor:

```ts
constructor(private extensionManager: ExtensionManager) {}
```

### 1. Same-Module Dependency

To retrieve data from an extension/group in the current module:

```ts
const stage1ExtensionMeta = await this.extensionManager.stage1(ExtensionToken);
// Use stage1ExtensionMeta.groupData (array of payloads)
```

### 2. Application-Wide Dependency (Cross-Module)

If your extension requires aggregated data from an extension across the entire application, you must pass `this` as the second argument:

```ts
const stage1ExtensionMeta = await this.extensionManager.stage1(
  ExtensionToken,
  this,
);
```

- A separate instance of your extension is created for each module. Passing `this` guarantees that the framework waits until the target extension executes across all modules before resolving.

### Handling `delay`

When retrieving application-wide data, you **MUST** check the `delay` property:

```ts
const stage1ExtensionMeta = await this.extensionManager.stage1(
  OtherExtension,
  this,
);
if (stage1ExtensionMeta.delay) {
  return; // Stop execution: Framework will invoke this extension again later
}

// Safe to access aggregated data:
stage1ExtensionMeta.groupDataPerApp.forEach((totalStage1Meta) => {
  totalStage1Meta.groupData.forEach((payload) => {
    // ...
  });
});
```

---

## Dynamic Provider Addition

Extensions frequently dynamically register providers.

1. **In `stage1()`**: Providers can be dynamically added to any level (app, module, route, request).
2. **In `stage2(injectorPerMod)`**: Providers can still be dynamically added, but **only to levels lower than the module** (e.g., route or request level).

### Code Pattern

To read configs before registering interceptors/providers, resolve them using a temporary injector hierarchy:

```ts
async stage1() {
  const stage1ExtensionMeta = await this.extensionManager.stage1(RestRouteExtension);
  stage1ExtensionMeta.groupData.forEach((metadataPerMod3) => {
    const { providersPerMod } = metadataPerMod3.baseMeta;

    metadataPerMod3.aControllerMetadata.forEach(({ providersPerReq, scope }) => {
      // 1. Setup temporary injector hierarchy to resolve config
      const injectorPerApp = Injector.resolveAndCreate(this.providersPerApp, 'App');
      const injectorPerMod = injectorPerApp.resolveAndCreateChild(providersPerMod);

      // 2. Read configuration
      const config = injectorPerMod.get(ConfigToken, {});

      // 3. Conditionally register provider/interceptor
      if (config.enableFeature) {
        providersPerReq.push({
          token: HTTP_INTERCEPTORS,
          useClass: MyInterceptor,
          multi: true
        });
      }
    });
  });
}
```

Ensure your extension runs in the correct sequence (e.g., after the metadata is collected by `RestRouteExtension` but before it's finalized by `PreRouterExtension`).

---

## Choosing a Registration Configuration

Use this decision guide to determine how to configure the extension in a module's metadata (e.g., `@featureModule`, `@restModule`) `extensions` array:

- **Need the extension only in this module**: `extensions: [MyExtension]`.
- **Need ordering constraints**: object form with `beforeExtensions` or `afterExtensions`.
- **Need importing modules to run it**: add `export: true`.
- **Need to export without running in host module**: use `exportOnly: true`.
- **Need to contribute data to an existing pipeline group**: use `groups`.
- **Need to replace an imported extension**: use `overrideExtension`.

---

## Troubleshooting

- **Extension does not run in importing module**: Verify it has `export: true` or `exportOnly: true` in its host module.
- **Extension runs too early**: Add `afterExtensions` for the target extension it depends on.
- **Extension runs too late**: Add `beforeExtensions` for the target extension that consumes its output.
- **Group data is missing**: Verify that you registered with `groups` using the correct extension class token.
- **Cyclic dependency**: Check the error log; `ExtensionManager` will print the full loop path.
- **Aggregated data is incomplete/stale**: Verify you passed `this` as the second argument to `extensionManager.stage1(Token, this)` and handled `delay` by returning early.
