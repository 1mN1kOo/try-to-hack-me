# 🛡️ Try to Hack Me (`try-to-hack-me`)

[![License: MIT](https://img.shields.io/badge/License-MIT-blue.svg)](LICENSE)
[![Antigravity Skill](https://img.shields.io/badge/Antigravity-Skill-orange.svg)](https://github.com/1mN1kOo/try-to-hack-me)
[![Scope: Localhost](https://img.shields.io/badge/scope-localhost%20only-green.svg)](#-safety-first)

> **Autonomous security auditing and hardening for your web projects.**  
> Proves vulnerabilities with harmless probes, applies minimal fixes, writes regression tests, and repeats up to 5 passes until achieving **0 open findings in the tested area**.

---

## ⚡ Quick Start

Install into your repository:

```bash
mkdir -p .agents/skills/try-to-hack-me
curl -fsSL -o .agents/skills/try-to-hack-me/SKILL.md https://raw.githubusercontent.com/1mN1kOo/try-to-hack-me/main/SKILL.md
```
*(For Claude Code, place in `.claude/skills/try-to-hack-me`)*

### Prompts

- **Audit Mode (Read-only):**  
  > *"Try to hack my site and find any vulnerabilities."*
- **Fix Mode (Loop-to-Zero):**  
  > *"Harden this app and bring all vulnerabilities to zero."*

---

## 📊 Benchmark

![Benchmark Results](benchmark.png)

Tested on **Mollra Music** (full-stack web app on localhost):
- **Initial flaws found:** 34
- **Remaining after hardening:** 4 *(host-level headers or client markers)*
- **Net reduction:** **-88%** (0 open in tested area)

---

## 🔁 How it Works (Two-Phase Hardening)

- **Phase 1 (Localhost Loop-to-Zero):**  
  1. **Map & Scan:** Discovers inputs, sinks, routes; runs defensive tools (`semgrep`, `gitleaks`, `npm audit`).
  2. **Harmless Probes:** Sends benign test markers (e.g. `window.__xss=1`, max 15 requests, localhost only).
  3. **Patch & Test:** Implements minimal fixes and writes regression tests into `security.test.js`.
  4. **Iterate (Up to 5 passes):** Re-scans the full codebase until reaching clean status (**0 open in tested area**).
  5. **Clean Hygiene:** Purges test data and verifies with `git status`.
- **Phase 2 (Host Rollout — User Confirmation Gate):**  
  Optional, strictly gated deployment of verified patches and `needs-host-config` tasks (CDN, headers, secret rotation) with snapshots, individual confirmations, and instant rollback.

---

## 🎯 Coverage (10 Attack Classes)

- **XSS & Client Injection:** Reflected, stored, DOM-based, context-aware escaping, strict CSP.
- **Server Injection:** SQL, NoSQL operators, shell, template, and CSV formula export injection.
- **Broken Access Control & IDOR:** Route-by-role matrix, mass assignment protection, private API responses.
- **SSRF & Path Traversal:** Internal IP/metadata blocking, safe path resolution (`path.resolve`).
- **Auth & Session Security:** Argon2id/PBKDF2 (100k+), timing-safe comparisons, user enumeration defense.
- **JWT & OAuth:** Fixed algorithm allowlists, negative token matrices, OAuth PKCE and state verification.
- **Webhooks & WebSockets:** HMAC signatures with replay timestamps, handshake `Origin` checks.
- **Security Headers & Cookies:** Live verification (`curl -sI`), `HttpOnly`/`Secure`/`SameSite`, ready-to-use host configs.
- **Secrets & Leaks:** Git history scanning, `.env` leakage, build artifact protection.
- **DoS & Logic Flaws:** ReDoS detection, race condition mitigation, and CDN/WAF readiness reviews (never simulated).

---

## 🔒 Safety First

- **Localhost only:** Never probes live production or external IPs.
- **Non-destructive:** No stress tests, brute-force, or fuzzers (rate-limit checks capped at 15 sequential requests).
- **Zero test residue:** Validated via `git status` cleanup after every run.
- **Strict Findings Registry:** Every finding is tracked as `open`, `fixed+verified`, `needs-host-config`, or `untestable`.

---

## 📄 License

[MIT License](LICENSE) © 2026 **[1mN1kO](https://github.com/1mN1kOo)**.
