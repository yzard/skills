# JWT authentication and TOML configuration

Read when implementing or changing authentication or configuration. Use existing components
for an unrelated handler change instead of starting an authentication/configuration migration.

## JWT

- Receipt Master uses separate Bearer API keys for its client and OCR service rather than JWT;
  keep that established contract when working on `backend_api`.
- For JWT-based services, user APIs use Bearer JWTs and a shared typed authentication extractor. Reuse the project's
  password hashing, token issuance, refresh, and revocation contracts.
- Verify signatures with the configured key and an explicit allowed algorithm; never trust
  the token's claimed algorithm or treat decoding as verification. Validate expiration and
  the issuer/audience required by the service's token contract.
- Return the established unauthenticated error for absent/invalid/expired credentials.
  Resource authorization is a separate check; a valid token alone does not grant access.
- Keep keys in typed startup configuration; do not hardcode secrets, log tokens, or ship
  placeholder credentials. Service API keys and protocol authentication remain explicit
  separate mechanisms, not substitutes for user JWTs.
- Changed auth paths need relevant tests for valid, invalid, expired, and unauthorized access,
  plus refresh/revocation when those behaviors change.

## TOML

Follow [background service credentials](../../general-coding/SKILL.md#background-service-credentials):
all service secrets/passwords are inline TOML values, without credential-file references or
environment/CLI secret overrides.

- Require the repository's explicit CLI configuration input, parse TOML into typed settings,
  and validate required values and cross-field invariants before serving. Receipt Master
  requires `--data-dir ABSOLUTE_DIRECTORY` and loads its fixed `config.toml` there;
  other projects may use `-c|--config PATH`. Do not fall back to another file or YAML schema.
- Add settings to the source-owned TOML/config types and update affected templates/callers.
  Configuration reaches business code explicitly; noncredential loader-only overrides may remain; credentials are embedded in `config.toml` exclusively.
- Keep config initialization explicit and preserve an existing operator file. Runtime updates
  use the repository's config manager and atomic persistence/publication contract.
- Test missing/malformed/invalid config and changed initialization/update behavior. Reuse
  the existing manager for unchanged routes without rebuilding configuration infrastructure.
