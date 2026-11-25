---
description: Document a library API, an HTTP API as OpenAPI 3, a connector's configuration keys, or the source API it calls
argument-hint: "[module] [library | http | config | source]"
allowed-tools: Read, Grep, Glob, Write, Edit, Bash(./gradlew:*), Bash(npx @redocly/cli:*), Bash(npx @apidevtools/swagger-cli:*)
disable-model-invocation: true
---

# API documentation

**Target:** $ARGUMENTS

Produce the API documentation that fits the module. The target may name the module, the kind of documentation, or both. With no kind given, detect it (step 1), say which kinds apply, and produce those. With no module given, use the one in the current directory.

| Kind | For | Output |
|---|---|---|
| `library` | a jar other modules compile against | Javadoc on the public surface, plus an overview page with "getting started" |
| `http` | a module that serves HTTP endpoints | an OpenAPI 3 contract (`openapi.yaml`) versioned with the code |
| `config` | a connector or service that operators configure | a configuration reference: every key, type, default, sensitivity, validation and example |
| `source` | a connector that calls an external API | notes on how that API really behaves: auth, paging, filtering, change tracking, limits, errors |

## 1. Preflight

1. **Find out which kinds apply, from the code.** Public types in a published jar mean `library`. Controllers or handlers mean `http`. Config-key constants and validators mean `config`. HTTP calls out to a vendor mean `source`. If the module serves no HTTP endpoints, say the OpenAPI part doesn't apply; don't invent one.
2. **Find the existing docs and generators.** Look for `openapi.yaml` or `openapi.json`, `docs/`, generated reference pages, and config tables in the README or `CLAUDE.md`. Extend them rather than creating a parallel document. Never hand-edit generated output.
3. **Detect the framework before suggesting tooling.** Suggest springdoc only if the build already uses Spring Boot, and JAX-RS tooling only if JAX-RS is there. For plain Java modules, hand-author the OpenAPI document, which is framework-neutral. Never add a framework dependency to a module that doesn't use that framework.
4. **Document only what's true.** Take signatures, keys, defaults and status codes from the code. For source behaviour, mark each fact as *documented* (vendor docs) or *measured* (observed on a real instance, with the date). If something is unknown, say so.

## 2. Library API

1. Write full Javadoc on the public surface: a specific first-sentence summary; `<p>` paragraphs on behaviour, nullability and thread-safety; every `@param`, `@return` and `@throws`; `{@link}` and `{@code}` for identifiers. Avoid the doclint errors that fail the build on JDK 21: self-closing tags such as `<p/>`, a raw `<` or `&` in text, headings out of sequence, and links to types javadoc can't find. (The `javadoc` command covers this in depth, if the user wants to run it.)
2. Add an overview page with a "getting started" section. In Gradle:
   ```groovy
   javadoc {
     options.overview = file('src/main/javadoc/overview.html').absolutePath   // the file must exist
   }
   ```
   ```html
   <!DOCTYPE html>
   <html>
   <head><title>Overview</title></head>
   <body>
   <p>One sentence on what this library does for its callers.</p>
   <h1>Getting started</h1>
   <p>Create a {@link com.example.demo.Greeter} and call {@code greet}.</p>
   </body>
   </html>
   ```
   This was generated with Gradle 8.14 and Java 21. In an overview file, headings start at `<h1>`; an `<h2>` first is a doclint error there, unlike in class docs.
3. Ship the docs with the jar using `java { withJavadocJar() }`. Note that `javadoc` then runs in `assemble`, so Javadoc errors fail the build.
4. Track how the API evolves: `@since` on new public API, `@deprecated` naming the replacement (with the `@Deprecated` annotation), and breaking changes in the changelog or release notes.

## 3. HTTP API contract (OpenAPI 3, framework-neutral)

1. Describe the contract in an OpenAPI 3 document versioned next to the code. Hand-author it, or produce it with the generator the project already uses.
2. For every operation give a summary and an `operationId`; parameters with a description, constraints and an example; request and response schemas; and every status code, including the errors.
3. Define the error model once and reference it from every error response. The RFC 7807 problem-details shape works well.
4. Keep the API's schemas separate from internal types, and give realistic examples that match the schemas.
5. Declare the security schemes (bearer token, API key) and mark which operations are protected.
6. Version the API, in the path or a header. Keep the spec diffable, so contract changes show up in pull requests.

This template validates as OpenAPI 3.0.3 with `swagger-cli` and Redocly. Redocly's two warnings are about placeholders you replace: the `example.com` server and a licence with no URL.

```yaml
openapi: 3.0.3
info:
  title: Example Connector Control API
  version: 1.0.0
  description: Read-only status endpoints for a running connector.
  license:
    name: Proprietary
servers:
  - url: https://connector.example.com/api/v1
security:
  - bearerAuth: []
paths:
  /objects/{name}/status:
    get:
      summary: Returns the extraction status of one data object.
      operationId: getObjectStatus
      tags: [status]
      parameters:
        - name: name
          in: path
          required: true
          description: Data object name, as discovered from the source.
          schema:
            type: string
            example: orders
      responses:
        '200':
          description: The current status.
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/ObjectStatus'
              example:
                name: orders
                stage: CHANGE_FEED
                lastCursor: '2026-10-14T05:00:00Z'
                recordsDelivered: 1200
        '401':
          $ref: '#/components/responses/Problem'
        '404':
          $ref: '#/components/responses/Problem'
components:
  securitySchemes:
    bearerAuth:
      type: http
      scheme: bearer
  responses:
    Problem:
      description: An error, in RFC 7807 problem-details shape.
      content:
        application/problem+json:
          schema:
            $ref: '#/components/schemas/Problem'
  schemas:
    ObjectStatus:
      type: object
      required: [name, stage]
      properties:
        name:
          type: string
        stage:
          type: string
          enum: [DISCOVERY, SNAPSHOT, CHANGE_FEED]
        lastCursor:
          type: string
          nullable: true
        recordsDelivered:
          type: integer
          format: int64
          minimum: 0
    Problem:
      type: object
      required: [title, status]
      properties:
        type:
          type: string
          format: uri
        title:
          type: string
        status:
          type: integer
        detail:
          type: string
        instance:
          type: string
```

## 4. Configuration reference

Every key an operator can set goes in one table. If the code declares keys with metadata (type, default, a "secret" flag), generate the table from the code rather than maintaining it by hand.

| Key | Type | Required | Default | Sensitive | Validation | Example | What it does |
|---|---|---|---|---|---|---|---|
| `source.baseUrl` | URL | yes | — | no | `https://` only | `https://acme.example.com` | The instance to read from. |
| `source.clientSecret` | string | yes | — | **yes**, masked | not blank | `****` | OAuth client secret. Never logged. |
| `cdc.lagSeconds` | integer | no | `30` | no | ≥ 0 | `30` | Safety lag behind the source clock. Lowering it risks missed updates. |

- State which layer applies each default (core, connector or host), and what happens to an unknown or misspelled key: refused, or silently ignored.
- Document the precedence between config sources (bundled file, external file, environment variables), and whether paths must be absolute.

## 5. Source API notes

Write one section per topic, and tag each fact *documented* or *measured (date, instance)*:
- **Auth:** grant type, token lifetime and refresh, the permissions or roles needed.
- **Endpoints used:** method, path, purpose and key parameters.
- **Paging:** the mechanism, the maximum page size, the end-of-data signal, and whether it's stable while rows change.
- **Filtering and ordering:** the syntax, and what happens with an unknown field. Some APIs silently drop the clause.
- **Change tracking:** timestamps and watermarks, audit or event tables, how deletes are observable, how restores behave.
- **Time:** the server's timezone, the format of time literals, and which clock to trust.
- **Limits and errors:** rate limits, payload and query-length limits, status codes, retry hints.
- **Schema:** where types and metadata come from, and what's missing (precision, scale).

## Avoid these errors

- **Validate the OpenAPI document** before handing it over: `npx @redocly/cli lint openapi.yaml` or `npx @apidevtools/swagger-cli validate openapi.yaml`, if Node is available. (`@apidevtools/swagger-parser` is a library with no command-line tool.) Otherwise say the document is unvalidated.
- **Version-specific syntax.** In OpenAPI 3.0, `nullable: true` is valid. In 3.1, write `type: [string, 'null']` instead. Don't mix the two.
- **References.** Every `$ref` must resolve, and every example must match its schema.
- **Javadoc overview.** The `overview` path must exist, or `javadoc` fails. Its headings start at `<h1>`.
- **Keep it current.** A stale contract is worse than none. Update the spec in the same change that alters the API.

## Verify

- The OpenAPI validator passes, and a request built from each example works against a test instance.
- `./gradlew javadoc` builds with the overview, and the index page shows the getting-started section.
- Config table: every key in the code appears in the table, and every key in the table exists in the code.
