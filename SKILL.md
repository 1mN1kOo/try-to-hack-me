---
name: try-to-hack-me
description: Attack simulation against the user's own web project. Looks for common hacker entry points (XSS, injection, SSRF, path traversal, broken access control, open redirects, clickjacking, CSRF and CORS, missing security headers, exposed secrets, unsafe uploads, weak sign-in, JWT and OAuth mistakes, vulnerable dependencies, and DDoS exposure), proves each hole with harmless probes, closes it, and re-tests. Use this skill whenever the user asks for a security check, a pentest, a vulnerability search, "try to hack my site", "find holes", "make it secure", or asks to harden a site or app, even if they don't use the word "security".
---

# Try to Hack Me

Act as an attacker against the user's own project to find holes, then close them and prove they are closed. Keep fixes small and simple.

## Scope and safety, always

- Test only the user's own project: its files, and a local server you start yourself (localhost). Never probe a live site, server, or third-party service, even if the user pastes its address, unless they state it is their own and the environment allows it. Never run floods, brute-force attempts, or anything that could take a service down or touch real accounts and data.
- Use minimal, harmless probes: unique marker strings and standard test inputs such as `"><img src=x onerror=window.__xss=1>` to see whether input is escaped. Do not write exploit tools, scanners, brute-forcers, payload collections, or malware.
- Never run a DDoS or load test, not even against the user's own site or a local copy: no request floods, no concurrency ramps, no stress-testing tools. A real denial-of-service test needs dedicated infrastructure and the hosting provider's permission, and doing it from here could take the site down or look like an attack to the host or their upstream network. DDoS is reviewed as a readiness check (below), never simulated.
- Don't read, send, or store real secrets or personal data found along the way. Report where they are, not what they are.
- Comments: don't add any to the code you change.
- Avoid em dashes (—) in everything you write. Use a period, comma, colon, or parentheses.

## Audit or fix

- Audit mode: the user says check, test, audit, find holes, or asks what is wrong. Don't edit, create, or delete project files. Starting a local server and sending probes is fine if it changes nothing in the project. Probes that write data (stored XSS and similar) run only against a temporary copy or a throwaway data store; if that isn't possible, list them as untested. End with a report and an offer to apply the fixes.
- Fix mode: the user says fix, harden, secure, or close the holes. Apply the fixes.
- If the wording is ambiguous, default to audit mode and ask before editing.

## Method

1. Read the project: entry points, config, `package.json`, server code if present, hosting config.
2. Map the attack surface: every input (forms, URL parameters, hash, cookies, `postMessage`, uploads, fetched data), every dangerous sink, every endpoint, storage use, and third-party script.
3. Trace each input to its sink in the code and note where it is not validated or encoded.
4. If the project runs locally, send the harmless probes through the real inputs and check the result in the page source and the DOM. Put a unique marker in every probe so test data can be found later.
5. Clean up. If a probe wrote data (a local database, a JSON file, `localStorage`, an uploads folder), delete exactly the records and files carrying your marker once the check is done. Then confirm with `git status` or a before/after comparison that the project holds no leftover test data.
6. In fix mode, fix each confirmed hole with the smallest change, then repeat the same probe, confirm it is closed, and clean up again. Check that the fix didn't break the page.
7. Report every finding.

If the project can't run locally (no `.env`, an external database, missing services, or only fragments of a project), don't burn attempts on debugging the environment. Try at most a couple of quick setup steps, using obvious dummy values in a temporary setup and never real credentials. If it still won't start, switch at once to full static analysis: trace the data flow from every input to every sink by reading the code, and list the dynamic checks you skipped under "could not be tested".

## Dangerous sinks to look for

`innerHTML`, `outerHTML`, `insertAdjacentHTML`, `document.write`, `eval`, `new Function`, `setTimeout` or `setInterval` with a string, `dangerouslySetInnerHTML`, `location` assignments from data, string-built URLs, CSS, SQL, shell commands, or templates that receive anything non-constant, server-side `fetch()`, `axios`, or `http.request` with a URL built from input, and file functions (`fs.readFile`, `createReadStream`, `sendFile`, static file serving, uploads) with a path built from input.

## Attack classes to try

In rough order of how often they appear.

**Cross-site scripting.** Reflected (URL parameters), stored (data saved then shown), and DOM-based (`location.hash`, `postMessage`, `innerHTML`). Fix: `textContent` and `createElement`, output encoding, no string-built HTML, a Content-Security-Policy.

**Injection.** Untrusted values in URLs, CSS, HTML attributes, and, if the project has a backend, in SQL, shell commands, and templates. Fix: parameterized queries, allowlists, no string concatenation.

**Unverified webhooks and callbacks.** Endpoints that receive payment, auth, or third-party events (`/webhook`, `/callback`) and trust the JSON body without verifying the provider's cryptographic signature against the raw request body, or without checking the event timestamp. Fix: verify the HMAC or signature header against the raw body using a constant-time comparison, reject stale timestamps to prevent replay attacks, and record processed event IDs for idempotency.

**SSRF (server-side request forgery).** The server fetches a URL that comes from a user: avatar by URL, link previews, webhooks, import from URL, image proxies. Attackers point it at internal targets such as `127.0.0.1`, `localhost`, `169.254.169.254` (cloud metadata), and private ranges (`10.0.0.0/8`, `172.16.0.0/12`, `192.168.0.0/16`, and their IPv6 equivalents). Fix: allow only `http:` and `https:`, prefer an allowlist of hosts, resolve the hostname and block loopback, link-local, and private addresses before connecting, re-check on every redirect (or disable redirects), set timeouts and size limits, and never return raw internal responses to the user. Review the code for this; don't send requests to real internal addresses.

**Path traversal.** User input reaches a file path (downloads, uploads, static serving, templates, `?file=` parameters) with `../`, encoded variants (`%2e%2e%2f`), absolute paths, or null bytes. Fix: resolve the final path with `path.resolve()` (and `fs.realpath()` to follow symlinks) and require that it stays strictly inside the intended folder (compare with the base folder plus a path separator), reject anything else, and generate server-side file names instead of using user-supplied ones. Don't serve dotfiles or source folders.

**Broken access control, data exposure, and database leaks.** Actions or data restricted only by hiding buttons in the interface, guessable ids in API calls (IDOR), admin routes without a server-side check, prices or roles taken from the client, mass assignment (passing a raw request body straight into a database create or update call), API responses that return full database records with sensitive fields (password hashes, email, internal flags) instead of a filtered view, and the database itself reachable from outside: an exposed admin panel (phpMyAdmin, a database GUI), a public cloud storage bucket, or a downloadable backup or dump file. Fix: enforce authorization on the server, allowlist writable fields on input, return only public fields in responses, and keep databases, admin panels, and backups off the public internet or behind authentication.

**Unsafe URLs, redirects, and links.** Validate each URL by where it is used, with an allowlist, and parse it with `new URL()` instead of checking the string.
- `a[href]`: `http:`, `https:`, `mailto:`, `tel:`, relative URLs, and `#` anchors.
- `img[src]`, `source[src]`, `video[poster]`, CSS `url()`: `http:`, `https:`, and relative URLs only. `data:` only for small images the developer embeds, never for values from users or fetched data, and never `data:image/svg+xml` from untrusted input.
- `script[src]`, `iframe[src]`, `link[href]`, `form[action]`, `fetch()` and `XMLHttpRequest` targets: `https:` or relative only, and only known hosts. Add `sandbox` to iframes with untrusted content.
- Reject `javascript:`, `vbscript:`, and `data:text/html` everywhere.
- Add `rel="noopener noreferrer"` to `target="_blank"` links. Check redirects built from query parameters for open redirects.

**Clickjacking.** The page can be framed by any site. Fix: `frame-ancestors` in CSP or `X-Frame-Options`.

**CSRF and CORS.** State-changing requests without tokens or `SameSite`, `Access-Control-Allow-Origin: *` together with credentials. Fix: tokens, `SameSite=Lax` or `Strict`, an explicit origin list.

**Security headers and cookies.** Check for `Content-Security-Policy`, `Strict-Transport-Security`, `X-Content-Type-Options: nosniff`, `Referrer-Policy`, and `Permissions-Policy`. Cookies need `HttpOnly`, `Secure`, and `SameSite`. A `<meta>` CSP helps on static pages, but real headers need the server or host config, so say which file or setting to change.

**Exposed files and secrets.** API keys, database passwords, signing secrets, or tokens hardcoded in source or config files, a committed `.env`, environment variables copied into the browser bundle (`VITE_`, `NEXT_PUBLIC_`, `REACT_APP_`, `NUXT_PUBLIC_`, `PUBLIC_`, `window.__ENV__`, public config files), or exposed `.git` folders, source maps, backups, debug endpoints, directory listings, and verbose error pages. Only publishable keys (such as a domain-restricted analytics or maps key) belong in the client. Fix: keep all real secrets on the server inside `.env` (or the host's secret manager) read via `process.env`, list `.env` in `.gitignore`, ship a `.env.example` with placeholder values, remove and block exposed files, and tell the user to rotate any leaked secret rather than just deleting it.

**Prototype pollution and unsafe merges.** User JSON merged into objects, keys like `__proto__` and `constructor`. Fix: schema validation, `Object.create(null)`, a key blocklist.

**Cross-window and WebSocket trust.** `postMessage` listeners that don't check `event.origin`, and WebSocket servers that don't verify the `Origin` header during the handshake, don't re-check authorization on individual socket messages, or accept unbounded message sizes (browser CORS doesn't cover WebSocket connections, so a page on any site can open one as the visitor). Fix: enforce a strict origin check on both `postMessage` and WebSocket handshakes, authenticate the session and authorize every socket action on the server, and set maximum payload limits.

**Uploads.** Type, size, and name checked only on the client, SVG or HTML files served from the same origin. Fix: server-side validation, a separate origin, `Content-Disposition`.

**Weak sign-in and account takeover protection.** No rate limiting or lockout against credential stuffing, tokens in `localStorage`, passwords in URLs or logs, verbose login errors, passwords stored in plaintext or hashed with a fast algorithm (`MD5`, `SHA-1`, `SHA-256`), and active sessions or refresh tokens that stay valid after a password or email change. Fix: enforce server-side rate limits, store tokens in `HttpOnly` cookies, use generic error messages, hash passwords with `Argon2id`, `bcrypt`, or `scrypt`, and revoke all existing sessions and tokens whenever a user changes their password or email. Review the configuration; don't hammer a login.
- JWT: the server must verify the signature with a fixed algorithm allowlist (reject `alg: none` and never trust the algorithm named in the token header), and check `exp`, `nbf`, `iss`, and `aud`. Use short lifetimes and rotate refresh tokens. Never put tokens in URLs or logs, and never decode a token without verifying it and then trust its contents.
- OAuth and OpenID: a random `state` value is generated, stored, and checked on return (protects against login CSRF and account mix-ups), PKCE is used for public clients and single-page apps, `redirect_uri` is matched exactly against a registered list, and the `nonce` is checked for OpenID. Don't use the implicit flow.
- Weak randomness: `Math.random()`, timestamps, or sequential counters used for session ids, password reset links, invite codes, CSRF tokens, or OTPs, which are predictable. Secrets or HMACs compared with `===` instead of a constant-time function. Fix: `crypto.randomBytes()`, `crypto.randomUUID()`, or `crypto.getRandomValues()` for any security-sensitive token, and `crypto.timingSafeEqual()` for comparing digests.

**Dependencies and third-party code.** Run `npm audit` or the equivalent, look for outdated libraries and CDN scripts without a pinned version or `integrity`. Fix: update, pin, add SRI, or remove. Remove a dependency only after the build and tests pass without it.

**DDoS readiness (review only, never simulated).** Don't send load. Read the project and its deployment config and report what is and isn't in place:

- CDN or reverse proxy in front of the origin (Cloudflare, Fastly, AWS CloudFront/Shield, Vercel/Netlify edge network) so the real server IP isn't exposed and traffic is absorbed upstream. If the origin's real IP is discoverable (old DNS records, a direct IP link in the code, an email header, a subdomain that bypasses the proxy), flag it: an exposed origin defeats a CDN's protection.
- Rate limiting at the edge or in the app on expensive routes: login, search, form submissions, password reset, any endpoint that hits a database or an external API. Look for a limiter in the server code or edge config, and flag routes with none.
- A Web Application Firewall or bot-management rule set enabled on the proxy (most CDNs ship one), and a basic bot-challenge (CAPTCHA or JS challenge) on forms that are costly to process or easy to script.
- Caching for static assets and, where content allows it, for public pages, so a traffic spike hits the cache instead of the origin. Never let the CDN or a shared cache store a response that carries one visitor's private data or token: pages and API responses that are personal or session-specific need `Cache-Control: private, no-store` (or equivalent) and must not be cached by key alone when the response actually varies by cookie or auth header (web cache deception). Check that cache keys include the parts of the request that make a response private.
- Sane timeouts, body-size limits, and upper bounds on pagination or array inputs on the server, so a single slow, oversized, or unbounded request can't tie up a worker (protects against Slowloris-style and large-payload exhaustion, which look like small-scale DoS rather than volumetric DDoS).
- Separate, rate-limited credentials for any third-party API the app calls, so a flood of requests can't exhaust a shared budget or quota.

Where the project has none of this, recommend putting the public site behind a CDN with DDoS protection (Cloudflare's free tier covers the basics) rather than trying to build the equivalent from scratch, and say that volumetric DDoS protection is an infrastructure decision outside what code changes alone can fix.


## Report format

Findings grouped by severity (critical, high, medium, low). Each one has:

1. File and line.
2. What the hole is and how the probe triggered it.
3. The fix, and for fix mode, the result of the re-test.

Then three short lists: what was tested, what was fixed, and what could not be tested.

## Limits to state in every report

- A front-end project can't prove server-side safety.
- DDoS protection is reviewed by configuration, never tested live: an untested setup is not the same as a vulnerable one, and a real load test (if the user wants one) must go through the hosting provider or a dedicated testing service.
- Local probes don't replace a professional penetration test. For sites with payments or personal data, recommend one.
- Anything that could not be tested (no server, no network, no backend code) is listed as untested.

by 1mN1kO