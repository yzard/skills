# Entrypoint contracts

Read when changing container identity, config generation, or process lifecycle.

- Support the established `PUID`/`PGID`/`UMASK`/`TZ` interface. Validate values and handle
  existing user/group IDs; scope ownership changes to application-owned mounted paths.
  Receipt Master also uses `PUID`/`PGID` throughout its runtime and build contracts; see
  [its data and configuration contract](receipt-master-config.md).
- Fail on required setup errors instead of masking them with blanket success fallbacks.
  Use the image's privilege-drop mechanism and `exec` the application for signal delivery.
- Invoke the executable's actual CLI/TOML/YAML/JSON contract. Generate initial config only
  when absent and only through the project's source-owned initialization path; preserve
  persisted operator settings on subsequent starts.
- Preserve service-specific mounted data/temp paths. Do not repurpose host temporary-path
  conventions as container runtime settings.
- If multiple processes run, define supervision, cancellation, bounded shutdown, and child
  reaping. Do not background a process and exec an unrelated parent without lifecycle ownership.
- Validate changed startup behavior with disposable data: configured identity/permissions,
  malformed settings, config preservation, signal handling, and child cleanup as relevant.
  Do not exercise these cases against an existing user volume.
