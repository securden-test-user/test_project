# NHI risk score, secret strength, and severity — testing guide

Use this while testing Identities and Secrets for every data source. Values below are what the product **computes**, not what a security policy “should” be.

**Providers**

| ID | Data source |
|---|---|
| 1 | GitHub |
| 2 | AWS |
| 3 | Azure |
| 4 | Bitbucket |
| 5 | Kubernetes |
| 6 | Linux devices |
| 7 | IIS web server |

---

## 1. How the three fields differ

| Field | What it is | Scale | Where testers see it |
|---|---|---|---|
| **Risk score** | Additive 0–100 (then capped) | 0 = lowest, 100 = highest | Identity table / identity details. Secrets: AWS stores `metadata.risk.score`; GitHub/Bitbucket/IIS often have no 0–100 secret risk until activity refinement. |
| **Severity** | Bucket of the risk score (or a discovery rule) | **1 Critical, 2 High, 3 Medium, 4 Low** | Identity `severity`, secret `severity_score` |
| **Secret strength** | Strength of the **observed value** (`entropy_score`) | **1 Very High, 2 High, 3 Medium, 4 Low**. Lower number = stronger. `null` = Unknown / N/A | Secrets table Strength column |

**Identity severity from identity risk score** (`identity_risk.py`)

| Score | Severity | Label |
|---|---|---|
| ≥ 75 | 1 | Critical |
| ≥ 50 | 2 | High |
| ≥ 25 | 3 | Medium |
| 0–24 | 4 | Low |

**Secret severity from secret risk score** (`get_severity_level`) — used by AWS secrets and activity-log refinement

| Score | Severity | Label |
|---|---|---|
| ≥ 80 | 1 | Critical |
| ≥ 60 | 2 | High |
| ≥ 40 | 3 | Medium |
| 0–39 | 4 | Low |

---

## 2. Secret strength (all data sources)

Computed by `assess_secret_strength`. It estimates the **search space of the stored value**. It does not claim how the provider generated the secret.

### When strength is not a 1–4 score

| Status | Score shown | When |
|---|---|---|
| NOT_APPLICABLE | Unknown | Provider **variable** (`is_variable=true`), or **certificate** (category 6) |
| UNAVAILABLE | Unknown | Value was never returned (typical Azure app password, AWS IAM secret access key after create, SSH key with missing algorithm/size) |
| ERROR | Unknown | JSON credential with no recognized secret field |

### Plain / API / token values (GitHub, Bitbucket, AWS Secrets Manager value, Azure Key Vault value, IIS config values)

1. Strip known prefixes: `github_pat_`, `ghp_`, `gho_`, `ghu_`, `ghs_`, `ghr_`, `xoxb-`, `xoxp-`, `xoxa-`, `xoxr-`, `Bearer `.
2. If empty after that → **Low (4)**.
3. If common/weak pattern → **Low (4)**. Patterns include: `password`, `admin`, `changeme`, `qwerty`, `secret`, `test`, `welcome`, `letmein`, `default`, `password1`, `password1!`; only 1–2 distinct characters; repeated short block; sequences like `0123456789`, `abcdefgh`, `qwerty`.
4. Else **estimated bits = length × log2(character pool)**.

Character pool:

- Hex-only string (`0-9a-fA-F`) → pool **16**
- Else: +26 if lowercase, +26 if uppercase, +10 if digits, +33 if any other character

| Estimated bits | Strength | Label |
|---|---|---|
| ≥ 128 | 1 | Very High |
| ≥ 80 | 2 | High |
| ≥ 64 | 3 | Medium |
| &lt; 64 | 4 | Low |

JSON / paired credentials: only **sensitive keys** are scored (`password`, `secret`, `token`, `private_key`, `api_key`, `access_key`, `auth`, …). Non-secret keys (`username`, `client_id`, `access_key_id`, `url`, …) are ignored. The **weakest** sensitive field wins.

### SSH keys (category 4) — Linux / Git deploy / SSH auth

Uses algorithm + size from metadata, **not** the private-key string.

| Algorithm / size | Strength |
|---|---|
| Ed25519 | 1 Very High (~128 bits) |
| Ed448 | 1 Very High (~224 bits) |
| RSA &lt; 2048 | 4 Low |
| RSA 2048–3071 | 2 High (~112 bits) |
| RSA ≥ 3072 | 1 Very High (≥128 bits) |
| ECDSA curve &lt; 224 | 4 Low |
| ECDSA 224–383 | 2 High |
| ECDSA ≥ 384 | 1 Very High |
| DSA | 4 Low |
| RSA/ECDSA size missing | Unknown |
| Unknown algorithm | Unknown |

### Quick strength checks

| Input | Expected strength |
|---|---|
| `password` | Low |
| `aaaa` | Low |
| 8 hex chars | Low (8 × 4 = 32 bits) |
| 16 hex chars | Low (64 bits is Medium; 16×4=64 → Medium) |
| 20 hex chars | High (80 bits) |
| 32 hex chars | Very High (128 bits) |
| 16 mixed upper+lower+digit | Medium (16 × log2(62) ≈ 95 → High) |
| GitHub PAT `ghp_` + 36 random alphanumerics | Score the suffix only |

---

## 3. Identity risk score (all data sources)

Engine: `evaluate_identity_risk` (`securden_identity_risk_v2`). Recalculated on identity security assessment / persist.

**Formula**

```
raw = max(sum(factor points), native_floor, max attached policy total_risk_score)
if status = Inactive (2): raw = round(raw × 0.8)
if status = Disabled (3) or removed_in_remote: raw = round(raw × 0.6)
score = min(max(raw, 0), 100)
```

Then map score → severity with the **identity** table in section 1 (≥75 / ≥50 / ≥25).

### Common factors (every provider)

| Factor | Points | When to expect it |
|---|---|---|
| Active identity | +5 | Status = Active (1) |
| Production exposure | +10 | Identity environment = 1 |
| Staging exposure | +5 | Identity environment = 2 |
| Dormant active identity | +20 | Active, last activity ≥ 180 days |
| Stale active identity | +10 | Active, last activity ≥ 90 and &lt; 180 days |
| Business owner missing | +10 | Active and no business owner |
| Large secret blast radius | +20 | ≥ 10 mapped secrets |
| Moderate secret blast radius | +10 | 5–9 mapped secrets |
| Secret access | +3 | 1–4 mapped secrets |
| Incomplete assessment evidence | +3 per missing coverage item, max 15 | Missing any of: provider controls, credentials, activity, ownership, blast radius |

**CI/CD blast radius:** only secrets **in that pipeline/workflow file** count, not every secret in the repo.

**Native floor:** Kubernetes `metadata.risk.risk_score`, and/or the highest attached AWS policy `total_risk_score`. Score cannot go **below** that floor before the status multiplier.

**Confidence:** share of coverage flags that are present. High ≥80%, Medium ≥50%, else Low.

---

## 4. Identity scoring by data source

### 4.1 GitHub and Bitbucket (provider 1 / 4)

Always sets “provider controls” covered. Then adds:

| Identity type (stored ID) | Extra points | Condition |
|---|---|---|
| Deploy key (6) | +35 | `read_only` is **false** (write deploy key) |
| CI/CD (7) | +30 | GitHub `actions_permissions.default_workflow_permissions` = `write` |
| Any VCS type | +20 | Any `permissions_list` row contains `admin`, `write`, or `maintain` |

Also coded (type IDs in the scorer). Confirm the **stored** `identity_type` before expecting these:

| Scorer type ID | Factor | Points | Condition |
|---|---|---|---|
| 8 | Webhook TLS off | +50 | `insecure_ssl` true |
| 9 | Persistent Actions runner | +25 | `ephemeral` is not true |
| 10 | Ruleset bypass | +60 | Always if type is 10 |
| 14 | Unprotected environment | +35 | `protection_rules_count` = 0 |
| 15 | Repository access token | +50 / +35 / +20 | visibility `all`/`public` / `private`/`selected` / other |
| 17 | Public repository identity | +35 | `private` is false or visibility `public` |

Discovered types today: webhook **10**, runner **11**, ruleset bypass **12**, SSH auth **14**, codespace **15**, deployment env **18**, repo access token **19**, repository **21**. If a factor above does not fire, check that the stored type matches the scorer ID.

**Provider-native identity severity at discovery** (`derive_vcs_identity_severity`) — used until identity risk persist overwrites `severity`:

| Type | Critical (1) | High (2) | Medium (3) | Low (4) |
|---|---|---|---|---|
| Deploy key | — | write (`read_only=false`) | Bitbucket deploy key (read-only) | GitHub read-only |
| SSH auth / signing | — | `weak_key` | otherwise | — |
| GPG | — | `weak_key` | otherwise | revoked or expired |
| CI/CD | — | workflow token **write**, or admin/write/maintain | workflow token **read**, or default | — |
| Webhook | `insecure_ssl` | ≥ 10 events | ≥ 3 events | inactive or few events |
| Actions runner | — | not ephemeral | ephemeral | — |
| Ruleset bypass | always | — | — | — |
| GitHub App | — | always (2) | — | — |
| OAuth app | — | ≥ 10 scopes | &lt; 10 scopes | — |
| GitHub PAT | — | expired→4; admin/write/all-repos/dangerous scopes → 2 | else 3 | expired = 4 |
| Bitbucket access token | visibility all/public | private/selected | else | — |
| Workspace integration | — | — | always 3 | — |
| Codespace | — | — | available/running | other state |
| Deployment environment | — | no protection rules | — | has protection rules |
| Repository | — | public | private | archived/disabled |

### 4.2 AWS (provider 2)

**IAM user (type 1)** — `analyze_iam_user_credentials`, then **capped at 60** when added as a factor:

| Finding | Points |
|---|---|
| Two active access keys | +25 |
| Active key age ≥ 180 days | +35 |
| Active key age ≥ 90 days | +25 |
| Active key age ≥ 45 days and never used | +15 |
| Console password on, MFA off | +35 |

**IAM role (type 2)** — `analyze_iam_role_trust`, then **capped at 70**:

| Finding | Unrestricted | With restrictive condition |
|---|---|---|
| Trust principal `*` | +80 | +45 |
| External account `:root` | +50 | +25 |
| Trust policy unparseable | +20 | — |

Restrictive conditions looked for: `aws:PrincipalOrgId`, `aws:PrincipalArn`, `aws:SourceArn`, `aws:SourceAccount`, `aws:SourceVpce`, `aws:SourceVpc`, `sts:ExternalId`.

**Attached IAM policies (native floor)** — each **Allow** statement:

| Statement | Points |
|---|---|
| Any Allow | 10 |
| Wildcard action (`*` or `service:*`) or NotAction | +40 |
| Wildcard resource (`*`) or NotResource | +40 |
| Action in iam / sts / organizations / secretsmanager / kms | +10 |
| Deny | 0 |

Policy `total_risk_score` = max statement score. That value is a **floor** on identity risk.

### 4.3 Azure (provider 3)

**Privilege (from `permissions_list` text)**

| Graph permission (case-insensitive substring) | Points |
|---|---|
| `rolemanagement.readwrite.directory` or `approleassignment.readwrite.all` | +45 (stops here) |
| else `directory.readwrite.all` / `application.readwrite.all` / `group.readwrite.all` | +35 |
| else any `write` or `fullcontrol` | +20 |

**Credentials** (password + key credential lists)

| Condition | Points |
|---|---|
| N expired (`endDateTime` in the past) | min(45, 30 + (N−1)×5) |
| else M expire within 30 days | min(30, 15 + (M−1)×5) |
| K with no expiry date | min(20, 8 + (K−1)×3) |
| `sign_in_audience` set and not `azureadmyorg` | +10 |

Managed identities skip credential-list coverage (treated as covered).

**Discovery-time Azure identity severity** (overwritten after risk persist): disabled → 4; ≥3 credentials → 2; ≥1 credential → 3; none → 4.

### 4.4 Kubernetes (provider 5)

| Factor | Points | Condition |
|---|---|---|
| Cluster-admin | +55 | RBAC flags with both “cluster” and “admin” |
| else wildcard RBAC | +40 | any rule verb or resource is `*` |
| Token automount | +10 | `automount_service_account_token` / `automountServiceAccountToken` |

Plus **native floor** = agent `metadata.risk.risk_score`. Agent also sets discovery severity from `risk.risk_level`: CRITICAL→1, HIGH→2, MEDIUM→3, LOW→4. Identity type is service account (3). Environment defaults to 5 (Others).

### 4.5 Linux (provider 6)

Uses account name, UID, `sudo_permissions`, `file_permissions`, `permissions_list`.

| Factor | Points | Condition |
|---|---|---|
| Direct root | +45 | name `root`, UID 0, `is_root`, or `uid_zero` in permissions |
| else passwordless unrestricted sudo | +45 | `nopasswd` plus `all` / `all_commands` / `unrestricted` |
| else unrestricted sudo | +35 | `sudo_all`, `all_commands`, `unrestricted_sudo`, or sudo `(all)` + ` all` |
| else any sudo | +25 | `is_sudo_user`, `sudo_sources`, or “sudo” in permission text |
| Unsafe SSH file permissions | +25 | world/group writable, mode 777/666, private key or authorized_keys too open |

### 4.6 IIS (provider 7)

| Factor | Points | Condition |
|---|---|---|
| Privileged IIS process identity | +40 | app-pool identity is LocalSystem / LocalService / NetworkService |
| Custom IIS app-pool identity | +20 | custom identity, not ApplicationPoolIdentity |

**Discovery-time IIS identity severity** (if risk persist has not run): ≥10 secrets → Critical; ≥5 → High; ≥1 → Medium; custom pool → Medium; else Low.

---

## 5. Secret severity and secret risk by data source

### 5.1 GitHub and Bitbucket secrets (scanner + provider secrets)

**Severity is visibility-first.** If repo public/private is known:

| Repo | Severity |
|---|---|
| Public | **2 High** |
| Private | **3 Medium** |

If visibility is unknown, fall back to a point sum then the **secret** severity table (≥60 Crit, ≥40 High, ≥20 Med, else Low):

| Input | Points |
|---|---|
| Strength 1 Very High | +40 |
| Strength 2 High | +25 |
| Strength 3 Medium | +10 |
| Strength 4 / missing | +0 |
| Environment Production (1) | +30 |
| Staging (2) | +15 |
| Development (3) | +5 |
| `insecure_ssl` | +15 |
| `read_only` is false | +10 |

**No AWS-style 0–100 secret risk** is stored at discovery. Activity refinement (if events exist) can still write `metadata.risk.score` and retarget severity using the **secret** 80/60/40 table.

**Environment inference** (if not already set):

| Clue | Environment |
|---|---|
| Branch `main` / `master` / `release` | 1 Production |
| Branch `develop` / `development` / `dev` | 3 Development |
| Name/path contains production (not nonprod) | 1 |
| staging / uat | 2 |
| development / sandbox / local / `dev` | 3 |
| `-test` / `qa` | 3 |
| else | 4 (mapped as test/other in VCS helper) |

Secret environment IDs used in scoring: **1 prod, 2 staging, 3 dev, 4 uat, 5 qa, 6 test, 7 unknown**.

### 5.2 AWS Secrets Manager

**Base risk** (then × environment multiplier, cap 100):

| Signal | Points |
|---|---|
| Rotation disabled | +20 |
| Strength Low (4) | +20 |
| Strength Medium (3) | +10 |
| Rotation overdue (age &gt; 1.5 × interval) | +15 |
| else age &gt; 180 days | +15 |
| else age &gt; 90 days | +10 |
| Unused (created &gt; 90 days and not read in 90 days) | +10 |
| High-value category (DB / access key pair / SSH / encryption key) | +10 |
| Public policy `Principal=*` | +40, or +15 with any Condition |
| Cross-account whole account / `:root` | +25, or +10 with Condition |
| Cross-account specific principal | +15, or +5 with Condition |

Public and cross-account **do not stack** within the same kind; the strongest of each kind is kept.

**Environment multiplier** on the sum: prod 1.5, staging 1.2, dev 1.1, uat 1.0, qa 0.8, test 0.8, unknown 1.0.

Severity = `get_severity_level(score)` (80/60/40).

### 5.3 AWS IAM access-key secrets

Not scored from the key string (AWS never returns the secret after create → strength Unknown).

| Key state | Secret risk base |
|---|---|
| Active | Inherits the **IAM user’s** identity `risk.base_score` (signal `credential_of_identity`) |
| Inactive | **0** (signal `inactive_access_key`) |

Severity from that score via 80/60/40.

### 5.4 Azure

| Secret kind | Strength | Severity |
|---|---|---|
| App password / client secret (Graph) | Usually **Unknown** (value not returned). If a captured value exists, score that value. | From identity environment + later identity-risk persist; Graph path passes entropy 2 but upsert **recomputes** strength. Certificates (category 6) → strength N/A. |
| Key Vault secret | Measured from the vault value | No GitHub-style visibility rule. If `severity_score` stays empty, UI may show empty until activity/assessment. |

Azure identity environment from name/tags: prod→1, stag→2, dev→3, test/qa→4.

### 5.5 Kubernetes secrets

Imported with the agent payload. Identity native `risk.risk_score` / `risk_level` is the source of truth for the **identity**. Secret strength follows `assess_secret_strength` when a value is stored.

### 5.6 Linux SSH secrets

Strength = SSH table in section 2. Severity for Linux is **not** the GitHub visibility rule (`_resolve_vcs_iis_secret_fields` only runs for GitHub, Bitbucket, IIS). Expect severity from later assessment/activity, or empty until then.

### 5.7 IIS secrets

Same severity helper as GitHub/Bitbucket (`derive_vcs_secret_severity`) **plus +10** if metadata `provider` is `IIS_WEB_SERVER` on the fallback path. Visibility rule applies only if public/private is on the metadata (usually not for IIS), so fallback entropy + environment + 10 is typical.

---

## 6. Activity-log overlay (mainly AWS CloudTrail)

After discovery, `refine_risk_and_environment_from_activity` can change secret/identity severity.

```
final = min(base_score × env_multiplier_ratio + activity_points, 100)
severity = get_severity_level(final)   # 80 / 60 / 40
```

If environment was unknown and can be inferred from the other side, the base is rescaled by the environment multiplier ratio.

**Secret activity points**

| Signal | Points |
|---|---|
| Failed access ≥ 5 | +20 |
| Failed access ≥ 1 | +10 |
| Distinct accessors ≥ 10 | +20 |
| Distinct accessors ≥ 5 | +10 |
| Distinct accessors ≥ 3 | +5 |
| Source IPs ≥ 5 | +10 |
| Cross-environment access | +25 if secret is prod, else +10 |

**Identity activity points**

| Signal | Points |
|---|---|
| Failed API calls ≥ 5 / ≥ 1 | +20 / +10 |
| Source IPs ≥ 5 | +10 |
| Assumed by ≥ 5 principals | +10 |
| Privilege-escalation IAM calls | +25 |
| Other IAM writes | +10 |
| AdministratorAccess attached | +25 |
| Other policy change | +10 |
| Uses secrets of other environments | +15 |

Scanner identity events are ignored for secret “usage”.

---

## 7. Suggested test matrix

Work top to bottom. After each sync, open the identity/secret and record **risk score, severity, strength**.

### Identities

| # | Data source | Setup | Expect |
|---|---|---|---|
| I1 | Any | Active, no extra factors, no owner, 0 secrets, no activity | About 5 (active) + 10 (no owner) + incomplete-evidence points; Low or Medium |
| I2 | Any | Same + production environment | Extra +10 vs I1 |
| I3 | Any | Assign business owner | −10 vs previous |
| I4 | Any | Inactive | Score × 0.8 |
| I5 | Any | Disabled / removed | Score × 0.6 |
| I6 | Any | Last activity 100 days ago, still Active | Extra +10 stale |
| I7 | Any | Last activity 200 days ago, still Active | Extra +20 dormant |
| I8 | GitHub/BB | Write deploy key | Extra +35 |
| I9 | GitHub/BB | Read-only deploy key | No +35 |
| I10 | GitHub | CI/CD with default workflow token **write** | Extra +30 |
| I11 | GitHub/BB | `permissions_list` contains admin/write | Extra +20 |
| I12 | GitHub/BB | Public repo identity (if scorer type matches) | Extra +35 |
| I13 | AWS user | Password + no MFA | Extra up to 35, factor capped at 60 |
| I14 | AWS user | Two active keys, one ≥ 180 days | 25+35, still capped at 60 |
| I15 | AWS role | Trust `*` | Large trust factor, capped at 70 |
| I16 | AWS | Attach `Action=*` `Resource=*` policy | Policy floor 10+40+40=90 → identity score ≥ 90 → Critical |
| I17 | Azure | `Directory.ReadWrite.All` | +35 |
| I18 | Azure | Expired client secret | ≥ 30 |
| I19 | Azure | Multi-tenant audience | +10 |
| I20 | K8s | cluster-admin flag | +55 plus agent floor |
| I21 | Linux | root UID 0 | +45 |
| I22 | Linux | sudo ALL NOPASSWD | +45 |
| I23 | IIS | LocalSystem app pool | +40 |
| I24 | GitHub CI/CD | 8 repo secrets, 1 in the pipeline file | Blast radius +3 (not +10) |

### Secrets

| # | Data source | Setup | Strength | Severity / risk |
|---|---|---|---|---|
| S1 | GitHub/BB | Value `password` in **public** repo | Low | High (visibility) |
| S2 | GitHub/BB | 32-byte hex in **private** repo | Very High | Medium (visibility) |
| S3 | GitHub/BB | Same value, visibility unknown, prod branch `main` | From bits | Fallback: Very High +40 + prod +30 = 70 → Critical |
| S4 | AWS SM | Rotation off, Low strength, prod | Low | (20+20)×1.5 = 60 → High |
| S5 | AWS SM | `Principal=*` no condition, private/unknown env | — | +40 on base |
| S6 | AWS IAM key | Active | Unknown | Same risk bucket as the user |
| S7 | AWS IAM key | Inactive | Unknown | Risk 0 → Low |
| S8 | Azure Graph password | No stored value | Unknown | — |
| S9 | Azure Key Vault | Random 32-char mixed | Very High or High from bits | — |
| S10 | Azure cert | Category 6 | N/A / Unknown | — |
| S11 | Linux | Ed25519 | Very High | — |
| S12 | Linux | RSA 1024 | Low | — |
| S13 | Linux | RSA 2048 | High | — |
| S14 | Linux | DSA | Low | — |
| S15 | IIS | Weak password, unknown env | Low | Fallback +10 IIS; Low strength adds 0 |

### Strength-only (any source that stores the value)

| # | Value | Expected strength |
|---|---|---|
| E1 | `admin` | Low |
| E2 | `ghp_` + 40 hex | Score suffix only (40×4=160 → Very High) |
| E3 | 16 hex | Medium (64 bits) |
| E4 | 20 hex | High (80 bits) |
| E5 | 32 hex | Very High |
| E6 | JSON `{"username":"a","password":"x"}` | Low (password field) |
| E7 | JSON `{"username":"a","host":"db"}` | Error / Unknown if category is credential pair with no secret field |
| E8 | CI/CD **variable** (`is_variable`) | N/A Unknown |

---

## 8. What “Sync identity/secret security assessment” does

- **Identity:** recomputes `evaluate_identity_risk`, writes `metadata.risk_score`, `metadata.identity_risk_assessment`, and `severity`.
- **Secret (GitHub/Bitbucket/IIS):** recomputes strength; if severity empty, fills `derive_vcs_secret_severity`.
- **AWS secret assess:** reruns Secrets Manager policy + rotation + entropy, then activity overlay.

After changing owner, environment, pipeline file, or IAM policy, run assessment (or full data-source sync) before comparing numbers.

---

## 9. Field cheat sheet

| UI | DB / JSON | Strong / high risk |
|---|---|---|
| Identity risk score | `metadata.risk_score` / `identity_risk_assessment.score` | Higher is worse (0–100) |
| Identity severity | `NonHumanIdentity.severity` | 1 worse than 4 |
| Secret strength | `Secrets.entropy_score` | **1 better than 4** |
| Secret severity | `Secrets.severity_score` | 1 worse than 4 |
| Secret risk (AWS) | `secret_metadata.risk.score` | Higher is worse |
| Identity factors | `identity_risk_assessment.factors[].points` | Sum (then floor, multiplier, cap) |
