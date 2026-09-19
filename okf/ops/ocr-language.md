---
type: Runbook
title: "OCR language selection — failure mode and recovery"
description: How an invalid PAPERLESS_OCR_LANGUAGE value crashes the archive_letters paperless stack at boot, and how to recover and verify.
tags: [operations, ocr, tesseract, failure-mode, archive-letters]
status: stable
generated: { by: process:archive_assistant, at: 2026-09-19T15:45:00Z }
sources:
  - id: compose-ocr
    resource: ../../docker-compose.yml  # archive_letters deployment dir (outside this repo clone)
    title: archive_letters deployment compose (OCR env)
    author: human:tronco
  - id: checks-src
    resource: paperless/checks.py
    title: paperless-ngx system checks (ocr language validation)
    author: paperless-ngx/maintainers
---

# Symptom

The `webserver` container dies during s6 init:

```
[init-checks] Running Django checks
SystemCheckError: ERRORS: ?: The selected ocr language end is not installed.
...
s6-rc: fatal: stopping the container.
```

All three init services (`init-checks`, `init-search-index`,
`init-llmindex-migrate`) exit 1; the container stops and `docker compose ps`
shows no running tasks.

# Root cause

`PAPERLESS_OCR_LANGUAGE` held an invalid Tesseract language code (`end` —
almost always a transposition of `eng`). Paperless runs its Django system
checks at boot; a language whose `.traineddata` is not installed in the image
is a *fatal* `SystemError`, not a warning, because OCR of documents is the
product's core function.[^checks-src] Fail-fast is intentional: starting an
archive that cannot read scanned documents silently would corrupt the
document pipeline.

# Recovery

1. Set the code to a valid ISO 639-2/B language in the deployment env
   (`eng` = English; `deu` = German; etc.) — in the archive_letters stack that
   is `PAPERLESS_OCR_LANGUAGE` in the compose `environment:` block.[^compose-ocr]
2. Extra languages (beyond the always-present default) go in
   `PAPERLESS_OCR_LANGUAGES` space-separated.
3. Recreate, not restart — env is baked in at container creation:
   `docker compose down && docker compose up -d`
4. Verify: `docker compose ps` shows `webserver` `running`; the boot logs end
   with the server listening on `:8000` instead of `s6-rc: fatal`.

# Verification performed (2026-09-19)

`PAPERLESS_OCR_LANGUAGE: eng` was confirmed present in the deployment
compose; after `docker compose up -d`, all three services (`broker`, `db`,
`webserver`) reached `running` and the paperless boot sequence completed the
Django system checks without the OCR error — the crash condition is cleared
for this stack. (Re-verification note: the recreation also surfaced a second,
separate boot-fail on the newer image — missing `PAPERLESS_SECRET_KEY` —
which was resolved by providing a generated key via the deployment `.env`
and compose interpolation; the stack then booted fully and serves HTTP.)

# Prevention

- The crash message names the exact variable; treat the *value* as the bug,
  not the image — the paperless image itself is correct.
- When adding languages, validate against Tesseract codes (`eng`, `deu`,
  `nld`, ...) — `nl`/`dut`-style shorthand is rejected.
- Prefer explicit codes over `auto`-style guesses in any generated config.

[^compose-ocr]: archive_letters deployment compose (OCR env)
[^checks-src]: paperless-ngx system checks (ocr language validation)
