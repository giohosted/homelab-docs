# cloudflared

**Role:** Cloudflare Tunnel — external access to internal services  
**Host:** docker-prod-01 (192.168.30.11)  
**Tunnel name:** homelab-v3  
**Compose:** `/opt/stacks/cloudflared/compose.yaml`  
**Appdata:** None — stateless  
**Last Updated:** 2026-11-14

---

## Deployment

- **Image:** `cloudflare/cloudflared:latest`
- **Compose:** `/opt/stacks/cloudflared/compose.yaml`
- **Network:** `proxy`
- **Tunnel token:** Stored in `/opt/stacks/cloudflared/.env` (gitignored)

## Container Variables

| Variable | Value |
|---|---|
| TUNNEL_TOKEN | See `.env` |

---

## Overview

cloudflared creates an outbound tunnel from docker-prod-01 to Cloudflare's edge network. This allows external access to selected internal services without opening dedicated inbound ports on the UDM-SE firewall for those applications.

External requests travel through Cloudflare and the tunnel to Traefik, which handles hostname-based routing.

cloudflared is not a replacement for Traefik — it works alongside it.

The external request path is:

```text
Internet
  ↓
Cloudflare edge
  ↓
Cloudflare Access (where configured)
  ↓
cloudflared tunnel
  ↓
Traefik (192.168.30.11)
  ↓
Application
```

Traefik handles internal routing whether requests originate from cloudflared or directly from an authorized LAN client.

Cloudflare Access is an optional application-level authentication layer. A published tunnel route is not automatically protected by an Access policy.

---

## Exposed Services

The following inventory reflects the established deployment documentation, updated with the November 14, 2026 external-access review.

| Hostname | Cloudflare Route | CF Access | Notes |
|---|---|---|---|
| `auth.giohosted.com` | `https://192.168.30.11` | No | Authentik — application authentication |
| `audiobooks.giohosted.com` | `https://192.168.30.11` | Yes | Audiobookshelf — Authentik for browsers, Service Auth token for mobile |
| `request.giohosted.com` | `https://192.168.30.11` | No | Seerr — Plex authentication |
| `shelf.giohosted.com` | `https://192.168.30.11` | Yes | Shelfmark — Authentik OIDC through Cloudflare Access |
| `photos.giohosted.com` | Documented inactive | N/A | Immich — temporary wedding access previously removed |

The November 14 review directly verified the Seerr published route, the Shelfmark and Audiobookshelf Access application configurations, and their tested authentication behavior.

The full tunnel inventory, including Immich's inactive status, was not independently re-audited during this review.

### Intended Access Model

- **Seerr:** Externally accessible for authorized Plex users.
- **Shelfmark:** Externally accessible after Cloudflare Access authentication through Authentik.
- **Audiobookshelf:** Externally accessible through Cloudflare Access; mobile clients use a service token.
- **Authentik:** Provides identity services for applications that use it.
- **Immich:** Intended to remain LAN-only outside specifically approved temporary exposure periods.

The three reviewed application routers intentionally omit Traefik's `local-only@docker` middleware because external access is part of their design.

---

## Cloudflare Tunnel Configuration

Published routes are managed in the Cloudflare Zero Trust dashboard under:

**Networks → Tunnels → homelab-v3 → Published application routes**

### Route Settings

The established configuration uses:

- **Service URL:** `https://192.168.30.11`
- **No TLS Verify:** Enabled on the documented routes

The Seerr published route was directly verified in the Cloudflare dashboard on 2026-11-14.

### Why No TLS Verify?

Traefik's certificate is a hostname-based wildcard certificate (`*.giohosted.com`) issued by Let's Encrypt.

The documented cloudflared origin connection uses the IP address `192.168.30.11`.

Because that IP address is not covered by the certificate's hostname identity, ordinary TLS certificate name verification would fail.

The existing deployment therefore uses **No TLS Verify** for this origin connection.

### Security Considerations

The Cloudflare Tunnel is an encrypted transport between cloudflared and Cloudflare's edge.

The cloudflared-to-Traefik connection is on the local Services network.

However, disabling origin TLS verification also disables verification of the origin certificate. A private network reduces exposure but does not eliminate every possible interception or impersonation risk.

**Existing configuration:** No TLS Verify remains enabled.

**Future improvement opportunity:** Review whether cloudflared can validate the Traefik wildcard certificate using an appropriate origin server name and certificate verification settings, without disrupting the established routing architecture.

No origin TLS changes were made during the November 14 audit.

### Why HTTP Causes a Redirect Loop

The documented configuration uses Traefik's HTTP-to-HTTPS redirect.

Using `http://192.168.30.11` as the origin can cause a redirect loop:

1. cloudflared sends HTTP traffic to Traefik on port 80.
2. Traefik redirects the request to HTTPS.
3. The redirected request can be sent through the external hostname.
4. Repeated external-to-origin redirects can prevent the application from loading.

The established deployment uses HTTPS to avoid this behavior.

Do not change the origin protocol without testing the effective redirect and forwarded-header behavior.

---

## Cloudflare Access

Cloudflare Access provides an authentication and authorization layer in front of selected externally published applications.

**Verified application inventory — 2026-11-14:**

- Shelfmark
- Audiobookshelf

No Cloudflare Access application covering Seerr was identified in the reviewed application inventory.

### Shared Browser Policy — `allow-users-authentik`

The Shelfmark and Audiobookshelf Access applications use the shared policy:

`allow-users-authentik`

**Action:** Allow

The reviewed configuration includes:

- Authentik `users` entitlement as an Include condition.
- Explicitly authorized email addresses as another Include condition.
- United States geographic requirement.

Cloudflare Access evaluates the Include conditions as alternatives and also requires the configured geographic condition.

The operator confirmed that the Authentik `users` group contains only two intended personal accounts.

**Verification limitation:** The exact relationship between the Authentik group and the entitlement claim evaluated by Cloudflare was not independently tested.

MFA is configured in Authentik, but universal MFA enforcement across every possible login path was not independently established during this review.

### Shared Deny Policy — `block-all-others`

Both Access applications also contain:

`block-all-others`

**Action:** Block

This policy is intended to deny requests that do not satisfy the configured permitted access paths.

No changes were made to the shared Allow or Block policies during the audit.

### Shelfmark

**Hostname:** `shelf.giohosted.com`  
**Cloudflare Access application:** Shelfmark

The reviewed application contains:

| Policy | Action |
|---|---|
| `allow-users-authentik` | Allow |
| `block-all-others` | Block |

#### Functional Verification

On November 14, the operator tested Shelfmark from a phone and confirmed:

1. Cloudflare Access presented its login screen.
2. Authentik OIDC was the available login method.
3. Shelfmark was not displayed directly to the unauthenticated client.

**Result:** The browser authentication boundary was verified.

The review did not independently test every policy condition, including geographic restrictions or specific entitlement claims.

See `services/books/shelfmark.md`.

### Audiobookshelf

**Hostname:** `audiobooks.giohosted.com`  
**Cloudflare Access application:** Audiobookshelf

The reviewed application contains:

| Policy | Action | Purpose |
|---|---|---|
| `abs-mobile-token` | Service Auth | Authorize clients with the scoped service token |
| `allow-users-authentik` | Allow | Authenticate eligible browser users |
| `block-all-others` | Block | Deny other requests |

#### Browser Authentication Verification

On November 14, a fresh unauthenticated browser session displayed the Cloudflare Access login page and offered Authentik authentication.

**Result:** Browser authentication protection was verified.

#### Mobile Authentication Verification

The operator also tested the Audiobookshelf mobile application through an external cellular connection.

The mobile application connected successfully and audiobook playback worked.

**Result:** The documented mobile access path was functional during the test.

The review did not independently inspect the transmitted service-token headers or perform a token-rotation test.

See `services/books/audiobookshelf.md`.

---

## ABS Mobile App — Service Token Bypass

The Audiobookshelf mobile app uses a Cloudflare Access service token to satisfy a non-interactive Service Auth policy.

This allows mobile requests to avoid the Cloudflare browser login challenge.

- **Service token name:** `abs-mobile`
- **CF Access application:** `Audiobookshelf`
- **Policy:** `abs-mobile-token`
- **Action:** Service Auth
- **Token storage:** Password manager
- **Mobile app headers:** `CF-Access-Client-Id` and `CF-Access-Client-Secret`

The policy is scoped to the Audiobookshelf Access application.

### Security Boundary

The service token authenticates a client to Cloudflare Access.

It does not replace Audiobookshelf's own application authentication and authorization.

The shared browser policy's geographic requirement should not be assumed to apply to the separate Service Auth path.

Keep the service token confidential and review its validity and rotation arrangements periodically.

Do not create a broad anonymous Bypass policy as a replacement for the scoped service token.

---

## Seerr — Plex Authentication Without Cloudflare Access

**Hostname:** `request.giohosted.com`

Seerr is intentionally accessible through Cloudflare Tunnel without a Cloudflare Access application.

Access is controlled by Seerr's Plex authentication integration.

### Verified Configuration — 2026-11-14

- Cloudflare Tunnel route exists and targets `https://192.168.30.11`.
- No matching Cloudflare Access application was present in the reviewed application list.
- An incognito browser session was directed to Seerr's `/login` page.
- Plex Sign-In is enabled.
- New Plex Sign-In is enabled.
- Local Sign-In was disabled during the audit.
- Plex login continued working after Local Sign-In was disabled.
- Default administrative and user-management permissions were restricted.

The operator does not use local-password accounts for Seerr.

Only authorized Plex users are intended to receive Seerr access.

**Authentik OIDC is not configured for Seerr.**

### Security Decision

No additional Cloudflare Access policy or Traefik `local-only` middleware was introduced.

The existing Plex authentication model matches the operator's intended access workflow.

If the permitted Plex user population, application requirements, or threat model changes, reassess whether an additional Cloudflare Access layer is appropriate.

See `services/media/seerr.md`.

---

## Immich — Temporary Tunnel for Wedding (October)

Immich is LAN-only by default.

The established documentation records a temporary public route for `photos.giohosted.com`, used for the October wedding event and subsequently removed.

**Traefik file-provider route:** `/opt/appdata/traefik/config/immich.yml` — points to `http://192.168.30.14:2283`

The continued presence of the internal Traefik route does not itself mean the service is published through Cloudflare Tunnel.

### Historical Temporary Access Procedure

The original procedure was:

1. In Cloudflare Zero Trust → Networks → Tunnels → homelab-v3, add a public hostname:
   - Subdomain: `photos`
   - Domain: `giohosted.com`
   - Service type: `HTTPS`
   - Origin URL: `192.168.30.11`
   - TLS setting: No TLS Verify enabled
2. Test the shared album link from a device using cellular data.
3. Use Immich's shared-album protection for the intended guests.

After the event:

1. Remove the public `photos.giohosted.com` tunnel route.
2. Confirm that external access is no longer available.
3. Preserve internal Traefik routing for LAN clients.

**Current documentation status:** The tunnel route is recorded as removed. Its current absence was not independently rechecked as part of the November 14 three-service audit.

The Immich server's documented external domain is `https://photos.giohosted.com` under Administration → Server Settings.

---

## Adding a New Externally Exposed Service

1. Deploy the service with Traefik labels or an appropriate file-provider route.
2. Verify its intended audience and authentication requirements.
3. Add the required AdGuard DNS rewrite.
4. In Cloudflare Zero Trust → Networks → Tunnels → homelab-v3 → Published application routes, configure the hostname and origin.
5. Review whether Cloudflare Access should protect the service.
6. If Access is required, create or assign narrowly scoped policies.
7. Verify application authentication independently of Cloudflare Access.
8. Test access from an external client.
9. Confirm that unauthorized clients cannot access protected functionality.
10. Document the deployed route, policies, and verification results.

The established origin configuration uses:

- Service type: HTTPS
- Origin: `192.168.30.11`
- No TLS Verify: enabled

These settings reflect the existing deployment. Evaluate origin certificate verification when making future changes rather than assuming the current exception is universally appropriate.

**Important:** Publishing a tunnel hostname does not automatically add Cloudflare Access protection.

---

## Services NOT Exposed via Tunnel

The established documentation excludes these services from Cloudflare Tunnel:

| Service | Reason |
|---|---|
| Traefik dashboard | Admin tool — internal only |
| Dockman | Admin tool — internal only |
| Proxmox UI | Admin tool — internal only |
| Unraid UI | Admin tool — internal only |
| ARR stack | LAN only — no external access required |
| qBittorrent | LAN only — no external access required |
| qBitrr | LAN only — no external access required |
| CWA | LAN only — no external access required |
| PBS | Admin tool — internal only |
| Immich | LAN-only by default; previously published temporarily |
| Plex | Uses the separately documented direct port-forward arrangement on TCP 32400 |

Plex remains intentionally outside the Cloudflare Tunnel architecture. Review the provider's current service terms and technical suitability before considering any change to that design.

The table reflects documented exposure intent. It is not a substitute for periodic live verification of Cloudflare routes, DNS, firewall rules, and other remote-access mechanisms.

---

## November 2026 External Access Security Review

**Audit:** A-008 — Docker Update Management, external application exposure review  
**Date:** 2026-11-14

### Scope

The review focused on three Docker applications without Traefik's `local-only@docker` middleware:

- Seerr
- Shelfmark
- Audiobookshelf

### Results

| Application | Finding | Outcome |
|---|---|---|
| Seerr | Intentionally published; Plex login working | Local Sign-In disabled |
| Shelfmark | Cloudflare Access and Authentik login challenge confirmed | No configuration change required |
| Audiobookshelf | Cloudflare Access browser challenge and external mobile playback verified | No configuration change required |

### Security Controls Reviewed

- Cloudflare Access application inventory.
- Access policy actions and Include/Require conditions.
- Authentik login integration.
- Plex-only Seerr authentication.
- Seerr default administrative permissions.
- Audiobookshelf mobile Service Auth policy.
- External browser and mobile application functionality.

### Conclusions

The three applications are intentionally externally accessible.

Their reviewed authentication paths were functional, and no additional `local-only@docker` middleware was recommended.

Seerr's unused Local Sign-In method was disabled and Plex login was verified afterward.

The review did not identify a need to add more authentication layers solely because the applications are externally accessible.

### Remaining General Security Considerations

The following were not fully tested and should be revisited when relevant:

- Authentik MFA enforcement for every permitted user and authentication path.
- Exact matching of the Authentik `users` entitlement in Cloudflare Access.
- Cloudflare Access geographic-policy behavior under different client locations.
- Audiobookshelf service-token expiration and rotation.
- Origin certificate verification for cloudflared-to-Traefik connections.
- Periodic validation that published tunnel routes match documented intent.

These are follow-up considerations, not unverified claims that the tested access paths are insecure.

---

## Cloudflare DNS Changes Made During v3 Build

| Record | Action | Reason |
|---|---|---|
| `*` A record → 69.216.122.32 | Deleted | v2 NPM wildcard — no longer needed |
| `giohosted.com` A record → 69.216.122.32 | Deleted | v2 NPM — no longer needed |
| `photos` A record → 69.216.122.32 | Deleted | v2 NPM — no longer needed |
| `auth` Tunnel record → homelab-v2 | Deleted | Dead v2 tunnel — recreated in homelab-v3 tunnel |

These are historical migration actions, not newly performed DNS changes.

---

## Notes

- cloudflared is a stateless Docker container and has no separate application-data directory.
- The tunnel token is stored in `/opt/stacks/cloudflared/.env`, excluded from Git.
- The tunnel token and Cloudflare-managed route and policy configuration require separate recovery consideration.
- cloudflared connects to Traefik by IP; its use of the `proxy` Docker network is not required for that particular IP-based origin connection.
- Cloudflare Access protects Shelfmark and Audiobookshelf browser access.
- Audiobookshelf mobile clients use the scoped `abs-mobile` Service Auth policy.
- Seerr uses Plex authentication, not Authentik OIDC.
- Seerr Local Sign-In was disabled on November 14, 2026.
- MFA is configured in Authentik; universal enforcement was not separately audited.
- The shared Access browser policy includes a United States geographic requirement.
- The service-token authentication path has separate authorization conditions from the browser Allow policy.
- The existing No TLS Verify setting reduces origin certificate authentication assurance and remains a future hardening consideration.
- Plex is intentionally excluded from the tunnel and uses the established direct access arrangement.
- Changes to shared Cloudflare Access policies can affect multiple applications; inspect policy associations before editing.

---

## Related Documentation

- `services/infra/traefik.md` — Reverse-proxy routes, TLS configuration, and security hardening.
- `services/infra/authentik.md` — Identity provider and authentication.
- `services/media/seerr.md` — Plex authentication and external media requests.
- `services/books/shelfmark.md` — Book and audiobook request frontend.
- `services/books/audiobookshelf.md` — Audiobook streaming and mobile access.
- `network/firewall-rules.md` — Inter-VLAN and network security policies.
- `runbooks/updating-containers.md` — Docker maintenance and verification.