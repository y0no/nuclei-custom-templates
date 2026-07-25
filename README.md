# nuclei-custom-templates

A personal collection of custom [nuclei](https://github.com/projectdiscovery/nuclei)
templates built during authorized security assessments — full exploit PoCs,
version-based advisory triage, and exposure/misconfiguration checks — for products
not (yet) well covered by the official template set.

Layout mirrors the official `projectdiscovery/nuclei-templates` repo so it composes
cleanly with `nuclei -t` and standard tag/severity filtering.

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
