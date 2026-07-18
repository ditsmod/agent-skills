---
name: ditsmod-reflector
description: 'Creating custom decorators using Reflector (makeClassDecorator, makePropDecorator, makeParamDecorator), metadata collection (collectMeta, getClassLevelMeta), reading decorator and parameter chains (decoratorChain, paramChain), testing requirements (reflect-metadata/lite, tsconfig flags), and programmatic metadata writing.'
---

# Ditsmod Reflector

Use this skill to create custom decorators, extract decorator metadata, and inspect metadata or dependency chains in the Ditsmod codebase.

## Mental Model

Standard JavaScript `Reflect` API (supplemented by `reflect-metadata`) allows storing and retrieving arbitrary metadata keys on classes, properties, or parameters. However, it operates on low-level string keys (e.g., `design:paramtypes`) and does not natively support class inheritance merging or decorator structures.

Ditsmod's `Reflector` provides a unified, cached, and high-level abstraction:

- **Unified Decorators:** It generates type-safe decorators that automatically register metadata in internal registries when evaluated.
- **Inheritance Traversal:** It merges class, property, and parameter metadata from parent classes down to children.
- **Iterability:** `collectMeta(Cls)` returns a `MergedClassMeta` object which is iterable and provides property-by-property details.
- **Cache-Optimized:** Reflection metadata is resolved once and cached internally.

---

## Prerequisites & Compilation

For reflection metadata to work, ensure these requirements are met:

1. **tsconfig.json Flags:**
   ```json
   {
     "compilerOptions": {
       "experimentalDecorators": true,
       "emitDecoratorMetadata": true
     }
   }
   ```
2. **Runtime Polyfill:**
   `reflect-metadata/lite` must be imported before any decorators are evaluated. In typical Ditsmod applications, this is already imported automatically by `@ditsmod/core`, so you do **not** need to add it manually to the application entry points.
3. **Test Setup:**
   In test configurations (like Jest or Vitest), if `@ditsmod/core` is not imported early enough (or at all), ensure `reflect-metadata/lite` is loaded via `setupFilesAfterEnv` or an explicit top-level import.

---

## Creating Custom Decorators

Use the static factory methods of `Reflector` to create decorators.

### 1. Class Decorator

```ts
import { Reflector } from '@ditsmod/core/di';

// Without transformer (returns arguments as an array)
const classLevelA = Reflector.makeClassDecorator();

// With transformer (transforms arguments into a structured object)
interface MyConfig {
  name: string;
  debug?: boolean;
}
const classLevelB = Reflector.makeClassDecorator((config: MyConfig) => config);

@classLevelB({ name: 'users-service', debug: true })
class UsersService {}
```

#### Class Decorator Arguments (Reflector.makeClassDecorator)

`Reflector.makeClassDecorator()` accepts up to three arguments:

1. **`transformer`**: (Optional) A function that transforms the decorator arguments into a structured object (e.g., `(config) => config`).
2. **`name`**: (Optional) A string containing the name of the decorator.
3. **`decoratorId`**: (Optional) An identifier (typically another decorator factory function) to group related class decorators together. This is a general feature of `Reflector.makeClassDecorator()` that allows different class decorators to share a common ID, facilitating their collection and inspection under the same group. For example, in init-decorators, the substitute decorators (like `restModule`) pass the base modifier decorator (like `initRest`) as the `decoratorId` so Ditsmod knows they belong to the same group.

_Note: Class decorator factories capture the directory where they are executed, which is used by Ditsmod's module discovery to resolve relative paths._

### 2. Property / Method Decorator

```ts
import { Reflector } from '@ditsmod/core/di';

const propertyLevel = Reflector.makePropDecorator((role: string) => ({ role }));

class Controller {
  @propertyLevel('admin')
  secureData() {}
}
```

### 3. Parameter Decorator

```ts
import { Reflector } from '@ditsmod/core/di';

const paramLevel = Reflector.makeParamDecorator((token: any) => ({ token }));

class Handler {
  constructor(@paramLevel('CONFIG_TOKEN') config: any) {}
}
```

---

## Collecting Metadata

Retrieve decorator metadata using `Reflector.collectMeta()` or `Reflector.getClassLevelMeta()`.

### Reading Class-Level Decorators

To get decorators directly attached to a class:

```ts
import { Reflector } from '@ditsmod/core/di';
import { controller } from '@ditsmod/rest';

// Returns all decorators on the class
const decorators = Reflector.getClassLevelMeta(MyController);

// Filter by a type guard
const controllerDecorators = Reflector.getClassLevelMeta(MyController, (decor) => decor.decorator === controller);
if (controllerDecorators) {
  const metadataValue = controllerDecorators[0].value; // contains the decorator options
}
```

### Reading Full Class Metadata Iterator

`Reflector.collectMeta(Cls)` returns a `MergedClassMeta` object which can be iterated to get decorated properties.

```ts
import { Reflector } from '@ditsmod/core/di';

const metadata = Reflector.collectMeta(MyService);
if (metadata) {
  // Read constructor metadata
  const ctorMeta = metadata.constructor;
  console.log(ctorMeta.decorators); // Array of DecoratorMeta on the constructor
  console.log(ctorMeta.params); // Array of constructor parameters and their decorators

  // Iterate over decorated property/method keys
  for (const propName of metadata) {
    const propMeta = metadata[propName];
    console.log(`Property ${String(propName)} has type:`, propMeta.type);
    console.log(`Decorators:`, propMeta.decorators);
  }
}
```

---

## Inheritance and Chains

`Reflector` resolves metadata throughout the entire class inheritance hierarchy. If a child class extends a parent class and overrides properties or constructor parameters, `Reflector` tracks this in the `decoratorChain` and `paramChain` fields of `MergedClassPropMeta`.

```ts
import { Reflector } from '@ditsmod/core/di';

const customDecor = Reflector.makeClassDecorator((val?: string) => val);

@customDecor('parent-meta')
class Parent {
  constructor(a: string) {}
}

@customDecor('child-meta')
class Child extends Parent {
  constructor(a: string, b: number) {
    super(a);
  }
}

const childMeta = Reflector.collectMeta(Child);
if (childMeta) {
  const ctor = childMeta.constructor;

  // Flattened decorators/params for the child
  console.log(ctor.decorators); // Contains child's decorators

  // Inheritance chains
  console.log(ctor.decoratorChain); // Map<Class, DecoratorMeta[]> containing Parent & Child decorators
  console.log(ctor.paramChain); // Map<Class, ParameterMeta[]> containing Parent & Child parameter metadata
}
```

## Programmatic Metadata Writing (Experimental & Non-Public)

Ditsmod provides experimental helper methods to attach metadata programmatically without using decorators directly (useful for code generators, dynamic routing setups, or testing environments).

> [!IMPORTANT]
> These methods (`setClassMeta`, `setPropertyMeta`, `setParameterMeta`) are **`protected static`** in the `Reflector` class. They are not part of the public API.
> To use them, you must subclass `Reflector` to expose them publicly or access them via a type assertion/cast:

```ts
import { Reflector } from '@ditsmod/core/di';
import { controller, route } from '@ditsmod/rest';

// Exposing protected methods via subclassing:
class CustomReflector extends Reflector {
  static override setClassMeta(...args: Parameters<typeof Reflector.setClassMeta>) {
    return super.setClassMeta(...args);
  }
  static override setPropertyMeta(...args: Parameters<typeof Reflector.setPropertyMeta>) {
    return super.setPropertyMeta(...args);
  }
  static override setParameterMeta(...args: any[]) {
    return super.setParameterMeta(...args);
  }
}

class CustomController {
  tellHello() {
    return 'Hello!';
  }
}

// 1. Programmatically write class-level decorator
CustomReflector.setClassMeta(CustomController, controller, { routePrefix: 'custom' });

// 2. Programmatically write method-level decorator and compile design:type
CustomReflector.setPropertyMeta(
  CustomController,
  'tellHello',
  Function, // property type
  route,
  'GET',
  'hello', // decorator arguments
);
```

---

## Troubleshooting Checklist

1. **`Reflect.defineMetadata is not a function` error:**
   - Ensure `@ditsmod/core` is imported (which automatically loads `reflect-metadata/lite` at startup), or explicitly import `reflect-metadata/lite` in entry points/test configurations that do not load `@ditsmod/core` early enough (e.g. `setupFilesAfterEnv` in Jest).
2. **Missing metadata for constructor arguments or properties:**
   - Verify that the class is decorated (e.g. with `@injectable()`, `@controller()`, etc.). If a class or property has no decorators at all, the TypeScript compiler will not emit any design metadata, even with `emitDecoratorMetadata` enabled.
   - Verify that `"emitDecoratorMetadata": true` and `"experimentalDecorators": true` are enabled in `tsconfig.json`.
3. **TypeScript compiler loses types after compiling imports:**
   - Check if parameter types in the constructor are imported using `import type` or interfaces. TypeScript does not emit runtime metadata for type-only constructs. Use classes, or use the `@inject(MY_TOKEN)` decorator on the parameter alongside the corresponding TypeScript interface/type.
4. **Decorator decoratorId type checks fail:**
   - When creating complex decorator factories, ensure you provide a valid `decoratorId` function/symbol and match it with a corresponding `TypeGuard`.

---

## Further Reading

For full type signatures of `MergedClassMeta`, `MergedClassPropMeta`, `DecoratorMeta`, and helper classes, see [references/REFERENCE.md](references/REFERENCE.md).
