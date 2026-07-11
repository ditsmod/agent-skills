# Ditsmod Extensions — Technical Reference

## `Extension<T>` Interface

Source: `@ditsmod/core` — `extension/extension-types.ts`

```ts
interface Extension<T = any> {
  /**
   * Called at the stage when providers are dynamically added.
   * isLastModule === true when this is the last module that imports this extension.
   * The return value T becomes the payload in ExtensionGroupMeta.groupData[].
   */
  stage1?(isLastModule: boolean): Promise<T>;

  /**
   * Called after stage1() has executed for ALL modules.
   * Receives a module-level injector that already has all dynamic providers.
   */
  stage2?(injectorPerMod: Injector): Promise<void>;

  /**
   * Called after stage2() has executed for ALL modules.
   * No strict role — use for final aggregation or side effects.
   */
  stage3?(): Promise<void>;
}

type ExtensionClass<T = any> = Class<Extension<T>>;
```

All three methods are optional. The class must be decorated with `@injectable()`.

---

## `ExtensionGroupMeta<T>` Shape

Returned by `extensionManager.stage1(ExtCls)` (same-module call):

```ts
class ExtensionGroupMeta<T = any> {
  moduleName: string; // name of the module in which this meta was collected
  delay: boolean; // true when cross-module data is not yet complete
  countdown: number; // number of remaining modules that must run (0 when done)
  groupDebugMeta: ExtensionDebugMeta<T>[]; // debug info per extension instance
  groupData: T[]; // payloads returned by each extension.stage1() in this group/class
  groupDataPerApp: AppExtensionGroupMeta<T>[]; // populated for cross-module calls only
}
```

`groupData` contains the values returned by `stage1()` from every extension that belongs to the requested class or group. If an extension returned `undefined`/`void`, that slot is still present.

---

## `AppExtensionGroupMeta<T>` Shape

Each element of `groupDataPerApp` in a cross-module call. It is `ExtensionGroupMeta<T>` without the `groupDataPerApp` field itself:

```ts
type AppExtensionGroupMeta<T = any> = Omit<ExtensionGroupMeta<T>, 'groupDataPerApp'>;
// fields: moduleName, delay, countdown, groupDebugMeta, groupData
```

`groupDataPerApp` is an array with one entry per module that registered the target extension. Iterate it to aggregate data across all modules.

---

## `ExtensionDebugMeta<T>` Shape

Debug metadata for a single extension instance within a group/class call:

```ts
class ExtensionDebugMeta<T = any> {
  constructor(
    public extension: Extension<T>, // the instance
    public payload: T, // value returned by extension.stage1()
    public delay: boolean, // same meaning as ExtensionGroupMeta.delay
    public countdown: number, // remaining module count
  ) {}
}
```

---

## `ExtensionConfig` Union

Source: `@ditsmod/core` — `extension/extension-providers-and-configs.ts`

`ExtensionConfig` is a discriminated union of three shapes:

```ts
type ExtensionConfig = ExtensionConfig1 | ExtensionConfig2 | ExtensionConfig3;
```

### `ExtensionConfig1` — export in host and importers

```ts
interface ExtensionConfig1 {
  extension: ExtensionClass;
  beforeExtensions?: ExtensionClass[]; // run this before each listed class
  afterExtensions?: ExtensionClass[]; // run this after each listed class
  groups?: ExtensionClass[]; // join these group tokens
  export?: boolean; // run in host AND every importing module
  exportOnly?: never; // mutually exclusive with export
  overrideExtension?: never;
}
```

### `ExtensionConfig2` — export to importers only

```ts
interface ExtensionConfig2 {
  extension: ExtensionClass;
  beforeExtensions?: ExtensionClass[];
  afterExtensions?: ExtensionClass[];
  groups?: ExtensionClass[];
  export?: never;
  exportOnly?: boolean; // run ONLY in importing modules, not in host
  overrideExtension?: never;
}
```

### `ExtensionConfig3` — override

```ts
interface ExtensionConfig3 {
  extension: ExtensionClass; // replacement class
  overrideExtension: ExtensionClass; // class token to replace
  // NOTE: beforeExtensions, afterExtensions, groups, export, exportOnly are NOT available here
}
```

Internally, override is implemented as `{ token: overrideExtension, useClass: extension }`, so all consumers of the original token transparently receive the replacement.

---

## `ExtensionsMetaPerApp` Shape

A free-form map that extensions can use to store app-level aggregated results:

```ts
class ExtensionsMetaPerApp {
  [key: string]: AnyObj;
}
```

This class is available as a DI token. Extensions that need to publish app-wide results can inject it and write to a keyed slot.

---

## `ExtensionCounters`

Tracks how many modules still need to execute each extension/group:

```ts
class ExtensionCounters {
  mExtensions: Map<Provider, number>;
}
```

Used internally by `ExtensionManager`. `countdown === 0` means the extension has been called in all modules — `isLastModule` will be `true` on the next invocation.

---

## `ExtensionManager.stage1()` Overloads

```ts
// Same-module call — always returns full ExtensionGroupMeta<T> (delay always false)
stage1<T>(ExtCls: ExtensionClass<T>): Promise<ExtensionGroupMeta<T>>;

// Cross-module call — may return PartialExtensionGroupMeta<T> with delay: true
// PartialExtensionGroupMeta = OptionalProps<ExtensionGroupMeta<T>, 'groupDebugMeta' | 'groupData' | 'moduleName' | 'countdown'>
stage1<T>(ExtCls: ExtensionClass<T>, pendingExtension: Extension): Promise<PartialExtensionGroupMeta<T>>;
```

When `delay` is `false` on the cross-module call, the returned object has `groupDataPerApp` populated but `groupData`, `groupDebugMeta`, `moduleName`, and `countdown` are omitted. Always check `delay` before accessing `groupDataPerApp`.

---

## Error Types

| Error class                       | When thrown                                                                                  |
| --------------------------------- | -------------------------------------------------------------------------------------------- |
| `UndeclaredExtensionDependency`   | Extension A called `extensionManager.stage1(B)` but B is not listed in A's `afterExtensions` |
| `CyclicExtensions`                | A cycle was detected in the extension dependency graph                                       |
| `ExtensionExecutionFailure`       | Any unhandled error thrown inside an extension's stage method                                |

The full dependency chain is included in the error message.

---

## Stage Sequencing Summary

```
For each module (in topological order):
  stage1() called for all extensions in that module

After stage1() completes for ALL modules:
  stage2(injectorPerMod) called for all extensions (same order as stage1)

After stage2() completes for ALL modules:
  stage3() called for all extensions (same order)
```

The order of extensions within a module at stage2 and stage3 is the same as at stage1.
