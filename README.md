## Final working start-to-finish setup (Bitwarden Lite + SQL Server 2022 + Nginx Proxy Manager)

This is the **full, working**, **locked-down by default** process you just proved on your server.

### Assumptions

* Docker + Portainer already installed
* Ubuntu server has a **static LAN IP**
* Public DNS `bw.signatureimagewear.com` points to your **public WAN IP**
* UniFi forwards **80/443 → Ubuntu server LAN IP**
* Only **Nginx Proxy Manager (NPM)** exposes **80/443**
* Bitwarden is only reachable through NPM (TLS terminates at NPM)

---

## 1) Create the shared Docker network (one-time)

On Ubuntu:

```bash
docker network create proxy
```

This is the external network shared by NPM and Bitwarden.

---

## 2) Deploy Nginx Proxy Manager (Portainer Stack)

Portainer → **Stacks** → **Add stack** → Name: `reverse-proxy`

Paste:

```yaml
version: "3.9"

services:
  npm:
    image: jc21/nginx-proxy-manager:latest
    container_name: nginx-proxy-manager
    restart: unless-stopped
    ports:
      - "80:80"
      - "443:443"
      - "127.0.0.1:81:81"   # Admin UI only on localhost
    environment:
      DB_MYSQL_HOST: npm-db
      DB_MYSQL_PORT: 3306
      DB_MYSQL_USER: npm
      DB_MYSQL_PASSWORD: ${NPM_DB_PASSWORD}
      DB_MYSQL_NAME: npm
    volumes:
      - npm_data:/data
      - npm_letsencrypt:/etc/letsencrypt
    networks:
      - proxy

  npm-db:
    image: mariadb:10.11
    container_name: npm-mariadb
    restart: unless-stopped
    environment:
      MYSQL_ROOT_PASSWORD: ${NPM_DB_ROOT_PASSWORD}
      MYSQL_DATABASE: npm
      MYSQL_USER: npm
      MYSQL_PASSWORD: ${NPM_DB_PASSWORD}
    volumes:
      - npm_db:/var/lib/mysql
    networks:
      - proxy

volumes:
  npm_data:
  npm_db:
  npm_letsencrypt:

networks:
  proxy:
    external: true
```

### Environment variables for this stack

Set in Portainer:

* `NPM_DB_ROOT_PASSWORD` = strong password (avoid `;`)
* `NPM_DB_PASSWORD` = strong password (avoid `;`)

Deploy the stack.

### Access NPM admin (locked down)

From your PC:

```bash
ssh -L 81:127.0.0.1:81 ubuntu@linuxdocker.corp
```

Open:

* `http://127.0.0.1:81`

---

## 3) Deploy Bitwarden Lite + SQL Server (Portainer Stack)

Portainer → **Stacks** → **Add stack** → Name: `bitwarden`

Paste this **final working compose** (includes all fixes):

```yaml
version: "3.9"

services:
  mssql:
    image: mcr.microsoft.com/mssql/server:2022-latest
    container_name: bw-mssql
    restart: unless-stopped
    environment:
      ACCEPT_EULA: "Y"
      MSSQL_PID: "Developer"
      MSSQL_SA_PASSWORD: ${MSSQL_SA_PASSWORD}
    volumes:
      - mssql_data:/var/opt/mssql
    networks:
      - internal
    # Wait until SQL is actually accepting logins before init runs.
    # Uses tools18 sqlcmd + -No to avoid encryption/cert negotiation issues.
    healthcheck:
      test:
        [
          "CMD-SHELL",
          "/opt/mssql-tools18/bin/sqlcmd -No -S localhost -U sa -P \"$${MSSQL_SA_PASSWORD}\" -Q \"SELECT 1\" >/dev/null 2>&1 || exit 1"
        ]
      interval: 5s
      timeout: 4s
      retries: 60
      start_period: 20s

  # One-shot DB bootstrap that creates DB/login/user/permissions/default DB.
  # Uses /opt/mssql-tools18/bin/sqlcmd directly (no variable/path issues).
  mssql-init:
    image: mcr.microsoft.com/mssql/server:2022-latest
    container_name: bw-mssql-init
    restart: "no"
    depends_on:
      mssql:
        condition: service_healthy
    environment:
      MSSQL_SA_PASSWORD: ${MSSQL_SA_PASSWORD}
      BW_DB_DATABASE: ${BW_DB_DATABASE}
      BW_DB_USERNAME: ${BW_DB_USERNAME}
      BW_DB_PASSWORD: ${BW_DB_PASSWORD}
    networks:
      - internal
    command:
      - /bin/bash
      - -lc
      - |
        set -euo pipefail
        echo "SQL is healthy. Initializing Bitwarden database/login..."

        ls -la /opt/mssql-tools18/bin/sqlcmd

        /opt/mssql-tools18/bin/sqlcmd -No -S "mssql,1433" -U sa -P "$MSSQL_SA_PASSWORD" -b -Q \
          "IF DB_ID(N'${BW_DB_DATABASE}') IS NULL CREATE DATABASE [${BW_DB_DATABASE}];"

        /opt/mssql-tools18/bin/sqlcmd -No -S "mssql,1433" -U sa -P "$MSSQL_SA_PASSWORD" -b -Q \
          "IF NOT EXISTS (SELECT 1 FROM sys.sql_logins WHERE name = N'${BW_DB_USERNAME}')
             CREATE LOGIN [${BW_DB_USERNAME}] WITH PASSWORD = N'${BW_DB_PASSWORD}';
           ELSE
             ALTER LOGIN [${BW_DB_USERNAME}] WITH PASSWORD = N'${BW_DB_PASSWORD}';"

        /opt/mssql-tools18/bin/sqlcmd -No -S "mssql,1433" -U sa -P "$MSSQL_SA_PASSWORD" -b -Q \
          "ALTER LOGIN [${BW_DB_USERNAME}] WITH DEFAULT_DATABASE = [${BW_DB_DATABASE}];"

        /opt/mssql-tools18/bin/sqlcmd -No -S "mssql,1433" -U sa -P "$MSSQL_SA_PASSWORD" -b -d "${BW_DB_DATABASE}" -Q \
          "IF NOT EXISTS (SELECT 1 FROM sys.database_principals WHERE name = N'${BW_DB_USERNAME}')
             CREATE USER [${BW_DB_USERNAME}] FOR LOGIN [${BW_DB_USERNAME}];
           IF NOT EXISTS (
             SELECT 1
             FROM sys.database_role_members rm
             JOIN sys.database_principals r ON rm.role_principal_id = r.principal_id
             JOIN sys.database_principals m ON rm.member_principal_id = m.principal_id
             WHERE r.name = N'db_owner' AND m.name = N'${BW_DB_USERNAME}'
           )
             ALTER ROLE [db_owner] ADD MEMBER [${BW_DB_USERNAME}];"

        echo "MSSQL init complete."

  bw:
    image: ghcr.io/bitwarden/lite:latest
    container_name: bitwarden-lite
    restart: unless-stopped
    depends_on:
      - mssql-init
    environment:
      # IMPORTANT: host only (NO https://)
      BW_DOMAIN: ${BW_DOMAIN}

      # TLS terminates at NPM
      BW_ENABLE_SSL: "false"

      BW_INSTALLATION_ID: ${BW_INSTALLATION_ID}
      BW_INSTALLATION_KEY: ${BW_INSTALLATION_KEY}

      BW_DB_PROVIDER: sqlserver
      BW_DB_SERVER: mssql
      BW_DB_PORT: "1433"
      BW_DB_DATABASE: ${BW_DB_DATABASE}
      BW_DB_USERNAME: ${BW_DB_USERNAME}
      BW_DB_PASSWORD: ${BW_DB_PASSWORD}
    volumes:
      - bw_data:/etc/bitwarden
    networks:
      - internal
      - proxy

volumes:
  bw_data:
  mssql_data:

networks:
  internal:
    driver: bridge
  proxy:
    external: true
```

### Environment variables for this stack

Set in Portainer:

* `BW_DOMAIN=bw.signatureimagewear.com` ✅ **NO `https://`**

* `BW_INSTALLATION_ID=...`

* `BW_INSTALLATION_KEY=...`

* `BW_DB_DATABASE=bitwarden_vault`

* `BW_DB_USERNAME=bitwarden`

* `BW_DB_PASSWORD=...` (avoid `;`)

* `MSSQL_SA_PASSWORD=...` (avoid `;`, must meet complexity)

Deploy the stack.

---

## 4) Verify init completed, then restart Bitwarden (important)

On Ubuntu:

```bash
docker logs --tail 200 bw-mssql-init
```

Confirm it ends with:

* `MSSQL init complete.`

Then restart Bitwarden so it reconnects cleanly after DB bootstrap:

```bash
docker restart bitwarden-lite
```

---

## 5) Configure NPM Proxy Host for Bitwarden

NPM Admin → **Hosts → Proxy Hosts → Add Proxy Host**

### Details

* **Domain Names:** `bw.signatureimagewear.com`
* **Scheme:** `http`
* **Forward Hostname:** `bitwarden-lite`
* **Forward Port:** `8080`
* ✅ Websockets Support
* ✅ Block Common Exploits

### SSL

* Request/attach Let’s Encrypt cert for `bw.signatureimagewear.com`
* ✅ Force SSL (once cert is issued)
* ✅ HTTP/2 Support

### Advanced (Custom Nginx Configuration)

Paste:

```nginx
proxy_set_header Host $host;
proxy_set_header X-Real-IP $remote_addr;
proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
proxy_set_header X-Forwarded-Proto $scheme;
proxy_set_header X-Forwarded-Host $host;

add_header X-Content-Type-Options "nosniff" always;
add_header X-Frame-Options "DENY" always;
add_header Referrer-Policy "no-referrer" always;
add_header Permissions-Policy "geolocation=(), microphone=(), camera=()" always;
add_header Strict-Transport-Security "max-age=86400; includeSubDomains" always;
```

Save.

---

## 6) Validate locally (bypasses DNS/NAT guessing)

From Ubuntu:

```bash
curl -i -H "Host: bw.signatureimagewear.com" http://127.0.0.1/ | head -n 15
```

You should **not** see the NPM “Congratulations” page.

Check Bitwarden services:

```bash
docker exec -it bitwarden-lite supervisorctl status
```

---

## 7) Register + Login

Open (incognito recommended once):

* `https://bw.signatureimagewear.com/#/register`

Create a user and log in.

---

## 8) “Don’t get burned again” rules

1. **Never delete `bw_data` without also deleting `mssql_data`**
   They’re cryptographically paired (Bitwarden DataProtection keys ↔ encrypted DB values). Breaking the pair causes login/key-ring failures.

2. **`BW_DOMAIN` must be host-only**
   ✅ `bw.signatureimagewear.com`
   ❌ `https://bw.signatureimagewear.com` (causes origin/CORS/token errors like `https://https://...`)

3. Keep SQL init behavior:

   * use `sqlcmd -No`
   * connect to `mssql,1433`
   * set default DB + grant `db_owner`

---

## 9) Backup (minimum recommended)

Back up these two volumes **together**:

* `bw_data` (Bitwarden config + keys)
* `mssql_data` (SQL database)

Restoring only one of them will resurrect the crypto mismatch.
