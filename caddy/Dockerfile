FROM caddy:builder AS builder

RUN xcaddy build \
    --with github.com/caddy-dns/cloudflare \
    --with github.com/ueffel/caddy-brotli

FROM caddy:2.4.6-alpine

COPY --from=builder /usr/bin/caddy /usr/bin/caddy
