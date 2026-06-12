# Exploring the Evitree Phishing Kit

**Date:** June 11, 2026  
**Subject:** Credential-harvesting phishing campaign targeting Google account holders via fabricated Google Docs share notification  

**Scope.** Personal investigation following receipt of a phishing email. Analysis used publicly accessible infrastructure and browser-level interaction where the kit required it. On `mesh*.vu` deployments, fake credentials were submitted after clearing Cloudflare challenges to progress through kit stages (including `pass.php`). No real account credentials were used. No attempts were made to access operator backend systems beyond what any visitor could reach.

---

## 1. I hate phishing.

On June 11, 2026, I received a phishing email impersonating a Google Docs sharing notification. I decided to poke back a bit. Investigation revealed the lure to be a deployment of the **Evitree** phishing kit, hosted at `meshb.vu`.

Evitree is a Google credential harvesting kit named after the directory structure used by its developer on staging infrastructure (`siodg.vu/Evitree/`), captured and analyzed by Joe Sandbox. The kit is a static rip of the Google Sign-In page with PHP credential-handling backends bolted on, deployed across multiple rotating `.vu` domains. The developer backdoor pattern - their own email hardcoded into kit backends - is consistent with commercially distributed phishing kits where the seller retains a silent copy of harvested credentials (see Section 4.2 for evidence and scope).

This deployment runs a four-stage credential harvesting flow behind Cloudflare managed challenges, with real-time operator control via a Telegram-based C2 channel. Kit identification, the developer backdoor, and the endpoint fingerprint (`pass.php` + `mlog.php` + `check_telegram_updates.php`) are documented in Section 4. The same Google-credential flow matches a large-scale, publicly documented invitation-phishing campaign (Section 6.4).

---

## 2. Attack Chain Overview


| Stage | Endpoint                   | User Experience        | Technical Action                                                                                                             |
| ----- | -------------------------- | ---------------------- | ---------------------------------------------------------------------------------------------------------------------------- |
| 1     | `index.php`                | Google sign-in page    | POST captures email, session ID, hardcoded region and `usi` values                                                           |
| 2     | `pass.php`                 | Password entry page    | JavaScript `fetch()` sends password to `session/mlog.php` asynchronously before page renders                                 |
| 3     | Loading screen             | "Please wait…" spinner | `setInterval` polls `check_json_status.php` every 1 second; `setTimeout` fallback redirects after 2 seconds                  |
| 4     | `load.php?id={VISITOR_ID}` | Spinner / blank        | `redirect-checker.js` polls both `check_telegram_updates.php` and `check_json_status.php` awaiting operator redirect command |


**Key characteristic:** Credentials are exfiltrated via AJAX on Stage 2 *before* the loading animation completes. Closing the browser tab after clicking submit does not prevent capture.

---

## 3. Delivery Analysis

### 3.1 Email Headers


| Field        | Value                                                                     |
| ------------ | ------------------------------------------------------------------------- |
| From         | `Donita Turk <donitaturk@gmail.com>`                                      |
| To           | `undisclosed-recipients:;`                                                |
| Bcc          | `[recipient redacted]`                                                    |
| Subject      | "Cordially Invited"                                                       |
| Message-ID   | `<CAEPE9a-w6jL_p0U=Smz+=afkx9w0XRce9oJp=7g9NsgsEPvubA@mail.gmail.com>`    |
| Stated Date  | Thu, 11 Jun 2026 22:45:00 -0800                                           |
| SMTP Receipt | Thu, 11 Jun 2026 14:53:10 PDT (flagged: *Delivered after -31910 seconds*) |
| SPF          | PASS - IP `209.85.220.41` (Google SMTP `mail-sor-f41.google.com`)         |
| DKIM         | PASS - selector `20251104`, domain `gmail.com`                            |
| DMARC        | PASS                                                                      |


### 3.2 Authentication Notes

The sender `donitaturk@gmail.com` is a genuine Gmail account - either compromised and weaponized or purpose-created. All three email authentication mechanisms pass, and these results mean standard spam filters will not flag this email on authentication grounds alone.

The delivery IP `209.85.220.41` is a legitimate Google outbound mail server. There is no indication of spoofing; the account itself was the vector.

### 3.3 Social Engineering Techniques

**Forward-dating for inbox positioning.** The `Date` header was set approximately 8 hours and 51 minutes into the future relative to actual SMTP delivery. Gmail sorts inbox by the `Date` header value, not the SMTP receive time. This placed the email at the top of the recipient's inbox until real emails caught up. Google's header parser flagged the discrepancy as `Delivered after -31910 seconds`.

**Fabricated threading headers.** The `References` and `In-Reply-To` headers were crafted to contain fabricated Google-format Message-IDs:

```
References: <b53ca0f8-ca7e-429e-9107-bdd125736cb9@docs-share.google.com>
            <autogen-java-7eb3eca0-799d-43e3-b267-65b0d50ea646@google.com>
In-Reply-To: <autogen-java-7eb3eca0-799d-43e3-b267-65b0d50ea646@google.com>
```

Gmail's threading algorithm groups emails by these identifiers. If the target had previously received any real Google Docs notification, this email silently threads into that conversation, visually appearing as a legitimate follow-up from Google with no re-authentication prompt.

**Perfect template clone.** The HTML body is an accurate copy of a genuine Google Docs sharing notification email, using real Google CDN image URLs (`lh3.googleusercontent.com`, `ssl.gstatic.com`), correct Google fonts, and the official Google LLC footer. The only deviation: all links point to `https://meshb.vu/invite`.

**BCC mass-send.** All recipients were blind carbon copied. No individual recipient can infer campaign scale or see other targets.

---

## 4. Phishing Kit Analysis

### 4.1 Kit Identification

**Kit name:** Evitree  
**Kit type:** Static Google Sign-In page rip with PHP credential handling backend  
**Developer backdoor:** `xforgexxcoder22@gmail.com`  
**Reference / staging domain:** `siodg.vu/Evitree/`  
**Google template capture date:** March 9, 2025 at 18:25:50 UTC *(decoded from hardcoded `usi` field value `S-233362078:1741544750655901`)*

### 4.2 Attribution Basis

The Evitree attribution is supported by multiple independent sources:


| Evidence                                  | Source                                                     | Artifact                                                                                       |
| ----------------------------------------- | ---------------------------------------------------------- | ---------------------------------------------------------------------------------------------- |
| Kit explicitly named "Evitree"            | Joe Sandbox analysis of `siodg.vu/Evitree/`                | [Analysis 1919382](https://www.joesandbox.com/analysis/1919382/0/html)                         |
| `xforgexxcoder22@gmail.com` in `pass.php` | Joe Sandbox (same report)                                  | [Analysis 1919382](https://www.joesandbox.com/analysis/1919382/0/html) - visible in PHP source |
| `xforgexxcoder22@gmail.com` in live page  | Direct analysis of `meshb.vu` password page body           | Hidden `<input>` with `id="identifierId"` and `id="hiddenEmail"`                               |
| Identical file structure                  | `siodg.vu`, `lifn.digital`, `geteviteflow.com`, `meshb.vu` | `pass.php`, `assets/bscframe.html`, `assets/CheckConnection.html`                              |
| Identical polling architecture            | Multiple deployments                                       | `check_json_status.php`, `check_telegram_updates.php`                                          |


**Independently analyzed deployments.** The backdoor email `xforgexxcoder22@gmail.com` was confirmed directly in `pass.php` output (Joe Sandbox HTTP parser) on three separate domains, and in rendered HTML on a fourth:


| Deployment                             | How confirmed                                                                                                                                                                                                                                    | Source                                                                 |
| -------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ | ---------------------------------------------------------------------- |
| `siodg.vu/Evitree/`                    | Backdoor email in `pass.php`; directory literally named `Evitree`                                                                                                                                                                                | [Analysis 1919382](https://www.joesandbox.com/analysis/1919382/0/html) |
| `lifn.digital/myin/myinvite/`          | Backdoor email in `pass.php`                                                                                                                                                                                                                     | [Analysis 1904023](https://www.joesandbox.com/analysis/1904023/0/html) |
| `geteviteflow.com/evite/`              | Backdoor email in `pass.php`                                                                                                                                                                                                                     | [Analysis 1924593](https://www.joesandbox.com/analysis/1924593/0/html) |
| `cehvfihys.de/invitation/Login/gmail/` | Matching kit structure                                                                                                                                                                                                                           | Joe Sandbox (structural match)                                         |
| `meshb.vu/invite/`                     | Backdoor email in rendered HTML of password page (reached via browser with fake credentials; `pass.php` PHP source not directly retrieved)                                                                                                       | Direct analysis                                                        |
| `bytez.vu/.../accounts.s/`             | Identical static-capture fingerprint (see below)                                                                                                                                                                                                 | Direct analysis of rendered HTML                                       |
| `siodg.vu/docs/`                       | Identical static-capture fingerprint: same `usi` value, `fldUsername`, `region="US"`, `fidoUserHandle`, `pass.php?id={MD5}` form action; same operator JS customizations; Cloudflare beacon token shared with `siodg.vu/rsvp/` (see Section 6.5) | Direct analysis of rendered HTML                                       |


While mapping infrastructure from the `meshb.vu` incident, rendered HTML from `https://bytez.vu/check.bytez.vu/ru/vib/iv/accounts.s/` turned up another Evitree deployment under a separate, non-`mesh*` domain. It matches the email-entry stage of `meshb.vu/invite/index.php` on every static-capture marker:

- Identical hardcoded `usi`/`dsh` value `S-233362078:1741544750655901` - the same Google Sign-In snapshot (captured March 9, 2025 18:25:50 UTC), not an independently grabbed copy.
- Same `fldUsername` field name, static `region="US"`, empty `nonce=""` script attributes, and decoy blank `fidoUserHandle` input.
- Same form-action fingerprint: `pass.php?id={32-char MD5}` (observed: `pass.php?id=d6035b962035ff81564dc8aac7507fed`).

The backdoor email does not appear in this HTML. On `meshb.vu`, `xforgexxcoder22@gmail.com` showed up on the password page; here the capture is only the username stage, which lacks the `identifierId`/`hiddenEmail` inputs, and `pass.php` itself was not retrieved behind Cloudflare. Attribution for `bytez.vu` rests on the template fingerprint, not the email.

The operator-injected JavaScript on this page - `validateInput()` (phone-number formatting as `(NNN) NNN-NNNN`), a focus-trap that keeps `fldUsername` focused once typing starts, and `startDelayedSubmission()` on the Next button - is the same code present on `meshb.vu/invite/index.php`. It is not unique to `bytez.vu`; the same scripts also appear on `siodg.vu/docs/`.

The kit developer backdoor email appears in every deployment whose `pass.php` was successfully retrieved. For `meshb.vu` specifically, the password page was reached by submitting fake credentials after clearing the Cloudflare challenge; the email was confirmed in the rendered HTML. The raw `pass.php` PHP source could not be pulled via direct HTTP request (Cloudflare returned `403 cf-mitigated: challenge`):

```html
<input type="hidden" name="" value="xforgexxcoder22@gmail.com" jsname="m2Owvb" id="identifierId">
<input type="email" name="identifier" value="xforgexxcoder22@gmail.com" jsname="KKx9x" id="hiddenEmail">
```

### 4.3 File Structure


| File                          | Role                                                                     |
| ----------------------------- | ------------------------------------------------------------------------ |
| `index.php`                   | Stage 1 - email input, form action to `pass.php?id={MD5}`                |
| `pass.php`                    | Stage 2 - password input, AJAX POST to `session/mlog.php`                |
| `session/mlog.php`            | Credential logger (receives AJAX POST with `fldPassword` + `visitor_id`) |
| `check_json_status.php`       | Operator-controlled redirect flag (returns `{"redirect": false/true}`)   |
| `check_telegram_updates.php`  | Server-side Telegram Bot API proxy                                       |
| `load.php`                    | Stage 4 - polling stage, keyed by `VISITOR_ID`                           |
| `redirect-checker.js`         | Client-side polling engine                                               |
| `assets/CheckConnection.html` | Fake Google cross-origin iframe (non-functional outside Google domains)  |
| `assets/bscframe.html`        | Fake BSC/session iframe (same limitation)                                |


### 4.4 Indicators of Static Page Rip (Not AiTM)

Evitree is a static HTML/JavaScript capture of the Google Sign-In page with PHP backends appended. It's not an Adversary in the Middle proxy, and as far as I can tell, cannot rely live sessions or bypass MFA.

- **Empty nonce attributes:** All `<script>` tags carry `nonce=""`. Legitimate Google pages generate a unique nonce per request. Fixed empty nonces indicate a single captured static snapshot.
- **Hardcoded `usi` timestamp:** The `usi` session identifier (`S-233362078:1741544750655901`) decodes to March 9, 2025. This value is static across all sessions on the kit.
- **Google BotGuard failure:** Console log `Error: pb` was captured, which is the internal assertion Google's BotGuard emits when it detects execution outside a Google-owned origin.
- **Cross-origin `postMessage` failures:** The `CheckConnection.html` iframes attempt to communicate with `accounts.google.com` via `postMessage`, which is blocked by the browser's same-origin policy when loaded from `meshb.vu`.
- **Broken Google JavaScript:** `Uncaught ReferenceError: onCssLoad is not defined`, `ccTick`, `onJsLoad` - Google's page scripts require server-side initialization values not present in a static capture.
- **Hardcoded `region="US"` field:** The `index.php` form carries a static `region` value of `US`, indicating the campaign is configured to target US account holders.
- **Decoy `fidoUserHandle` field:** A blank WebAuthn/passkey `fidoUserHandle` input is present purely to make the hidden-field structure match a real Google page on casual source inspection. It is non-functional (no passkey flow exists in a static rip).

### 4.5 Command and Control Architecture

After a victim submits their password, the kit enters a polling loop waiting for the operator to issue a redirect. The operator controls this in real time:

```
Victim browser                   meshb.vu server              Operator (Telegram)
──────────────────────────────────────────────────────────────────────────────────
Submit password
  │
  ├── AJAX POST session/mlog.php ──►  Store credentials ──────► Telegram notification
  │                                                                      │
  │   Poll check_telegram_updates.php ◄── PHP proxies Telegram API ◄────┘
  │        (returns: 1)
  │
  │   Poll check_json_status.php?visitor_id=...
  │        (returns: {"redirect": false})
  │                      │
  │              [operator acts] ─────────────────────────────────────────►
  │                                      Sets redirect URL in session store
  │        (returns: {"redirect": true, "redirect_url": "https://..."})
  │
  └── window.location = redirect_url
         (victim lands on real Gmail / Google error page)
```

**Live C2 observations.** The following were directly observed during active interaction with the live kit:


| Endpoint                                         | Observed Response                             | Notes                                                                                                                                                                                                                                                                                                                                                         |
| ------------------------------------------------ | --------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `check_telegram_updates.php`                     | `1` (integer, kept polling every 1s)          | Confirmed by direct browser observation. The integer `1` is not standard Telegram Bot API JSON - the server-side PHP simplifies the response. Whether it represents a heartbeat, update count, or boolean true cannot be determined without the `check_telegram_updates.php` source. Indicates the endpoint is reachable and returning a consistent response. |
| `check_json_status.php?visitor_id=5299565306...` | `{"redirect": false}` (kept polling every 1s) | Confirmed by direct browser observation. No redirect was issued for this session during the observation window.                                                                                                                                                                                                                                               |


No redirect was issued during the observation window. The reason cannot be determined from client-side observation alone.

**Note on observed scope.** Ungated HTTP requests to `pass.php` return `403 cf-mitigated: challenge` - the kit executes only after a visitor clears the Cloudflare managed challenge. On `mesh*.vu` deployments, fake credentials were submitted in-browser after clearing that challenge; that is how `pass.php`, the loading stage, and the polling endpoints were reached. Live interaction confirmed `check_telegram_updates.php` and `check_json_status.php` behavior, and network traffic during password submission established the Stage 2 `fetch()` to `session/mlog.php`. The `pass.php` PHP source itself was not directly retrieved from `meshb.vu` (automated fetch still blocked). Joe Sandbox analysis of sibling deployments where `pass.php`/`mlog.php` were retrieved in full corroborates the backend structure.

---

## 5. Second Kit: `mesht.vu` Custom Google Sign-In Lure

While enumerating the `mesh*.vu` cluster (Section 6), a second active kit turned up on `mesht.vu` - unrelated to Evitree, but hosted on the same domain rotation pool. It runs under a different Cloudflare account (Browser Insights beacon identifier `03ae8bcbdb8d46c1a1f72420e776adb5`).

### 5.1 Attack Flow


| Stage | URL                                               | Content                                                                                                          |
| ----- | ------------------------------------------------- | ---------------------------------------------------------------------------------------------------------------- |
| 1     | `mesht.vu/invite/`                                | Fake Google "Browser verification" reCAPTCHA gate                                                                |
| 2     | `mesht.vu/invite/pages/login.php`                 | Custom Google Sign-In lookalike - email collection                                                               |
| 3     | `mesht.vu/invite/pages/login.php` (POST response) | Password page - left side echoes submitted email to increase trust                                               |
| 4     | `mesht.vu/invite/pages/login.php` (POST response) | "Processing" spinner; dual polling begins (`update_online.php` + `poll_status.php`)                              |
| 5     | Operator-determined                               | `poll_status.php` response status string becomes next page filename (e.g. `otp.php`, `wrong.php`, `success.php`) |


### 5.2 Kit Characteristics

This is a lightweight, custom-built kit - not a static page rip of the real Google Sign-In page. The HTML is purpose-written, making it smaller and harder to fingerprint against known static captures.


| Characteristic       | Value                                                                                                                          |
| -------------------- | ------------------------------------------------------------------------------------------------------------------------------ |
| Email field name     | `username`                                                                                                                     |
| Password field name  | `password`                                                                                                                     |
| Form action          | Posts to self (`pages/login.php`) across both stages                                                                           |
| Session token        | `sess_6a2b6a4ade2082.82505823` (hardcoded; consistent across all stages)                                                       |
| Heartbeat endpoint   | `../update_online.php?session_id=sess_6a2b6a4ade2082.82505823` every 3 seconds (active on both email and password pages)       |
| Email echo           | Submitted email is server-side echoed on the password page ("Welcome, [email]") - mimics real Google Sign-In to increase trust |
| Favicon path         | `res/img/fav435vsdvge5.png` (randomized suffix to evade file-path scanners)                                                    |
| Submit UX            | 2-second card animation before form POST (both stages)                                                                         |
| Stylesheet structure | Per-stage CSS: `style.css` (email), `pass-style.css` (password)                                                                |
| Cloudflare beacon    | `03ae8bcbdb8d46c1a1f72420e776adb5`                                                                                             |


### 5.3 Operator Control Mechanism

After the password is submitted, the kit enters a "Processing" spinner stage with two polling loops running simultaneously:

- `../update_online.php?session_id=sess_6a2b6a4ade2082.82505823` every **3 seconds** - heartbeat confirming the victim's session is still active
- `../poll_status.php?session_id=sess_6a2b6a4ade2082.82505823` every **2 seconds** - operator command channel

The `poll_status.php` response determines where the victim is sent next. When the response is `{"status": "waiting"}` the spinner continues. Any other status value is used directly as a PHP filename:

```javascript
window.location.href = data.status + '.php?session_id=' + sessionId;
```

This means the operator can route victims to any page in the kit in real time - for example:

- `{"status": "otp"}` - OTP/2FA interception page
- `{"status": "wrong"}` - fake incorrect-password page to capture a second attempt
- `{"status": "success"}` - redirect to real Google

This is a more flexible design than Evitree's `{"redirect": true, "redirect_url": "..."}` mechanism. Adding a new victim-routing option requires only adding a new PHP file; the JS routing logic requires no changes.

### 5.4 Developer Artifacts

**Broken hand-written obfuscation (Stage 1).** The reCAPTCHA gate page contains JavaScript structured to mimic professional obfuscator output (`_0x2efe`, `_0x179f`, `0x100` indexing, `while(!![])` array-rotation guard) but implemented incorrectly by hand. The rotation loop evaluates `3 === 3` on its first iteration and immediately breaks - the array is never rotated. The payload hides a two-element array containing a redirect URL and a timeout value. A comment left inside the "obfuscated" code reads `// dummy computation (equals 3)` - comments are stripped by real obfuscators; this one survived because the obfuscation was written manually. The developer understood the pattern well enough to imitate it, but not well enough to implement it correctly or to remove their own annotation.

**reCAPTCHA site key: `6LfTPlUUAAAAAGSUt1_LqpJXQpatx7_BzTDcU9On`.** Site keys are tied to a specific Google account at registration. This is both a forensic artifact linking all deployments using this key to one operator identity, and an actionable reporting target - disabling the key via Google reCAPTCHA abuse reporting breaks the anti-analysis gate on every domain this operator deploys the kit to.

**Hardcoded, shared session ID: `sess_6a2b6a4ade2082.82505823`.** This token appears identically across every stage and every victim session. It is not generated per-visitor. The format (14 hex characters + dot + 8 decimal digits) does not match any standard session format and appears custom. The hardcoding is an operational security failure: the `poll_status.php` redirect command applies to all active sessions simultaneously rather than to a specific victim.

**`fav435vsdvge5.png`.** The randomized favicon name is anti-fingerprinting by design, but is itself now a fingerprint. The string `435vsdvge5` does not match MD5, SHA-1, UUID, or any standard hash truncation format.

**Copied Google `origin-trial` meta tag.** The signed Origin Trial token in Stage 1 is valid only for `www.google.com` and is non-functional on `mesht.vu`. It is copied verbatim from a real Google page to make the HTML source appear more authentically Google-generated to a source inspector.

**Architectural split.** The broken obfuscation is confined to Stage 1 (client-side JS). The server-side routing logic (`data.status + '.php'`) is clean and well-designed. This inconsistency suggests either two contributors of different skill levels, or a developer significantly more comfortable with server-side PHP architecture than client-side JavaScript.

### 5.5 Prior Research Context

No public report specifically names or attributes the `mesht.vu` kit as a distinct threat family. The closest documented analog is the **"heartbeat/check_redirect variant"** of Doko's Panel, described in Push Security's June 2026 report [Inside a phishing panel used by ShinyHunters and BlackFile](https://pushsecurity.com/blog/inside-criminal-phishing-panel). That variant replaces Doko's Panel's standard single `ping` action with a split two-endpoint protocol:

- **Heartbeat** - POST to `backend.php` with `action=heartbeat` (analogous to `update_online.php`)
- **Check Redirect** - GET to `backend.php` with `action=check_redirect` (analogous to `poll_status.php`)

Push Security's characterization of that variant is directly applicable here: "This level of broken duplication indicates an inexperienced developer making modifications to unfamiliar code." Their report also documents extensive LLM-generated artifacts in the same variant - verbose self-documenting comments, broken duplication where an inline script and an external JS file independently schedule the same backend requests, and UUID generation using `Math.random()`. These signatures overlap with the broken obfuscation and developer annotation (`// dummy computation (equals 3)`) found in Stage 1 of the `mesht.vu` kit.

The `mesht.vu` kit does not match Doko's Panel's exact technical profile (different endpoint names, PHP-routed redirects instead of `backend.php`, different session token format), but the architectural DNA - operator-in-the-loop via dual heartbeat + poll endpoints, and low-skill client-side JS grafted onto competent server-side PHP - is the same pattern. Whether the `mesht.vu` kit is a direct fork, a parallel derivation, or an independent implementation of the same design is not determinable from available evidence.

### 5.6 Comparison with Evitree


|                                | Evitree (`meshb.vu`)                                                                             | `mesht.vu` kit                                                                                         |
| ------------------------------ | ------------------------------------------------------------------------------------------------ | ------------------------------------------------------------------------------------------------------ |
| Page weight                    | ~1MB (full Google page rip)                                                                      | Lightweight custom HTML                                                                                |
| Email field name               | `fldUsername`                                                                                    | `username`                                                                                             |
| Credential endpoint            | `pass.php` / `session/mlog.php`                                                                  | `pages/login.php`                                                                                      |
| C2 mechanism                   | Telegram Bot API proxy (`check_telegram_updates.php`)                                            | Dual polling: `update_online.php` (heartbeat, 3s) + `poll_status.php` (operator command, 2s)           |
| Operator routing               | Fixed `redirect_url` in JSON response                                                            | `data.status + '.php'` - status string becomes filename; operator can route to any kit page            |
| Anti-analysis gate             | Cloudflare Turnstile only                                                                        | Google reCAPTCHA v2 + Cloudflare Turnstile                                                             |
| Session tracking               | MD5 hash in URL params                                                                           | `sess_` token in query string                                                                          |
| Credential exfiltration timing | Async `fetch()` fires BEFORE page transition - closing tab after submit does not prevent capture | Standard form POST with 2-second animation delay - closing tab during animation may prevent submission |
| Developer backdoor             | `xforgexxcoder22@gmail.com` hardcoded                                                            | Not observed (different kit)                                                                           |
| Cloudflare account             | `c08d6ea6432746b9a07fe27c72f8ef1c`                                                               | `03ae8bcbdb8d46c1a1f72420e776adb5`                                                                     |


---

## 6. Infrastructure

### 6.1 Domain Cluster


| Domain     | Cloudflare Status (Jun 2026)    | Notes                                                                                                                                                                                                                                                                                                                                                                                                                                                 |
| ---------- | ------------------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `mesha.vu` | **Phishing flagged**            | Cloudflare interstitial; Turnstile bypass URL confirms `/invite/` path (cf-ray `a0a54b945c33d697`)                                                                                                                                                                                                                                                                                                                                                    |
| `meshb.vu` | Active - 403 CF challenge       | Primary domain; kit fully analyzed in this investigation                                                                                                                                                                                                                                                                                                                                                                                              |
| `meshc.vu` | 403 CF / no kit                 | Cloudflare challenge present; `/invite/` returns 404 and root returns blank page after challenge - domain reserved, no kit deployed                                                                                                                                                                                                                                                                                                                   |
| `meshd.vu` | 403 CF / no kit                 | Same as above                                                                                                                                                                                                                                                                                                                                                                                                                                         |
| `meshe.vu` | **Phishing flagged**            | Cloudflare phishing interstitial - kit was previously deployed, now burned                                                                                                                                                                                                                                                                                                                                                                            |
| `meshf.vu` | 403 CF / no kit                 | Domain reserved, no kit deployed                                                                                                                                                                                                                                                                                                                                                                                                                      |
| `meshg.vu` | 403 CF / no kit                 | Domain reserved, no kit deployed                                                                                                                                                                                                                                                                                                                                                                                                                      |
| `meshh.vu` | No DNS record                   | Not provisioned                                                                                                                                                                                                                                                                                                                                                                                                                                       |
| `meshi.vu` | **Phishing flagged**            | Cloudflare phishing interstitial - kit was previously deployed, now burned                                                                                                                                                                                                                                                                                                                                                                            |
| `meshj.vu` | 403 CF / no kit                 | Domain reserved, no kit deployed                                                                                                                                                                                                                                                                                                                                                                                                                      |
| `meshk.vu` | 403 CF / no kit                 | Domain reserved, no kit deployed                                                                                                                                                                                                                                                                                                                                                                                                                      |
| `meshl.vu` | No DNS record                   | Not provisioned                                                                                                                                                                                                                                                                                                                                                                                                                                       |
| `meshm.vu` | 403 CF / no kit                 | Domain reserved, no kit deployed                                                                                                                                                                                                                                                                                                                                                                                                                      |
| `meshn.vu` | 403 CF / no kit                 | Domain reserved, no kit deployed                                                                                                                                                                                                                                                                                                                                                                                                                      |
| `mesho.vu` | **Phishing flagged**            | Cloudflare phishing interstitial - kit was previously deployed, now burned                                                                                                                                                                                                                                                                                                                                                                            |
| `meshp.vu` | 403 CF / no kit                 | Domain reserved, no kit deployed                                                                                                                                                                                                                                                                                                                                                                                                                      |
| `meshq.vu` | 403 CF / no kit                 | Domain reserved, no kit deployed                                                                                                                                                                                                                                                                                                                                                                                                                      |
| `meshr.vu` | 403 CF / no kit                 | Domain reserved, no kit deployed                                                                                                                                                                                                                                                                                                                                                                                                                      |
| `meshs.vu` | 403 CF / no kit                 | Domain reserved, no kit deployed                                                                                                                                                                                                                                                                                                                                                                                                                      |
| `mesht.vu` | **Kit confirmed active**        | Custom reCAPTCHA + 4-stage login kit; full attack chain captured; beacon token `03ae8bcbdb8d46c1a1f72420e776adb5`                                                                                                                                                                                                                                                                                                                                     |
| `meshu.vu` | **Phishing flagged**            | Cloudflare phishing interstitial - kit was previously deployed, now burned                                                                                                                                                                                                                                                                                                                                                                            |
| `meshv.vu` | 403 CF / no kit                 | [Gridinsoft](https://gridinsoft.com/online-virus-scanner/url/meshv-vu) 1/100 trust; `check.meshv.vu` also flagged; kit not confirmed behind challenge                                                                                                                                                                                                                                                                                                 |
| `meshw.vu` | 403 CF / no kit                 | Domain reserved, no kit deployed                                                                                                                                                                                                                                                                                                                                                                                                                      |
| `meshx.vu` | **Phishing flagged**            | Cloudflare phishing interstitial; [Gridinsoft](https://gridinsoft.com/online-virus-scanner/url/meshv-vu) 1/100 trust - kit was previously deployed                                                                                                                                                                                                                                                                                                    |
| `meshy.vu` | 403 CF / no kit                 | Domain reserved, no kit deployed                                                                                                                                                                                                                                                                                                                                                                                                                      |
| `meshz.vu` | 403 CF / no kit                 | [Gridinsoft](https://gridinsoft.com/online-virus-scanner/url/check-meshz-vu) 1/100 trust; `check.meshz.vu` flagged; kit not confirmed behind challenge                                                                                                                                                                                                                                                                                                |
| `siodg.vu` | **Confirmed - two kits active** | Evitree kit at `/docs/` (same static-capture fingerprint as `meshb.vu`; shared Cloudflare beacon `50e4918df1e5493a9fbf6b916d373d02`) + multi-brand kit at `/rsvp/` (Adobe lure, `processmail.php`/`process.php`, *same* beacon token); developer reference/staging domain (`siodg.vu/Evitree/`, [Joe Sandbox 1919382](https://www.joesandbox.com/analysis/1919382/0/html)); shared beacon token across both kits links both to one Cloudflare account |
| `bytez.vu` | **Evitree kit confirmed**       | Identical static-capture fingerprint as `meshb.vu`; same hardcoded `usi`, `fldUsername`, `region="US"`, `fidoUserHandle`, `pass.php?id={MD5}` form action; same operator JS (`validateInput`, focus-trap, `startDelayedSubmission`) as `meshb.vu` index stage and `siodg.vu/docs/`; `check.bytez.vu` flagged by Gridinsoft (June 2026)                                                                                                              |
| `jsdns.vu` | Confirmed phishing              | Gridinsoft (2 blacklist hits, domain ~8 days old at detection); consistent with cluster operator pattern                                                                                                                                                                                                                                                                                                                                              |


**Full-alphabet pre-provisioning.** DNS enumeration of all 26 single-letter variants confirms that **24 of 26 `mesh*.vu` domains resolve** (all Cloudflare anycast IPs in the `104.21.x.x` / `172.67.x.x` ranges). Only `meshh.vu` and `meshl.vu` have no DNS record. This is not opportunistic domain cycling - the entire alphabet appears pre-provisioned as a rotation pool.

**Operational model: pre-registered pool, selective kit deployment.** The 403 challenge is present across all provisioned domains but does not indicate an active kit on each. Browser-level probing (solving the Cloudflare challenge) on the majority of domains yields a 404 at `/invite/` and a blank page at the root - domain infrastructure is live but no kit is deployed. The operator's model is to register the full alphabet in advance, deploy a kit to a domain when needed, and retire it when burned. Active deployments confirmed as of Jun 2026: `meshb.vu` (Evitree kit) and `mesht.vu` (custom reCAPTCHA kit).

**6 domains carry Cloudflare phishing interstitials** (`mesha`, `meshe`, `meshi`, `mesho`, `meshu`, `meshx`) - these were previously active deployments, now flagged and burned. These domains remain live but Cloudflare warns visitors before allowing access.

`meshv.vu` and `meshz.vu` carry independent phishing classifications (Gridinsoft, 1/100 trust). Both were registered through **Sav.com LLC** and both expose separately flagged `check.`* subdomains. Gridinsoft recorded their TLS certificates as issued within two minutes of each other on May 23, 2026 (12:04 and 12:06 UTC); this is suggestive of the two zones being added to Cloudflare in the same window, though certificate timing is Cloudflare-controlled (Appendix B) and is weak evidence on its own. The stronger shared signal is the common registrar and the identical `check.*` subdomain convention.

The `mesh*.vu` naming pattern (full alphabet registered via Sav.com LLC, all behind Cloudflare) represents a substantial pre-committed infrastructure investment, consistent with a long-running operation that burns domains on detection and rotates to the next letter.

**Sibling domains: `bytez.vu` and `jsdns.vu`.** `bytez.vu` is a confirmed Evitree deployment outside the `mesh*.vu` namespace (Section 4.2). `jsdns.vu` (Gridinsoft, 2 blacklist hits) shares the Sav.com + Cloudflare profile. The `check.*` subdomain convention (`check.meshv.vu`, `check.meshz.vu`, `check.bytez.vu`) is a shared operational fingerprint.

**TLS certificates are not a reliable pivot.** Certificate issuer and issuance time on Cloudflare-proxied domains reflect Cloudflare Universal SSL automation, not operator choice. Do not treat mixed Google Trust / Let's Encrypt issuance as evidence of a deliberate batch. Details in Appendix B.

**Multiple kits on the same pool.** `meshb.vu` (Evitree) and `mesht.vu` (custom reCAPTCHA kit, Section 5) use different Cloudflare Browser Insights beacon identifiers, which may indicate separate operators renting the same `mesh*.vu` rotation pool - or one operator with multiple Cloudflare accounts.

### 6.2 Hosting

- **CDN/Protection:** Cloudflare (anycast; IPs `172.67.172.146`, `104.21.80.11`; IPv6 `2606:4700:303x::` range)
- **Registrar:** Sav.com LLC (IANA ID 609) - cheap bulk registrar used across the `mesh*.vu` cluster
- **TLD:** `.vu` (Vanuatu - APNIC region; historically limited abuse-response infrastructure)
- **SSL Certificate:** Issued by Google Trust Services (CN `WE1`, ACME-issued), valid June 5 - September 3, 2026. SANs cover `meshb.vu` and the wildcard `*.meshb.vu` (the wildcard explains the live `check.meshb.vu` subdomain, which also returns a Cloudflare `403`). A Google Trust cert does not imply Google affiliation; it is auto-issued for any domain.
- **Cloudflare edge PoP:** Washington Dulles (IAD) - `cf-ray: a0a477f57dd556ce-IAD`

### 6.3 Operator Infrastructure Fingerprints

**Cloudflare Browser Insights beacon identifiers** are account-scoped tracking tokens embedded in page JavaScript (`beacon.min.js`). They identify a Cloudflare account, not an individual operator, but are useful for correlating domains hosted under the same account.

These artifacts uniquely identify operator Cloudflare accounts without identifying anyone by name:


| Artifact                                                           | Value                                      | Significance                                                                                                                                                                                                                      |
| ------------------------------------------------------------------ | ------------------------------------------ | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Cloudflare Beacon Token (`meshb.vu` operator)                      | `c08d6ea6432746b9a07fe27c72f8ef1c`         | Scoped to one Cloudflare account; present across `meshb.vu` pages                                                                                                                                                                 |
| Cloudflare Beacon Token (`mesht.vu` operator)                      | `03ae8bcbdb8d46c1a1f72420e776adb5`         | Different account from `meshb.vu`; present on `mesht.vu`; may indicate second operator using the cluster                                                                                                                          |
| Cloudflare Beacon Token (`siodg.vu` operator - shared across kits) | `50e4918df1e5493a9fbf6b916d373d02`         | Identical token confirmed on both `siodg.vu/docs/` (Evitree kit) and `siodg.vu/rsvp/` (multi-brand Adobe kit); since Cloudflare beacon tokens are scoped to a single account, both kits operate under the same Cloudflare account |
| reCAPTCHA site key (`mesht.vu`)                                    | `6LfTPlUUAAAAAGSUt1_LqpJXQpatx7_BzTDcU9On` | Registered to `mesht.vu` operator; used as anti-analysis gate and social engineering on fake Google verification page                                                                                                             |
| NEL Endpoint `s=` parameter                                        | (present in response headers)              | Cloudflare account-scoped Network Error Logging endpoint; usable for abuse escalation to Cloudflare                                                                                                                               |


### 6.4 Relationship to the Broader Invitation-Phishing Campaign

The Evitree kit's server-side fingerprint matches a large-scale, publicly documented phishing operation. In April 2026, [ANY.RUN](https://any.run/cybersecurity-blog/us-fake-invitation-phishing/) documented a US-targeted campaign that lures victims with fake event invitations (impersonating **Evite, Paperless Post, and Punchbowl**), funneling them through a Cloudflare challenge to multi-brand credential-harvesting pages. The same campaign was independently covered by [Sublime Security](https://sublime.security/blog/impersonated-evite-and-punchbowl-invitations-used-for-credential-phishing-and-malware-distribution/), Cofense, and an [FTC consumer warning](https://www.techtimes.com/articles/317390/20260529/fake-party-invitation-phishing-scam-spoofs-google-microsoft-oauth-logins-ftc-warns.htm).

**The linking evidence is the exact Google-flow endpoint triad.** ANY.RUN attributes the Google-credential path of that campaign specifically to `/pass.php` + `/mlog.php` + `/check_telegram_updates.php` - the precise three endpoints observed on `meshb.vu`. This is a high-specificity match.


| Campaign attribute (ANY.RUN et al.)                                                         | This deployment (`meshb.vu`)                                                                                                                                                                                                                                                    |
| ------------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Google flow: `pass.php` + `mlog.php` + `check_telegram_updates.php`                         | Exact match (observed live)                                                                                                                                                                                                                                                     |
| Lure theme: fake event/party invitations                                                    | Match (Google Docs / "Cordially Invited" invite lure)                                                                                                                                                                                                                           |
| Cloudflare challenge gating                                                                 | Match (`403 cf-mitigated: challenge`)                                                                                                                                                                                                                                           |
| Telegram real-time exfiltration                                                             | Match (`check_telegram_updates.php`)                                                                                                                                                                                                                                            |
| Primary TLD `.de`, ~80 domains from Dec 2025                                                | Partial - sibling `cehvfihys.de` fits; `mesh*.vu` appears to be a related parallel sub-cluster not yet enumerated in the `.de`-focused public reporting                                                                                                                         |
| Multi-brand (Google/Microsoft/Yahoo/AOL) + RMM delivery via `processmail.php`/`process.php` | **Confirmed on same operator cluster** - `meshb.vu` is the Google-credential variant; multi-brand Adobe-lure kit with `processmail.php`/`process.php` endpoints confirmed live at `siodg.vu/rsvp/` (see 6.5), sharing the same Cloudflare account as `siodg.vu/docs/` (Evitree) |


**Assessment.** The `meshb.vu` Evitree deployment is consistent with being one node of this broader operation, sharing its Google-credential kit fingerprint. The multi-brand and RMM-delivery behaviors documented campaign-wide were *not* observed on `meshb.vu` itself, but are confirmed present in the same cluster at `siodg.vu/rsvp/` (see 6.5). (`web3.meshwallets.com` is a superficial name match only - an unrelated crypto-drainer on a different registrar and host - and is **not** part of this cluster.)

### 6.5 `siodg.vu/rsvp/` - Multi-Brand Adobe Document Lure Kit

A second, structurally distinct phishing kit was discovered live at `https://siodg.vu/rsvp/`. This is **not** an Evitree variant; it is a separate kit co-hosted on the same operator domain.

**Lure and social engineering.** The landing page impersonates **Adobe Document Cloud**, displaying an Adobe logo and the message: *"To read the document, please enter with the valid email credentials that this file was sent to."* The victim selects their email provider from five brand buttons, which triggers a Bootstrap modal overlay.

**Provider selection.** Five email brands are offered, each with a matching logo asset:

- Outlook (`Image/office360.png`)
- Office365 (`Image/office.png`)
- Yahoo Mail (`Image/yahoo.png`)
- AOL (`Image/aol.png`)
- Other Mail (`Image/email.png`)

**Credential and OTP capture.** After brand selection, a modal form (Stage 1) posts to `processmail.php` via jQuery AJAX. The first submission *always* returns an "Incorrect Password" error regardless of input - a deliberate double-submission trap that captures the first password attempt and forces a second entry. On the second submission, the modal is dismissed and a second modal presents an OTP field that posts to `process.php`. The OTP `process.php` handler was commented out in the analyzed source, indicating the kit was in active development or the operator had not yet configured the OTP stage.

**Technical stack and client-side artifacts:**

- Bootstrap 4.0.0 (CDN: `maxcdn.bootstrapcdn.com`)
- Font Awesome 6.4.0 (CDN: `cdnjs.cloudflare.com`)
- jQuery 3.1.1 (CDN: `ajax.googleapis.com`)
- Background image: `Image/bg1.PNG` (blurred via CSS `filter: blur(8px)`)
- Brand icons served from `Image/` directory (local, not CDN)
- Form ID: `formSubmit` (Stage 1), `secondForm` (Stage 2)
- Hidden input `name="id"` passes brand-selection numeric ID (1-5) to `processmail.php`

**Cloudflare beacon - shared with Evitree kit.** The `siodg.vu/rsvp/` page loads the *identical* Cloudflare Beacon token (`50e4918df1e5493a9fbf6b916d373d02`) as the `siodg.vu/docs/` Evitree deployment (Section 6.3 row 3). Since Cloudflare beacon tokens are scoped to a single Cloudflare account, both kits are operating under the same Cloudflare account. The most parsimonious interpretation is that a **single operator** is running both the Evitree (Google-credential) kit and this multi-brand Adobe-lure kit on the same domain - consistent with the ANY.RUN campaign documentation (Section 6.4) that attributes both kit types to the same invitation-phishing operation.

**Relationship to the `mesh*.vu` Evitree cluster.** `siodg.vu` is registered via Sav.com LLC, sits behind Cloudflare, and shares the `50e4918df1e5493a9fbf6b916d373d02` beacon token with the Evitree instance at `siodg.vu/docs/`. It was already identified as the likely developer staging/reference domain (Joe Sandbox `siodg.vu/Evitree/` analysis). The presence of a second kit variant there is consistent with a PhaaS operator testing or deploying multiple kit modules from the same staging host.

---

## 7. Detection Notes

Practical hunt and triage signals from this investigation - specifics from the lure and kit.

### 7.1 Email Delivery

- **Future-dated `Date` header** relative to SMTP receipt (this lure: ~8h51m ahead; Gmail flagged `Delivered after -31910 seconds`)
- **BCC-only mass send** (`undisclosed-recipients:;` with targets in Bcc)
- **Fabricated Google threading** - `References`/`In-Reply-To` using `@docs-share.google.com` or `@google.com` Message-IDs to land in existing Google Docs conversation threads
- **Authentication passes on a compromised or throwaway Gmail** - SPF/DKIM/DMARC all pass; auth-only filtering misses this
- **Template clone with wrong links** - real Google CDN assets (`lh3.googleusercontent.com`, `ssl.gstatic.com`) but hrefs point to non-Google domains (`.vu`, `/invite/`)

### 7.2 Network and DNS

- **`.vu` domains** with `/invite/` path (Evitree convention on this cluster)
- **Evitree endpoint set:** `pass.php`, `session/mlog.php`, `check_telegram_updates.php`, `check_json_status.php`, `load.php`
- **Form action pattern:** `pass.php?id={32-char MD5}`
- **`check.*` subdomains** on cluster domains (`check.meshb.vu`, `check.bytez.vu`)
- **Cloudflare Browser Insights beacon identifiers** - account-scoped; values in Section 8.2 and Appendix A

### 7.3 Page Source (Evitree)

- Email field name **`fldUsername`** (not Google's `identifier`)
- Hidden inputs **`identifierId`** / **`hiddenEmail`** containing a non-victim email (developer backdoor - see Section 4.2)
- **Empty `nonce=""`** on all script tags
- **Hardcoded `usi`** value `S-233362078:1741544750655901` (March 9, 2025 snapshot)
- Static **`region="US"`** in form
- Decoy blank **`fidoUserHandle`** input
- **`redirect-checker.js`** on loading stage
- Console errors: BotGuard `Error: pb`, broken `onCssLoad` / `ccTick` / `onJsLoad`

### 7.4 Victim Behavior

- **Closing the tab after password submit does not prevent capture** - Evitree exfiltrates via AJAX before the loading animation (Section 2)
- Post-submit **polling** to `check_json_status.php` and `check_telegram_updates.php` indicates operator-in-the-loop routing

### 7.5 Post-Incident

- Review for **new OAuth grants** or **impossible travel** after any suspected credential submission
- This kit is a **static rip**, not AiTM - it harvests credentials at input time; it does not proxy live sessions or bypass MFA (Section 4.4)

---

## 8. Attribution

### 8.1 Kit Developer


| Attribute          | Value                                         | Confidence                                                                                               |
| ------------------ | --------------------------------------------- | -------------------------------------------------------------------------------------------------------- |
| Developer email    | `xforgexxcoder22@gmail.com`                   | High - confirmed in live code and independent sandbox reports                                            |
| Kit name           | Evitree                                       | High - explicitly named in Joe Sandbox report for `siodg.vu/Evitree/`                                    |
| Distribution model | Consistent with phishing-as-a-service (PhaaS) | Inferred - developer backdoor pattern is common in commercially distributed kits; not directly confirmed |


Evidence for the developer backdoor and cross-deployment confirmation is in Section 4.2. Where `pass.php` was recovered, the developer's email appears as a hardcoded recipient - meaning any operator running the kit may deliver a silent copy of harvested credentials to the kit author. Whether operators are told about this is unknown.

**TTP precedent (not Evitree-specific).** Fortinet's 2018 analysis of the [Bindweed](https://www.fortinet.com/blog/threat-research/bindweed--digging-down-to-a-root-of-a-hidden-phishing-network) phishing network traced hundreds of Dropbox credential-harvesting pages to a single published kit ZIP. Inside that kit, the original publisher had hardcoded their own email addresses into the PHP exfiltration scripts - distinct from the addresses used by the operators actually running the campaign. Fortinet noted these could be genuine publisher backdoors or deliberate false-flag inserts. The pattern is the same class of artifact as Evitree's `xforgexxcoder22@gmail.com` backdoor: a commercially distributed kit that silently copies harvested credentials to the kit author. Bindweed is a Dropbox-targeting kit with no observed link to Evitree, `xforgexxcoder22@gmail.com`, or this `.vu` cluster; it is cited here only as independent precedent for the PhaaS backdoor TTP.

### 8.2 Campaign Infrastructure Artifacts

The following artifacts are scoped to the operator's accounts and can be used to correlate other domains or campaigns run by the same actor:


| Artifact                             | Value                                                                          | Use                                                                                                                                                                                                                                                        |
| ------------------------------------ | ------------------------------------------------------------------------------ | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Cloudflare beacon token (`meshb.vu`) | `c08d6ea6432746b9a07fe27c72f8ef1c`                                             | Unique to one Cloudflare account; search for this value across other suspected phishing domains to identify shared infrastructure                                                                                                                          |
| Cloudflare beacon token (`siodg.vu`) | `50e4918df1e5493a9fbf6b916d373d02`                                             | Shared across `siodg.vu/docs/` (Evitree) and `siodg.vu/rsvp/` (multi-brand); distinct from `meshb.vu` operator                                                                                                                                             |
| Cloudflare beacon token (`mesht.vu`) | `03ae8bcbdb8d46c1a1f72420e776adb5`                                             | Distinct third Cloudflare account on the same domain cluster                                                                                                                                                                                               |
| Telegram C2                          | `check_telegram_updates.php` returned `1` (bot live)                           | Confirms active Telegram bot at time of analysis; bot token embedded server-side only                                                                                                                                                                      |
| Delivery account                     | `donitaturk@gmail.com`                                                         | Email distribution vector - compromised or throwaway; low correlation value on its own                                                                                                                                                                     |
| Endpoint fingerprint                 | `pass.php` + `mlog.php` + `check_telegram_updates.php`                         | High-specificity match to the ANY.RUN-documented invitation-phishing campaign's Google-credential flow (see 6.4)                                                                                                                                           |
| Registrar pattern                    | Sav.com LLC (IANA 609) across the `mesh*.vu` cluster and sibling `.vu` domains | Pivot to find sibling `mesh*.vu` and related cluster domains. Note: TLS certificate issuer/timing is *not* a reliable pivot - certs are auto-provisioned by Cloudflare Universal SSL (mixed Google Trust / Let's Encrypt) and are not operator-controlled. |


---

## Appendix A: Indicators of Compromise

### Domains

`mesh*.vu` cluster (24 of 26 letters provisioned; `meshh.vu` and `meshl.vu` have no DNS record):

- `mesha.vu` (confirmed phishing - Cloudflare interstitial; `/invite/` path confirmed)
- `meshb.vu` (primary, directly analyzed)
- `meshc.vu` `meshd.vu` `meshe.vu` `meshf.vu` `meshg.vu` (live, probable)
- `meshi.vu` `meshj.vu` `meshk.vu` (live, probable)
- `meshm.vu` `meshn.vu` `mesho.vu` `meshp.vu` `meshq.vu` `meshr.vu` `meshs.vu` `mesht.vu` `meshu.vu` (live, probable)
- `meshv.vu` (confirmed phishing - Gridinsoft)
- `meshw.vu` `meshx.vu` `meshy.vu` (live, probable)
- `meshz.vu` (confirmed phishing - Gridinsoft)
- `siodg.vu` (developer reference / staging; two kits active)
- `bytez.vu` (confirmed Evitree deployment)
- `jsdns.vu` (confirmed phishing - Gridinsoft)
- Subdomains: `check.meshv.vu`, `check.meshz.vu`, `check.meshb.vu`, `check.bytez.vu`

**`mesht.vu` kit URLs:**

- `https://mesht.vu/invite/` (reCAPTCHA gate)
- `https://mesht.vu/invite/pages/login.php` (email + password collection)
- `https://mesht.vu/invite/update_online.php` (session heartbeat, 3s interval)
- `https://mesht.vu/invite/poll_status.php` (operator command channel, 2s interval)
- `https://mesht.vu/invite/pages/otp.php` (probable OTP collection - inferred)
- `https://mesht.vu/invite/pages/wrong.php` (probable incorrect-password loop - inferred)
- `https://mesht.vu/invite/pages/success.php` (probable exit/redirect - inferred)

Other confirmed Evitree deployments (`pass.php` backdoor confirmed unless noted):

- `lifn.digital/myin/myinvite/`
- `geteviteflow.com/evite/`
- `cehvfihys.de/invitation/Login/gmail/` (structural match)
- `ciotg.vu/Dateinvite/` (same cluster; structural match)
- `bytez.vu/check.bytez.vu/ru/vib/iv/accounts.s/` (static-capture fingerprint; backdoor email not recovered)
- `siodg.vu/docs/` (static-capture fingerprint; backdoor email not recovered; shared beacon with `/rsvp/`)

**`siodg.vu` multi-brand kit URLs (`/rsvp/`):**

- `https://siodg.vu/rsvp/` (Adobe Document Cloud lure)
- `https://siodg.vu/rsvp/processmail.php` (credential submission)
- `https://siodg.vu/rsvp/process.php` (OTP submission)

Related broader campaign (ANY.RUN, examples only):

- `festiveparty.us`
- `getceptionparty.de`
- `celebratieinvitiee.de`

### Email Addresses

- `donitaturk@gmail.com` - phishing delivery account
- `xforgexxcoder22@gmail.com` - kit developer backdoor (see Section 4.2)

### URLs

**`meshb.vu` (Evitree):**

- `https://meshb.vu/invite`
- `https://meshb.vu/invite/pass.php`
- `https://meshb.vu/invite/session/mlog.php`
- `https://meshb.vu/invite/check_json_status.php`
- `https://meshb.vu/invite/check_telegram_updates.php`
- `https://meshb.vu/invite/load.php`

**`bytez.vu` (Evitree):**

- `https://bytez.vu/check.bytez.vu/ru/vib/iv/accounts.s/`

**`siodg.vu` (Evitree + multi-brand):**

- `https://siodg.vu/docs/`
- `https://siodg.vu/rsvp/`

### Tokens / Hashes

- Cloudflare Browser Insights beacon (`meshb.vu` operator): `c08d6ea6432746b9a07fe27c72f8ef1c`
- Cloudflare Browser Insights beacon (`mesht.vu` operator): `03ae8bcbdb8d46c1a1f72420e776adb5`
- Cloudflare Browser Insights beacon (`siodg.vu` operator): `50e4918df1e5493a9fbf6b916d373d02`
- reCAPTCHA site key (`mesht.vu` kit): `6LfTPlUUAAAAAGSUt1_LqpJXQpatx7_BzTDcU9On`
- PHPSESSID (observed): `0ee466ff7c5a1b812510bbb03053434b`
- visitor_key cookie (observed): `bab9fdb3a634c852fa9d9ff939803ef8`

### File Indicators

- `assets/CheckConnection.html` and `assets/bscframe.html` in Google Sign-In lookalikes
- Hidden input `id="identifierId"` containing non-victim email
- Hidden input `id="hiddenEmail"` with same value
- Form action: `pass.php?id={32-char MD5}`
- Script src `redirect-checker.js` on loading pages
- `usi` value containing timestamp `1741544750655901` (March 9, 2025)

### Campaign-Level Indicators

Google-credential flow endpoints:

- `pass.php`, `session/mlog.php`, `check_telegram_updates.php`, `check_json_status.php`, `load.php`

Additional endpoints (confirmed on `siodg.vu/rsvp/`; not on `meshb.vu`):

- `processmail.php`, `process.php`, `blocked.html`, `Image/*.png`

Threat-hunting query (ANY.RUN TI Lookup):

```
url:"/blocked.html" AND url:"/favicon.ico" AND url:"/Image/*.png"
```

---

## Appendix B: TLS Certificate Notes

Live certificate inspection (June 2026) across the cluster:

| Domain | CA | Issued |
| ------ | -- | ------ |
| `meshb.vu`, `mesht.vu` | Google Trust Services WE1 | June 5, 2026 (~19 minutes apart) |
| `siodg.vu` | Let's Encrypt E8 | May 24, 2026 |
| `ciotg.vu` | Let's Encrypt E8 | May 11, 2026 |
| `bytez.vu` | Let's Encrypt E8 | May 25, 2026 |

This mix is expected Cloudflare Universal SSL behavior - certificates are auto-provisioned from Google Trust or Let's Encrypt at Cloudflare's discretion, not under operator control. The E7/E8/WE1 labels in third-party scanners are CA intermediate names. Near-simultaneous issuance for `meshb.vu` and `mesht.vu` is mild evidence those zones were onboarded in the same window, not evidence of a deliberate operator batch.

---

*Report prepared from direct artifact analysis: original phishing email headers, live kit walkthrough on `mesh*.vu` deployments (fake credentials submitted in-browser after Cloudflare challenge), rendered HTML and network traffic from all four Evitree stages, browser console logs, and live DNS/HTTP probes of the cluster (June 2026). Corroborated by Joe Sandbox automated analyses, Gridinsoft URL reputation data, public reporting on the broader invitation-phishing campaign (ANY.RUN, Sublime Security, FTC), Push Security's June 2026 analysis of real-time operator phishing panels, and Fortinet's Bindweed kit analysis as TTP precedent for developer backdoors in commercially distributed phishing kits.*
