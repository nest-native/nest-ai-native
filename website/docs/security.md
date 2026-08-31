# Security

Security is a first-class concern for this package: every PR includes an explicit
security pass, and the design exists partly to fix a security-relevant `@Sse`
defect. This page covers the application-security surface of streaming AI SDK
endpoints.

## Pre-Stream Auth Cannot Be Bypassed

The headline guarantee: guards run **before** the stream opens, so an auth
rejection is a real HTTP `401`/`403`. This is the opposite of `@Sse`, which opens
the connection first and turns a failed pre-flight check into an SSE error *event*
on an already-`200` connection. With `@AiStream`, a client that fails auth never
receives a streamed `200`.

The package's test suite asserts this directly — pre-stream guard rejection is
covered on both Express and Fastify. Do not work around it by moving auth into the
handler body; keep it in a guard so the rejection stays an HTTP error.

## Secret Leakage In Streaming Responses

API keys and provider credentials must never appear in user-visible output. Two
rules:

1. **In-stream error frames default to a generic message.** The AI SDK's default
   `onError` is `() => 'An error occurred.'`, which hides server-side detail. If
   you override it, map only to vetted, non-sensitive strings — never raw provider
   errors, which can contain credentials. See [Error Mapping](error-mapping.md).
2. **Never echo configuration into the stream.** Do not write headers, env vars,
   or model configuration into the streamed body.
3. **Client-safe is not log-safe.** The mapper only controls what goes on the
   wire. The raw provider error stays server-side — `streamText`'s own `onError`
   option defaults to `console.error(error)` — so redacted, structured error
   logging is the application's job, not the mapper's. See
   [Error Mapping](error-mapping.md).

## Prompt Injection

Request input flows into the model. Treat it as untrusted:

- Validate and constrain input with a pipe before it reaches the AI SDK call (see
  [Enhancer Pipeline](enhancer-pipeline.md)).
- Keep system prompts server-side; do not let request input override them.
- Scope any tools the model can call to the authenticated principal.

The samples demonstrate safe input handling — request bodies are validated with a
Zod pipe or a DTO before reaching `streamText`.

## Cost As A Security Concern

Uncontrolled generation is a denial-of-wallet vector. The cost-control patterns
in [Production Patterns](production-patterns.md) — abort-on-disconnect, output
caps, and rate-limiting guards — are part of the security posture, not just
performance tuning.

## Supply Chain

The published package keeps `"dependencies": {}` empty. The AI SDK and NestJS
packages are peers, so the package adds no transitive runtime supply chain of its
own. The AI SDK is a fast-moving peer; review its changelog at every bump. See
[Quality and CI](quality-and-ci.md) for the audit gates.

## Reporting

Report vulnerabilities per
[`SECURITY.md`](https://github.com/nest-native/ai-sdk/blob/main/SECURITY.md) in
the repository. Do not open public issues for security reports.
