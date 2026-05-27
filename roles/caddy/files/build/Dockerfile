# Use the official Caddy 'builder' image as a temporary build stage
FROM caddy:2-builder AS builder

# Use the xcaddy tool to build a new Caddy binary with our desired plugin
RUN xcaddy build \
    --with github.com/caddy-dns/cloudflare

# Start a new, clean stage with the standard Caddy image
FROM caddy:latest

# Copy the custom-built Caddy binary from the 'builder' stage over the original one
COPY --from=builder /usr/bin/caddy /usr/bin/caddy