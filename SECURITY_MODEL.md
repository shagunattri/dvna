# Security model: DVNA

## System overview
DVNA (Damn Vulnerable NodeJS Application, by Appsecco) is a deliberately vulnerable OWASP Top 10 training app. Single Node.js process (`server.js`): Express 4 + EJS views, passport-local auth with express-session, Sequelize 4 ORM against MySQL 5.7 (docker-compose) or SQLite (`appsecco/dvna:sqlite` image / `DATABASE_URL`). Listens on `0.0.0.0:9090` (`config/server.js`). `docs/solution/*.md` documents the intended vulnerabilities and their fixes.

**Deployment:** runs in a **shared training environment reachable by students over a network**. No separate infrastructure repo exists — the in-repo `Dockerfile`, `Dockerfile-dev`, `docker-compose.yml` and entrypoint scripts are the deployment.

## Components
| Component | Code | Notes |
|---|---|---|
| **Auth** | `core/passport.js`, `core/authHandler.js`, `routes/main.js` (login/register/forgotpw/resetpw/logout) | passport-local, bcrypt hashes, md5(login) reset token, session secret `keyboard cat` in `server.js` |
| **App features** | `core/appHandler.js`, `routes/app.js` | user search, ping, products, product edit, user edit, calc, admin API, redirect, bulk XML / legacy serialized import |
| **Web UI** | `views/**`, `public/assets/**` | EJS templates, jQuery 3.2.1, showdown |
| **Learning content** | `routes/main.js` `/learn*`, `views/vulnerabilities/**`, `config/vulns.js`, `docs/**` | static training material |
| **Data layer** | `models/**`, `config/db.js` | Sequelize models `User` (login, email, bcrypt password, role), `Product` |
| **Deployment** | `Dockerfile`, `Dockerfile-dev`, `docker-compose.yml`, `entrypoint*.sh`, `wait-for-it.sh`, `package.json`, `server.js` config | node:carbon image, MySQL 5.7, `vars.env` credentials, pinned old deps (mathjs 3.10.1, node-serialize 0.0.4, libxmljs 0.19) |

## Trust boundaries and entry points
- **Browser → app (HTTP :9090)**, reachable by everyone on the shared training network: unauthenticated routes `/login`, `/register`, `/forgotpw`, `/resetpw`, `/logout`, `/app/redirect`. Everything else under `/app/*` and `/learn*` requires any logged-in session (`authHandler.isAuthenticated`); registration is open, so "authenticated" = anyone who can reach the port.
- No role checks: `/app/admin/usersapi` and `/app/admin/users` only require login; `/app/admin` renders based on `req.user.role`.
- **App → OS**: `child_process.exec` in `ping`, `mathjs.eval` in `calc`, `node-serialize.unserialize` in `bulkproductslegacy`, libxmljs with `noent:true` in `bulkproducts`.
- **App → DB**: raw SQL in `userSearch`; ORM elsewhere.
- **Container → host/network**: app runs as root in `node:carbon`; docker-compose mounts the repo into the container (dev) and exposes 9090. On a shared network, compromise of the container reaches the training network.

## Assets
- The host/container running DVNA and the shared training network it sits on.
- Other students' accounts and sessions; user records and bcrypt hashes in the DB.
- DB credentials in `vars.env`.
(No single asset is designated most critical.)

## Attacker profiles (all in scope)
- Unauthenticated attacker on the training network who can reach :9090 (can self-register → becomes an authenticated user).
- Any registered/logged-in user (student).
- Third-party content targeting a logged-in user (CSRF, XSS, open redirect).

## Existing controls
bcrypt password hashing; passport session auth on `/app/*`; Sequelize parameterization on ORM queries. `csurf` and `x-xss-protection` are dependencies but not wired in. Session cookie `secure:false`, no helmet, no rate limiting.

## Documented training vulnerabilities
Documented in `docs/solution/`: A1 injection (SQLi `userSearch`, command injection `ping`), A2 broken auth (md5 reset token, user enumeration), A3 sensitive data exposure (`/app/admin/usersapi` returns hashes), A4 XXE (`bulkProducts`), A5 broken access control (admin API, IDOR in `useredit`), A6 misconfig (hardcoded session secret, `X-Powered-By`, error leakage), A7 XSS (product search/listing), A8 insecure deserialization (`bulkProductsLegacy`), A9 vulnerable components (mathjs RCE in `calc`), A10 insufficient logging, CSRF (all forms), open redirect (`/app/redirect`).

**Decision:** these are **real issues — keep the scanner's severity**. They are not accepted risk.

## Out of scope / accepted risk
- Nothing is ruled out.

## Prioritization rule
All attackers matter, nothing is accepted risk, and no asset was named as critical: findings keep the scanner's severity.

## Open questions
- None outstanding.
