# whispergate.ludus_nginx_redirector

An Ansible role for Ludus to deploy and configure a malleable Nginx-based HTTP/HTTPS redirector. This role is designed for use in Ludus ranges to proxy C2 traffic with extensive customization options for routing, rate limiting, and operational security.

Although many operators prefer Apache2 with mod_rewrite, Nginx provides superior performance and flexibility for high-throughput C2 redirector infrastructure. This role provides a solid foundation for building malleable redirectors with custom routing logic.

**Table of Contents:**
- [Features](#-features)
- [Installation](#-installation-in-ludus)
- [Usage](#-usage)
- [Role Variables](#-role-variables)
- [How Nginx Redirector Works](#how-nginx-redirector-works)
- [Security Considerations](#security-considerations)
- [Troubleshooting](#troubleshooting)

## Features

- **Installs and configures Nginx** with full SSL/TLS support
- **Malleable redirector logic** via custom location blocks and routing rules
- **Custom upstream proxying** with health checks and failover
- **Dynamic header manipulation** for C2 traffic blending
- **Gzip compression** support for bandwidth optimization
- **Flexible logging** for operational awareness and troubleshooting

## Installation in Ludus

### Install via Ansible Galaxy:

```sh
ludus ansible role add whispergate.ludus_nginx_redirector
```

### Or clone directly:

```sh
git clone https://github.com/Whispergate/ludus_nginx_redirector.git /opt/ludus_nginx_redirector
ludus ansible role add /opt/ludus_nginx_redirector
```

### Customizing Templates and Files

If you want to **customize the Nginx configuration templates** or **static files**, clone the role:

- **Nginx configuration templates** are located in:
  - `ludus_nginx_redirector/templates/nginx.conf.j2` - Main Nginx config
  - `ludus_nginx_redirector/templates/c2_upstream.conf.j2` - Upstream definitions
  - `ludus_nginx_redirector/templates/c2_vhost.conf.j2` - Virtual host config
  - `ludus_nginx_redirector/templates/rate_limit.conf.j2` - Rate limiting config

- **Landing page** (deployed to web root):
  - `ludus_nginx_redirector/files/index.html` - Professional financial services SPA

Edit these files as needed, then run your playbook. Your custom templates will be deployed to the target server.

### Default Landing Page

The role automatically deploys a professional-looking **Pinnacle Financial Solutions** Single Page Application (SPA) as the default landing page. This provides operational security benefits:

- **OPSEC**: Presents a legitimate business front when accessed by analysts/scanners
- **Realistic Design**: Professional financial services website with contact info, services, and company details
- **Responsive**: Mobile-friendly design that looks genuine across all devices
- **Low Suspicion**: Appears as a normal business website, reducing operational risk

**Key Features of the Landing Page:**
- Portfolio management and wealth advisory services
- Professional contact information and office details
- Interactive UI with smooth scrolling and animations
- Financial industry disclaimers and regulatory information
- Social media links and company branding

**To customize the landing page**, replace `files/index.html` with your own HTML file before deploying the role. You can create any type of front:
- E-commerce site
- Corporate website  
- Blog/news site
- SaaS product landing page

The page is automatically deployed to the nginx web root (`/usr/share/nginx/html` on RedHat or `/var/www/html` on Debian).

## Usage

### Example Ludus Configuration

```yaml
ludus:
  # Nginx Redirector
  - vm_name: "{{ range_id }}-nginx-redir01"
    hostname: "{{ range_id }}-REDIR01"
    template: ubuntu-22.04-x64-server
    vlan: 200
    ip_last_octet: 11
    ram_gb: 2
    cpus: 2
    linux: true
    dns_rewrites:
      - redir.ludus
      - '*.redir.ludus'
    roles:
      - whispergate.ludus_nginx_redirector
    role_vars:
      c2_upstream_servers:
        - address: 'c2.ludus:8080'
          weight: 1
          max_fails: 2
          fail_timeout: 15s
      enable_ssl: true
      enable_rate_limiting: true
      rate_limit_requests: '50r/s'
      custom_locations:
        - path_regex: '^/(admin|console|wp-admin)'
          return_code: 404
        - path: '/api/v1'
          upstream: 'c2_backend'
        - path: '/beacon'
          upstream: 'c2_backend'

  # C2 Server (Mythic example)
  - vm_name: "{{ range_id }}-Mythic"
    hostname: "{{ range_id }}-Mythic"
    template: ubuntu-22.04-x64-server-template
    vlan: 100
    ip_last_octet: 52
    dns_rewrites:
      - c2.ludus
      - '*.c2.ludus'
    ram_gb: 16
    cpus: 4
    linux: true
    roles:
      - ludus_mythic_teamserver

network:
  rules:
    - name: Allow Mythic to Redirector
      vlan_src: 200
      vlan_dst: 100
      protocol: "all"
      ports: "all"
      action: ACCEPT
    - name: Allow Redirector to Mythic
      vlan_src: 100
      vlan_dst: 200
      protocol: "all"
      ports: "all"
      action: ACCEPT
    - name: Allow Target to Redirector HTTP
      vlan_src: 10
      vlan_dst: 100
      protocol: tcp
      ports: 80
      action: ACCEPT
    - name: Allow Target to Redirector HTTPS
      vlan_src: 10
      vlan_dst: 100
      protocol: tcp
      ports: 443
      action: ACCEPT
  inter_vlan_default: DROP # Drop all by default
  external_default: ACCEPT  # Allow outbound internet if needed
```

## Role Variables

All variables are defined in `defaults/main.yml` and can be overridden in your Ludus playbook under `role_vars`.

### Available Variables

| Variable | Default | Description |
|----------|---------|-------------|
| `nginx_install_from_repository` | `true` | Install Nginx from OS repositories |
| `nginx_apt_update_cache` | `true` | Run apt cache update during Debian/Ubuntu Nginx install |
| `nginx_remove_default_vhost` | `true` | Remove default Nginx virtual host |
| `nginx_user` | `localuser` | User that Nginx worker processes run as |
| `nginx_worker_processes` | `auto` | Number of worker processes (auto = CPU count) |
| `nginx_worker_connections` | `768` | Max concurrent connections per worker |
| `c2_upstream_name` | `c2_backend` | Name of upstream group in Nginx config |
| `c2_upstream_servers` | `[{address: 'localhost:8080', ...}]` | List of upstream C2 servers with health checks |
| `nginx_connect_timeout` | `60` | Upstream connection timeout (seconds) |
| `nginx_send_timeout` | `60` | Upstream send timeout (seconds) |
| `nginx_read_timeout` | `60` | Upstream read timeout (seconds) |
| `nginx_proxy_buffering` | `off` | Disable proxy buffering for real-time C2 |
| `enable_ssl` | `false` | Enable HTTPS/TLS |
| `ssl_certificate_path` | `/etc/nginx/ssl/cert.pem` | Path to SSL certificate |
| `ssl_key_path` | `/etc/nginx/ssl/key.pem` | Path to SSL private key |
| `ssl_protocols` | `TLSv1.2 TLSv1.3` | Allowed TLS versions |
| `enable_rate_limiting` | `false` | Enable rate limiting |
| `rate_limit_requests` | `10r/s` | Requests per second per IP |
| `rate_limit_burst` | `20` | Burst size before rate limiting |
| `custom_locations` | `[]` | Custom location blocks for malleable logic |
| `enable_gzip` | `true` | Enable gzip compression |
| `nginx_hide_version` | `true` | Hide Nginx version in headers |

### Custom Locations (Malleable Configuration)

Use `custom_locations` to implement malleable redirector logic:

```yaml
role_vars:
  custom_locations:
    # Return 404 for admin/scanner patterns
    - path: '/admin'
      return_code: 404
    - path: '/phpmyadmin'
      return_code: 403
    
    # Regex-based blocking
    - path_regex: '^/(\.git|\.svn|\.hg)'
      return_code: 404
    - path_regex: '^.*(union|select|drop|insert)'
      return_code: 403
    
    # Route specific C2 paths
    - path: '/api/v1'
      upstream: 'c2_backend'
    - path: '/upload'
      upstream: 'c2_backend'
    - path: '/download'
      upstream: 'c2_backend'
```

### Upstream Configuration

Configure C2 upstream servers with health checks:

```yaml
role_vars:
  c2_upstream_servers:
    - address: '10.{{ range_second_octet }}.50.52:8080'  # Internal C2
      weight: 1
      max_fails: 3      # Mark down after 3 failures
      fail_timeout: 30s # How long before retrying
```

## How Nginx Redirector Works

This role configures Nginx as a transparent HTTP/HTTPS proxy redirector:

1. **Client connects** → Redirector listens on HTTP/HTTPS
2. **Request routing** → Matches against custom location rules
3. **Malleability** → Returns 404/403 for suspicious patterns, proxies C2 paths
4. **Upstream proxying** → Valid C2 requests forwarded to backend with health checks
5. **Response passthrough** → Upstream responses returned to client with custom headers

**Typical Traffic Flow:**
- Legitimate client → Redirector → Upstream C2 server
- Scanner/analyst → Redirector → 404 response
- C2 beacon → Redirector → Upstream (proxied transparently)

## Configuration Examples

### Example 1: Basic HTTP Redirector

```yaml
role_vars:
  c2_upstream_servers:
    - address: 'c2.ludus:8080'
      weight: 1
```

### Example 2: HTTPS with Rate Limiting

```yaml
role_vars:
  enable_ssl: true
  ssl_certificate_path: '/etc/nginx/ssl/cert.pem'
  ssl_key_path: '/etc/nginx/ssl/key.pem'
  enable_rate_limiting: true
  rate_limit_requests: '100r/s'
  rate_limit_burst: 200
```

### Example 3: Malleable Redirector with Custom Routing

```yaml
role_vars:
  c2_upstream_servers:
    - address: 'c2.ludus:8080'
  custom_locations:
    # Block scanners
    - path_regex: '^/(admin|console|wp-admin|phpmyadmin|cgi-bin)'
      return_code: 404
    - path_regex: '(nikto|sqlmap|nmap|masscan)'
      return_code: 403
    # Route C2 traffic
    - path: '/api/schedule'
      upstream: 'c2_backend'
    - path: '/api/task'
      upstream: 'c2_backend'
    - path: '/api/output'
      upstream: 'c2_backend'
```

## Security Considerations

1. **SSL Certificates**: For production in Ludus, integrate with your organization's PKI or use Let's Encrypt
2. **Upstream Security**: Ensure C2 is not accessible from target VLANs (only via redirector)
3. **Rate Limiting**: Configure to match your C2 traffic patterns (don't rate-limit actual beacons)
4. **Logging**: Access logs contain full request data; manage log rotation and retention
5. **Version Hiding**: Nginx version is hidden by default

## Troubleshooting

### Nginx fails to start

Check syntax and logs:
```bash
ssh <redirector-vm>
sudo nginx -t
sudo tail -f /var/log/nginx/error.log
```

### C2 traffic not reaching backend

Verify:
- Upstream address and port in `c2_upstream_servers`
- Network rules allow Redirector VLAN → C2 VLAN
- C2 server is running and accessible on expected port
- Check access logs: `sudo tail -f /var/log/nginx/c2_access.log`

### High latency or timeouts

- Verify timeouts: `nginx_connect_timeout`, `nginx_read_timeout`, `nginx_send_timeout`
- Check proxying: ensure `nginx_proxy_buffering: 'off'` for real-time traffic
- Monitor upstream server health via nginx health checks

### 404 responses for valid C2 traffic

Check your `custom_locations`:
- Verify regex patterns don't match legitimate C2 paths
- Ensure C2 beacon path is in proxied locations
- Test via `curl -v http://redirector/<beacon-path>`

## Files Modified

This role manages:
- `/etc/nginx/nginx.conf` - Main Nginx configuration
- `/etc/nginx/conf.d/*.conf` - Virtual hosts and features
- `/etc/nginx/ssl/` - SSL certificates (if enabled)
- `/etc/nginx/upstream.d/` - Upstream definitions
- `/var/log/nginx/` - Access and error logs

## License

See LICENSE file
