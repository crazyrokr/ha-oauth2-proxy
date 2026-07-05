# Home Assistant Add-on: OAuth2 Proxy

[![License][license-shield]][license-url]
![Supports aarch64 Architecture][aarch64-shield]
![Supports amd64 Architecture][amd64-shield]
[![CI/CD Status][cicd-shield]][cicd-url]

A specialized Home Assistant Add-on running [OAuth2 Proxy](https://github.com/oauth2-proxy/oauth2-proxy). OAuth2 Proxy acts as a secure authentication gateway, enabling Single Sign-On (SSO) and Multi-Factor Authentication (MFA) via major identity providers (such as Google, GitHub, Keycloak, Okta, and others) to protect your Home Assistant instance or other local HTTP endpoints.

---

## 🏗️ Architecture & Internals

Understanding how this add-on operates under the hood is crucial for robust configuration and debugging.

### 1. Multi-Stage Container Design
The add-on uses a dual-source Docker build:
* **Base Environment:** Built on the secure, stable [Home Assistant Community Add-on Base Image (`ghcr.io/hassio-addons/base`)](https://github.com/hassio-addons/addon-base) which includes the essential `bashio` helper library and an s6-overlay runtime environment.
* **Service Binary:** Extracted directly from the official, highly secure [OAuth2 Proxy Distroless/Alpine image (`quay.io/oauth2-proxy/oauth2-proxy`)](https://quay.io/repository/oauth2-proxy/oauth2-proxy) using an explicit SHA256 digest pinning.

### 2. Process Management via s6-overlay v3
Instead of running a basic entry point script, the add-on is fully integrated with **s6-overlay v3** for enterprise-grade service supervision:
* **Initialization (`/etc/s6-overlay/s6-rc.d/oauth2-proxy/run`):**
  1. Utilizes Home Assistant's native `bashio` command-line utility to retrieve configured options from `/data/options.json`.
  2. Dynamically maps and exports configuration parameters to their respective `OAUTH2_PROXY_*` environment variables.
  3. Seamlessly boots up the daemon via `/usr/bin/bootstrap`.
* **Teardown & Crash Monitoring (`/etc/s6-overlay/s6-rc.d/oauth2-proxy/finish`):**
  1. Tracks the active daemon's exit codes.
  2. If the process crashes (exits with any status other than `0` or `256`), it logs a warning and forces a complete container halt.
  3. Otherwise, it allows s6-overlay to perform clean restarts.

---

## 🚀 Installation

1. Open your Home Assistant frontend and navigate to **Settings > Add-ons > Add-on Store**.
2. Click the three vertical dots (top-right corner) and select **Repositories**.
3. Add the following repository URL:
   ```
   https://github.com/crazyrokr/hassio-addons
   ```
4. Search for **Oauth2 Proxy** in the list, click **Install**, and wait for the process to complete.

---

## ⚙️ Configuration

Configure the add-on via the **Configuration** tab in the Home Assistant UI.

### Options & Environment Variable Mappings

The Home Assistant configuration options are mapped dynamically to standard `OAUTH2_PROXY_*` environment variables during s6 initialization:

| Parameter | Type | Required | Default | Target Environment Variable | Description |
| :--- | :--- | :---: | :--- | :--- | :--- |
| `auth_url` | string | No | `(None)` | `OAUTH2_PROXY_REDIRECT_URL` | Callback URL for your identity provider. |
| `ssl` | bool | Yes | `false` | *(Internal)* | Enable TLS/SSL. |
| `cert_file` | string | If `ssl` | `fullchain.pem` | `OAUTH2_PROXY_TLS_CERT_FILE` | SSL cert file (in `/ssl`). |
| `key_file` | string | If `ssl` | `privkey.pem` | `OAUTH2_PROXY_TLS_KEY_FILE` | SSL key file (in `/ssl`). |
| `client_id` | string | **Yes** | `(None)` | `OAUTH2_PROXY_CLIENT_ID` | OAuth2 Client ID. |
| `client_secret` | string | **Yes** | `(None)` | `OAUTH2_PROXY_CLIENT_SECRET` | OAuth2 Client Secret. |
| `oidc_issuer_url` | string | **Yes** | `(None)` | `OAUTH2_PROXY_OIDC_ISSUER_URL` | OIDC Issuer URL. |
| `cookie_secret` | string | **Yes** | `(None)` | `OAUTH2_PROXY_COOKIE_SECRET` | Secure cookie encryption secret. |
| `upstreams` | string | **Yes** | `(None)` | `OAUTH2_PROXY_UPSTREAMS` | Downstream service URL. |
| `provider` | string | **Yes** | `oidc` | `OAUTH2_PROXY_PROVIDER` | Auth provider (e.g., `oidc`). |
| `scope` | string | **Yes** | `openid email profile groups` | `OAUTH2_PROXY_SCOPE` | Auth scopes. |
| `reverse_proxy` | bool | Yes | `true` | `OAUTH2_PROXY_REVERSE_PROXY` | Running behind a reverse proxy. |
| `email_domains` | string | **Yes** | `(None)` | `OAUTH2_PROXY_EMAIL_DOMAINS` | Allowed email domains. |
| `insecure_oidc_allow_unverified_email` | bool | Yes | `true` | `OAUTH2_PROXY_INSECURE_OIDC_ALLOW_UNVERIFIED_EMAIL` | Allow unverified OIDC emails. |
| `cookie_secure` | bool | Yes | `true` | `OAUTH2_PROXY_COOKIE_SECURE` | Enable secure cookies. |
| `http_address` | string | **Yes** | `0.0.0.0:4180` | `OAUTH2_PROXY_HTTP_ADDRESS` | Listen address. |
| `skip_provider_button` | bool | Yes | `true` | `OAUTH2_PROXY_SKIP_PROVIDER_BUTTON` | Skip the login button page. |
| `cookie_name` | string | No | `(None)` | `OAUTH2_PROXY_COOKIE_NAME` | Custom cookie name. |

---

## 🌐 NGINX Integration (`auth_request`)

To secure Home Assistant behind OAuth2 Proxy, configure NGINX using the `auth_request` module. This setup intercepts incoming traffic, routes authentication checks to the proxy, and propagates user headers once authenticated.

Below is a robust NGINX configuration snippet:

```nginx
server {
    listen 443 ssl;
    server_name your-domain.com;

    # SSL Configuration (using certificates located in your local proxy host)
    ssl_certificate /etc/letsencrypt/live/your-domain.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/your-domain.com/privkey.pem;

    # Intercept and protect Home Assistant endpoints
    location / {
        auth_request /oauth2/auth;
        error_page 401 = /oauth2/sign_in;

        # Propagate user metadata from OAuth2 Proxy to downstream Home Assistant
        auth_request_set $user   $upstream_http_x_auth_request_user;
        auth_request_set $email  $upstream_http_x_auth_request_email;
        proxy_set_header X-User  $user;
        proxy_set_header X-Email $email;

        proxy_pass http://127.0.0.1:8123; # IP and port of Home Assistant Core
        
        # Standard Home Assistant proxy headers
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "upgrade";
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }

    # Redirect auth endpoint to OAuth2 Proxy
    location /oauth2/ {
        proxy_pass http://127.0.0.1:4180; # Add-on exposed port
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Scheme $scheme;
        proxy_set_header X-Auth-Request-Redirect $request_uri;
    }
}
```

---

## 🛡️ Home Assistant Trusted Proxy Configuration

When using an external reverse proxy (like NGINX) to route and authenticate traffic, Home Assistant's internal security layer requires explicit trust configurations. Add the following block to your `configuration.yaml` file to prevent requests from being blocked:

```yaml
http:
  use_x_forwarded_for: true
  trusted_proxies:
    - 127.0.0.1       # Localhost (if NGINX is on the same machine)
    - 172.30.33.0/24  # Standard Home Assistant Supervisor docker network (adjust as needed)
```

---

## 🛠️ Local Development & CI/CD

If you wish to contribute or build this add-on locally, please keep the following rules in mind:

### 1. Docker Pinning
The project utilizes an automated dependency lock workflow `.github/workflows/cicd.yaml` that triggers `gha-workflows/ha-addon-auto-locker`. This action parses your `Dockerfile` and locks the base images by their strict cryptographic digest (`sha256`), preventing unpredictable upstream runtime modifications.

### 2. Multi-Architecture Builds
The build setup supports compilation on both `amd64` and `aarch64` architectures. To test builds locally, use the standard [Home Assistant Add-on Developer Tools (Hass.io CLI compiler)](https://developers.home-assistant.io/docs/add-ons/testing/).

---

## 🤝 Support & Contributions

If you find any bugs, have feature requests, or want to contribute configuration scripts:
1. Search or open a ticket on the [GitHub Issue Tracker](https://github.com/crazyrokr/ha-oauth2-proxy/issues).
2. For code modifications, please submit a Pull Request.

---

## 📄 License

This project is open-source and licensed under the terms of the [MIT License](LICENSE).

[license-shield]: https://img.shields.io/github/license/crazyrokr/ha-oauth2-proxy.svg
[license-url]: LICENSE
[aarch64-shield]: https://img.shields.io/badge/aarch64-yes-green.svg
[amd64-shield]: https://img.shields.io/badge/amd64-yes-green.svg
[cicd-shield]: https://github.com/crazyrokr/ha-oauth2-proxy/workflows/CI/CD/badge.svg
[cicd-url]: https://github.com/crazyrokr/ha-oauth2-proxy/actions
