FROM quay.io/oauth2-proxy/oauth2-proxy:v7.15.3@sha256:10a1165743a192e1940b4708fb9647027185ce11a681a1c5519b442ff7f1f561 AS oauth2-proxy
FROM ghcr.io/hassio-addons/base:21.0.0@sha256:26c14772d1eb72c8aa7f0b6866a2bc13ada705f77b8c263fcfa09116a5e00e25
COPY rootfs /
COPY --from=oauth2-proxy /usr/bin/oauth2-proxy /usr/bin/oauth2-proxy
LABEL \
  org.opencontainers.image.title="Home Assistant Oauth2 Proxy" \
  org.opencontainers.image.description="Runs Oauth2 Proxy, a middleware enabling SSO authentication for any HTTP-endpoint" \
  org.opencontainers.image.source="https://github.com/crazyrokr/ha-oauth2-proxy" \
  org.opencontainers.image.licenses="MIT License"
