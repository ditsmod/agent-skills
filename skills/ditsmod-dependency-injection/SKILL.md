---
name: ditsmod-dependency-injection
description: Practical guidance for using dependency injection in Ditsmod applications. Use when building or fixing application services, controllers, guards, interceptors, modules, provider configuration, typed InjectionToken values, provider overrides, multi-providers, request-scoped data, Context, ctxProviders, and parameter decorators in apps built with @ditsmod/rest or other Ditsmod application packages.
---

# Ditsmod Dependency Injection

Use this skill when writing application code that consumes Ditsmod DI. Keep the focus on app ergonomics: clear provider placement, typed tokens, predictable overrides, and request-safe data flow.

## Application Mental Model

Ditsmod app provider arrays map to injector levels:

```txt
providersPerApp -> providersPerMod -> providersPerRou -> providersPerReq
```

Place values by lifetime and override needs:

- Use `providersPerApp` for app-wide singletons and default configuration.
- Use `providersPerMod` for module-specific services or module-level configuration overrides.
- Use `providersPerRou` for route-level behavior.
- Use `providersPerReq` for request-scoped services and values that must not be shared across HTTP requests.

If a service depends on a config or helper, register that dependency in the same provider array or in an ancestor array. Do not put a dependency only in a child scope.

## Tokens And Typed Config

Use a class as a token only when the dependency is actually that class. For interfaces, arrays, plain objects, and configuration values, create an `InjectionToken<T>`.

```ts
// tokens.ts
import { InjectionToken } from '@ditsmod/core';

export interface AuthConfig {
  issuer: string;
  audience: string;
}

export const AUTH_CONFIG = new InjectionToken<AuthConfig>('AUTH_CONFIG');
```

Inject non-class values with `@inject()`:

```ts
import { inject, injectable } from '@ditsmod/core';
import { AUTH_CONFIG, type AuthConfig } from './tokens.js';

@injectable()
export class AuthService {
  constructor(@inject(AUTH_CONFIG) private config: AuthConfig) {}
}
```

Do not use TypeScript-only entities as runtime tokens: `interface`, `type`, `declare`, or imports written with `import type`.

## Provider Recipes

Use `useValue` for static config:

```ts
import { type Provider } from '@ditsmod/core';
import { AUTH_CONFIG } from './tokens.js';

export const authProviders: Provider[] = [
  { token: AUTH_CONFIG, useValue: { issuer: 'https://auth.example.com', audience: 'api' } },
];
```

Use `useClass` for swapping implementations:

```ts
export const authProviders = [
  { token: AuthService, useClass: JwtAuthService },
];
```

Use `useToken` for aliases:

```ts
export const authProviders = [
  JwtAuthService,
  { token: AuthService, useToken: JwtAuthService },
];
```

Use a class factory when the factory has dependencies or decorated parameters:

```ts
import { factoryMethod, inject } from '@ditsmod/core';
import { AUTH_CONFIG, type AuthConfig } from './tokens.js';

class AuthClientFactory {
  @factoryMethod()
  create(@inject(AUTH_CONFIG) config: AuthConfig) {
    return new AuthClient(config.issuer, config.audience);
  }
}

export const authProviders = [
  { token: AuthClient, useFactory: [AuthClientFactory, AuthClientFactory.prototype.create] },
];
```

Use a function factory only when plain token dependencies are enough; `deps` accepts tokens, not providers.

## Provider Placement

When a user asks where to put a provider, choose the highest scope that is still correct:

- Put shared defaults in `providersPerApp`.
- Put a feature module's private services in `providersPerMod`.
- Put route behavior and route-specific overrides in `providersPerRou`.
- Put request-owned mutable state in `providersPerReq` or in `Context`, not in app/module singletons.

If a child scope overrides a config but the service using that config is registered in a parent scope, the service will still use the parent-visible config. Register the service in the child scope too when it must consume the child override.

## Overrides

For feature-specific configuration, override the token at the feature scope and make sure consumers are created at that same scope or lower:

```ts
export const defaultProviders = [
  { token: AUTH_CONFIG, useValue: { issuer: 'https://auth.example.com', audience: 'api' } },
];

export const adminRouteProviders = [
  AuthService,
  { token: AUTH_CONFIG, useValue: { issuer: 'https://auth.example.com', audience: 'admin-api' } },
];
```

If the same regular token appears more than once in one provider array, the last provider wins. Use this deliberately for app-level replacement, but avoid accidental duplicates in large arrays.

## Multi-Providers

Use multi-providers for extensible ordered lists such as interceptors or plugin-style handlers:

```ts
import { InjectionToken } from '@ditsmod/core';

export interface AuditHook {
  run(event: string): void;
}

export const AUDIT_HOOKS = new InjectionToken<AuditHook[]>('AUDIT_HOOKS');

export const auditProviders = [
  { token: AUDIT_HOOKS, useClass: ConsoleAuditHook, multi: true },
  { token: AUDIT_HOOKS, useClass: MetricsAuditHook, multi: true },
];
```

Do not mix regular and multi-providers for the same token in one injector. If a child injector defines multi-providers for the same token, it returns its own array instead of merging parent values.

For substitutable defaults from external modules, prefer a multi-provider entry that points to a class via `useToken`, then override that class:

```ts
export const providers = [
  { token: AUDIT_HOOKS, useToken: DefaultAuditHook, multi: true },
  DefaultAuditHook,
  { token: DefaultAuditHook, useClass: AppAuditHook },
];
```

## Request Data And Context

Use `Context` for values produced after injector creation, especially data produced by interceptors or guards and consumed by controllers or services later in the same request.

Use `getSymbol<T>()` for typed context keys:

```ts
import { Context, getSymbol } from '@ditsmod/core';

export interface CurrentUser {
  id: string;
  roles: string[];
}

export const CURRENT_USER = getSymbol<CurrentUser>('CURRENT_USER');

export class CurrentUserWriter {
  constructor(private ctx: Context) {}

  setUser(user: CurrentUser) {
    this.ctx.set(CURRENT_USER, user);
  }
}
```

Use `@ctx(KEY)` in method parameters when a value should be read from `Context`. Ensure `ctxProviders` are available unless the REST module already provides them through `CtxModule`.

## Parameter Decorators

- Use `@inject(token)` for non-class tokens and implementation aliases.
- Use `@optional()` when a missing provider should produce `undefined` instead of an error.
- Use `@inject(token, inputData)` with `@input` for contextual construction data; the dependency created with input data is not cached.
- Use `@fromSelf()` only when a dependency must come from the current provider scope.
- Use `@skipSelf()` only when a dependency must come from an ancestor scope.

## App Debugging Checklist

1. Confirm the failing token exists at runtime; replace interfaces/types with `InjectionToken<T>`.
2. Confirm the consumer service/controller is registered at a scope that can see all its dependencies.
3. If an override is ignored, check whether the consuming service is created in a parent injector.
4. If request data leaks across requests, move mutable data out of app/module singletons and into request scope or `Context`.
5. If a multi-provider result is incomplete, check whether a child scope replaced the parent multi-provider array.
6. If a function factory fails, verify every item in `deps` has its own provider.
