# @nest-native/ai-sdk

<p align="center">Decorator-first NestJS streaming primitive for the Vercel AI SDK that preserves the full Nest enhancer pipeline.</p>

<p align="center">
  <a href="https://www.npmjs.com/package/@nest-native/ai-sdk"><img src="https://img.shields.io/npm/v/@nest-native/ai-sdk.svg" alt="NPM Version" /></a>
  <a href="https://opensource.org/licenses/MIT"><img src="https://img.shields.io/badge/license-MIT-green.svg" alt="Package License" /></a>
</p>

> [!NOTE]
> **Status: `0.5.0` (`0.x`).** Functional and fully tested (100% coverage), and
> usable today — but the public API may still change before `1.0`, so pin a
> version (per semver, `0.x` minor releases can include breaking changes). See the
> [support policy](https://nest-native.github.io/ai-sdk/docs/support-policy).
> `@AiStream` streams AI SDK results on
> **both Express and Fastify** while keeping the Nest enhancer pipeline intact,
> `@AiAbortSignal` cancels the AI SDK call when the client disconnects mid-stream,
> `@AiContext` injects request-scoped context (`{ request, response, signal }`) so
> an AI SDK tool's `execute` closure can reach the current request mid-stream, and
> pre-stream vs in-stream errors are mapped correctly (HTTP errors vs documented
> stream error frames). Samples cover `streamText`, `streamObject`, the v5
> generative-UI equivalent of `streamUI` (custom data parts via
> `createUIMessageStream`), and request-scoped tool context. A
> [migration guide](../../MIGRATION.md) ports the official AI SDK NestJS cookbook
> recipe to `@AiStream`, and full
> [documentation](https://nest-native.github.io/ai-sdk/) is published.

## What This Is

`@nest-native/ai-sdk` is a community NestJS integration that will make Vercel AI
SDK streaming responses feel like a first-class Nest primitive — replacing the
"raw `@Res()` + manual piping" pattern with a decorator that keeps the Nest
enhancer pipeline (guards, pipes, interceptors, filters) intact.

The headline goal: a pre-stream guard rejection returns an HTTP error, not an
SSE error frame, and a client disconnect propagates an `AbortSignal` to the AI
SDK call.

## Compatibility

| Runtime | Supported line |
| --- | --- |
| Node.js | `>=22` (required by `ai@7`) |
| NestJS | `11.x` |
| Vercel AI SDK (`ai`) | `^7` (tracks the current major; older majors not supported) |
| HTTP adapter | Express and Fastify (parity shipped and tested) |

The published package has no runtime dependencies. The Vercel AI SDK and the
NestJS packages are declared as `peerDependencies`, so applications install only
the ecosystems they actually use.

## Installation

```bash
npm i @nest-native/ai-sdk ai
```

Required peers:

```bash
npm i @nestjs/common @nestjs/core reflect-metadata rxjs
```

## Usage

Register the module once, then decorate a handler with `@AiStream`. The handler
returns an AI SDK stream result (for example from `streamText`); the decorator
wires it to the active HTTP adapter's response.

```ts
import { Module } from '@nestjs/common';
import { AiModule } from '@nest-native/ai-sdk';

@Module({
  imports: [
    AiModule.forRoot({
      defaultHeaders: { 'x-powered-by': 'nest-native-ai-sdk' },
    }),
  ],
})
export class AppModule {}
```

```ts
import { AiStream } from '@nest-native/ai-sdk';
import { Body, Controller, Post, UseGuards } from '@nestjs/common';
import { streamText } from 'ai';

@Controller('chat')
export class ChatController {
  @Post()
  @AiStream()
  @UseGuards(ApiKeyGuard)
  chat(@Body() body: ChatDto) {
    // ApiKeyGuard runs BEFORE the stream opens, so a rejection is an HTTP
    // 401/403 — never an SSE error frame.
    return streamText({ model, prompt: body.prompt });
  }
}
```

`@AiStream` serializes to the AI SDK UI message protocol by default
(`pipeUIMessageStreamToResponse()`), the format `@ai-sdk/react`'s `useChat`
consumes. Pass `format: 'text'` for a plain text delta stream, or set `headers`
/ `status` per route:

```ts
@Post('text')
@AiStream({ format: 'text', headers: { 'x-stream': 'text' } })
chatText(@Body() body: ChatDto) {
  return streamText({ model, prompt: body.prompt });
}
```

Method-level `headers` merge over the module's `defaultHeaders` (method keys
win). Guards, pipes, and exception filters all run ahead of the stream, so
pre-stream rejections surface as ordinary HTTP responses.

### Stream types

`@AiStream` serializes any AI SDK result that exposes a `pipe*ToResponse`
method, so the three v5 streaming shapes all work through the same decorator:

- **`streamText`** — the default `ui-message` format
  (`pipeUIMessageStreamToResponse()`). See `sample/00-showcase`.
- **`streamObject`** — streams a structured object as partial-JSON text deltas.
  Its result exposes only `pipeTextStreamToResponse`, so serve it with
  `@AiStream({ format: 'text' })`. See
  [`sample/04-stream-object`](../../sample/04-stream-object/README.md).
- **`streamUI` (v5 equivalent)** — `ai/rsc`'s `streamUI` was removed in v5. The
  supported replacement is a UI message stream with custom `data-*` parts built
  via `createUIMessageStream`; wrap the returned `ReadableStream` so it exposes
  `pipeUIMessageStreamToResponse` and serve it with the default `@AiStream()`.
  See [`sample/05-stream-ui`](../../sample/05-stream-ui/README.md).

### Express and Fastify

The same handler works on either adapter — `@AiStream` writes to the underlying
Node `ServerResponse` (Express exposes it directly; Fastify exposes it via
`reply.raw`), so you do not touch the raw response yourself. The only difference
is the adapter you pass to `NestFactory.create`:

```ts
import { FastifyAdapter } from '@nestjs/platform-fastify';

const app = await NestFactory.create(AppModule, new FastifyAdapter());
```

See [`sample/01-fastify-parity`](../../sample/01-fastify-parity/README.md) for
the same controller streaming identically on both adapters.

### Cancel on disconnect with `@AiAbortSignal`

When a client disconnects mid-stream you usually want to stop generating — the
tokens go nowhere, but the provider keeps billing. `@AiAbortSignal()` injects an
`AbortSignal` derived from the client's connection; forward it into your AI SDK
call and a disconnect tears the upstream model request down immediately:

```ts
import { AiAbortSignal, AiStream } from '@nest-native/ai-sdk';
import { Body, Controller, Post } from '@nestjs/common';
import { streamText } from 'ai';

@Controller('chat')
export class ChatController {
  @Post()
  @AiStream()
  chat(@Body() body: ChatDto, @AiAbortSignal() signal: AbortSignal) {
    return streamText({ model, prompt: body.prompt, abortSignal: signal });
  }
}
```

The signal is derived from the underlying Node response, so it works identically
on Express and Fastify, and it is memoized per request — declaring
`@AiAbortSignal()` more than once yields the same signal. See
[`sample/02-abort-signal`](../../sample/02-abort-signal/README.md) for a smoke
test that disconnects mid-stream and asserts the model call is cancelled on both
adapters.

### Request-scoped tool context with `@AiContext`

An AI SDK tool's `execute` closure runs *inside* the stream, after the handler
returns, so it cannot use ordinary Nest parameter decorators to reach the current
request. `@AiContext()` injects a request-scoped `AiExecutionContext` —
`{ request, response, signal }` — that you capture in the handler and close over,
giving a tool request-scoped access (auth/headers a guard attached) plus the
client-disconnect signal:

```ts
import { AiContext, AiExecutionContext, AiStream } from '@nest-native/ai-sdk';
import { Controller, Post, UseGuards } from '@nestjs/common';
import { jsonSchema, streamText, tool } from 'ai';

@UseGuards(ApiKeyGuard) // attaches request.user pre-stream
@Controller('chat')
export class ChatController {
  @Post()
  @AiStream()
  chat(@AiContext() ctx: AiExecutionContext) {
    const request = ctx.request as { user?: { id: string } };

    return streamText({
      model,
      prompt: 'who am I?',
      tools: {
        whoami: tool({
          description: 'Return the authenticated caller.',
          inputSchema: jsonSchema({ type: 'object', properties: {} }),
          execute: async () => ({ user: request.user }),
        }),
      },
    });
  }
}
```

The context is minimal by design — `request`, `response`, and the
client-disconnect `signal` (the same one `@AiAbortSignal()` resolves). It is not
a broad context bus. Works identically on Express and Fastify and is memoized per
request. See
[`sample/07-tool-context`](../../sample/07-tool-context/README.md) for a smoke
test where a tool's `execute` reads the guard-attached user via `@AiContext` on
both adapters.

### Error mapping: pre-stream vs in-stream

`@AiStream` has a two-sided error model that mirrors when the failure happens
relative to the first byte:

- **Pre-stream errors** — thrown by a guard, a pipe, or the handler *before* it
  returns a stream — flow through the Nest enhancer pipeline and become ordinary
  HTTP errors (401 / 403 / 400 / 429 / …). The stream is never opened. This is
  the headline differentiator over the raw `@Res()` cookbook recipe, where an
  auth failure leaks into the SSE body instead.
- **In-stream errors** — thrown *during* stream production, after the first byte
  — can no longer be HTTP errors: the status and headers are already on the wire.
  Instead the AI SDK emits a **documented error frame** inside the stream
  protocol, never a partial-then-broken response. Use `onError` to map the thrown
  error to that frame's message:

```ts
@Post()
@AiStream({ onError: () => 'The model is temporarily unavailable.' })
chat(@Body() body: ChatDto) {
  return streamText({ model, prompt: body.prompt });
}
```

`onError` can also be set module-wide via `AiModule.forRoot({ onError })`; a
method-level mapper overrides the module default. When neither is set, the AI
SDK's secret-safe default (`'An error occurred.'`) is used, so raw provider
errors — which may contain credentials — never reach the client. **Only ever map
to vetted, non-sensitive messages.**

The mapper guards the wire, not your logs. The raw error stays server-side, and
`streamText`'s own `onError` option — a different callback that happens to share
the name — defaults to `console.error(error)`, printing the provider error
verbatim. Client-safe and log-safe are separate concerns: configure redacted,
structured error logging in the application, and never assume `@AiStream`'s
`onError` keeps credentials out of your logs.

> The `text` format has no error frame: `pipeTextStreamToResponse` accepts only
> status/headers and silently drops non-text events, so `onError` is ignored for
> `format: 'text'`. Use the default `ui-message` format when you need in-stream
> error reporting.
>
> Response-transform-style interceptors (e.g. a classic
> `ResponseTransformInterceptor` that rewrites the final body) are incompatible
> with streaming and are not auto-skipped — do not apply them to `@AiStream`
> routes.

See [`sample/03-error-mapping`](../../sample/03-error-mapping/README.md) for both
paths exercised on Express and Fastify.

## Testing

The `@nest-native/ai-sdk/testing` entrypoint ships deterministic, fully offline
AI SDK v4 mock language models — no provider, no API keys — so streaming
handlers can be exercised end-to-end in tests:

```ts
import { createMockLanguageModel } from '@nest-native/ai-sdk/testing';
import { streamText } from 'ai';

const result = streamText({
  model: createMockLanguageModel({ text: 'You said: ping' }),
  prompt: 'ping',
});
```

`createMockLanguageModel` streams `text` as v4 protocol chunks (word deltas for
a `string`, one delta per element for a `string[]`), fails mid-stream with the
documented in-stream error frame when `error` is set, and — with
`respectAbortSignal: true` and a `chunkDelayInMs` — honors `doStream`'s abort
signal the way a real provider does, exposing `capturedSignal()` / `started()`
/ `settled()` observers so a test can prove a client disconnect cancelled the
model call. `createToolCallingModel(toolName)` emits a single tool call so a
tool `execute` closure runs. The entrypoint adds no runtime dependencies
(`"dependencies": {}` stays). Full reference:
[testing documentation](https://nest-native.github.io/ai-sdk/docs/testing).

## Migrating from the official cookbook

Already using the official AI SDK NestJS recipe (raw `@Res()` +
`pipe*ToResponse`)? The [migration guide](../../MIGRATION.md) ports every recipe
in [`ai-sdk.dev/cookbook/api-servers/nest`](https://ai-sdk.dev/cookbook/api-servers/nest)
to `@AiStream` one-for-one, and [`sample/06-migration`](../../sample/06-migration/README.md)
ships the before/after side by side with a smoke test proving the streams are
byte-identical.

## Links

- Documentation: [nest-native.github.io/ai-sdk](https://nest-native.github.io/ai-sdk/)
- Source and issues: [github.com/nest-native/ai-sdk](https://github.com/nest-native/ai-sdk)
- Migration guide: [MIGRATION.md](../../MIGRATION.md)
- Changelog: [CHANGELOG.md](../../CHANGELOG.md)
- Vercel AI SDK: [ai-sdk.dev](https://ai-sdk.dev)
- The nest-native family: [@nest-native/drizzle](https://www.npmjs.com/package/@nest-native/drizzle), [@nest-native/trpc](https://www.npmjs.com/package/@nest-native/trpc)
