# Ditsmod DI: Reference Guide

This file contains detailed reference material loaded on demand. Use it when:

- Diagnosing complex `Resolution path` errors
- Needing the complete provider type definitions
- Looking up injector hierarchy simulation code

## Reading `Resolution path` Errors

When DI cannot resolve a token, it throws an error with a `Resolution path`. Understanding this path is critical for diagnosing hierarchy misconfigurations.

### Format

```
Error: No provider for [MissingToken in injectorName]!
Resolution path: [TokenA in injectorX >> injectorY] -> [MissingToken in injectorY]
```

- **`TokenA in injectorX >> injectorY`** — DI searched for `TokenA` starting in `injectorX`, did not find it there, and continued upward through `>>` until it found it in `injectorY`.
- **`->`** — separates the found provider from the dependency it requires. The dependency search starts from the injector where the provider was found (not from where the request originated).
- **`MissingToken in injectorY`** — the dependency `MissingToken` was searched starting from `injectorY` (upward) and was not found anywhere.

### Injector Naming

Ditsmod auto-names injectors as `injector1` (highest/app level), `injector2`, `injector3`, etc. (lower = higher number). Pass an explicit second argument to `Injector.resolveAndCreate(providers, 'MyName')` for clearer diagnostics. In a full Ditsmod application the levels are named `App`, `Mod`, `Rou`, `Req`.

### Diagnostic Examples

**Example 1 — dependency placed in child injector:**

```ts
// Setup
const parent = Injector.resolveAndCreate([Service], 'parentInjector');
const child  = parent.resolveAndCreateChild([{ token: Config, useValue: {...} }], 'childInjector');

child.get(Service);
// Error: No provider for [Config in parentInjector]!
// Resolution path: [Service in childInjector >> parentInjector] -> [Config in parentInjector]
```

Reading this: `Service` was not found in `childInjector`, was found in `parentInjector`. From `parentInjector`, DI searched for `Config` — also in `parentInjector` and its ancestors — but found nothing. Fix: move `Config` to `parentInjector` or add `Service` to `childInjector` (so it uses the child's `Config`).

**Example 2 — Ditsmod application level names:**

```
Error: No provider for [Config in Mod >> App]!
Resolution path: [Service in Req >> Rou >> Mod] -> [Config in Mod >> App]
```

Reading this: `Service` was searched in `Req`, `Rou`, and finally found in `Mod`. From `Mod`, DI searched for `Config` through `Mod` and `App` but found nothing. Fix: register `Config` in `providersPerMod` or `providersPerApp`.

---

## Complete Provider Type Reference

```ts
import { Class } from '@ditsmod/core';

type Provider =
  | Class<any> // TypeProvider (shorthand)
  | { token: any; useValue?: any; multi?: boolean } // ValueProvider
  | { token: any; useClass: Class<any>; multi?: boolean } // ClassProvider
  | { token?: any; useFactory: [Class<any>, Function]; multi?: boolean } // ClassFactoryProvider
  | { token?: any; useFactory: (...args: any[]) => any; deps: any[]; multi?: boolean } // FunctionFactoryProvider
  | { token: any; useToken: any; multi?: boolean }; // TokenProvider
```

Named provider types importable from `@ditsmod/core`:

```ts
import { TypeProvider, ValueProvider, ClassProvider, FactoryProvider, TokenProvider } from '@ditsmod/core';
```

### `useFactory` token is optional

For factory providers, `token` is optional because DI can use the factory function or the class method itself as a token.

---

## Injector Hierarchy Simulation

Full four-level hierarchy simulation matching a real Ditsmod application:

```ts
import { Injector, Provider } from '@ditsmod/core';

const providersPerApp: Provider[] = [];
const providersPerMod: Provider[] = [];
const providersPerRou: Provider[] = [];
const providersPerReq: Provider[] = [];

const injectorPerApp = Injector.resolveAndCreate(providersPerApp, 'App');
const injectorPerMod = injectorPerApp.resolveAndCreateChild(providersPerMod, 'Mod');
const injectorPerRou = injectorPerMod.resolveAndCreateChild(providersPerRou, 'Rou');
const injectorPerReq = injectorPerRou.resolveAndCreateChild(providersPerReq, 'Req');

injectorPerApp === injectorPerMod.parent; // true
```

Ditsmod repeats this procedure for each module, route, and HTTP request.

---

## `Context.getInScope()` vs `Context.get()`

Both methods retrieve values by key, but they differ in traversal scope:

- **`ctx.get(key)`** — retrieves the value from the current `Context` instance only.
- **`ctx.getInScope(key, injector)`** — traverses **up** the injector hierarchy, checking each `Context` instance at each level, returning the first match found. Useful when `Context` instances exist at multiple injector levels (e.g., parent module sets a value, child module reads it).

```ts
const parent = Injector.resolveAndCreate([Context], 'parent level');
const child = parent.resolveAndCreateChild([Context], 'child level');

parent.get(Context).set('key1', 'value1');
child.get(Context).set('key2', 'value2');

child.get(Context).getInScope('key1', child); // 'value1'  (found in parent Context)
child.get(Context).getInScope('key2', child); // 'value2'  (found in child Context)
```

---

## `@inject` + `@input` Full Example

```ts
import { injectable, inject, input, Injector } from '@ditsmod/core';

@injectable()
class Dependency1 {
  constructor(@input public inputParameter: string) {}
}

@injectable()
class Service1 {
  constructor(@inject(Dependency1, 'input-data') public dependency1: Dependency1) {}
}

const injector = Injector.resolveAndCreate([Service1, Dependency1]);
const service1 = injector.get(Service1);
service1.dependency1.inputParameter; // 'input-data'
```

Key rules:

- `@input` (without parentheses) is shorthand for `@inject(input)`.
- When a second argument is passed to `@inject()`, **no cache** is created for that dependency — a new instance is created each time.
- `input` itself can be used as a token in `deps` arrays of function factories.
