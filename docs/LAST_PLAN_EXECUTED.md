---
PLAN: "refactor!: transformer becomes encoder (module webtyp.com/encoder, package encoder) and uses webtyp.com/nn"
TAG: v0.2.0
EXECUTOR: jules
REVIEWER: none
---

> This plan is dispatched via the CodeJob workflow. See skill: agents-workflow.
>
> Part of
> [`AGENT_ECOSYSTEM_MASTER_PLAN.md`](https://github.com/webtyp/agent/blob/main/docs/AGENT_ECOSYSTEM_MASTER_PLAN.md).
> **Blocked until `webtyp.com/nn` v0.1.0 is published.** `webtyp/bekko` waits for this tag.

# Plan — `webtyp/transformer` is renamed `webtyp/encoder`

## 0. Context

This repository computes the **encoder** of an embedding model (ModernBERT: token ids in, one
vector out). It was called `transformer`, a name that promises every transformer, including
the decoder that generates text. That decoder is now being built in its own repository,
`webtyp/decoder`. With both next to each other, "transformer" would be ambiguous. The GitHub
repository has already been renamed to `webtyp/encoder`. This plan renames the Go module and
package to match.

The stateless operations (`MatmulT`, `LayerNorm`, `RMSNorm`, `Softmax`, `GELU`, `SiLU`, `Add`,
`RoPE`) were copied, unchanged, to `webtyp.com/nn` v0.1.0 so the decoder and the speech models
can share them. This plan deletes the copy here and calls `nn`.

## Development rules (inline)

- Primary runtime: browser, TinyGo/WASM. Every file compiles under `GOOS=js GOARCH=wasm` and TinyGo.
- **Behaviour must not change.** The encoder was verified to cosine ≥ 0.999 against the real
  `bekko-embedding-v1-a8m`. This plan only renames and re-imports.
- **Never import:** `fmt`, `errors`, `strings`, `strconv` (use `webtyp.com/fmt`), `context`,
  `encoding/json`, `sort`, `map[K]V`, `os`, `log`. `math` is allowed.
- Plain Go, one implementation: no SIMD, no build tags.
- Tests: `testing` + `math` only. Do **not** run `gopush`/`codejob`.

## Design gate (api-design — five answers)

1. **Prior art.** **Hugging Face `transformers`** separates `*Encoder` and `*Decoder` classes
   (`BertEncoder`, `T5Stack` as encoder or decoder). **ONNX exports** of seq2seq models ship
   `encoder_model.onnx` and `decoder_model.onnx`. **whisper.cpp** has `whisper_encode` and
   `whisper_decode`. They all name the half, not the architecture family.
2. **Novice-name test.** `encoder.Encode(cfg, w, tokenEmbeds, seqLen)` reads as "encode these
   tokens". `transformer.Encode` next to a future `decoder.Generate` would leave a reader
   asking whether "transformer" also decodes.
3. **Complexity ledger.**
   ```
   Concepts the developer must learn   +0 / −0
   Files they must touch to do X       +0 / −1   (kernels.go gone)
   Lines at the call site              +0 / −0   (import path and package name)
   Ways to do the same thing           +0 / −1   (the 8 operations exist only in nn)
   ```
4. **Where it belongs.** The encoder graph (`Config`, `Weights`, `Encode`, `GatedFFN`,
   `Activation`, pooling) stays here. The stateless operations live in `nn`.
5. **What it deletes.** `kernels.go`, `kernels_test.go`, the module path
   `webtyp.com/transformer` and the package name `transformer`.

## Stage 1 — module and package

- `go.mod`: `module webtyp.com/encoder`. `go get webtyp.com/nn@v0.1.0`.
- Every `package transformer` → `package encoder` (including test files).
- Every error prefix `"transformer: …"` → `"encoder: …"`. The rest of each message is unchanged.

## Stage 2 — use nn

- Delete `kernels.go` and `kernels_test.go`. Their tests now live in `webtyp/nn`.
- In `encode.go`, import `webtyp.com/nn` and call `nn.MatmulT`, `nn.LayerNorm`, `nn.RoPE`,
  `nn.Softmax`, `nn.GELU`, `nn.SiLU` (and `nn.RMSNorm` / `nn.Add` if used). Update the comment
  on `GatedFFN` ("the SiLU/GELU kernels in kernels.go" → "the SiLU/GELU operations of
  webtyp.com/nn").
- `webtyp.com/vector` stays a dependency only if something other than the deleted `MatmulT`
  still uses it. Otherwise `go mod tidy` removes it.

## Stage 3 — docs

- `README.md`, `AGENTS.md`: title and every `transformer` reference → `encoder`. Add one sentence:
  "Formerly `webtyp/transformer`; the stateless operations it used are in `webtyp/nn`."
- `docs/LAST_PLAN_EXECUTED.md` is history: do not edit it.

## Stages

| Stage | Files | Acceptance |
|---|---|---|
| 1 | `go.mod`, all `.go` files | `grep -rn "package transformer\|\"transformer:" --include=*.go .` → empty |
| 2 | `encode.go`; `kernels.go`, `kernels_test.go` deleted | `test ! -e kernels.go`; `grep -n "nn\." encode.go` finds the calls |
| 3 | `README.md`, `AGENTS.md` | `grep -n "transformer" README.md AGENTS.md` only finds the "Formerly" sentence |
| all | — | `gotest` and `gotest -tinygo` pass; `BenchmarkEncode_20x12x384` still runs |
