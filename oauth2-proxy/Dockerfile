FROM quay.io/oauth2-proxy/oauth2-proxy:v7.15.4@sha256:b1b2021fe8f4004573e8d690dec6c7bb29cc44364572cf8510a05bf3a0ae2ded AS oauth2-proxy
FROM ghcr.io/hassio-addons/base:21.0.5@sha256:2329e9f61ecbfaa9a2b1b2b260b5a692872f95634fbc58c1a5a685b4bec566cb
COPY rootfs /
COPY --from=oauth2-proxy /usr/bin/oauth2-proxy /usr/bin/oauth2-proxy
LABEL \
  org.opencontainers.image.title="Home Assistant Oauth2 Proxy" \
  org.opencontainers.image.description="Runs Oauth2 Proxy, a middleware enabling SSO authentication for any HTTP-endpoint" \
  org.opencontainers.image.source="https://github.com/crazyrokr/ha-oauth2-proxy" \
  org.opencontainers.image.licenses="MIT License"
