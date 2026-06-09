# appsec.cc

A hands-on **dev-security training** (OWASP Top 10 Web + API) and the reproducible infrastructure that serves it online. Two complementary repos: the training **content** and its **deployment**.

🔗 Training server: **https://appsec.cc/**

---

## 📚 [`formation-dev-secu-java-vue`](https://github.com/appsec-cc/formation-dev-secu-java-vue) — the content

A hands-on **dev-security course** built around Java backends and Vue/Vite frontends. It's organized as 13 numbered case studies, each mapping to an **OWASP Top 10** (Web + API) category:

| # | Theme |
|---|-------|
| 01 | Broken Access Control / IDOR |
| 02 | Security Misconfiguration (Spring Boot, H2 console, Actuator) |
| 03 | Supply Chain Failures (Nexus npm registry, Vite, exfil listener) |
| 04 | Cryptographic Failures |
| 05 | Injection / XSS |
| 06–10 | API: BOLA, Broken Auth, BOPLA, Resource Consumption, BFLA |
| 11–13 | Combined 2021/2025 OWASP categories |

Each project typically runs on `localhost:8080` (embedded Tomcat or Spring Boot, in-memory H2), so they're meant to be launched **one at a time**. Project 03 is the exception (Nexus on 8081, Vite on 5173). The README also captures practical environment notes (WSL DNS/PTR lookup fix, switching JDKs on Windows).

## ⚙️ [`conf-as-code`](https://github.com/appsec-cc/conf-as-code) — the deployment / infra

An **Ansible-based provisioning repo** that turns those projects into a live training server reachable at `https://appsec.cc/`. Two-phase logic:

1. **`apps`** — rsync sources from *both* repos: the formation repo (the `NN-*` projects + their READMEs) and this repo's `front/` (a Next.js static doc site), then pre-build everything (Maven with the right `JAVA_HOME`, Node builds), and install systemd units (`appsec-app@…`).
2. **`switch`** — flip the single active project: stop current → start new → swap the nginx config. Run with:
   ```bash
   ansible-playbook -i inventory.ini playbook.yml --tags switch -e project=XX
   ```

**Roles**: `common` (Java 8+17, Maven, Node 20, UFW), `nginx` (reverse proxy + TLS via certbot/Let's Encrypt), `nexus` (bare-metal Nexus for project 03), `apps`, `switch`. UFW whitelists trainee IPs.

## 🔗 The relationship between the two

- **`formation-dev-secu-java-vue`** = the **vulnerable apps + course material** (the AppSec course content).
- **`conf-as-code`** = the **infrastructure** that builds, serves, and hot-swaps them behind nginx/TLS on a shared server.

The two cross-reference each other (`formation_repo` is a sibling dir by default), and the doc site in `conf-as-code/front` reads the projects' `README.md` files via `APPSEC_CONTENT_DIR`.
