FROM quay.io/oauth2-proxy/oauth2-proxy:v7.15.5@sha256:8498b0d0ef0a7b29686414000a08aee467f02d0299c9ed1e006a8f33fc017916 AS oauth2-proxy
FROM ghcr.io/hassio-addons/base:21.0.4@sha256:e72908cc2e9d90bef85210a927d5e0fd11a7b3e61f608d3d99a95c107eef9a3f
COPY rootfs /
COPY --from=oauth2-proxy /usr/bin/oauth2-proxy /usr/bin/oauth2-proxy
LABEL \
  org.opencontainers.image.title="Home Assistant Oauth2 Proxy" \
  org.opencontainers.image.description="Runs Oauth2 Proxy, a middleware enabling SSO authentication for any HTTP-endpoint" \
  org.opencontainers.image.source="https://github.com/crazyrokr/ha-oauth2-proxy" \
  org.opencontainers.image.licenses="MIT License"
