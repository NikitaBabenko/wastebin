# VibeNest pilot review

**Outcome: rejected at static review for the current pilot scope, 26 October 2026.**
This is an audit fork, not a deployable VibeNest integration. No live demo exists.
The Free target (256 MB RAM, 0.5 platform vCPU, 4 GB nominal storage, trial) is
unverified. Do not expose the upstream server directly to the internet.

## Source and runtime identity

- Upstream reviewed: `matze/wastebin`, master
  `c0ec9687ba41e8806654f34841e9a739f0264ced`.
- Fork: <https://github.com/NikitaBabenko/wastebin>.
- Latest release inspected: 3.9.0, source
  `c829df9d116368459541b145350ea319d7a04b26`, published 15 October 2026.
- Upstream documents `quxfoo/wastebin` as its official image repository.
  At inspection, both `3.9.0` and `latest` resolve to index
  `sha256:d441b009827b896b67d7df5d7bf849cc5cff970c505cb50d72c946c3d3fd0a5e`.
  The linux/amd64 descriptor is
  `sha256:3ee18e1ee23d490770cdecd3a97283f333812f4818a35265d5764b81d8a6f4c0`.
- These digests establish immutable registry identity, not source provenance.
  The inspected image config has no source/revision labels; no attestation or
  reproducible source-to-binary comparison was established. This image is **not
  approved**. The release source predates the burn-extension XSS fix `b703cb7`
  and open-redirect fix `c0ec968`. Fixes in this fork's source cannot be assumed
  to exist in the released image.
- Source, tests, Dockerfile and MIT license remain intact. No Rust compilation,
  container build or application execution was performed. Fork Actions are disabled.

## Why this needs more than a proxy

Upstream explicitly warns that authentication and comprehensive DoS mitigation
are absent. Blanket authentication and request limits are necessary, but would
not repair the following application semantics among authenticated team members:

- An owner-token handoff becomes the first uid for a visitor without a cookie;
  subsequent form inserts use that first uid. This lets the token's existing
  holder delete those later pastes. See [PR 229](https://github.com/matze/wastebin/pull/229).
- QR and burn pages use a plaintext title from metadata before checking an
  encryption password. See [PR 232](https://github.com/matze/wastebin/pull/232).
- Raw/download GET handlers call the database read that deletes burn-after-reading
  entries, without an interstitial. See [PR 233](https://github.com/matze/wastebin/pull/233).
- GET/HEAD form extraction accepts query parameters, including password and burn
  confirmation. See [PR 227](https://github.com/matze/wastebin/pull/227).

These are source-level observations, not executed exploit tests. PRs 224–234
were all open, unmerged, and their heads were not ancestors of the reviewed SHA.
Other claims include per-entry encryption salts, unsafe link schemes, retention,
filename/terminal rendering and expensive nested Markdown. Their exact runtime
impact, including the reported CPU duration, has not been independently measured.
No blanket vulnerability clearance is claimed.

Stripping owner handoffs or disabling encrypted, raw and Markdown views would
remove parts of the intended acceptance scenario. Maintaining multiple security
patches across handlers, storage and rendering exceeds this pilot's minimal
configuration scope. No mass security fork was attempted. This is a rejection
of the current pilot revision, not an allegation of malicious software or a
permanent judgment on the project.

## Preparation requirements for a future, separately reviewed revision

These are design requirements, **not implemented deployment instructions**.

1. Review the resolved security changes and establish source/image provenance for
   an immutable official runtime. Do not silently substitute the release image
   for reviewed master. Use an authorized build executor if a new build is needed.
2. Authenticate every content and API route at a reverse proxy before exposure.
   Keep the wastebin port internal with no bypass route. Any health exception
   must return only a static readiness result. Missing credentials must fail closed.
   Use HTTPS, bounded request rate/concurrency/body/timeouts, and verified proxy
   identity; do not trust arbitrary forwarded client-IP headers. A small proxy
   can solve perimeter access but cannot repair the issues listed above.
3. Configure SQLite explicitly: `WASTEBIN_DATABASE_PATH=/data/state.db`, with a
   persistent writable volume at `/data` for uid/gid 10001. The upstream default
   is `:memory:` and loses data at replacement. Its scratch image has no shell
   and no `TMPDIR`; configure a writable temporary directory if migrations need
   one. The volume and permissions must be verified through the supported path.
4. Set `WASTEBIN_BASE_URL` to the final HTTPS origin, without a path prefix. The
   upstream default derives from the container hostname and is unsuitable for
   public QR links. Preserve secure ownership cookies across HTTPS proxying.
5. Supply a persistent `WASTEBIN_SIGNING_KEY` of at least **64 UTF-8 bytes**. Use
   a cryptographically random value (for example, 64 random bytes encoded as
   128 hex characters) saved only in protected local storage and runtime secrets.
   Do not print it, put it in command arguments, URLs, Git, build args or reports.
   Without a configured key, upstream generates one at startup, invalidating
   deletion ownership cookies at replacement. VibeNest's inspected generic
   generator produces 48 ASCII hex characters and is too short for this setting.
   Validate the supplied byte length; never rely on automatic generation here.
6. Persist a randomly chosen instance salt in protected runtime ENV when retaining
   the current encryption scheme. `WASTEBIN_PASSWORD_SALT` defaults to `somesalt`.
   An instance-specific salt does not provide per-entry salts or resolve PR 224;
   changing it can make existing encrypted data unreadable.
7. Bound memory inputs. Upstream defaults are 1 MiB body, 128 rendered cache
   entries and a five-second HTTP timeout. A future trial can start conservatively
   at 16 KiB body and four cache entries, with a matching proxy body cap; these
   values are unmeasured. Cache size must be positive; zero is not a disable switch.
   Entry count is not a cache-byte cap, and HTTP timeout does not prove blocking
   rendering/Argon2 work has stopped. Current Argon2 parameters request 64 MiB per
   derivation; concurrency must fit the total 256 MiB envelope including proxy.
8. VibeNest can classify a plain `BASE_URL` as the public origin and an internal
   `ADMIN_PASSWORD` as a generated secret, if an eventual adapter deliberately
   consumes them. Current upstream consumes neither alias. Its prefixed
   `WASTEBIN_BASE_URL` and salt need explicit values; its signing key needs the
   length treatment above. Never invent external authentication-provider credentials.

An inert environment worksheet is provided at
[`deploy/vibenest.env.example`](deploy/vibenest.env.example). It provides no
authentication, proxy, volume, resource enforcement or deployment manifest.
Do not deploy it as a complete configuration.

## Acceptance required before enabling the proposed button

Only a coordinator-assigned slot and a newly accepted revision can reopen runtime
work. Recheck the exact project/owner, `IsBlocked`, `SafetyHoldId` and any dossier,
hardware, capacity and shared locks immediately before the ordinary atomic gated
dispatch. A clear read-only snapshot is not launch permission. Never bypass the
gate through Docker, Coolify, SQL, a clone or changed policy.

- Follow the README button through anonymous access, registration/login and
  preserved repository selection. Test a clean project separately from reuse
  with old ENV/volumes. Record all manual steps; do not call configured reuse
  clean one-click deployment.
- Verify public HTTPS, missing/wrong/correct authentication, no direct app-port
  bypass, cookie flags, create/view/delete, cross-user ownership rejection,
  encrypted source/rendered/raw/download/QR behavior and burn confirmation.
  Upstream has no in-place edit endpoint: a revised paste must be created as a
  new entry and the old one deleted. Do not report in-place editing as supported.
- Replace the runtime through the gated path and verify data, encryption and
  ownership-cookie continuity on the same volume with new container identity.
- Measure actual total RAM/CPU caps, startup/idle/peak use, CPU, disk, OOM events,
  restart count and a small bounded load. Translate CPU using the target's
  `targetCpuQuotaScale`; an upgraded voucher is not Free. Distinguish nominal
  4 GB storage, measured usage and enforced hard quota.
- Record exact upstream/fork/managed source/image identities and the first
  actual gate response. Preview and prepared files do not count as acceptance.

No platform defect or resource-fit result was established in this static review.
The proposed deploy URL is
`https://vibenest.net/deploy?repo=https://github.com/NikitaBabenko/wastebin`;
the badge URL is `https://vibenest.net/deploy-to-vibenest.svg`.
