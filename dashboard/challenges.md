# Key challenges when MFE‑A encodes a bookmark for Dashboard

- **Contract and versioning**
  - Different releases must agree on schema; fields come/go.
  - Mitigate: add `v` (schema version), use additive changes, ignore unknowns.

- **Deterministic serialization**
  - JSON key order, whitespace, and transient fields change the payload and signatures.
  - Mitigate: canonicalize JSON (stable key order), exclude ephemeral fields.

- **Cross‑MFE semantics**
  - IDs/types in MFE‑A may be unknown to Dashboard; widget/feature flags may differ.
  - Mitigate: use globally resolvable IDs and type registries; graceful fallbacks.

- **Time, locale, and precision**
  - Relative dates (“last‑30‑days”), timezones, currency/locale formatting, float precision.
  - Mitigate: encode absolute timestamps + timezone; store raw numbers, format on display.

- **Size limits**
  - URLs often break around 2–8 KB (browsers, proxies, servers).
  - Mitigate: deflate/gzip then Base64url; or store server‑side and pass a short `bookmarkId`.

- **URL safety and collisions**
  - `+`/`/`/`=` in Base64, reserved characters, query param name clashes.
  - Mitigate: Base64url, or wrap with `encodeURIComponent`; namespace params (e.g., `dash_state`).

- **Security and privacy**
  - Base64 is not secrecy; URLs leak via logs, Referer headers, and browser history.
  - Tampering risk if the Dashboard trusts client‑provided state.
  - Mitigate: do not include secrets/PII; sign payloads (HMAC) for integrity; encrypt if sensitive; enforce authZ when resolving `bookmarkId`.

- **Persistence model**
  - Where the bookmark lives (URL vs server), TTL, migration, and multi‑tenant scoping.
  - Mitigate: server persistence with owner/tenant, `createdBy`, `updatedAt`, soft‑delete, migrations.

- **Compatibility and capability negotiation**
  - Dashboard may not support features encoded by MFE‑A.
  - Mitigate: include `capabilities`/`minDashboardVersion`; ignore/placeholder unknown widgets.

- **Error handling and resilience**
  - Corrupt/expired payloads, missing resources, partial loads.
  - Mitigate: strict validation with helpful errors; default safe state; telemetry.

- **Performance**
  - Compression/decoding cost on load; large state delaying first paint.
  - Mitigate: keep state minimal; lazy load widget configs; precompute heavy filters server‑side.

- **Governance and testing**
  - Drift between teams/repos.
  - Mitigate: shared schema package, contract tests, sample fixtures, and changelog.

- **Auditing and observability**
  - Need traceability of who created/used a bookmark.
  - Mitigate: audit fields and server logs without logging sensitive payloads.

### Minimal, robust envelope (if embedding)

- Include: `v`, `type`, `createdAt`, `tz`, `capabilities`, and a `state` object.
- Use: canonical JSON → deflate → Base64url; add `sig=HMAC(secret, payload)` for integrity.

### Preferable alternative (server‑stored)

- Persist bookmark server‑side with owner/tenant and version; share `bookmarkId` in the URL.
- Dashboard fetches, validates, and renders; easier to migrate and control access.

- In short: the hard parts are contract/versioning, deterministic serialization, safety (size, URL, privacy), and cross‑MFE compatibility; mitigate with versioned, canonical, signed payloads—or store server‑side and reference by ID.
