# Nginx — Production Configuration Skill

## Role

You are a **senior infrastructure engineer**. You configure Nginx for production: security headers, rate limiting, SSL, and performance. Default Nginx config is not production-ready — you always harden it.

---

## Production nginx.conf

```nginx
# /etc/nginx/nginx.conf
user nginx;
worker_processes auto;
worker_rlimit_nofile 65535;
error_log /var/log/nginx/error.log warn;
pid /var/run/nginx.pid;

events {
    worker_connections 1024;
    use epoll;
    multi_accept on;
}

http {
    include /etc/nginx/mime.types;
    default_type application/octet-stream;

    # Security
    server_tokens off;  # don't reveal Nginx version

    # Logging with request ID
    log_format main '$remote_addr - $remote_user [$time_local] "$request" '
                    '$status $body_bytes_sent "$http_referer" '
                    '"$http_user_agent" "$http_x_forwarded_for" '
                    'rt=$request_time id=$request_id';

    access_log /var/log/nginx/access.log main;

    # Performance
    sendfile on;
    tcp_nopush on;
    tcp_nodelay on;
    keepalive_timeout 65;
    types_hash_max_size 2048;
    client_max_body_size 20M;

    # Compression
    gzip on;
    gzip_vary on;
    gzip_proxied any;
    gzip_comp_level 6;
    gzip_types
        text/plain
        text/css
        text/xml
        text/javascript
        application/json
        application/javascript
        application/xml+rss
        application/atom+xml
        image/svg+xml;

    # Rate limiting zones
    limit_req_zone $binary_remote_addr zone=api:10m rate=100r/m;
    limit_req_zone $binary_remote_addr zone=auth:10m rate=10r/m;
    limit_req_zone $binary_remote_addr zone=global:10m rate=1000r/m;

    include /etc/nginx/conf.d/*.conf;
}
```

### Application Server Block

```nginx
# /etc/nginx/conf.d/app.conf

# Redirect HTTP to HTTPS
server {
    listen 80;
    server_name example.com www.example.com;
    return 301 https://$host$request_uri;
}

server {
    listen 443 ssl http2;
    server_name example.com www.example.com;

    root /var/www/html/public;
    index index.php;

    # SSL
    ssl_certificate /etc/letsencrypt/live/example.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/example.com/privkey.pem;
    ssl_session_timeout 1d;
    ssl_session_cache shared:SSL:50m;
    ssl_session_tickets off;
    ssl_protocols TLSv1.2 TLSv1.3;
    ssl_ciphers ECDHE-ECDSA-AES128-GCM-SHA256:ECDHE-RSA-AES128-GCM-SHA256:ECDHE-ECDSA-AES256-GCM-SHA384:ECDHE-RSA-AES256-GCM-SHA384;
    ssl_prefer_server_ciphers off;

    # HSTS
    add_header Strict-Transport-Security "max-age=31536000; includeSubDomains; preload" always;

    # Security headers
    add_header X-Frame-Options "SAMEORIGIN" always;
    add_header X-Content-Type-Options "nosniff" always;
    add_header X-XSS-Protection "1; mode=block" always;
    add_header Referrer-Policy "strict-origin-when-cross-origin" always;
    add_header Permissions-Policy "camera=(), microphone=(), geolocation=()" always;
    add_header Content-Security-Policy "default-src 'self'; script-src 'self' 'nonce-$request_id'; style-src 'self' 'unsafe-inline'; img-src 'self' data: https:; font-src 'self'; connect-src 'self'; frame-ancestors 'none';" always;

    # Request ID for tracing
    add_header X-Request-ID $request_id always;

    # Rate limiting — global
    limit_req zone=global burst=50 nodelay;

    # Laravel/App routing
    location / {
        try_files $uri $uri/ /index.php?$query_string;
    }

    # PHP-FPM
    location ~ \.php$ {
        fastcgi_pass unix:/run/php/php8.3-fpm.sock;
        fastcgi_index index.php;
        fastcgi_buffers 16 16k;
        fastcgi_buffer_size 32k;
        include fastcgi_params;
        fastcgi_param SCRIPT_FILENAME $realpath_root$fastcgi_script_name;
        fastcgi_param DOCUMENT_ROOT $realpath_root;
        fastcgi_read_timeout 60;
    }

    # API rate limiting (stricter)
    location /api/ {
        limit_req zone=api burst=20 nodelay;
        try_files $uri $uri/ /index.php?$query_string;
    }

    # Auth endpoints (strictest)
    location ~* ^/api/v[0-9]+/(login|register|forgot-password|reset-password) {
        limit_req zone=auth burst=5 nodelay;
        try_files $uri $uri/ /index.php?$query_string;
    }

    # Static assets — long cache, immutable
    location ~* \.(css|js|png|jpg|jpeg|gif|ico|svg|woff2|woff|ttf)$ {
        expires 1y;
        add_header Cache-Control "public, immutable";
        access_log off;
    }

    # Block access to sensitive files
    location ~ /\. {
        deny all;
        access_log off;
        log_not_found off;
    }

    location ~ \.(env|log|sh|sql|bak)$ {
        deny all;
        access_log off;
    }

    # Health check — no rate limit, no auth
    location /api/health {
        limit_req off;
        try_files $uri /index.php?$query_string;
        access_log off;
    }
}
```

---

## Common Anti-Patterns

```nginx
# REJECT: Missing server_tokens off (leaks Nginx version)
# Attackers use version info to target known CVEs

# REJECT: SSLv3 or TLS 1.0/1.1
ssl_protocols SSLv3 TLSv1;

# REJECT: No rate limiting on auth endpoints
location /api/login {
    # unlimited requests — brute force welcome
}

# REJECT: Serving .env files
location / {
    try_files $uri $uri/ /index.php?$q; # .env is accessible if in webroot
}

# REJECT: No HSTS
# Without HSTS, users can be downgraded to HTTP via MITM

# REJECT: Overly permissive CSP
add_header Content-Security-Policy "default-src *";
```

---

## Deployment Gates

- [ ] `nginx -t` passes
- [ ] HTTPS redirect active (HTTP → HTTPS)
- [ ] SSL Labs score: A+ ([ssllabs.com/ssltest](https://www.ssllabs.com/ssltest/))
- [ ] `server_tokens off`
- [ ] All security headers present (check with [securityheaders.com](https://securityheaders.com))
- [ ] Rate limiting on `/api/` and auth endpoints
- [ ] `.env`, `.git`, `.log` files blocked
- [ ] Health check endpoint returns 200 without auth
- [ ] Gzip enabled
- [ ] Static asset caching configured
