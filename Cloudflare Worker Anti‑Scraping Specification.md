Cloudflare Worker Anti‑Scraping Specification
1. Purpose
This document defines the full technical specification for the Cloudflare Worker Anti‑Scraping module. It includes architecture diagrams, functional and non‑functional requirements, security policies, and references to related Business Factory documents. The Worker protects public endpoints from scraping, automated bots, and abusive traffic while integrating with Cloudflare Tunnel, Access, WAF, Wazuh, and Prometheus.
2. Architectural Overview
2.1 High‑Level Architecture Diagram
Client
  |
  v
+---------------------------+
|     Cloudflare Edge       |
|  WAF / Access / Firewall  |
+-------------+-------------+
              |
              v
+---------------------------+
|  Anti‑Scraping Worker     |
|  (Bot Detection Layer)    |
+-------------+-------------+
              |
              v
+---------------------------+
| Cloudflare Tunnel         |
+-------------+-------------+
              |
              v
+---------------------------+
| Router → RAG / n8n / CRM |
+---------------------------+
2.2 Integration With Business Factory Network
+---------------------------+
| Cloudflare Worker Layer   |
+-------------+-------------+
              |
              v
+-------------------+   +-------------------+   +-------------------+
| Factory Core (TS) |   | Payments VPN      |   | Blockchain VPN    |
| RAG / Router /    |   | PSP / Billing DB  |   | Validators / RPC  |
| Vault / n8n       |   | (Isolated)        |   | (Isolated)        |
+-------------------+   +-------------------+   +-------------------+
3. Functional Requirements
- Bot & scraper detection (UA blacklist, header anomalies, behavioral analysis, geo anomalies, honeypots).
- Response actions (soft block, hard block, challenge, Wazuh logging, Bot Fight Mode).
- HTML & API obfuscation (DOM randomization, honeypots, noise fields).
4. Non‑Functional Requirements
- Execution < 5 ms.
- TTFB overhead < 10%.
- Global scaling on Cloudflare Edge.
- Wazuh‑compatible logging.
5. Logging & Monitoring
- Wazuh fields: IP, UA, geo, violation type, threat score, timestamp, path.
- Prometheus metrics: blocked count, suspicious count, response time, geo distribution.
6. Policies
6.1 User‑Agent Blacklist
curl
python-requests
wget
scrapy
aiohttp
okhttp
selenium
puppeteer
playwright
6.2 Rate Limiting Policy
- 10 req/sec → soft block
- 20 req/sec → hard block
- 50 req/sec → 10‑minute ban
6.3 Cloudflare Access Policy
- SSO required for admin endpoints.
- MFA enforced.
- Session expiration: 12 hours.
6.4 VPN & Network Policy
- Worker cannot access Payments or Blockchain networks.
- Worker interacts only with Router via Tunnel.
6.5 Wazuh Policy
- All Worker logs forwarded to Wazuh.
- Alerts for repeated scraping attempts.
7. Worker Logic (Pseudocode)
if isBlacklistedUA(request): return block()
if isSuspiciousHeaders(request): return softBlock()
if isRateLimitExceeded(ip): return hardBlock()
if isBotBehavior(request): return challenge()
return fetch(request)
8. Related Documents
- `Security white paper.md`
- `RAG‑сервис.md`
- `Agent Context Search Specification.md`
9. Summary
This Worker provides a robust anti‑scraping layer for the Business Factory. It integrates seamlessly with Cloudflare, Tailscale, Wazuh, and Prometheus while maintaining compliance‑ready logging and zero‑trust principles.
