---
name: docker-build
description: Create or change Python/Rust Docker images, Android APK builds, entrypoints, and build or publish scripts.
allowed-tools: Write, Read, Bash, Glob, Grep
---

# Docker builds

## Client navigation and editing

For client application tabs and editors, keep the original view state and open editors as overlays.
Do not reload the original view on tab return or editor close.
Follow the [client navigation and editing contract](../add-modify-codebase/references/client-navigation.md) when this work affects those flows.
Apply it to designs, implementation, reviews, or changed-client release checks within this skill's existing scope.


All background-service credentials are embedded in the owning service's `config.toml`;
follow [the shared credential constraint](../general-coding/SKILL.md#background-service-credentials).
Do not add credential files, Docker secret mounts, or environment/CLI key/password overrides.


Use repository-owned build scripts and mandatory platform entrypoints. Preserve existing
runtime/deployment contracts; a generic template is not a reason to replace them.

- Default layout is `docker/` for image files and root `build_docker.sh`, with repository
  exceptions respected. Build context stays at the git root; COPY uses paths inside it.
- Build scripts resolve their own location, validate arguments, and fail on command errors.
  Local build is the default. Publishing requires an explicit publish command/flag and user
  authorization; do not add an unconditional registry push to a build script.
- Preserve the project's tag scheme. Date tags plus latest are an option for a new script,
  not a reason to change an existing release contract.
- Cache dependency installation separately from source changes; use lockfiles and the
  project's toolchain. Choose a base compatible with native/GPU dependencies rather than
  mandating Alpine. Preserve existing Python uv/pip/virtual-environment decisions.
- Inspect actual binary linkage and include required runtime libraries, certificates, and
  timezone data. Installing static libraries does not prove a binary is statically linked.
- Use Dockerfile-specific ignore files beside each Dockerfile. Exclude secrets/runtime data
  while retaining required COPY inputs, including entrypoints. Compose bind/context paths
  resolve relative to the Compose file; update build/CI/docs when moving them.

## Android APK builds

- Build Android APKs inside Docker. A repository `build_android.sh` orchestrates the
  container build and artifact export; it must not compile with host Flutter, Java, or
  Android SDK installations. This rule does not apply to iOS/Xcode builds.
- Pin the container's Flutter and Android toolchain versions, honor dependency lockfiles,
  and cache dependency downloads separately from application source.
- Export APKs to the repository's build output directory with ownership usable by the
  invoking user. Preserve the development signing identity across builds, supplying keys
  through runtime mounts or build secrets; exclude keys and host build state from context.
- Update callers, CI, and documentation to use the Docker entrypoint, and actually build
  the APKs to validate the path. Do not claim a container build based on host artifacts.

## Combined mobile and backend builds

Apply these script contracts when a repository includes mobile applications and a Docker
backend; do not introduce absent platforms or services merely to satisfy the convention.

- Root `build_docker.sh` is the combined build entrypoint. It calls `build_android.sh`
  (not `android.sh`), stages the resulting APK for backend download, and builds the backend
  image. A successful combined build must include a freshly built or valid Docker-cached
  APK; do not silently reuse an unrelated host artifact or skip Android.
- Root `build_android.sh` owns mobile workspace preparation, Android checks/tests and
  containerized APK compilation/export. Fold `prepare_mobile.sh` into this entrypoint;
  a preparation-only mode may be shared by iOS without compiling Android on the host.
- Provide a separate root `build_ios.sh` for iOS checks/tests and compilation on
  macOS/Xcode. `build_docker.sh` must not call it. State whether its output is unsigned;
  do not report iOS compilation as verified on Linux.
- Fold `run_*checks.sh` into the relevant `build_*.sh` workflow or required Docker build
  stages. Checks must fail the build on errors. CI calls those maintained entrypoints or
  their check stages rather than keeping a duplicate standalone check workflow.
- Root `run_playground.sh` is a no-argument foreground entrypoint, without `up`, `down`,
  `logs`, or `status` subcommands. Every invocation calls `build_docker.sh` to build the
  current image and APK, then initializes any image-dependent secrets and starts Docker
  Compose attached to continuously stream service logs (equivalent to following logs).
  Use `exec docker compose ... up --no-build` so Ctrl+C reaches Compose, stops the
  playground containers, and exits. Do not detach or leave services running after Ctrl+C;
  preserve volumes and data. Build failure must exit before starting/replacing services.
  Replace service-specific names such as `build_backend_ocr.sh` and `run_backend_ocr.sh`
  with the combined entrypoints, updating callers, CI and docs and removing obsolete scripts.

### Receipt Master build and publication

Receipt Master's `build_docker.sh` defaults to local builds and supports
`--publish dockerhub|github --username NAME` for explicit publication. Build both
backends with `YYYYMMDD`, captured once at script start in the build host's timezone.
After all checks pass, mark `latest` as an alias of each corresponding tested image ID;
Compose and playground use `:latest`. Push both date tags, then both `latest` tags
from those same IDs. No custom `--tag` or unconditional remote pushes. `github` maps
to GHCR; credentials come from the operator's existing Docker login.
For changes to this CLI, registry publishing, or APK packaging, read
[build, publication and embedded APK contracts](references/receipt-master-build-release.md).

### Playground networking and Compose entrypoint

- `run_playground.sh` starts the stack directly with root-relative
  `docker/docker-compose.yaml`; keep one canonical Compose file and update callers/docs
  when renaming it. Retain foreground logs and Ctrl+C shutdown.
- Publish the API on host `0.0.0.0:<published-port>`, explicitly mapped to its actual
  container service port. The service inside the container must also listen on `0.0.0.0`.
- Print the mapping and usable host network IP + published port URLs at startup, including
  the APK download URL when present. Derive ports from the Compose configuration; do not
  duplicate them in the script. `0.0.0.0` is a bind address, not a client destination.
  Exclude loopback and Docker bridge/container interfaces from external address discovery;
  show a clear fallback if discovery fails. Do not label a LAN address as a public Internet IP
  or fetch a public IP from an external service just to print local access URLs.
- `backend_api` and `backend_ocr` communicate through Docker service DNS and container ports
  over a shared `internal: true` network. Publish no OCR host ports and do not route their
  requests through host/public URLs. Only API joins the ingress network for host access;
  the OCR container keeps its inference engine on loopback.

### Receipt Master APK delivery contract

For Receipt Master, the combined build atomically stages the Android arm64 APK as
`build/mobile/receipt_master.apk`. The playground backend serves it at
`/receipt_master.apk` on port 5000, listening on `0.0.0.0`. Link users to the actual host
or LAN IP; `0.0.0.0` is the bind address. The initial APK, update manifest and immutable
update APKs must be copied into the `backend_api` image's `/artifacts/` during the
combined build, after Android compilation. Do not supply them through a runtime volume
or bind mount. Preserve readable files and traversable directories for the service UID;
verify downloads with only `/data` mounted. The OCR image contains its model, without
APK files. Receipt Master uses exactly two independent images/containers: `backend_api`
(`src/backend_api/`, Rust API, configurable general/store prompts, JSON validation,
arithmetic/unit conversion, SQLite and async jobs; no model or GPU) and `backend_ocr`
(`src/backend_ocr/`, Qwen3.8-27B NVFP4 through pinned NInfer for receipt extraction, logo localization and visual reference matching). Package only the pinned NInfer model in the OCR image. Start only the lightweight HTTP service at container startup. Launch NInfer and load the model on demand, serialize requests, and stop/reap its process group after the configured `idle_timeout_seconds` (playground default 300 seconds) without pending or running work. Health checks must remain successful while unloaded and must not wake the model or reset the idle clock. Cold loading and inference share the request timeout; timeout/cancellation must not leave native generation running. Remove SuperPoint/LightGlue, their weights, feature caches, thresholds and dedicated matching endpoint. Remove the old
Unlimited-OCR/PP-OCR runtimes and receipt parsers; Git/history may retain evaluation outputs.
The API first localizes/crops the logo and resolves its preserved image Alias, then combines
`[[general]]` and the matching `[[store]]` prompt from `/data/prompt.toml`. Logo localization,
logo comparison, and receipt prompts all live in this one operator TOML file. Unknown stores
use general prompts. Keep thinking enabled, all receipt photos in one bounded request, and
serial inference. Logo matching uses the evaluated concise prompt, opaque reference IDs, 768-pixel crops and at most 15 references per batch; compare all batches and return unknown for conflicting merchant identities. Validate returned IDs against their exact batch; never accept generated store names. The API may strip a whole Markdown fence and project extra metadata fields
out of the model response, auditing removals; it must still reject missing/wrong required
fields, truncation and invalid associations. Never correct amounts or invent balancing items.
Preserve item-discount relationships, multi-photo overlap handling and async jobs. All
processes remain inside two containers communicating over internal REST.
`build_docker.sh` builds both
images as well as Android. Do not recreate separate API and structure containers.

Receipt Master also bundles reviewed merchant logo crops and store names in the API image;
the build installs the default prompt template in `/data/prompt.toml` on first setup.
Fresh database initialization installs this catalog without contacting OCR; existing
catalogs are never overwritten on restart or image upgrade. Keep user-added/edited aliases
in /data. Build only reviewed source-controlled assets, never automatically snapshot the
playground database or full receipt photos into a runtime image. Validate empty-database
matching and preservation of edits/deletions across restarts.


The downloaded APK includes only a build-configured initial backend API origin. It never contains user credentials,
JWTs, refresh tokens, JWT signing secrets or shared client/OCR API keys. The user logs
into Web to download, installs Android with its server address prefilled, and logs into
Android separately with a username and password. On app upgrades, preserve the saved
server URL and device-held login credentials; only fresh installs use the new APK default.
Derive a phone-accessible host and published port for playground; support
`RECEIPT_BACKEND_ENDPOINT=https://actual-domain/` for deployment. Changing the endpoint
invalidates APK build inputs and requires rebuilding and publishing the API image.
If the trial backend uses HTTP, limit the exception to its provisioned endpoint and Android
host policy. An upgrade may continue using an HTTP origin already saved after a successful
login; do not broaden this exception to arbitrary newly entered HTTP hosts.

Initial APK, update manifest and immutable APK URLs require a valid user JWT or Web
session cookie. Unauthenticated downloads return 401; accounts that must change their
password return 403. Preserve authenticated GET, HEAD and Range, and `private, no-store`
responses. Android updates send the current user JWT to the currently authenticated server origin,
without cross-origin redirects. Verify downloaded bytes match the built artifact and
that revoked sessions lose download access. Downloads remain available while OCR loads.

When changing identity, configuration initialization, or process startup/shutdown, read
[entrypoint contracts](references/entrypoints.md). Ordinary image dependency edits do not
require redesigning the entrypoint.

Validate the changed script/config/image path and relevant startup behavior. Complete local
checks within the authorized task scope. Build instructions alone do not authorize deploying,
publishing, or modifying an existing data volume.

### Receipt Master Android client updates

- `backend_api` serves both the initial APK and all client updates. Include a visible
  "Check for updates" button in Android settings; compare installed version code and
  SHA256 with the API's update manifest and select the matching device ABI APK.
- Provision an initial server origin (scheme, phone-accessible host and published port)
  at build time; Android cannot recover the browser's download URL after ordinary APK
  installation. A new install prefills this value. An upgrade preserves the user's saved
  server URL and device-held credentials, even if a later APK has a different default.
  Never use Docker service DNS, the container's internal port, `0.0.0.0`, or the phone's
  `localhost` as the initial destination.
- Fetch `/android-update.json` and `/updates/<sha256>.apk` from the currently
  authenticated server URL, with the current JWT, and reject cross-origin redirects.
  Do not introduce an external APK hosting service. A changed build default requires
  rebuilding the APK and API image, but must not silently replace saved server settings.
- `build_android.sh` owns persistent version state, source/config/signing fingerprints and
  final APK hashes. Keep the version for identical inputs and artifacts; increment it when
  inputs or APK bytes change. Serialize builds and commit state only after success.
- Publish the current version's hash-addressed APKs before atomically publishing
  `android-update.json`, then delete every older APK from development, staging and
  update output directories. Keep only the latest version for each supported ABI;
  API images contain only these current updates and the fixed initial-install APK.
  Do not archive older APKs or add a retention/grace period. Failed builds preserve
  the last valid release; preserve signing keys and version state.
- Validate size, SHA256, package identity and signing certificate before invoking Android's
  installer. Never downgrade. Installation remains subject to Android user confirmation.
- Verify unchanged-build stability, changed-input/hash version increments, hosted artifact
  equality, and the update UI/install path. Preserve release state and signing keys.

### Receipt Master v2 persistent data

- Android and iOS use the unified `backend_api` for every business read/write, report,
  recognition job and backup. The backend owns SQLite and original receipt photos.
  No client business SQLite or legacy test-data import is required.
- `backend_api` uses `/data` as its persistent data root; Compose binds repository
  `playground/backend_api/` to `/data` read-write. Keep it out of images, Git and build cleanup.
  Do not confuse this host-relative directory with the host filesystem's `/data`.
- Receipt Master `run_playground.sh` exports the invoking user's numeric `PUID` and `PGID`
  to both services. Each entrypoint validates the IDs and drops privileges before starting
  the service. API startup adjusts ownership only inside its `/data` mount so files created
  by the service remain owned by the host user; OCR's separate config mount is read-only.
- Retain original images and their receipt relationships; rotations produce separate
  files with correct hashes. Mobile devices retain unacknowledged uploads for retry.
- Keep `/artifacts` inside the API image and read-only to the service user; no artifact
  mounts. Keep APK/update URLs on the same published API origin.
  The OCR container mounts its separate `playground/backend_ocr/` read-only at `/data`,
  has no business data mount, and talks only over the internal network.
- Data/API availability must survive model failure. Verify persistence across container
  recreation, authenticated media access, concurrency conflicts and interrupted uploads.


For Receipt Master configuration, directory layout, key ownership, prompt ownership,
bootstrap, and runtime UID/GID, follow the complete
[data and configuration contract](references/receipt-master-config.md). In particular,
each service reads its own `/data/config.toml` from the required `--data-dir /data`;
API config has no `data_dir`, model settings, pricing, budget, or Logo-enable flag.
Do not package a model choice in the mobile bootstrap or expose OCR/pricing/budget
controls in Android/iOS. Clients authenticate, capture/upload, submit jobs or edit
actions, and display server results. Weight conversions, receipt calculations,
calendar period boundaries and statistics belong in `backend_api`; retain the
user-requested shared display-unit preference.
