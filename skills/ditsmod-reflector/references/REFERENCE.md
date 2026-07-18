# Ditsmod Reflector — Technical Reference

This guide contains detailed API and type definitions for Ditsmod Reflector internals. Use it when:

- Designing complex custom decorator validation pipelines.
- Accessing deep properties of `MergedClassMeta` or `MergedClassPropMeta` in extensions.
- Integrating programmatic metadata definitions with custom testing fixtures.

---

## Technical Types Reference

These are core interfaces and classes used by `Reflector` (primarily imported from `@ditsmod/core/di` or `@ditsmod/core`):

### 1. `DecoratorMeta<Value>`

Stores the identifier of a decorator and the transformed value it returned:

```ts
class DecoratorMeta<Value = any> {
  constructor(
    public decorator: AnyFn, // The decorator factory function
    public value: Value, // The output returned by the decorator transformer
    public decoratorId?: AnyFn, // Optional group/type identifier for typeguards
    public declaredInDir?: string, // Directory in which the decorator was evaluated (class-level only)
  ) {}
}
```

### 2. `MergedClassMeta<DecorValue, Proto>`

The return type of `Reflector.collectMeta(Cls)`. It maps every decorated property of the prototype to its metadata and provides constructor metadata:

```ts
type MergedClassMeta<DecorValue = any, Proto extends object = object> = {
  [P in keyof Proto]: MergedClassPropMeta<DecorValue>;
} & {
  constructor: MergedClassPropMeta<DecorValue>;
} & {
  [Symbol.iterator]: () => Generator<string | symbol>; // yields decorated property names
};
```

### 3. `MergedClassPropMeta<DecorValue>`

Contains metadata about a constructor, property, or method. Supports inheritance hierarchies by storing metadata chains:

```ts
class MergedClassPropMeta<DecorValue = any> extends ClassPropMeta<DecorValue> {
  constructor(
    type: Class, // Property type or class constructor type
    decorators: DecoratorMeta<DecorValue>[], // Flattened array of decorators (merged from chain)
    params: (ParameterMeta | null)[], // Constructor/Method parameters metadata (merged)
    public decoratorChain: Map<Class, DecoratorMeta<DecorValue>[]>, // Key: Class, Value: class-specific decorators
    public paramChain: Map<Class, (ParameterMeta | null)[]>, // Key: Class, Value: class-specific parameters
  ) {}
}
```

### 4. Parameter Metadata Types

```ts
type ParameterItem<Value = any> = DecoratorMeta<Value> | InjectionToken<any> | Class;
type ParameterMeta<Value = any> = [Class, ...ParameterItem<Value>[]] | [...ParameterItem<Value>[]] | [];
```

---

## Detailed Example: Parsing `collectMeta` Output

When `Reflector.collectMeta(Child)` is executed, the following data structure is generated:

```ts
import { Reflector } from '@ditsmod/core/di';

class ServiceA {}
class ServiceB {}

class Parent {
  constructor(a: ServiceA) {}
}

class Child extends Parent {
  constructor(a: ServiceA, b: ServiceB) {
    super(a);
  }
}

const metadata = Reflector.collectMeta(Child);
/*
metadata structure:
{
  constructor: MergedClassPropMeta {
    type: Child,
    decorators: [],
    params: [
      [ServiceA], // Parameter 0: Type is ServiceA, no decorators
      [ServiceB]  // Parameter 1: Type is ServiceB, no decorators
    ],
    decoratorChain: Map {
      [class Parent] => [],
      [class Child] => []
    },
    paramChain: Map {
      [class Parent] => [ [ServiceA] ],
      [class Child] => [ [ServiceA], [ServiceB] ]
    }
  }
}
*/
```

---

## Custom Decorator with Complex Signatures

When writing a decorator that accepts multiple argument lists, use an interface to define its call signatures and let `Reflector.make*Decorator` transform the arguments:

```ts
import { Reflector } from '@ditsmod/core/di';

interface RouteOptions {
  method: string;
  path: string;
}

interface RouteDecorator {
  (path: string): MethodDecorator;
  (method: string, path: string): MethodDecorator;
}

const route: RouteDecorator = Reflector.makePropDecorator((arg1: string, arg2?: string) => {
  if (arg2) {
    return { method: arg1, path: arg2 } satisfies RouteOptions;
  }
  return { method: 'GET', path: arg1 } satisfies RouteOptions;
}, 'route');
```

## Class Decorator with decoratorId (Grouping Decorators)

When creating multiple class decorators that belong to a single logical group, you can pass a reference to the base decorator as the third argument (`decoratorId`). This makes it easy to filter/inspect them collectively at runtime:

```ts
import { DecoratorMeta, Reflector } from '@ditsmod/core/di';

// 1. Create a base decorator (acts as the Group ID)
export const base = Reflector.makeClassDecorator((data?: any) => data, 'base');

// 2. Create related decorators, passing `base` as the third argument (decoratorId)
export const decorator1 = Reflector.makeClassDecorator((data?: any) => data, 'decorator1', base);
export const decorator2 = Reflector.makeClassDecorator((data?: any) => data, 'decorator2', base);

// 3. Apply the decorators
@decorator1({ one: 1 })
class SomeModule {}

// 4. Retrieve and inspect metadata by grouping ID
const decorators = Reflector.getClassLevelMeta(
  SomeModule,
  (decor): decor is DecoratorMeta<{ one: number }> => decor.decoratorId === base,
);
if (decorators && decorators.length > 0) {
  console.log(decorators[0].value); // { one: 1 }
}
```
