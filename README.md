# nuclei-custom-templates

A personal collection of custom [nuclei](https://github.com/projectdiscovery/nuclei)
templates built during authorized security assessments — full exploit PoCs,
version-based advisory triage, and exposure/misconfiguration checks — for products
not (yet) well covered by the official template set.

Layout mirrors the official `projectdiscovery/nuclei-templates` repo so it composes
cleanly with `nuclei -t` and standard tag/severity filtering.

> **Upstream-first — why this repo exists.** It holds only templates that do **not**
> (yet) meet the official [ProjectDiscovery CONTRIBUTING](https://github.com/projectdiscovery/nuclei-templates/blob/main/CONTRIBUTING.md)
> requirements: version-based triage (PD accepts complete PoCs only), pentest-practical
> variants (e.g. session-cookie/SSO auth), or work-in-progress checks. Any template that
> *does* qualify — a complete, verified PoC — should be contributed to
> `projectdiscovery/nuclei-templates` **as soon as possible**. This is a staging /
> overflow area, not a competitor: the goal is to give back to open source whenever a
> template is upstream-ready.

## Layout

```
http/
├── cves/<year>/CVE-*.yaml          # complete, verified exploit PoCs (upstream-quality)
├── technologies/                   # product + version fingerprinting
├── exposures/                      # exposed endpoints / misconfigurations
└── version-triage/<product>/       # version-based advisory mapping (see note below)
workflows/                          # ordered, tag-based orchestration per product
```

### Two philosophies, on purpose

| Kind | Auth | Proves exploitability | FP risk | Upstream-acceptable |
|------|------|-----------------------|---------|---------------------|
| `http/cves/` full PoC | often yes | **yes** | very low | ✅ yes |
| `http/version-triage/` | no | no (exposure only) | medium | ❌ no (PD requires complete PoCs) |
| `http/exposures/`, `http/technologies/` | no | n/a (detection) | low | ✅ often |

> **Why keep version-triage at all?** ProjectDiscovery does not accept version-only
> CVE templates upstream, but for an engagement they are invaluable: fingerprint the
> version once and get a prioritised, severity-tagged checklist of every advisory in
> range. They live in their own tree so it is always clear they assert *exposure*,
> not *exploitability*.

## Usage

```bash
# everything for a product, by tag
nuclei -t http/ -tags dolibarr -u https://target

# just the exposure + fingerprint sweep (unauthenticated)
nuclei -t http/technologies/ -t http/exposures/ -u https://target

# version-based advisory triage (unauthenticated), high/critical only
nuclei -t http/version-triage/ -severity critical,high -u https://target

# a full exploit PoC (authenticated — pass creds)
nuclei -t http/cves/2026/CVE-2026-34036.yaml -u https://target \
  -var username=USER -var password=PASS

# ordered workflow
nuclei -w workflows/dolibarr-workflow.yaml -t http/ -u https://target
```

## Coverage

### Dolibarr (built/validated against 20.0.4)
- `http/cves/2026/CVE-2026-34036.yaml` — **authenticated LFI** in `selectobject.php` (full PoC, verified)
- `http/technologies/dolibarr-version-fingerprint.yaml` — exact version extraction
- `http/exposures/` — installer takeover exposure, public endpoint enumeration, cron-by-url endpoint
- `http/version-triage/dolibarr/` — 6 advisories affecting 20.x, each at its real severity/CVSS
  (critical `MAIN_ODT_AS_PDF` RCE, 3× high RCE, medium LFI, low injection)

## Authentication options (for authenticated templates)

Authenticated templates ship in two flavours so they work against any auth backend:

| Flavour | How auth is provided | Requests | Use when |
|---------|----------------------|----------|----------|
| **Form login** (`CVE-*.yaml`) | `-var username=` / `-var password=` — the template fetches the CSRF token and posts the login form | 3 | native Dolibarr auth, LDAP — anywhere the login form accepts a username/password |
| **Session cookie / SSO** (`CVE-*-cookie.yaml`) | `-var cookie='DOLSESSID_<hash>=<value>'` — you supply an already-authenticated session | 1 | **SSO (SAML/OIDC/CAS/OAuth), 2FA, or any backend where the login form can't be scripted** |

```bash
# form login (native auth)
nuclei -t http/cves/2026/CVE-2026-34036.yaml -u https://target \
  -var username=USER -var password=PASS

# session cookie (SSO-friendly) — grab DOLSESSID from an authenticated browser
# session (DevTools > Application > Cookies, or your intercepting proxy)
nuclei -t http/cves/2026/CVE-2026-34036-cookie.yaml -u https://target \
  -var cookie='DOLSESSID_9a96...=abcdef0123456789'
```

Guidance for new authenticated templates: provide the **form-login** variant for
upstream submission (PD convention is `{{username}}`/`{{password}}`), and add a
**`-cookie`** companion here for SSO/2FA targets. The cookie variant is a single
request and matches only when the session is valid, so a bogus/expired cookie yields
no false positive.

## Conventions for new templates

- Follow the official [TEMPLATE-CREATION-GUIDE](https://github.com/projectdiscovery/nuclei-templates/blob/main/TEMPLATE-CREATION-GUIDE.md).
- Put real exploit PoCs under `http/cves/<year>/` with `id: CVE-XXXX-YYYY`, full
  `classification`, `verified: true`, and ≥2 non-trivial matchers — so they can be
  submitted upstream as-is.
- Put version-only checks under `http/version-triage/<product>/` and say so in the
  description.
- Always add a `<product>` tag so `-tags` sweeps work.
- Validate before committing: `nuclei -validate -t .`

## Authorized use only

For defensive assessment against systems you own or are explicitly authorized to test.
