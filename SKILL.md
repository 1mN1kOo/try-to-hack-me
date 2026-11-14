---
name: try-to-hack-me
description: Attack simulation against the user's own web project. Looks for common hacker entry points (XSS, injection, SSRF, path traversal, broken access control, open redirects, clickjacking, CSRF and CORS, missing security headers, exposed secrets, unsafe uploads, weak sign-in, JWT and OAuth mistakes, vulnerable dependencies), proves each hole with harmless probes, closes it, and re-tests. Use this skill whenever the user asks for a security check, a pentest, a vulnerability search, "try to hack my site", "find holes", "make it secure", or asks to harden a site or app, even if they don't use the word "security".
---

# Try to Hack Me

Act as an attacker against the user's own project to find holes, then close them and prove they are closed. Keep fixes small and simple.

## Scope and safety, always

- Test only the user's own project: its files, and a local server you start yourself (localhost). Never probe a live site, server, or third-party service, even if the user pastes its address, unless they state it is their own and the environment allows it. Never run floods, brute-force attempts, or anything that could take a service down or touch real accounts and data.
- Use minimal, harmless probes: unique marker strings and standard test inputs such as `"><img src=x onerror=window.__xss=1>` to see whether input is escaped. Do not write exploit tools, scanners, brute-forcers, payload collections, or malware.
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

**SSRF (server-side request forgery).** The server fetches a URL that comes from a user: avatar by URL, link previews, webhooks, import from URL, image proxies. Attackers point it at internal targets such as `127.0.0.1`, `localhost`, `169.254.169.254` (cloud metadata), and private ranges (`10.0.0.0/8`, `172.16.0.0/12`, `192.168.0.0/16`, and their IPv6 equivalents). Fix: allow only `http:` and `https:`, prefer an allowlist of hosts, resolve the hostname and block loopback, link-local, and private addresses before connecting, re-check on every redirect (or disable redirects), set timeouts and size limits, and never return raw internal responses to the user. Review the code for this; don't send requests to real internal addresses.

**Path traversal.** User input reaches a file path (downloads, uploads, static serving, templates, `?file=` parameters) with `../`, encoded variants (`%2e%2e%2f`), absolute paths, or null bytes. Fix: resolve the final path with `path.resolve()` (and `fs.realpath()` to follow symlinks) and require that it stays strictly inside the intended folder (compare with the base folder plus a path separator), reject anything else, and generate server-side file names instead of using user-supplied ones. Don't serve dotfiles or source folders.

**Broken access control.** Actions or data restricted only by hiding buttons in the interface, guessable ids in API calls, admin routes without a server-side check, prices or roles taken from the client. Fix: enforce on the server.

**Unsafe URLs, redirects, and links.** Validate each URL by where it is used, with an allowlist, and parse it with `new URL()` instead of checking the string.
- `a[href]`: `http:`, `https:`, `mailto:`, `tel:`, relative URLs, and `#` anchors.
- `img[src]`, `source[src]`, `video[poster]`, CSS `url()`: `http:`, `https:`, and relative URLs only. `data:` only for small images the developer embeds, never for values from users or fetched data, and never `data:image/svg+xml` from untrusted input.
- `script[src]`, `iframe[src]`, `link[href]`, `form[action]`, `fetch()` and `XMLHttpRequest` targets: `https:` or relative only, and only known hosts. Add `sandbox` to iframes with untrusted content.
- Reject `javascript:`, `vbscript:`, and `data:text/html` everywhere.
- Add `rel="noopener noreferrer"` to `target="_blank"` links. Check redirects built from query parameters for open redirects.

**Clickjacking.** The page can be framed by any site. Fix: `frame-ancestors` in CSP or `X-Frame-Options`.

**CSRF and CORS.** State-changing requests without tokens or `SameSite`, `Access-Control-Allow-Origin: *` together with credentials. Fix: tokens, `SameSite=Lax` or `Strict`, an explicit origin list.

**Security headers and cookies.** Check for `Content-Security-Policy`, `Strict-Transport-Security`, `X-Content-Type-Options: nosniff`, `Referrer-Policy`, and `Permissions-Policy`. Cookies need `HttpOnly`, `Secure`, and `SameSite`. A `<meta>` CSP helps on static pages, but real headers need the server or host config, so say which file or setting to change.

**Exposed files and secrets.** Keys and tokens in code, a committed `.env`, environment variables that the build copies into the browser bundle (`VITE_`, `NEXT_PUBLIC_`, `REACT_APP_`, `NUXT_PUBLIC_`, `PUBLIC_`, `window.__ENV__`, a public config file), secrets visible in the built output or source maps, a `.git` folder, source maps, backups, debug endpoints, directory listing, verbose error pages. Only publishable keys (for example a public analytics or maps key restricted by domain) belong in the client. Fix: move secrets to the server, remove and block the files. Tell the user that a leaked key must be rotated, not just deleted.

**Prototype pollution and unsafe merges.** User JSON merged into objects, keys like `__proto__` and `constructor`. Fix: schema validation, `Object.create(null)`, a key blocklist.

**Cross-window trust.** `postMessage` listeners that don't check `event.origin`. Fix: a strict origin check.

**Uploads.** Type, size, and name checked only on the client, SVG or HTML files served from the same origin. Fix: server-side validation, a separate origin, `Content-Disposition`.

**Weak sign-in handling.** No rate limiting or lockout, tokens in `localStorage`, passwords in URLs or logs, verbose login errors. Fix: server-side limits, `HttpOnly` cookies, generic error messages. Review the configuration; don't hammer a login.
- JWT: the server must verify the signature with a fixed algorithm allowlist (reject `alg: none` and never trust the algorithm named in the token header), and check `exp`, `nbf`, `iss`, and `aud`. Use short lifetimes and rotate refresh tokens. Never put tokens in URLs or logs, and never decode a token without verifying it and then trust its contents.
- OAuth and OpenID: a random `state` value is generated, stored, and checked on return (protects against login CSRF and account mix-ups), PKCE is used for public clients and single-page apps, `redirect_uri` is matched exactly against a registered list, and the `nonce` is checked for OpenID. Don't use the implicit flow.

**Dependencies and third-party code.** Run `npm audit` or the equivalent, look for outdated libraries and CDN scripts without a pinned version or `integrity`. Fix: update, pin, add SRI, or remove. Remove a dependency only after the build and tests pass without it.

## Report format

Findings grouped by severity (critical, high, medium, low). Each one has:

1. File and line.
2. What the hole is and how the probe triggered it.
3. The fix, and for fix mode, the result of the re-test.

Then three short lists: what was tested, what was fixed, and what could not be tested.

## Limits to state in every report

- A front-end project can't prove server-side safety.
- Local probes don't replace a professional penetration test. For sites with payments or personal data, recommend one.
- Anything that could not be tested (no server, no network, no backend code) is listed as untested.

by 1mN1kO
