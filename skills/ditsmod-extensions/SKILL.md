---
name: ditsmod-extensions
description: Practical guidance for creating, registering, ordering, exporting, grouping, and overriding Ditsmod extensions. Use when writing custom extensions with stage1/stage2/stage3 hooks, injecting ExtensionManager to retrieve same-module or app-wide data (handling delay), dynamically registering providers, configuring module extensions metadata, or resolving cyclic extension dependencies.
compatibility: Ditsmod project with @ditsmod/core >=3.0. Node.js >=24.
---

# Ditsmod Extensions

## Mental Model

Extensions run **after** Ditsmod collects static metadata from decorators but **before** request handlers are created. Think of them as "infrastructure setup" hooks that operate on the fully assembled module graph.

- Business logic belongs in services, not extensions.
- Extensions prepare infrastructure: dynamically add interceptors/providers, build route metadata, set up OpenAPI docs, initialize DB connections.
- Extensions can be async and depend on each other through `ExtensionManager`.

---

## Implementing An Extension

An extension is a class decorated with `@injectable()` that implements the `Extension<T>` interface. All three stage methods are **optional** — implement only the stages you need:

```ts
import { injectable } from '@ditsmod/core';
import type { Extension, ExtensionManager, Injector } from '@ditsmod/core';

@injectable()
export class MyExtension implements Extension<MyPayload | void> {
  constructor(private extensionManager: ExtensionManager) {}

  async stage1(isLastModule: boolean): Promise<MyPayload | void> {
    // Stage 1: called once per module while providers are being dynamically collected.
    // isLastModule === true when this is the last module that imports this extension.
    // Return a payload that other extensions can consume via ExtensionManager.stage1().
  }

  async stage2(injectorPerMod: Injector): Promise<void> {
    // Stage 2: called after stage1 finishes for ALL modules.
    // Receives a ready module-level injector.
  }

  async stage3(): Promise<void> {
    // Stage 3: called after stage2 finishes for ALL modules.
    // No strict role — use for final post-processing.
  }
}
```

### Constructor Injection Constraints

- The injector for the extension constructor is created **before any extension runs**.
- You can only inject providers that were **statically** registered (module-level or app-level) before extensions execute.
- You **cannot** inject providers that are dynamically added by other extensions.

### `isLastModule` Parameter

`stage1(isLastModule: boolean)` receives `true` only for the final module that imports this extension. Use this flag to defer aggregation work (e.g., finalizing a data structure) until all modules have contributed.

---

## Registering An Extension

Add extensions to the `extensions` array of any module decorator (`@featureModule`, `@restModule`, etc.).

### Direct Class Registration

Simplest form — runs only in the current module with no ordering constraints:

```ts
@restModule({
  extensions: [MyExtension],
})
export class AppModule {}
```

### Config Object Registration

Use the config object form for ordering, exporting, grouping, or overriding:

```ts
@restModule({
  extensions: [
    {
      extension: MyExtension,
      beforeExtensions: [ConsumerExtension], // MyExtension runs before ConsumerExtension
      afterExtensions: [ProducerExtension], // MyExtension runs after ProducerExtension
      groups: [RestRouteExtension], // MyExtension joins the RestRouteExtension group
      export: true, // also runs in every module that imports this module
    },
  ],
})
export class FeatureModule {}
```

`export` and `exportOnly` are mutually exclusive. `beforeExtensions`, `afterExtensions`, and `groups` are **not** available when using `overrideExtension` (see below).

---

## Configuration Properties Reference

### Ordering

| Property           | Meaning                                          |
| ------------------ | ------------------------------------------------ |
| `beforeExtensions` | This extension runs **before** each listed class |
| `afterExtensions`  | This extension runs **after** each listed class  |

Both accept an array of `ExtensionClass` values.

### Export Behaviour

| Property           | Runs in host module? | Runs in importing modules? |
| ------------------ | -------------------- | -------------------------- |
| _(default)_        | Yes                  | No                         |
| `export: true`     | Yes                  | Yes                        |
| `exportOnly: true` | **No**               | Yes                        |

> [!TIP]
> When registering extensions that are only intended to run in importing modules (such as in wrapper, library, or shared modules), always prefer `exportOnly: true` over `export: true`. This prevents the extension from executing unnecessarily in the host module itself, which can lead to redundant execution or runtime errors due to missing configuration/routes in the host.

### Overriding

Replace an imported extension with a custom one. The override form **only** accepts `extension` and `overrideExtension` — `beforeExtensions`, `afterExtensions`, `groups`, `export`, and `exportOnly` are **not** allowed in this form:

```ts
extensions: [{ extension: MyExtension, overrideExtension: ExternalExtension }];
```

`MyExtension` is registered under the token `ExternalExtension`, so all existing consumers transparently receive `MyExtension`.

---

## Extension Groups

A group lets multiple extensions contribute data under a single shared token.

```ts
extensions: [
  {
    extension: OpenApiRouteExtension,
    groups: [RestRouteExtension], // joins the RestRouteExtension group
    export: true,
  },
];
```

When another extension calls `extensionManager.stage1(RestRouteExtension)`, it receives the aggregated `groupData` from **both** `RestRouteExtension` and `OpenApiRouteExtension`.

---

## Using ExtensionManager

`ExtensionManager` manages extension dependencies, caches stage1 results, and detects cyclic dependencies.

Inject it in the constructor:

```ts
constructor(private extensionManager: ExtensionManager) {}
```

### Same-Module Data

Retrieves the `stage1` output of `TargetExtension` (or its group) within the current module only:

```ts
const meta = await this.extensionManager.stage1(TargetExtension);
// meta.groupData — array of T values returned by each extension in the group/class
// meta.delay     — always false for same-module calls
```

### App-Wide Data (Cross-Module)

Pass `this` as the second argument to receive aggregated data from **all** modules:

```ts
const meta = await this.extensionManager.stage1(TargetExtension, this);
```

A separate instance of your extension is created per module. Passing `this` tells the framework to wait until `TargetExtension` has completed stage1 across all modules before resolving.

#### Handling `delay`

When requesting app-wide data you **must** check `meta.delay`. If `true`, not all modules have run yet — return early; the framework will call your extension again later:

```ts
const meta = await this.extensionManager.stage1(OtherExtension, this);
if (meta.delay) {
  return; // not ready yet — framework will re-invoke this extension
}

// meta.groupDataPerApp: AppExtensionGroupMeta<T>[]
//   Each entry = result from one module that registered OtherExtension
meta.groupDataPerApp.forEach((perModMeta) => {
  // perModMeta.groupData: T[]  — payloads from that module
  perModMeta.groupData.forEach((payload) => {
    // process payload
  });
});
```

For the full `ExtensionGroupMeta` type shape, see [references/REFERENCE.md](references/REFERENCE.md).

---

## Dynamic Provider Registration

Extensions can dynamically push into provider arrays that are still being assembled.

| Stage    | Allowed provider levels               |
| -------- | ------------------------------------- |
| `stage1` | Any: app, module, route, request      |
| `stage2` | Only **below** module: route, request |

### Typical Pattern: Reading Config, Pushing Interceptors

```ts
async stage1(): Promise<void> {
  const meta = await this.extensionManager.stage1(RestRouteExtension);

  meta.groupData.forEach((routeExtensionMeta) => {
    const { providersPerMod } = routeExtensionMeta.normalizedModuleMeta;

    routeExtensionMeta.aControllerMetadata.forEach(({ providersPerReq }) => {
      // Build a temporary injector to resolve module-level config
      const injectorPerApp = Injector.resolveAndCreate(this.providersPerApp, 'App');
      const injectorPerMod  = injectorPerApp.resolveAndCreateChild(providersPerMod);
      const config = injectorPerMod.get(MyConfig, null);

      if (config?.enableFeature) {
        providersPerReq.push({
          token: HTTP_INTERCEPTORS,
          useClass: MyInterceptor,
          multi: true,
        });
      }
    });
  });
}
```

Always ensure your extension is ordered **after** the extension that produces the metadata (e.g., after `RestRouteExtension`) and **before** the extension that consumes it (e.g., before `DispatcherExtension`).

---

## Registration Decision Guide

| Goal                                    | Configuration                                           |
| --------------------------------------- | ------------------------------------------------------- |
| Run only in this module                 | `extensions: [MyExtension]`                             |
| Control execution order                 | Object form with `beforeExtensions` / `afterExtensions` |
| Run in this module AND importers        | `export: true`                                          |
| Run in importers only (not host)        | `exportOnly: true`                                      |
| Contribute to an existing data pipeline | `groups: [TargetExtension]`                             |
| Replace an imported extension           | `{ extension: MyExt, overrideExtension: OriginalExt }`  |

---

## Troubleshooting

| Symptom                                    | Fix                                                                                                          |
| ------------------------------------------ | ------------------------------------------------------------------------------------------------------------ |
| Extension does not run in importing module | Add `export: true` or `exportOnly: true` in host module                                                      |
| Extension runs too early                   | Add `afterExtensions: [ProducerExtension]`                                                                   |
| Extension runs too late                    | Add `beforeExtensions: [ConsumerExtension]`                                                                  |
| Group data is empty or missing             | Verify `groups: [TargetExtension]` uses the correct class token                                              |
| `groupDataPerApp` is incomplete            | Ensure `this` is passed as second arg to `extensionManager.stage1()` and `delay` is handled                  |
| Circular dependency error                  | Check the error log — `ExtensionManager` prints the full loop path; use `afterExtensions` to break the cycle |
| `UndeclaredExtensionDependency` error       | Extension A called `extensionManager.stage1(B)` but B was not listed in A's `afterExtensions`                |

---

## Further Reading

For full type definitions (`Extension<T>`, `ExtensionGroupMeta<T>`, `AppExtensionGroupMeta<T>`, `ExtensionConfig` union), see [references/REFERENCE.md](references/REFERENCE.md).
