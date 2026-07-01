---
name: ditsmod-extensions
description: Practical guidance for using, configuring, ordering, exporting, grouping, and overriding Ditsmod extensions in applications. Use when a Ditsmod app developer needs to import modules with extensions, register extensions, configure beforeExtensions/afterExtensions/groups/export/exportOnly/overrideExtension, integrate route metadata extensions, or troubleshoot extension execution order.
---

# Ditsmod Extensions

Use this skill when writing application code that consumes Ditsmod extensions. Keep the focus on module metadata configuration, integration of external modules, and practical extension ordering.

## Application Mental Model

Extensions run after Ditsmod has collected static metadata from decorators, module imports/exports, and providers. They usually run before request handlers are created.

Use extensions to make application infrastructure understand metadata. For example:

- route extensions interpret route decorators;
- OpenAPI extensions add route documentation metadata;
- body parser extensions add interceptors before handlers are created;
- custom extensions can initialize infrastructure or expose startup metadata.

Do not use an extension for per-request business logic. Put request-time behavior in controllers, services, guards, or interceptors.

## Registering An Extension

For a local extension that only needs to run in the current module, pass the class directly:

```ts
import { restModule } from '@ditsmod/rest';
import { AppMetadataExtension } from './app-metadata.extension.js';

@restModule({
  extensions: [AppMetadataExtension],
})
export class AppModule {}
```

Use object form when the extension needs ordering, grouping, export behavior, or override behavior:

```ts
import { restModule, RestRouteExtension } from '@ditsmod/rest';

@restModule({
  extensions: [
    {
      extension: AppRouteExtension,
      afterExtensions: [RestRouteExtension],
      export: true,
    },
  ],
})
export class FeatureModule {}
```

## Ordering

Use `afterExtensions` when your extension consumes metadata produced by another extension. Use `beforeExtensions` when another extension must consume your extension's output.

For route-related metadata, a common pattern is:

```ts
extensions: [
  {
    extension: AppRouteExtension,
    afterExtensions: [RestRouteExtension],
    beforeExtensions: [PreRouterExtension],
    exportOnly: true,
  },
]
```

This means:

- `RestRouteExtension` first produces route metadata.
- `AppRouteExtension` augments that metadata.
- `PreRouterExtension` later creates final routing infrastructure from the augmented metadata.

## Exporting Extensions

Use `export: true` when importing modules should also receive and run the extension.

Use `exportOnly: true` when the extension should be available to importing modules but should not run in the module where it is declared. This is common in reusable infrastructure modules.

Prefer direct local registration when the extension is purely app-local. Prefer exported registration when building a reusable feature module that should affect consumers.

## Extension Groups

Use `groups` when your extension contributes the same kind of metadata as another extension. The group token is an extension class.

```ts
extensions: [
  { extension: AppOpenapiRouteExtension, groups: [RestRouteExtension], export: true },
]
```

This lets consumers that request `RestRouteExtension` data receive data from both `RestRouteExtension` and `AppOpenapiRouteExtension`.

Only join a group when your extension returns data compatible with the group's base interface. Extending the interface is fine; narrowing or changing the meaning is not.

## Overriding External Extensions

Use `overrideExtension` when an imported module registers an extension and the app needs to replace it:

```ts
extensions: [
  { extension: AppRouteExtension, overrideExtension: ExternalRouteExtension },
]
```

Use override for replacement semantics. Use `groups` when both extensions should contribute compatible data. Use ordering only when your extension should run before or after another one without replacing it.

## Choosing A Configuration

Use this decision guide:

- Need the extension only in this module: `extensions: [MyExtension]`.
- Need ordering: object form with `beforeExtensions` or `afterExtensions`.
- Need importing modules to receive it: add `export: true`.
- Need to export without running in the host module: use `exportOnly: true`.
- Need to contribute compatible metadata to an existing workflow: use `groups`.
- Need to replace an imported extension: use `overrideExtension`.

## Troubleshooting

If an extension does not run where expected, check whether it is registered only in its host module and lacks `export` or `exportOnly`.

If an extension runs too early, add `afterExtensions` for the producer extension it depends on.

If an extension runs too late, add `beforeExtensions` for the target extension that consumes its output.

If grouped metadata is missing, verify that the extension uses the correct group token. Group tokens are extension classes, not arbitrary strings.

If replacing behavior does not work, use `overrideExtension` instead of only registering another extension in the same group.

If per-request behavior seems stale, move that behavior to an interceptor, guard, controller, service, request-scoped provider, or `Context`; extensions usually run before request handlers are created.
