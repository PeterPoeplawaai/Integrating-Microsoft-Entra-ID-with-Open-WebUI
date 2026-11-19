# Integrating Microsoft Entra ID with Open WebUI

A complete, step-by-step guide to enabling Single Sign-On (SSO) for [Open WebUI](https://github.com/open-webui/open-webui) with **Microsoft Entra ID** (formerly Azure AD) using OpenID Connect (OIDC) and OAuth 2.0.

> [!NOTE]
> Open WebUI changes quickly. Variable names and behaviour can differ between releases. Always cross-check with the official documentation:
> - <https://docs.openwebui.com/features/access-security/auth/sso>
> - <https://docs.openwebui.com/troubleshooting/sso>

## Table of Contents

- [Overview](#overview)
- [Prerequisites](#prerequisites)
- [Choose an integration method](#choose-an-integration-method)
- [Step 1: Register the application in Entra ID](#step-1-register-the-application-in-entra-id)
- [Step 2: Add the redirect URI](#step-2-add-the-redirect-uri)
- [Step 3: Create a client secret](#step-3-create-a-client-secret)
- [Step 4: Configure API permissions](#step-4-configure-api-permissions)
- [Step 5: Collect the required values](#step-5-collect-the-required-values)
- [Step 6: Configure Open WebUI](#step-6-configure-open-webui)
- [Step 7: Restrict access](#step-7-restrict-access)
- [Step 8: Map Entra app roles to Open WebUI roles (optional)](#step-8-map-entra-app-roles-to-open-webui-roles-optional)
- [Step 9: Sync Entra groups (optional)](#step-9-sync-entra-groups-optional)
- [Step 10: Reverse proxy configuration](#step-10-reverse-proxy-configuration)
- [Step 11: Test the login](#step-11-test-the-login)
- [Step 12: Harden the deployment](#step-12-harden-the-deployment)
- [Secret rotation](#secret-rotation)
- [Troubleshooting](#troubleshooting)
- [Environment variable reference](#environment-variable-reference)
- [References](#references)
- [License](#license)

## Overview

Open WebUI supports two ways to sign users in with Entra ID:

| Method | Callback path | Key variables |
|--------|---------------|---------------|
| **A. Native Microsoft provider** | `/oauth/microsoft/callback` | `MICROSOFT_CLIENT_ID`, `MICROSOFT_CLIENT_SECRET`, `MICROSOFT_CLIENT_TENANT_ID` |
| **B. Generic OIDC provider** | `/oauth/oidc/callback` | `OAUTH_CLIENT_ID`, `OAUTH_CLIENT_SECRET`, `OPENID_PROVIDER_URL` |

Both use the standard **Authorization Code flow**:

```text
User -> Open WebUI -> Entra ID (login + MFA) -> redirect back with authorization code
Open WebUI <- tokens (exchanged server-side using the client secret)
```

With `ENABLE_OAUTH_SIGNUP=true`, users are created automatically on their first login.

## Prerequisites

- A running Open WebUI instance reachable over **HTTPS** at a stable URL, for example `https://chat.example.com`.
- An Entra ID tenant and an account allowed to create app registrations (**Application Administrator** or **Cloud Application Administrator**).
- A role that can grant tenant-wide admin consent (**Privileged Role Administrator** or **Global Administrator**).
- Access to the Open WebUI environment variables (Docker, Docker Compose, Kubernetes or systemd).
- A fixed, strong `WEBUI_SECRET_KEY`, so sessions survive restarts.

## Choose an integration method

- **Method A (native Microsoft)** is the simplest option if you only use Entra ID.
  Redirect URI: `https://<your-domain>/oauth/microsoft/callback`
- **Method B (generic OIDC)** treats Entra ID like any other OIDC provider and gives more control over scopes.
  Redirect URI: `https://<your-domain>/oauth/oidc/callback`

> [!IMPORTANT]
> Pick **one** method and use its matching redirect URI everywhere. Open WebUI supports only one generic OIDC provider (`OPENID_PROVIDER_URL`) at a time.

## Step 1: Register the application in Entra ID

1. Sign in to the [Microsoft Entra admin center](https://entra.microsoft.com/).
2. Go to **Entra ID > App registrations**.
3. Click **New registration**.
4. Fill in the form:
   - **Name:** `Open WebUI`
   - **Supported account types:** *Accounts in this organizational directory only (Single tenant)*
   - **Redirect URI:** platform **Web**, using the URI for your chosen method (see Step 2)
5. Click **Register**.

## Step 2: Add the redirect URI

1. Open the app registration and go to **Manage > Authentication**.
2. Under **Platform configurations**, click **Add a platform > Web** (or edit the existing Web platform).
3. Add the redirect URI that matches your method:

   | Method | Redirect URI |
   |--------|--------------|
   | A. Native Microsoft | `https://chat.example.com/oauth/microsoft/callback` |
   | B. Generic OIDC | `https://chat.example.com/oauth/oidc/callback` |

4. Click **Configure** or **Save**.

> [!WARNING]
> The redirect URI must match **exactly**: scheme, host and path. A mismatch causes error `AADSTS50011`.

## Step 3: Create a client secret

1. In the app registration, go to **Manage > Certificates & secrets**.
2. Open the **Client secrets** tab and click **New client secret**.
3. Enter a description such as `open-webui-prod` and choose an expiry.
4. Click **Add**.
5. Copy the secret **Value** immediately. It is shown only once. Do **not** copy the *Secret ID*.

> [!CAUTION]
> When the secret expires, nobody can log in. Set a calendar reminder before the expiry date. See [Secret rotation](#secret-rotation).

## Step 4: Configure API permissions

1. Go to **Manage > API permissions**.
2. Click **Add a permission > Microsoft Graph > Delegated permissions**.
3. Select:
   - `openid`
   - `email`
   - `profile`
   - `offline_access` (recommended)
   - `User.Read` (usually present by default)
4. Click **Add permissions**.
5. Click **Grant admin consent for your tenant** and confirm.

## Step 5: Collect the required values

From the app registration **Overview** page, note down:

| Value | Where to find it | Used as |
|-------|------------------|---------|
| Application (client) ID | Overview | `MICROSOFT_CLIENT_ID` (A) or `OAUTH_CLIENT_ID` (B) |
| Directory (tenant) ID | Overview | `MICROSOFT_CLIENT_TENANT_ID` (A) and part of `OPENID_PROVIDER_URL` |
| Client secret value | Certificates & secrets | `MICROSOFT_CLIENT_SECRET` (A) or `OAUTH_CLIENT_SECRET` (B) |

The OIDC discovery URL for your tenant is:

```text
https://login.microsoftonline.com/<TENANT_ID>/v2.0/.well-known/openid-configuration
```

## Step 6: Configure Open WebUI

Use **Method A** or **Method B**. Do not combine both for Entra ID.

### Method A: Native Microsoft provider

#### Docker Compose

```yaml
services:
  open-webui:
    image: ghcr.io/open-webui/open-webui:main   # pin a specific version in production
    container_name: open-webui
    restart: unless-stopped
    ports:
      - "3000:8080"
    volumes:
      - open-webui-data:/app/backend/data
    environment:
      # Core
      WEBUI_URL: "https://chat.example.com"
      WEBUI_SECRET_KEY: "${WEBUI_SECRET_KEY}"

      # Create users automatically on first SSO login
      ENABLE_OAUTH_SIGNUP: "true"

      # Microsoft Entra ID
      MICROSOFT_CLIENT_ID: "${MICROSOFT_CLIENT_ID}"
      MICROSOFT_CLIENT_SECRET: "${MICROSOFT_CLIENT_SECRET}"
      MICROSOFT_CLIENT_TENANT_ID: "${MICROSOFT_CLIENT_TENANT_ID}"
      MICROSOFT_REDIRECT_URI: "https://chat.example.com/oauth/microsoft/callback"
      MICROSOFT_OAUTH_SCOPE: "openid email profile offline_access"

      # Needed for a proper sign-out
      OPENID_PROVIDER_URL: "https://login.microsoftonline.com/${MICROSOFT_CLIENT_TENANT_ID}/v2.0/.well-known/openid-configuration"

      # Account handling
      OAUTH_MERGE_ACCOUNTS_BY_EMAIL: "true"

volumes:
  open-webui-data:
```

Create a `.env` file next to `docker-compose.yml` and **never commit it**:

```env
WEBUI_SECRET_KEY=replace-with-a-long-random-string
MICROSOFT_CLIENT_ID=00000000-0000-0000-0000-000000000000
MICROSOFT_CLIENT_SECRET=your-client-secret-value
MICROSOFT_CLIENT_TENANT_ID=11111111-1111-1111-1111-111111111111
```

Generate a secret key:

```bash
openssl rand -hex 32
```

Start the stack:

```bash
docker compose up -d
```

#### docker run

```bash
docker run -d -p 3000:8080 \
  -v open-webui:/app/backend/data \
  -e WEBUI_URL="https://chat.example.com" \
  -e WEBUI_SECRET_KEY="<random-secret>" \
  -e ENABLE_OAUTH_SIGNUP=true \
  -e MICROSOFT_CLIENT_ID="<client-id>" \
  -e MICROSOFT_CLIENT_SECRET="<client-secret>" \
  -e MICROSOFT_CLIENT_TENANT_ID="<tenant-id>" \
  -e MICROSOFT_REDIRECT_URI="https://chat.example.com/oauth/microsoft/callback" \
  -e OPENID_PROVIDER_URL="https://login.microsoftonline.com/<tenant-id>/v2.0/.well-known/openid-configuration" \
  -e OAUTH_MERGE_ACCOUNTS_BY_EMAIL=true \
  --name open-webui --restart always \
  ghcr.io/open-webui/open-webui:main
```

### Method B: Generic OIDC provider

Register `/oauth/oidc/callback` as the redirect URI in Entra ID when using this method.

```yaml
services:
  open-webui:
    image: ghcr.io/open-webui/open-webui:main
    restart: unless-stopped
    ports:
      - "3000:8080"
    volumes:
      - open-webui-data:/app/backend/data
    environment:
      WEBUI_URL: "https://chat.example.com"
      WEBUI_SECRET_KEY: "${WEBUI_SECRET_KEY}"

      ENABLE_OAUTH_SIGNUP: "true"
      OAUTH_PROVIDER_NAME: "Microsoft"
      OAUTH_CLIENT_ID: "${OAUTH_CLIENT_ID}"
      OAUTH_CLIENT_SECRET: "${OAUTH_CLIENT_SECRET}"
      OPENID_PROVIDER_URL: "https://login.microsoftonline.com/${TENANT_ID}/v2.0/.well-known/openid-configuration"
      OPENID_REDIRECT_URI: "https://chat.example.com/oauth/oidc/callback"
      OAUTH_SCOPES: "openid email profile"
      OAUTH_MERGE_ACCOUNTS_BY_EMAIL: "true"

volumes:
  open-webui-data:
```

### Kubernetes

Store the credentials in a Kubernetes `Secret` and reference them:

```yaml
env:
  - name: WEBUI_URL
    value: "https://chat.example.com"
  - name: ENABLE_OAUTH_SIGNUP
    value: "true"
  - name: MICROSOFT_CLIENT_ID
    valueFrom:
      secretKeyRef:
        name: openwebui-entra
        key: client-id
  - name: MICROSOFT_CLIENT_SECRET
    valueFrom:
      secretKeyRef:
        name: openwebui-entra
        key: client-secret
  - name: MICROSOFT_CLIENT_TENANT_ID
    valueFrom:
      secretKeyRef:
        name: openwebui-entra
        key: tenant-id
  - name: MICROSOFT_REDIRECT_URI
    value: "https://chat.example.com/oauth/microsoft/callback"
  - name: OPENID_PROVIDER_URL
    value: "https://login.microsoftonline.com/<TENANT_ID>/v2.0/.well-known/openid-configuration"
```

> [!WARNING]
> Make sure variable names contain **no trailing whitespace** (for example `'OAUTH_CLIENT_ID '`). Open WebUI silently ignores misspelled variables.

### Persistent configuration

Open WebUI stores some settings in its database after the first start (`PersistentConfig`). If you change an OAuth variable later and nothing happens, either change it in **Admin Panel > Settings**, or set `ENABLE_OAUTH_PERSISTENT_CONFIG=false` so environment variables always win, then restart the container.

## Step 7: Restrict access

By default, anyone in your tenant can sign in. Restrict this with one or more of the options below.

### Option 1: Require user assignment in Entra ID (recommended)

1. Go to **Entra ID > Enterprise applications** and open **Open WebUI**.
2. Under **Manage > Properties**, set **Assignment required?** to **Yes** and save.
3. Under **Manage > Users and groups**, click **Add user/group** and assign the allowed users or security groups.

Unassigned users now receive error `AADSTS50105`.

### Option 2: Restrict by e-mail domain in Open WebUI

```env
OAUTH_ALLOWED_DOMAINS=example.com
```

Separate multiple domains with commas, for example `example.com,subsidiary.com`.

### Option 3: Conditional Access

Use Entra ID **Conditional Access** policies targeting the Open WebUI enterprise application to require MFA, compliant devices or trusted locations.

### Default role for new users

```env
# New users must be approved by an admin
DEFAULT_USER_ROLE=pending
# Or: new users are active immediately
# DEFAULT_USER_ROLE=user
```

## Step 8: Map Entra app roles to Open WebUI roles (optional)

### 8.1 Create app roles in Entra ID

1. Open the app registration and go to **Manage > App roles > Create app role**.
2. Create the following roles:

   | Display name | Allowed member types | Value |
   |--------------|----------------------|-------|
   | Open WebUI Admin | Users/Groups | `admin` |
   | Open WebUI User | Users/Groups | `user` |

3. Make sure both roles are enabled, then save.

### 8.2 Assign roles

Go to **Enterprise applications > Open WebUI > Users and groups**, select an assignment and choose **Edit assignment > Select role**.

### 8.3 Enable role management in Open WebUI

```env
ENABLE_OAUTH_ROLE_MANAGEMENT=true
OAUTH_ROLES_CLAIM=roles
OAUTH_ALLOWED_ROLES=user,admin
OAUTH_ADMIN_ROLES=admin
```

Users with `admin` in the `roles` claim become Open WebUI admins. Users with neither role are denied access.

## Step 9: Sync Entra groups (optional)

### 9.1 Add the groups claim in Entra ID

1. Open the app registration and go to **Manage > Token configuration**.
2. Click **Add groups claim**.
3. Choose **Security groups**, or **Groups assigned to the application** to keep tokens small.
4. Keep the default **Group ID** option for ID tokens and click **Add**.

> [!NOTE]
> Entra emits group **object IDs (GUIDs)**, not display names. Name your Open WebUI groups after those IDs.

> [!WARNING]
> Users in too many groups (roughly more than 200 in a JWT) trigger a *group overage* and the list is omitted from the token. Use **Groups assigned to the application** to avoid this.

### 9.2 Enable group management in Open WebUI

```env
ENABLE_OAUTH_GROUP_MANAGEMENT=true
OAUTH_GROUP_CLAIM=groups
# Set to true to auto-create groups found in the token
ENABLE_OAUTH_GROUP_CREATION=false
```

## Step 10: Reverse proxy configuration

Make sure `WEBUI_URL`, the redirect URI and the public URL all agree.

### Nginx

```nginx
server {
    listen 443 ssl http2;
    server_name chat.example.com;

    ssl_certificate     /etc/letsencrypt/live/chat.example.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/chat.example.com/privkey.pem;

    location / {
        proxy_pass http://127.0.0.1:3000;
        proxy_http_version 1.1;

        proxy_set_header Host              $host;
        proxy_set_header X-Real-IP         $remote_addr;
        proxy_set_header X-Forwarded-For   $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;

        # WebSocket support
        proxy_set_header Upgrade    $http_upgrade;
        proxy_set_header Connection "upgrade";

        proxy_read_timeout 300s;
        proxy_buffering off;
    }
}
```

### Caddy

```caddyfile
chat.example.com {
    reverse_proxy 127.0.0.1:3000
}
```

### Traefik

```yaml
labels:
  - "traefik.enable=true"
  - "traefik.http.routers.openwebui.rule=Host(`chat.example.com`)"
  - "traefik.http.routers.openwebui.entrypoints=websecure"
  - "traefik.http.routers.openwebui.tls.certresolver=letsencrypt"
  - "traefik.http.services.openwebui.loadbalancer.server.port=8080"
```

> [!TIP]
> Always set `MICROSOFT_REDIRECT_URI` (Method A) or `OPENID_REDIRECT_URI` (Method B) explicitly. Otherwise Open WebUI derives it from the request, which can yield an `http://` URL behind a proxy and break the login.

## Step 11: Test the login

1. Restart Open WebUI so the new variables are loaded:

   ```bash
   docker compose down && docker compose up -d
   docker compose logs -f open-webui
   ```

2. Open a private/incognito window and browse to `https://chat.example.com`.
3. Confirm that a **Continue with Microsoft** button (Method A) or a button with your `OAUTH_PROVIDER_NAME` (Method B) appears.
4. Click it and sign in with an Entra ID account, completing MFA if required.
5. You are redirected back to Open WebUI and logged in.
6. Check **Admin Panel > Users** to confirm the account exists with the correct role.

> [!IMPORTANT]
> On a fresh instance, the first user to sign up becomes the administrator. Make sure this is you or an intended admin, or use role management from Step 8.

## Step 12: Harden the deployment

After SSO works:

- **Disable local password login** (SSO only):

  ```env
  ENABLE_PASSWORD_AUTH=false
  ENABLE_LOGIN_FORM=false
  ```

  Make sure at least one Entra user is an admin **before** doing this, otherwise you can lock yourself out.

- **Disable local sign-ups:**

  ```env
  ENABLE_SIGNUP=false
  ```

- **Pin the container image** to a specific version instead of `:main`.
- **Keep secrets out of Git.** Use `.env` files listed in `.gitignore`, Docker secrets, Kubernetes secrets or a vault.
- **Use HTTPS only** and add HSTS at the reverse proxy.
- **Require assignment** in Entra (Step 7) and enforce **MFA** through Conditional Access.
- **Back up** the `/app/backend/data` volume.

## Secret rotation

Client secrets expire. To rotate without downtime:

1. In Entra ID, create a **new** client secret under **Certificates & secrets** and keep the old one active.
2. Update `MICROSOFT_CLIENT_SECRET` (or `OAUTH_CLIENT_SECRET`) in your deployment.
3. Restart Open WebUI and verify that login works.
4. Delete the old secret in Entra ID.

## Troubleshooting

| Symptom / error | Likely cause | Fix |
|-----------------|--------------|-----|
| No SSO button on the login page | Variables missing or misspelled, or persisted config overriding them | Check names for typos and trailing spaces, restart, or set `ENABLE_OAUTH_PERSISTENT_CONFIG=false` |
| `AADSTS50011` redirect URI mismatch | URI in Entra differs from what Open WebUI sends | Make them identical, set the redirect URI variable explicitly, check `https` vs `http` behind a proxy |
| `AADSTS7000215` invalid client secret | Secret ID copied instead of Value, or secret expired | Create a new secret and use its **Value** |
| `AADSTS700016` application not found | Wrong client ID or tenant ID | Re-copy the values from the Overview page |
| `AADSTS50105` user not assigned | "Assignment required" is on and the user is not assigned | Assign the user or group to the enterprise application |
| `AADSTS65001` consent required | Admin consent not granted | Grant admin consent under **API permissions** |
| "Access prohibited" after login | `ENABLE_OAUTH_SIGNUP` not `true`, domain not allowed, or role not in `OAUTH_ALLOWED_ROLES` | Check `ENABLE_OAUTH_SIGNUP`, `OAUTH_ALLOWED_DOMAINS` and `OAUTH_ALLOWED_ROLES` |
| Duplicate accounts or "email already registered" | A local account with the same e-mail exists | Set `OAUTH_MERGE_ACCOUNTS_BY_EMAIL=true` (only if you trust the IdP's e-mail claim) |
| Logout does not fully sign the user out | `OPENID_PROVIDER_URL` missing | Set it to your tenant discovery URL |
| Users are logged out after restarts | `WEBUI_SECRET_KEY` changes on every start | Set a fixed `WEBUI_SECRET_KEY` |
| Groups not syncing | Groups claim missing, group overage, or names differ from object IDs | Add the groups claim, use "Groups assigned to the application", match group IDs |
| Role mapping not working | `roles` claim missing from the token | Create app roles, assign them, set `OAUTH_ROLES_CLAIM=roles` |
| Works locally but not behind a proxy | `X-Forwarded-Proto` missing or WebSockets blocked | Use the proxy examples in Step 10 |

### Debugging tips

- Read the container logs: `docker compose logs -f open-webui`
- Inspect tokens of **test accounts** with Microsoft's viewer at <https://jwt.ms>. Never paste production tokens into third-party sites.
- Check **Entra ID > Monitoring > Sign-in logs** for the exact failure reason.
- Verify that the discovery document loads in your browser:
  `https://login.microsoftonline.com/<TENANT_ID>/v2.0/.well-known/openid-configuration`

## Environment variable reference

### General

| Variable | Purpose | Example |
|----------|---------|---------|
| `WEBUI_URL` | Public URL of Open WebUI | `https://chat.example.com` |
| `WEBUI_SECRET_KEY` | Secret used to sign sessions | output of `openssl rand -hex 32` |
| `ENABLE_OAUTH_SIGNUP` | Auto-create users on first SSO login | `true` |
| `OAUTH_MERGE_ACCOUNTS_BY_EMAIL` | Link SSO logins to existing accounts with the same e-mail | `true` |
| `OAUTH_ALLOWED_DOMAINS` | Allowed e-mail domains | `example.com` |
| `DEFAULT_USER_ROLE` | Role assigned to new users | `pending` or `user` |
| `ENABLE_OAUTH_PERSISTENT_CONFIG` | Persist OAuth settings in the database | `false` |

### Method A: Microsoft provider

| Variable | Purpose | Example |
|----------|---------|---------|
| `MICROSOFT_CLIENT_ID` | Application (client) ID | GUID |
| `MICROSOFT_CLIENT_SECRET` | Client secret value | n/a |
| `MICROSOFT_CLIENT_TENANT_ID` | Directory (tenant) ID | GUID |
| `MICROSOFT_REDIRECT_URI` | Redirect URI | `https://chat.example.com/oauth/microsoft/callback` |
| `MICROSOFT_OAUTH_SCOPE` | Requested scopes | `openid email profile offline_access` |
| `OPENID_PROVIDER_URL` | Discovery URL (needed for proper logout) | `https://login.microsoftonline.com/<TENANT_ID>/v2.0/.well-known/openid-configuration` |

### Method B: Generic OIDC

| Variable | Purpose | Example |
|----------|---------|---------|
| `OAUTH_CLIENT_ID` | Application (client) ID | GUID |
| `OAUTH_CLIENT_SECRET` | Client secret value | n/a |
| `OPENID_PROVIDER_URL` | Discovery URL (mandatory) | see above |
| `OPENID_REDIRECT_URI` | Redirect URI | `https://chat.example.com/oauth/oidc/callback` |
| `OAUTH_PROVIDER_NAME` | Login button label | `Microsoft` |
| `OAUTH_SCOPES` | Requested scopes | `openid email profile` |

### Roles and groups

| Variable | Purpose | Example |
|----------|---------|---------|
| `ENABLE_OAUTH_ROLE_MANAGEMENT` | Map token roles to Open WebUI roles | `true` |
| `OAUTH_ROLES_CLAIM` | Claim that contains roles | `roles` |
| `OAUTH_ALLOWED_ROLES` | Roles allowed to log in | `user,admin` |
| `OAUTH_ADMIN_ROLES` | Roles that become admins | `admin` |
| `ENABLE_OAUTH_GROUP_MANAGEMENT` | Sync groups from the token | `true` |
| `OAUTH_GROUP_CLAIM` | Claim that contains groups | `groups` |
| `ENABLE_OAUTH_GROUP_CREATION` | Auto-create groups from the token | `false` |

### Hardening

| Variable | Purpose | Example |
|----------|---------|---------|
| `ENABLE_PASSWORD_AUTH` | Allow local password authentication | `false` |
| `ENABLE_LOGIN_FORM` | Show the local login form | `false` |
| `ENABLE_SIGNUP` | Allow local sign-ups | `false` |

## References

- [Open WebUI SSO documentation](https://docs.openwebui.com/features/access-security/auth/sso)
- [Open WebUI SSO troubleshooting](https://docs.openwebui.com/troubleshooting/sso)
- [Open WebUI environment variable reference](https://docs.openwebui.com/getting-started/env-configuration)
- [Microsoft Entra: register an application](https://learn.microsoft.com/entra/identity-platform/quickstart-register-app)
- [Microsoft Entra: restrict an app to a set of users](https://learn.microsoft.com/entra/identity-platform/howto-restrict-your-app-to-a-set-of-users)
- [Microsoft Entra: optional claims and group claims](https://learn.microsoft.com/entra/identity-platform/optional-claims)
- [Microsoft Entra: app roles](https://learn.microsoft.com/entra/identity-platform/howto-add-app-roles-in-apps)

## License

This guide is provided as-is under the [MIT License](https://opensource.org/licenses/MIT). Feel free to adapt it for your own deployment.
