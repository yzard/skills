# Receipt Master data and configuration contract

Use this contract when changing Receipt Master's Compose deployment, build/bootstrap
scripts, backend configuration, prompts, or mobile provisioning. It describes the
current local NInfer deployment, not a generic Docker convention.

## Two independent data roots

| Service | Host data root | Container root | Access |
| --- | --- | --- | --- |
| `backend_api` | `playground/backend_api/` | `/data` | read/write |
| `backend_ocr` | `playground/backend_ocr/` | `/data` | read-only |

The host service directory **is** the data root. Do not add a nested `data/` or
`config/` directory, mount the shared `playground/data/` tree, or expose API business
data to OCR. API keeps SQLite, original and derived receipt images, recognition
evidence, staging files, backups, `config.toml`, and `prompt.toml` under its root.
OCR's data root contains its own `config.toml`; the NInfer model weights are packaged
in the OCR image, not copied to either data root. `/artifacts/` is part of the API
image and contains the initial APK, update manifest and immutable update APKs. Do not
mount a host artifact directory. See the [build and publication contract](receipt-master-build-release.md).

Both programs require an explicit absolute `--data-dir /data` startup argument and
load the **fixed** path `<data-dir>/config.toml`. A missing directory or config,
malformed TOML, or invalid settings must fail startup. Neither service accepts a
config filename/path argument or a `data_dir` setting inside TOML. Runtime config
is TOML only; do not reintroduce YAML or a second config location.

## Keep settings with their owner

- API `config.toml` has `[general]` for API listener/runtime settings and `jwt_secret`
  for user sessions; `[ocr]` contains **only** `url` and `api_key` for authenticating to
  the internal OCR REST service. Clients use user authentication; do not restore a
  shared client API key or embed the JWT signing secret in an APK.
- OCR `config.toml` has `[general]` for OCR listener, service `api_key`, request and
  idle timeouts; `[engine]` owns the model path/name, output token limits, thinking,
  context and image capacity, and other inference settings. API obtains image capacity
  through OCR's authenticated capabilities endpoint rather than copying model settings.
- JWT signing secrets and every service key/password stay embedded in the owning
  service's `config.toml`. API `BACKEND_OCR_URL` may override the noncredential URL;
  `BACKEND_OCR_API_KEY` and other secret environment/CLI overrides are not supported.
  API `[ocr].api_key` must match OCR `[general].api_key`. Keep secret-bearing TOMLs
  owner-readable only (0600), excluded from Git/images/APKs, and out of logs and exports.
- API's separate `/data/prompt.toml` contains Logo localization/matching prompts,
  the general receipt prompt, and per-store prompts. OCR does not own receipt prompts.
  Every recognition attempts Logo identification before selecting the matching store
  prompt; an unknown store uses the general prompt. Do not add `[logos].enabled` or
  another switch to skip Logo recognition.
- Do not put OCR model, output-token, thinking, or image settings in API config. Do not
  restore local-model token pricing, fee estimates, or monthly budget reminders. The
  mobile clients have no OCR/model/pricing/budget controls; they authenticate to API,
  capture/upload, submit jobs, edit, and display server results.

Bootstrap editable templates only when missing. Preserve operator edits and stable
keys across rebuilds/restarts. A one-time **filesystem layout** move may relocate
existing data and config into the two service roots, but must preserve SQLite,
photos, backups, and key values; this is not a database-schema migration. Stop
containers before moving a live SQLite directory, fail on conflicting destinations,
and verify record and media counts after the move.

## Host identity and verification

Use the standard `PUID` / `PGID` environment variable pair throughout Receipt Master:
entrypoints, build scripts, Compose, tests, examples and deployment documentation.
`run_playground.sh` exports its invoking user's numeric `PUID` and `PGID` to both Compose services.
Their entrypoints validate those values and run the service and NInfer processes under
that UID/GID. API may adjust ownership only within its writable `/data` mount;
OCR's `/data` stays read-only. When NVIDIA creates root-only capability device nodes
inside the OCR container, grant the inference UID access **inside that container**;
do not change host GPU device ownership or run inference as root. Direct Compose
invocations must provide `PUID` and `PGID` explicitly, or use an operator environment
file such as NAS `env_file: ./lsio.env`. Do not add a separate group-ID variable or
an alternate legacy spelling. On the NAS, preserve `lsio.env`'s shared identity values;
set any `BACKEND_OCR_URL` override explicitly in the receipts service's Compose
`environment`; embed the OCR key directly in API/OCR TOMLs, never an operator `.env` file.
The API has no `init: true`, Compose `healthcheck`, `ports`, or `expose`.
Declare the API's default service port with `EXPOSE 8000` in its
Dockerfile so Caddy's container discovery works without Compose port declarations.
Image `EXPOSE` metadata does not publish a host port.

Validate the changed contract with config/bootstrap tests, Docker builds, separate
Compose mount inspection, process UID/GID and file ownership checks, and service
startup. When runtime identity or model-file permissions change, complete a real
cold-start inference request as a non-root OCR process. Confirm the API still reads
the existing receipts and serves the APK.
