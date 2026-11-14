---
name: try-to-hack-me
description: Attack simulation against the user's own web project. Looks for common hacker entry points (XSS, injection, SSRF, path traversal, broken access control, open redirects, clickjacking, CSRF and CORS, missing security headers, exposed secrets, unsafe uploads, weak sign-in, JWT and OAuth mistakes, vulnerable dependencies, business logic flaws, and DDoS exposure), proves each hole with harmless probes, closes it, adds a regression test, and repeats until a full pass finds nothing open. Use this skill whenever the user asks for a security check, a pentest, a vulnerability search, "try to hack my site", "find holes", "make it secure", "bring it to zero", or asks to harden a site or app, even if they don't use the word "security".
---

# Try to Hack Me

Act as an attacker against the user's own project to find holes, then close them and prove they are closed. Keep fixes small and simple. In fix mode, keep going until a full pass finds zero open items in the tested area.

## Scope and safety, always

- Test only the user's own project: its files, and a local server you start yourself (localhost). Never probe a live site, server, or third-party service, even if the user pastes its address, unless they state it is their own and the environment allows it. Never run floods, brute-force attempts, or anything that could take a service down or touch real accounts and data.
- Use minimal, harmless probes: unique marker strings and standard test inputs such as `"><img src=x onerror=window.__xss=1>` to see whether input is escaped. Do not write exploit tools, scanners, brute-forcers, payload collections, or malware.
- Existing, well-known defensive tools may be run against the project and localhost: `semgrep`, `npm audit` (or the ecosystem equivalent), `gitleaks` or `trufflehog` (including git history), `eslint-plugin-security`, and a headless browser (Playwright) to observe whether a probe executed. Install them only through standard package registries. Never use them against anything but the local project, and never use offensive scanners, fuzzers, or exploit frameworks.
- Never run a DDoS or load test, not even against the user's own site or a local copy: no request floods, no concurrency ramps, no stress-testing tools. A real denial-of-service test needs dedicated infrastructure and the hosting provider's permission, and doing it from here could take the site down or look like an attack to the host or their upstream network. DDoS is reviewed as a readiness check (below), never simulated.
- Rate-limit and lockout checks are capped at 15 sequential requests to localhost, one connection at a time, only to see whether a 429 or lockout appears. Never go above the cap, never parallelize.
- Don't read, send, or store real secrets or personal data found along the way. Report where they are, not what they are.
- Comments: don't add any to the code you change.
- Avoid em dashes in everything you write. Use a period, comma, colon, or parentheses.

## Audit or fix

- Audit mode: the user says check, test, audit, find holes, or asks what is wrong. Don't edit, create, or delete project files. Starting a local server and sending probes is fine if it changes nothing in the project. Probes that write data (stored XSS and similar) run only against a temporary copy or a throwaway data store; if that isn't possible, list them as untested. End with a report and an offer to apply the fixes.
- Fix mode: the user says fix, harden, secure, close the holes, "close everything", "bring it to zero", or similar. Apply the fixes and run the loop below.
- If the wording is ambiguous, default to audit mode and ask before editing.

## Method

1. Read the project: entry points, config, `package.json`, lockfile, server code if present, hosting config, git history for secrets.
2. Map the attack surface: every input (forms, URL parameters, hash, cookies, headers, `postMessage`, WebSocket messages, uploads, fetched data, exports), every dangerous sink, every endpoint, storage use, and third-party script. Build the route list with the roles that should reach each route.
3. Run the existing tools (semgrep, dependency audit, secret scan, eslint security rules) and add every result to the findings registry. Treat tool output as leads, then confirm each one by hand.
4. Trace each input to its sink in the code and note where it is not validated or encoded.
5. If the project runs locally, send the harmless probes through the real inputs and check the result in the raw response, the DOM, and (for XSS) whether the headless browser actually executed it. Put a unique marker in every probe so test data can be found later.
6. Clean up. If a probe wrote data (a local database, a JSON file, `localStorage`, an uploads folder), delete exactly the records and files carrying your marker once the check is done. Then confirm with `git status` or a before/after comparison that the project holds no leftover test data.
7. In fix mode, fix each confirmed hole with the smallest change that closes the whole class, not only the one payload. Add a regression test for it, repeat the same probe, confirm it is closed, and clean up again. Check that the fix didn't break the page or the existing tests.
8. Loop. After a fix round, repeat steps 2 to 7 over the whole project, not only the touched files, because a fix can open a new hole or expose one that was hidden behind it. Stop when a full pass produces no new `open` finding. Hard cap: 5 passes. If the cap is hit with open items left, report them as unresolved with the reason.
9. Report every finding.

If the project can't run locally (no `.env`, an external database, missing services, or only fragments of a project), don't burn attempts on debugging the environment. Try at most a couple of quick setup steps, using obvious dummy values in a temporary setup and never real credentials. If it still won't start, switch at once to full static analysis: trace the data flow from every input to every sink by reading the code, and list the dynamic checks you skipped under "could not be tested".

## Findings registry and what "zero" means

Keep a registry through the whole run. Every finding has an id, a class, file and line, and exactly one status:

- `open`: confirmed or strongly suspected, not yet closed.
- `fixed+verified`: fixed, the original probe now fails, a regression test exists and passes.
- `needs-host-config`: closing it requires a change outside the project (CDN, host panel, DNS, database permissions, secret rotation). Give the exact setting or file, and where possible put a ready config in the project.
- `untestable`: could not be checked here, with the reason.

"Zero" means no `open` findings in the tested area after a clean full pass. Never describe the project as having zero vulnerabilities. Say "0 open in the tested area" and list every `needs-host-config` and `untestable` item next to it.

## Dangerous sinks to look for

`innerHTML`, `outerHTML`, `insertAdjacentHTML`, `document.write`, `eval`, `new Function`, `setTimeout` or `setInterval` with a string, `dangerouslySetInnerHTML`, `location` assignments from data, string-built URLs, CSS, SQL, shell commands, or templates that receive anything non-constant, server-side `fetch()`, `axios`, or `http.request` with a URL built from input, file functions (`fs.readFile`, `createReadStream`, `sendFile`, static file serving, uploads) with a path built from input, and regular expressions applied to user input.

## Attack classes to try

In rough order of how often they appear.

**Cross-site scripting.** Reflected (URL parameters), stored (data saved then shown), and DOM-based (`location.hash`, `postMessage`, `innerHTML`). Probe every context separately, because escaping that is right for one context is wrong for another: HTML body, quoted and unquoted attribute, JavaScript string, URL, CSS, and template or framework expression. Confirm execution in a headless browser by checking the `window.__xss` flag, not by reading the source. Fix: `textContent` and `createElement`, context-correct output encoding, no string-built HTML, DOMPurify for any HTML that must stay rich. Structural close: a strict Content-Security-Policy using nonces or hashes with no `unsafe-inline` and no `unsafe-eval`, plus `require-trusted-types-for 'script'` where the browser support fits.

**Injection.** Untrusted values in URLs, CSS, HTML attributes, and, if the project has a backend, in SQL, NoSQL operators (`$ne`, `$gt` in JSON bodies), shell commands, LDAP, header values (CRLF), and templates. Fix: parameterized queries, schema validation that rejects objects where strings are expected, allowlists, no string concatenation, no shell with interpolated input.

**Unverified webhooks and callbacks.** Endpoints that receive payment, auth, or third-party events (`/webhook`, `/callback`) and trust the JSON body without verifying the provider's cryptographic signature against the raw request body, or without checking the event timestamp. Fix: verify the HMAC or signature header against the raw body using a constant-time comparison, reject stale timestamps to prevent replay attacks, and record processed event IDs for idempotency.

**SSRF (server-side request forgery).** The server fetches a URL that comes from a user: avatar by URL, link previews, webhooks, import from URL, image proxies. Attackers point it at internal targets such as `127.0.0.1`, `localhost`, `169.254.169.254` (cloud metadata), and private ranges (`10.0.0.0/8`, `172.16.0.0/12`, `192.168.0.0/16`, and their IPv6 equivalents), including through decimal or hex IP forms, DNS rebinding, and redirects. Fix: allow only `http:` and `https:`, prefer an allowlist of hosts, resolve the hostname and block loopback, link-local, and private addresses before connecting and connect to the resolved address, re-check on every redirect (or disable redirects), set timeouts and size limits, and never return raw internal responses to the user. Review the code for this; don't send requests to real internal addresses.

**Path traversal.** User input reaches a file path (downloads, uploads, static serving, templates, `?file=` parameters) with `../`, encoded variants (`%2e%2e%2f`, double encoding), absolute paths, backslashes, or null bytes. Fix: resolve the final path with `path.resolve()` (and `fs.realpath()` to follow symlinks) and require that it stays strictly inside the intended folder (compare with the base folder plus a path separator), reject anything else, and generate server-side file names instead of using user-supplied ones. Don't serve dotfiles or source folders.

**Broken access control, data exposure, and database leaks.** Logic bugs that no scanner finds, so test them against a matrix:

- Build a table of every route and action against every role (anonymous, user A, user B, admin) with the expected result, then send the real requests locally and compare the actual result. Every mismatch is a finding. Use two throwaway test users on the local instance.
- IDOR: replay user A's requests with ids that belong to user B, in the path, query, body, and headers.
- Admin routes without a server-side check, prices or roles taken from the client, mass assignment (passing a raw request body straight into a database create or update call), HTTP method tampering (a `GET` allowed where only `POST` was meant), and API responses that return full database records with sensitive fields (password hashes, email, internal flags) instead of a filtered view.
- The database itself reachable from outside: an exposed admin panel (phpMyAdmin, a database GUI), a public cloud storage bucket, or a downloadable backup or dump file.

Fix: a deny-by-default authorization middleware applied to every route, with explicit opt-out for public routes, instead of per-handler checks. Allowlist writable fields on input, return only public fields in responses, and keep databases, admin panels, and backups off the public internet or behind authentication.

**Business logic and race conditions.** Negative or zero quantities and amounts, skipping steps of a multi-step flow, reusing a one-time code or invite, applying a coupon twice, double spend through two near-simultaneous requests (send two, never a burst). Fix: server-side validation of every state transition, atomic updates or database transactions, unique constraints, and idempotency keys.

**Unsafe URLs, redirects, and links.** Validate each URL by where it is used, with an allowlist, and parse it with `new URL()` instead of checking the string.
- `a[href]`: `http:`, `https:`, `mailto:`, `tel:`, relative URLs, and `#` anchors.
- `img[src]`, `source[src]`, `video[poster]`, CSS `url()`: `http:`, `https:`, and relative URLs only. `data:` only for small images the developer embeds, never for values from users or fetched data, and never `data:image/svg+xml` from untrusted input.
- `script[src]`, `iframe[src]`, `link[href]`, `form[action]`, `fetch()` and `XMLHttpRequest` targets: `https:` or relative only, and only known hosts. Add `sandbox` to iframes with untrusted content.
- Reject `javascript:`, `vbscript:`, and `data:text/html` everywhere.
- Add `rel="noopener noreferrer"` to `target="_blank"` links. Check redirects built from query parameters for open redirects, including `//evil.example`, `\\evil.example`, and `https:evil.example` forms.

**Clickjacking.** The page can be framed by any site. Fix: `frame-ancestors` in CSP or `X-Frame-Options`.

**CSRF and CORS.** State-changing requests without tokens or `SameSite`, `Access-Control-Allow-Origin: *` together with credentials, an origin reflected back without an allowlist, and `null` origin accepted. Fix: tokens, `SameSite=Lax` or `Strict`, an explicit origin list.

**Security headers and cookies.** Check the real response, not the code: start the server and read `curl -sI` output for the main page, an API route, and a static file. Required: `Content-Security-Policy`, `Strict-Transport-Security`, `X-Content-Type-Options: nosniff`, `Referrer-Policy`, `Permissions-Policy`, and no `X-Powered-By` or version leaks in `Server`. List every `Set-Cookie` from live responses and check `HttpOnly`, `Secure`, `SameSite`, and a sensible `Path` and lifetime. A `<meta>` CSP helps on static pages, but real headers need the server or host config. In fix mode, always write the ready config into the project for the stack in use (`helmet()` for Express, `_headers` for Netlify or Cloudflare Pages, `vercel.json`, an nginx or Caddy snippet), then re-check the live response. If the real host is not the local one, mark the item `needs-host-config` and name the file to deploy.

**Exposed files and secrets.** API keys, database passwords, signing secrets, or tokens hardcoded in source or config files, a committed `.env`, environment variables copied into the browser bundle (`VITE_`, `NEXT_PUBLIC_`, `REACT_APP_`, `NUXT_PUBLIC_`, `PUBLIC_`, `window.__ENV__`, public config files), secrets that exist only in old git commits, or exposed `.git` folders, source maps, backups, debug endpoints, directory listings, and verbose error pages. Only publishable keys (such as a domain-restricted analytics or maps key) belong in the client. Fix: keep all real secrets on the server inside `.env` (or the host's secret manager) read via `process.env`, list `.env` in `.gitignore`, ship a `.env.example` with placeholder values, remove and block exposed files, and tell the user to rotate any leaked secret rather than just deleting it, including secrets found only in git history (mark these `needs-host-config` because rotation happens at the provider).

**Prototype pollution and unsafe merges.** User JSON merged into objects, keys like `__proto__` and `constructor`. Fix: schema validation, `Object.create(null)`, a key blocklist.

**Cross-window and WebSocket trust.** `postMessage` listeners that don't check `event.origin`, and WebSocket servers that don't verify the `Origin` header during the handshake, don't re-check authorization on individual socket messages, or accept unbounded message sizes (browser CORS doesn't cover WebSocket connections, so a page on any site can open one as the visitor). Fix: enforce a strict origin check on both `postMessage` and WebSocket handshakes, authenticate the session and authorize every socket action on the server, and set maximum payload limits.

**Uploads.** Type, size, and name checked only on the client, SVG or HTML files served from the same origin, content type trusted from the request instead of detected from the file, archives unpacked without path checks. Fix: server-side validation by content, a separate origin, `Content-Disposition`, generated file names.

**Weak sign-in and account takeover protection.** No rate limiting or lockout against credential stuffing, tokens in `localStorage`, passwords in URLs or logs, verbose login errors, user enumeration through different responses for existing and missing accounts (login, reset, register), passwords stored in plaintext or hashed with a fast algorithm (`MD5`, `SHA-1`, `SHA-256`), and active sessions or refresh tokens that stay valid after a password or email change. Fix: enforce server-side rate limits, store tokens in `HttpOnly` cookies, use generic error messages and equal response shapes, hash passwords with `Argon2id`, `bcrypt`, or `scrypt`, and revoke all existing sessions and tokens whenever a user changes their password or email. Check the limiter with the capped probe from the safety section; don't hammer a login.
- JWT: the server must verify the signature with a fixed algorithm allowlist (reject `alg: none` and never trust the algorithm named in the token header), and check `exp`, `nbf`, `iss`, and `aud`. Run a fixed set of negative token tests locally and expect rejection each time: `alg: none`, a tampered payload with the old signature, an expired `exp`, a wrong `iss`, a wrong `aud`, a token signed with a different key, an HS256 token signed with the public key of an RS256 setup, and a token issued before the last password change. Use short lifetimes and rotate refresh tokens. Never put tokens in URLs or logs, and never decode a token without verifying it and then trust its contents.
- OAuth and OpenID: a random `state` value is generated, stored, and checked on return (protects against login CSRF and account mix-ups), PKCE is used for public clients and single-page apps, `redirect_uri` is matched exactly against a registered list, and the `nonce` is checked for OpenID. Don't use the implicit flow.
- Weak randomness: `Math.random()`, timestamps, or sequential counters used for session ids, password reset links, invite codes, CSRF tokens, or OTPs, which are predictable. Secrets or HMACs compared with `===` instead of a constant-time function. Fix: `crypto.randomBytes()`, `crypto.randomUUID()`, or `crypto.getRandomValues()` for any security-sensitive token, and `crypto.timingSafeEqual()` for comparing digests.

**Regular expression and parser denial of service (ReDoS).** Nested quantifiers or overlapping alternations applied to user input, unbounded JSON or XML depth, decompression bombs. Find them by reading the code and with `semgrep` or `eslint-plugin-security`; confirm with one slow-input timing on a short string, never a flood. Fix: simpler patterns, input length limits before matching, a safe regex engine or a parser limit.

**Export and template injection.** CSV or spreadsheet exports where a cell starting with `=`, `+`, `-`, or `@` is executed as a formula, and server-side templates that render user input as code. Fix: prefix such cells with a single quote or tab, and pass user data to templates only as values.

**Dependencies and third-party code.** Run `npm audit` or the equivalent, including transitive packages, using the lockfile (`npm ci`, not a fresh resolve). Look for outdated libraries and CDN scripts without a pinned version or `integrity`, and for install scripts and typosquatted package names. Fix: update (`npm audit fix`, then targeted upgrades), use `overrides` or `resolutions` for a vulnerable transitive package, pin, add SRI, or remove. Remove a dependency only after the build and tests pass without it. If no patched version exists, mark it `needs-host-config` with the advisory id and the workaround.

**DDoS readiness (review only, never simulated).** Don't send load. Read the project and its deployment config and report what is and isn't in place:

- CDN or reverse proxy in front of the origin (Cloudflare, Fastly, AWS CloudFront/Shield, Vercel/Netlify edge network) so the real server IP isn't exposed and traffic is absorbed upstream. If the origin's real IP is discoverable (old DNS records, a direct IP link in the code, an email header, a subdomain that bypasses the proxy), flag it: an exposed origin defeats a CDN's protection.
- Rate limiting at the edge or in the app on expensive routes: login, search, form submissions, password reset, any endpoint that hits a database or an external API. Look for a limiter in the server code or edge config, and flag routes with none.
- A Web Application Firewall or bot-management rule set enabled on the proxy (most CDNs ship one), and a basic bot-challenge (CAPTCHA or JS challenge) on forms that are costly to process or easy to script.
- Caching for static assets and, where content allows it, for public pages, so a traffic spike hits the cache instead of the origin. Never let the CDN or a shared cache store a response that carries one visitor's private data or token: pages and API responses that are personal or session-specific need `Cache-Control: private, no-store` (or equivalent) and must not be cached by key alone when the response actually varies by cookie or auth header (web cache deception). Check that cache keys include the parts of the request that make a response private.
- Sane timeouts, body-size limits, and upper bounds on pagination or array inputs on the server, so a single slow, oversized, or unbounded request can't tie up a worker (protects against Slowloris-style and large-payload exhaustion, which look like small-scale DoS rather than volumetric DDoS).
- Separate, rate-limited credentials for any third-party API the app calls, so a flood of requests can't exhaust a shared budget or quota.

Where the project has none of this, recommend putting the public site behind a CDN with DDoS protection (Cloudflare's free tier covers the basics) rather than trying to build the equivalent from scratch, and say that volumetric DDoS protection is an infrastructure decision outside what code changes alone can fix. Mark these items `needs-host-config`.

## Regression tests

Every `fixed+verified` finding gets an automated test in the project's own test setup (for example `security.test.js`), named after the finding id, that repeats the original probe and asserts it fails. Use the framework the project already has; if none exists, use the runtime's built-in test runner. Tests must use throwaway data and clean up after themselves. Run the whole suite at the end of every pass. A fix without a test is reported as `fixed` but not `verified`.

## Report format

Start with one line: passes run, and the counts per status (`open`, `fixed+verified`, `needs-host-config`, `untestable`), before and after.

Then findings grouped by severity (critical, high, medium, low). Each one has:

1. Id, file and line.
2. What the hole is and how the probe triggered it.
3. The fix, the regression test name, and for fix mode, the result of the re-test.

Then four short lists: what was tested, what was fixed, what needs host or provider action (with the exact setting, file, or rotation step), and what could not be tested.

## Limits to state in every report

- "0 open" means zero in the tested area, not zero vulnerabilities.
- A front-end project can't prove server-side safety.
- DDoS protection is reviewed by configuration, never tested live: an untested setup is not the same as a vulnerable one, and a real load test (if the user wants one) must go through the hosting provider or a dedicated testing service.
- Local probes don't replace a professional penetration test. For sites with payments or personal data, recommend one.
- Anything that could not be tested (no server, no network, no backend code) is listed as untested.

by 1mN1kO
