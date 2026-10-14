# Receipt Master build, publication and embedded APK contracts

Use for `build_docker.sh`, its registry options, Android artifact packaging, or
Compose artifact mounts. These are Receipt Master requirements, not defaults for
unrelated projects.

## Build entrypoints and order

- `build_docker.sh` is the combined entrypoint. No arguments build/test locally;
  `run_playground.sh` calls that mode and remains a foreground, no-subcommand runner.
- Keep mandatory Web, backend, Flutter, Android and release checks in the build
  workflow or its required Docker stages. A failure stops the build.
- Call `build_android.sh` to prepare the mobile workspace and compile/sign/check
  APKs inside Docker. Do not compile Android on the host or skip this step when
  publishing. Valid Docker-cached builds are acceptable.
- Android publishes artifacts to `build/mobile/` before the API runtime image is
  built. Build both independent backend images: `backend_api` and `backend_ocr`.
  Capture the build date once at script start in the build host's timezone as
  `YYYYMMDD`, and build `receipt-master-backend-api:YYYYMMDD` and
  `receipt-master-backend-ocr:YYYYMMDD`. Use this same date for both images and
  publication even if the build crosses midnight. After all checks pass, set
  each image's `:latest` alias to its corresponding tested image ID. Compose,
  playground helper containers and the smoke tool's default use `:latest`.
- `build_ios.sh` remains independent, requires macOS/Xcode, and is not called by
  the combined Docker build.

## Registry CLI

```bash
./build_docker.sh
./build_docker.sh --publish dockerhub --username YOUR_USERNAME
./build_docker.sh --publish github --username YOUR_USERNAME
```

- `--publish` selects only `dockerhub` or `github`, resolving to `docker.io` or
  `ghcr.io`. `github` means GitHub Container Registry, not a GitHub release asset.
- `--username` is required for publication and selects the registry user or
  organization namespace, normalized to lowercase. There is no custom `--tag`.
  For each backend, publish exactly the captured `YYYYMMDD` tag and the `latest`
  alias, both pointing to the same image. Do not use `--all-tags` to push unrelated
  historical builds. Successful builds on the same day replace that day's tag
  and its `latest` alias together.
- Validate missing/invalid/duplicate options before starting build work.
  `--username` requires `--publish`; `--help` exits without building.
- The operator logs in with `docker login docker.io` or `docker login ghcr.io`.
  Reuse those credentials. Do not add password/token CLI arguments, credentials
  to source/images, or secret values to logs. The namespace can be an authorized
  organization distinct from the login account.

Each selected registry receives these two repositories:

```text
docker.io/your_username/receipt-master-backend-api:YYYYMMDD
docker.io/your_username/receipt-master-backend-api:latest
docker.io/your_username/receipt-master-backend-ocr:YYYYMMDD
docker.io/your_username/receipt-master-backend-ocr:latest
ghcr.io/your_username/receipt-master-backend-api:YYYYMMDD
ghcr.io/your_username/receipt-master-backend-api:latest
ghcr.io/your_username/receipt-master-backend-ocr:YYYYMMDD
ghcr.io/your_username/receipt-master-backend-ocr:latest
```

Record each built image ID. Run the API/container smoke check against that exact
API image ID, then tag and push those built/tested IDs. `latest` is another tag
for the same image, not a separately rebuilt image. Do not resolve mutable date
or `latest` tags again at publication time. Builds and all checks, including HTTP
artifact verification, finish before changing local `latest` or the first remote
push. First push both dated backend tags, then both corresponding `latest` tags
(four pushes total). A push failure exits nonzero and stops further pushes;
do not claim all tags succeeded or silently
delete an already published image. Do not initiate a remote publication merely to
test or document this feature.

Keep publish regression tests in `tests/docker/test_build_docker.py` and run them
through the combined build. Use command substitutes to exercise both registries,
local-only mode, single date capture across midnight, identical image IDs for date
and latest, four correctly ordered pushes, validation errors, build/check failures
that leave latest unchanged, and date/latest push failures without network
mutations. If the Android test stage discovers that directory, include
the actual `build_docker.sh` as a test input in its Docker context/COPY rules.

## APK files belong to the API image

Copy these completed build outputs into the runtime `backend_api` image:

| Host build output | Image path |
| --- | --- |
| `build/mobile/receipt_master.apk` | `/artifacts/receipt_master.apk` |
| `build/mobile/android-update.json` | `/artifacts/android-update.json` |
| `build/mobile/updates/*.apk` | `/artifacts/updates/*.apk` |

The API's Dockerfile-specific ignore file must allow these inputs. Exclude
signing keys, secret-bearing TOML, databases, receipt photos, debug APKs and
unrelated build files from the artifact COPY. Keep only the latest Android version,
including during development: the current manifest's ABI packages, the latest
named/debug development outputs and the fixed install APK. After verifying the
new APKs and switching the manifest, delete all superseded APKs from
`build/mobile/updates/`, `build/mobile/` and `build/android-staging/`. Do not archive
old APKs or retain a download grace period. A rejected/failed candidate must keep
the last valid manifest and APKs. Preserve signing keys and monotonic version state.
Old update URLs return 404 after deployment; clients obtain the current manifest
for a new update attempt.

APK/manifest files are readable by the service UID (0644); artifact directories
need traversal permissions (0755). Docker `COPY --chmod=0644` can also affect
new destination directories, so explicitly preserve directory permissions.
The non-root service must not be able to overwrite these image artifacts.

Compose mounts only `playground/backend_api/` at the API's writable `/data`;
never mount `build/mobile` or another host APK directory at `/artifacts`.
The OCR container keeps its separate config `/data` and its image-bundled model.
No business database, config or signing secret is packaged in the API image.
Keep fixed APK/update URLs on the API's published origin, authenticated as
described in `SKILL.md`.

Updating host APK output alone cannot change a running API image. Publishing new
APKs requires rebuilding and deploying the API image. Configure the fresh-install device-facing
origin before Android compilation via `RECEIPT_BACKEND_ENDPOINT`; the registry
username/host is not the backend service address. Domain/port changes require
rebuilding the APK and image. Existing installs keep their saved server URL and
device-held credentials, and fetch updates from that authenticated server.
Usernames, passwords and user sessions are never transferred from Web into the APK.

## Verify with a self-contained image

`tests/web/http_smoke.py` starts the built API image with disposable `/data`,
without any artifact mount. Inspect actual mounts and validate logged-in APK
bytes against the signed build output, the embedded manifest against generated
metadata, and every architecture's update size and SHA256. Assert the update
directory's APK set equals the current manifest exactly, on the host and in the
image, and removed update URLs return 404 for authenticated users. Check forced-password
and anonymous download rejection, normal-user access, and revoked-session
rejection. Existing authenticated HEAD/Range coverage must remain passing.

Complete the combined local build when changing executable build/packaging
behavior. Report local/offline validation separately from real registry pushes;
documentation-only skill edits do not require another build or push.
