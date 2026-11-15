# Audiobookshelf

**Host:** docker-prod-01 (192.168.30.11)  
**Status:** Active  
**Last Updated:** 2026-11-14

---

## Overview

Audiobookshelf (ABS) is the audiobook server.

Users request audiobooks through Shelfmark, which can submit audiobook torrents to qBittorrent. Once downloaded, Shelfmark imports the completed content into the ABS library, using hardlinks where supported.

ABS indexes the imported content and makes it available for playback.

qBitrr manages torrent seeding and cleanup independently of Audiobookshelf.

**Important:** Audiobook torrents do not seed indefinitely by default. The verified public-torrent policy is approximately one hour, with separate protection configured for MyAnonamouse private-tracker torrents.

---

## Deployment

| Field | Value |
|---|---|
| **Image** | `ghcr.io/advplyr/audiobookshelf:latest` |
| **Compose** | `/opt/stacks/books/compose.yaml` |
| **Appdata** | `/opt/appdata/audiobookshelf/` |
| **URL** | `https://audiobooks.giohosted.com` |
| **Internal port** | 80 |
| **PUID/PGID** | 2000:2000 |
| **TZ** | America/Chicago |

---

## Volumes

| Host Path | Container Path | Purpose |
|---|---|---|
| `/data/media/books/audiobooks` | `/audiobooks` | Audiobook library |
| `/opt/appdata/audiobookshelf/config` | `/config` | ABS config and database |
| `/opt/appdata/audiobookshelf/metadata` | `/metadata` | Cover art, metadata cache |

The library storage is part of the shared `/data` NFS architecture on docker-prod-01.

The single-share arrangement supports the intended hardlink workflow between torrent downloads and imported library files.

---

## Network & Access

| Field | Value |
|---|---|
| **Docker network** | `proxy` (external, shared with Traefik) |
| **Traefik router** | `audiobookshelf` |
| **Middleware** | No `local-only@docker` — intentionally externally accessible |
| **External access** | Cloudflare Tunnel at `audiobooks.giohosted.com` |
| **Cloudflare Access** | Enabled |
| **Browser access** | Cloudflare Access with Authentik OIDC |
| **Mobile access** | Cloudflare Access service token plus application authentication |

The external-access design was reviewed and functionally tested on November 14, 2026.

The absence of the Traefik `local-only@docker` middleware is intentional because remote browser and mobile access are required.

### Verified Browser Access

In a fresh unauthenticated browser session, the operator visited:

`https://audiobooks.giohosted.com`

Cloudflare Access presented its login page, offering Authentik OIDC.

**Result:** The tested unauthenticated browser could not proceed directly to the Audiobookshelf interface without encountering Cloudflare Access.

### Cloudflare Access Policies

The Audiobookshelf Access application has three configured policies:

| Policy | Action | Purpose |
|---|---|---|
| `abs-mobile-token` | Service Auth | Authenticate clients presenting the valid `abs-mobile` token |
| `allow-users-authentik` | Allow | Permit eligible browser users through Authentik |
| `block-all-others` | Block | Deny requests that do not satisfy the permitted authentication paths |

These policies were inspected in the Cloudflare dashboard during the November 14 security review.

The `allow-users-authentik` policy is shared with Shelfmark.

The reviewed Allow policy uses:

- An Authentik `users` entitlement or explicitly permitted email addresses.
- A United States geographic requirement.

The operator confirmed that the Authentik `users` group contains only the two intended personal accounts.

The exact mapping between that group and the entitlement returned to Cloudflare was not independently tested.

**Important:** The geographic requirement belongs to the shared browser Allow policy. Do not assume it also restricts service-token-authenticated mobile requests.

---

## Authentication

- **Method:** OIDC via Authentik
- **Authentik application:** `Audiobookshelf`
- **Authentik provider:** Carried forward from v2 backup — existing configuration and accounts restored
- **Admin group:** `admins`
- **Auto-provision users:** Enabled
- **Local admin:** Retained as break-glass — do not delete

The operator confirmed that MFA is configured in Authentik.

The security review did not independently establish MFA enforcement for every account or every possible authentication path.

### Authentik Redirect URIs

The following redirect URIs are documented as registered on the Authentik provider:

- `https://audiobooks.giohosted.com/audiobookshelf/auth/openid/callback` (strict)
- `https://audiobooks.giohosted.com/audiobookshelf/auth/openid/mobile-redirect` (strict)
- `audiobooth://oauth` (strict) — required for iOS/Android mobile app OIDC login

These are retained from the established authentication configuration documentation.

### ABS Server Redirect URIs

`audiobooth://oauth` must also be added in ABS itself under **Settings → Authentication → OpenID Connect → Mobile Redirect URIs**.

Both Authentik and ABS must have the required redirect URI for the documented mobile SSO flow.

---

## Mobile App External Access (CF Access Service Token)

The ABS mobile app uses a Cloudflare Access service token to authenticate non-interactive requests without requiring the browser-based Cloudflare login challenge.

**CF Access application:** `Audiobookshelf` (`audiobooks.giohosted.com`)  
**Service token name:** `abs-mobile`  
**Policy:** `abs-mobile-token`  
**Action:** Service Auth  
**Token location:** Password manager (Client ID and Client Secret)

The service token allows clients possessing the valid credentials to satisfy the Cloudflare Access Service Auth policy.

It does **not** automatically authenticate a user inside Audiobookshelf. Application-level authentication and authorization remain separate.

### CF Access Policy Order

| Order | Policy | Action |
|---|---|---|
| 1 | `abs-mobile-token` | SERVICE AUTH — valid service token |
| 2 | `allow-users-authentik` | ALLOW — eligible Authentik users |
| 3 | `block-all-others` | BLOCK — other requests |

The existing policy documentation notes that Service Auth policies have separate evaluation behavior from interactive Allow policies.

The service-token rule must remain scoped to the intended Audiobookshelf Access application.

### Mobile App Configuration

In the ABS iOS/Android app, open the server connection settings and configure the following **Custom Headers**:

| Header Name | Value |
|---|---|
| `CF-Access-Client-Id` | Client ID from the `abs-mobile` service token |
| `CF-Access-Client-Secret` | Client Secret from the `abs-mobile` service token |

Never place these values in Git, documentation, screenshots, or application logs.

### Functional Verification — 2026-11-14

The Cloudflare Access policy configuration was reviewed.

The operator then:

1. Confirmed unauthenticated browser access displayed the Cloudflare Access login screen.
2. Used the Audiobookshelf mobile application through an external cellular connection.
3. Confirmed the mobile application worked and audiobook playback was available.

**Result:** Both the browser-protected access path and the externally accessible mobile application worked during the test.

The review did not independently inspect individual service-token HTTP headers, token expiration, or token rotation history.

### Security Considerations

- Protect the `abs-mobile` credentials as application secrets.
- Keep the Service Auth policy scoped to Audiobookshelf.
- Rotate or revoke the token if it is suspected to be compromised.
- Review token expiration and rotation arrangements periodically.
- Do not replace the Service Auth policy with a broad Bypass rule.
- Verify both browser and mobile access after changing Cloudflare Access policies.
- Keep Audiobookshelf's application authentication enabled.
- Consider the service token a sensitive credential even though Cloudflare Access is not itself the application's user authorization layer.

---

## Library Configuration

| Library | Path |
|---|---|
| Audiobooks | `/audiobooks` |

> **Note:** After restoring from the v2 backup, the library path was still pointing to the old path (`/books/audiobooks`). It was manually repointed to `/audiobooks` in ABS → Settings → Libraries.

---

## Audiobook Download Workflow

The current torrent-based workflow is:

1. User requests an audiobook in Shelfmark.
2. Shelfmark submits a torrent to qBittorrent with category `audiobooks`.
3. qBittorrent downloads the content to `/data/downloads/books/audiobooks/downloads`.
4. Shelfmark imports the completed audiobook using a hardlink into `/audiobooks`, mapped to `/data/media/books/audiobooks`.
5. ABS detects the imported content and makes it available for playback.
6. qBittorrent continues seeding the torrent according to the applicable settings.
7. qBitrr applies the configured category and tracker-specific seeding policy.
8. When the applicable cleanup conditions are satisfied, qBitrr can remove the torrent and its download-side data.
9. The separate hardlinked audiobook library entry remains available when import completed successfully.

### Public Torrent Seeding

The verified qBitrr category-level policy uses:

```toml
[qBit.CategorySeeding]
MaxUploadRatio = -1
MaxSeedingTime = 3600
RemoveTorrent = 2
```

This represents an intended default public-torrent seeding period of approximately one hour, with automatic cleanup managed by qBitrr.

It replaces the original documentation's claim that audiobook torrents seed permanently.

### Private Tracker — MyAnonamouse

MyAnonamouse (MAM) torrents have a separate configured policy:

```toml
[[qBit.Trackers]]
Name = "MyAnonamouse"
URI = "myanonamouse.net"
Priority = 10
MaxUploadRatio = 1.0
MaxSeedingTime = 1209600
HitAndRunMode = "and"
MinSeedRatio = 1.0
MinSeedingTimeDays = 14
TrackerUpdateBuffer = 3600
AddTrackerIfMissing = false
RemoveIfExists = false
```

The intended policy requires both:

- At least 14 days of seeding.
- At least a 1.0 upload ratio.

**Verification status:** The tracker-specific configuration has been reviewed, but the required MAM behavior has not been verified end-to-end using a real private-tracker torrent.

Do not treat the intended configuration as proof of tracker compliance.

See `services/torrent/qbitrr.md` for authoritative settings and any subsequent changes.

### Verified Public Audiobook Workflow — November 2026

A public audiobook torrent, *Threshing Day*, was used to test the complete workflow.

The following were confirmed:

- Shelfmark submitted the audiobook torrent.
- qBittorrent downloaded it and began seeding.
- Shelfmark imported the audiobook to the ABS library.
- Audiobookshelf successfully played the audiobook.
- qBitrr automatically removed the public torrent after the configured seeding period.
- The audiobook remained present and playable after cleanup.

**Result:** Public audiobook import, playback, automatic torrent cleanup, and retention were verified.

This does not establish the untested MAM-specific seeding requirements.

---

## Backup & Restore

The established documentation records:

- **Appdata backup destination:** `/mnt/user/backups/` through the Phase 5 rsync workflow.
- **v2 backup location:** `/mnt/user/backups/docker-v2/appdata/books-stack/audiobookshelf/`
- **Restore procedure:** Copy `config/` and `metadata/` to `/opt/appdata/audiobookshelf/`, set ownership to `2000:2000`, start the container, and verify the library path.

These are documented recovery arrangements, not evidence of a freshly tested Audiobookshelf restoration during the November 14 security review.

Before restoring:

1. Verify a suitable backup exists.
2. Preserve the current configuration if needed for troubleshooting.
3. Restore compatible configuration and metadata.
4. Confirm ownership and permissions.
5. Verify the `/audiobooks` mount and library path.
6. Validate application authentication.
7. Test browser and mobile playback.

Cloudflare Access policies and service tokens are managed separately from Audiobookshelf appdata. A Docker appdata restore alone does not recreate all external-access configuration.

---

## Notes

- No `local-only@docker` middleware — ABS is externally accessible by design.
- Browser access is protected by Cloudflare Access with Authentik OIDC.
- Mobile access uses a scoped Cloudflare Access Service Auth token.
- The browser authentication challenge and external mobile playback were verified on 2026-11-14.
- The service token must be protected and never committed to Git.
- The shared browser policy includes a US geographic requirement; the Service Auth path is separately evaluated.
- Audiobookshelf retains application authentication independently of Cloudflare Access.
- Public audiobook torrents use qBitrr-managed seeding and cleanup, not indefinite seeding.
- A public audiobook remained playable after its torrent was removed.
- MAM-specific seeding protection is configured but has not been verified end-to-end.
- qBittorrent's previous global 60-minute stop rule was disabled to avoid conflicts with private-tracker policies.
- Podcasts are not currently used — audiobooks only.

---

## Related Documentation

- `services/infra/cloudflared.md` — Cloudflare Tunnel, Access applications, and external authentication.
- `services/infra/traefik.md` — Reverse proxy and external router behavior.
- `services/books/shelfmark.md` — Ebook and audiobook discovery, download, and import.
- `services/torrent/qbittorrent.md` — Torrent downloading and storage architecture.
- `services/torrent/qbitrr.md` — Seeding policies, tracker overrides, and cleanup.
- `runbooks/updating-containers.md` — Docker updates and recovery procedures.