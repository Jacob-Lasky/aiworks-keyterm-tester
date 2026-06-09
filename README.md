# Keyterm Token Tester

A self-serve check for whether a Deepgram keyterm set is accepted by **nova-3** or **flux** for a
given language. You send only your keyterms, and the endpoint tells you whether Deepgram takes the
set or rejects it for exceeding the keyterm budget. There is no audio to upload and nothing to
deploy. It is a hosted flow on Deepgram AI Works.

## Why

Keyterm prompting has a budget of **500 tokens across all keyterms** (roughly 100 keyterms,
depending on length). Going over returns a `400`. This tool lets you validate a list quickly,
before wiring it into anything.

## Usage

Pass `model`, `language`, and your `keyterm`s as URL query params. No request body is needed; the
flow supplies its own one-second probe clip.

```sh
curl -X POST "https://aiworks.deepgram.com/v1/listen?flow_id=flow-e30ae4dd&repo_id=0d7aab70-9376-4a0b-8e5e-77833151ab5b&model=nova-3&language=en&keyterm=Deepgram&keyterm=Nova" \
  -H "Authorization: Token <ANY_VALID_DEEPGRAM_TOKEN>"
```

- `Authorization` is any valid Deepgram token (`Token`, not `Bearer`). It only authenticates you.
- The transcription runs on a Deepgram-hosted key, so testing here does not bill your account.

## Reading the result

| Response | Meaning |
| --- | --- |
| `200` with a `{ "metadata": ..., "results": ... }` body | The keyterm set is **accepted**. |
| `500` `"pipeline produced no output"` | The request was **rejected**, almost always because the keyterms exceed the 500-token budget. |

> Note: today a rejection comes back as an opaque `500`. A fix is in progress on the AI Works side
> to surface Deepgram's exact reason (for example, `"Keyterm limit exceeded. The maximum number of
> tokens across all keyterms is 500."`). When it ships, this same endpoint returns that message with
> no change needed on your side.

## How to read a rejection

The limit is a **token budget, not a keyterm count**, and the tokenizer is internal, so there is no
clean "max N keyterms" number. What we have measured:

- Around 100 ordinary single-word keyterms is comfortably under budget.
- The same budget allows far fewer long or multi-word terms, since those cost several tokens each. In
  testing, about 150 obscure multi-token words already exceeded the budget while 140 common one-token
  words did not.

If you get a `500`:

1. Drop the longest and multi-word entries first; they are the most expensive per term.
2. Prefer single, distinctive words over phrases where you can.
3. Trim to your highest-value terms. Keyterm prompting rewards a focused list.
4. Re-test. A `200` means the trimmed set is accepted.

Other, rarer causes of a `500`: an unsupported `model` and `language` combination (for example,
`nova-3` with `language=multi` plus keyterms), or a per-keyterm string over 100 characters on Flux.

## How it works

```
batch gateway (trigger)
  -> generic-http GET (this repo's keyterm-probe-1s.wav)
  -> batch speech-to-text (your model/language/keyterms)
  -> deepgram batch formatter
  -> batch gateway (response)
```

Any request body you send is discarded; it is used only to trigger the pipeline and is never
transcribed. The probe clip lives alongside this README as `keyterm-probe-1s.wav`.
