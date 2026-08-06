---
id: globals
title: Global Properties
---

## Available Global Properties

| Property                                          | Description                                                                                                                   | Acceptable value                                     |
|---------------------------------------------------|-------------------------------------------------------------------------------------------------------------------------------|------------------------------------------------------|
| debugOpenAPI                                      | Dumps JSON formatted and fully parsed OpenAPI document during generation                                                      | none                                                 |
| debugModels                                       | Dumps JSON formatted template-bound model information during generation                                                       | none                                                 |
| debugOperations                                   | Dumps JSON formatted template-bound operation information during generation                                                   | none                                                 |
| debugSupportingFiles                              | Dumps JSON formatted Supporting File information during generation                                                            | none                                                 |
| verbose                                           | Defines the verbosity                                                                                                         | `true` or `false`                                    |
| generateAliasAsModel                              | Defines whether primitive types defined at the model/schema level will be wrapped in a model                                  | `true` or `false`                                    |
| org.openapitools.codegen.utils.oncelogger.enabled | Enable/disable the "OnceLogger" which reduces noise for select repeated logs                                                  | `true` or `false`                                    |
| supportingFiles                                   | Allows the user to define which supporting files will be generated. Prefer using the more robust `.openapi-generator-ignore`. | no value, or a colon-separated string of file names  |
| models                                            | Allows the user to define which models will be generated. Prefer using the more robust `.openapi-generator-ignore`.           | no value, or a colon-separated string of model names |
| apis                                              | Allows the user to define which apis will be generated. Prefer using the more robust `.openapi-generator-ignore`.             | no value, or a colon-separated string of api names   |
| apiDocs                                           | Allows the user to define if api docs will be generated. Prefer using the more robust `.openapi-generator-ignore`.            | `true` or `false`                                    |
| modelDocs                                         | Allows the user to define if model docs will be generated. Prefer using the more robust `.openapi-generator-ignore`.          | `true` or `false`                                    |
| apiTests                                          | Allows the user to define if api tests will be generated. Prefer using the more robust `.openapi-generator-ignore`.           | `true` or `false`                                    |
| modelTests                                        | Allows the user to define if model tests will be generated. Prefer using the more robust `.openapi-generator-ignore`.         | `true` or `false`                                    |
| splitOperationsByContentType                      | Generates one operation per request/response content-type when an operation exposes several with different schemas            | `true` or `false`                                    |


## Note on splitOperationsByContentType

An operation may declare several request or response content-types backed by *different* schemas. Only the
first one is normally kept, which leaves the others unreachable. With `splitOperationsByContentType=true`
such an operation is generated once per content-type instead — the cartesian product of the request and
response axes, deduplicated by schema — each with a typed, collision-free operation id built from the base
one: `With<Subtype>` for the request axis, `As<Subtype>` for the response axis, as in
`createReportWithXmlAsPdf`.

The content-type declared first on each axis is the default one, consistently with the rest of the
generator. The option is opt-in and off by default, because it changes the shape of the generated API.

Each generated operation carries `x-content-type-variant-*` extensions recording the group it was split
from, the content-type it was narrowed to on each axis and the rank of that content-type in its axis. A
generator whose language can express the whole matrix in a single construct uses them to merge the variants
back together while keeping each one's natively resolved types. `typescript-fetch` does exactly that: it
emits one method whose request type is a union discriminated by `contentType` and whose return type is
selected by overloads on `accept`.

```ts
export type CreateReportRequest =
    | { contentType?: 'application/json'; report?: Report; reportXml?: never; }
    | { contentType: 'application/xml'; reportXml?: ReportXml; report?: never; }
;

async createReport(requestParameters: CreateReportRequest & { accept?: 'application/json' }, initOverrides?: RequestInit | runtime.InitOverrideFunction): Promise<Receipt>;
async createReport(requestParameters: CreateReportRequest & { accept: 'application/pdf' }, initOverrides?: RequestInit | runtime.InitOverrideFunction): Promise<Blob>;
```

Each member declares the other members' bodies as `never`. Without it nothing stops a caller from handing an
XML body to the JSON member and having it silently sent as JSON: excess property checking, which would
normally reject the surplus property, treats a key present in *any* member of a union as known, so it never
fires here — for an object literal no more than for a variable. What rejects most shapes is unrelated: weak
type detection when every property of a member is optional, a missing required property otherwise. A member
with a required parameter and an optional body has neither.

Two cases are left split rather than merged, with a warning. An operation accepting several request
content-types one of which is a form or multipart body, because such a body is spread over individual
parameters rather than gathered in one, and so cannot be folded into a discriminated union — merging the
response axis of an operation with a form body is unaffected, since it only adds `accept` to the request
object. And every operation when `useSingleRequestParameter` is off, since the parameters are then spread
over the signature and there is no request object to carry the discriminant.

In both cases the separate, individually typed methods the split produced are generated as they are, which
is what a statically-typed generator emits anyway.


## Note on Global Property declaration

There are _two ways_ to provide selective generation properties or "global properties". First, these can be passed as Java System Properties. Second, these can be passed via the global property tooling option (`--global-property` in CLI and `globalProperty` in Maven and Gradle configurations). This differentiation is new in version 5.0 with the removal of the `-D` CLI option and the renaming of `systemProperties`. If you're upgrading to OpenAPI Generator 5.0+

While the examples seen in [Customization](./customization.md) use the Java System Property syntax, keep in mind that the following are equivalent:

```sh
java -Dmodels {jar} generate {opts}
```

and

```sh
java {jar} generate {opts} --global-property=models
```

Why the two differing ways to provide the same properties? We previously accepted a `-D` tooling option which resembled Java System Property declaration. In older versions of OpenAPI Generator, the option modified the SystemProperties collection directly and was truly a "system property". This option changed during the 4.x release in an effort to make OpenAPI Generator thread-safe and isolate its configuration via thread locals. We no longer mutate System Properties. In the 4.x release and earlier, specifying the tooling `-D` option with system properties intended for other tools like swagger-parser rather than passing them as true Java System Properties would lead to unexpected behavior for the user; if our tool set the system property _after_ invoking certain code, it would seem to the user like Java System Properties weren't working! 
