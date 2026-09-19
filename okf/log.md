# Bundle Update Log

## 2026-09-19
* **Initialization**: Created the OKF bundle for the paperless-ngx fork (operations + security notes for the archive_letters deployment).
* **Creation**: Added the [OCR language selection runbook](/ops/ocr-language.md) after the `PAPERLESS_OCR_LANGUAGE=end` crash-loop was diagnosed and cleared (recovered with `eng`, verified by stack recreation).
* **Creation**: Added the [Docker image supply-chain assessment](/security/docker-images.md) covering provenance, the `latest`-tag weakness, and exposure of the three running images.
* **Update**: Opened [PR #1](https://github.com/Troncooooo/paperless-ngx/pull/1) (branch `docs/okf-ops-security` → `dev`) carrying this bundle.
