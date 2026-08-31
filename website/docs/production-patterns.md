# Production Patterns

Streaming LLM endpoints have failure and cost characteristics that ordinary JSON
endpoints do not. These patterns keep an `@AiStream` route safe and affordable in
production. All of them are application-owned — the package gives you the hooks;
your app sets the policy.

## Cancel On Disconnect (cost control)

The single most important pattern: forward `@AiAbortSignal()` into every AI SDK
call. A client that closes the tab otherwise leaves the model generating tokens
you pay for.

```ts
@Post()
@AiStream()
chat(@Body() body: ChatDto, @AiAbortSignal() signal: AbortSignal) {
  return streamText({ model, prompt: body.prompt, abortSignal: signal });
}
```

See [@AiAbortSignal](abort-signal.md).

## Cap Output Length

Bound the model's output so a single request cannot run away. The AI SDK exposes
this on the call itself:

```ts
return streamText({
  model,
  prompt: body.prompt,
  abortSignal: signal,
  maxOutputTokens: 1024,
});
```

## Rate Limiting

Apply rate limiting as a **guard** so a rejection is a pre-stream HTTP `429`,
never a stream frame. `@nestjs/throttler` works unchanged because guards run
before the stream opens:

```ts
@Post()
@AiStream()
@UseGuards(ThrottlerGuard)
chat(@Body() body: ChatDto) {
  return streamText({ model, prompt: body.prompt });
}
```

## Input Validation

Validate the request body with a pipe so malformed input is a clean pre-stream
HTTP `400`. Both class-validator DTOs and Zod schemas are supported. See
[Enhancer Pipeline](enhancer-pipeline.md).

## Observability

Pre-stream interceptors run before the first byte, so use them for logging,
metrics, and tracing. A response-transform interceptor is incompatible with
streaming — keep it off `@AiStream` routes (see
[Enhancer Pipeline](enhancer-pipeline.md)). For in-stream telemetry, instrument
inside the AI SDK call (the AI SDK exposes its own telemetry hooks) rather than
trying to wrap the response.

## In-Stream Error Hygiene

Set a module-wide `onError` mapper so mid-stream failures emit a stable,
non-sensitive message rather than the AI SDK's generic default. Never surface raw
provider errors. The mapper covers the wire only — the raw error still reaches
your logs, so pair it with redacted error logging. See
[Error Mapping](error-mapping.md).

```ts
AiModule.forRoot({
  onError: () => 'The assistant is temporarily unavailable.',
});
```

## Checklist

- [ ] Forward `@AiAbortSignal()` into every AI SDK call.
- [ ] Set `maxOutputTokens` (or the provider equivalent) on every call.
- [ ] Rate-limit with a guard so over-limit is a pre-stream `429`.
- [ ] Validate input with a pipe so bad input is a pre-stream `400`.
- [ ] Set a module-wide `onError` that only emits vetted messages.
- [ ] Redact errors in your logger too — `onError` does not sanitize logs.
- [ ] Keep response-transform interceptors off streaming routes.

See [Security](security.md) for the auth and secret-leakage angle.
