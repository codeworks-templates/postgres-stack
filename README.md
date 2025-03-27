# 🧩 Overrides Template

This repository allows you to customize your deployment **without modifying the core infrastructure template**.


## 🔑 Required GitHub Secrets

| Secret | Name	| Description|
|--------|--------|------------|
| `VAULT_YML` | `VAULT_YML` | Plaintext content of the vault YAML config |
| `SSH_USER` | `SSH_USER` | SSH login user for server |
| `SSH_PRIVATE_KEY` | `SSH_PRIVATE_KEY` | SSH key with access
| `STAGING_HOST` | `STAGING_HOST` | IP or domain of staging server |
| `PROD_HOST` | `PROD_HOST` | IP or domain of prod server |


## 🔧 How it works

1. All your environment-specific settings and secrets go in an **encrypted** file: `group_vars/overrides/vault.yml`
2. Ansible decrypts that file and uses Jinja2 templates to:
   - Create a `.env` file
   - Inject your `docker-compose.override.yml`
   - Generate a `Caddyfile` with proper subdomains

## 📁 Example Template

### `group_vars/overrides/vault.yml`

Contains your env secrets and config:

```yaml
# This is an example of a vault file for Ansible group_vars
# To keep your secrets secure create a GitHub secret named VAULT_YML
# and store the contents of this file in it.
# Add this file to your .gitignore to prevent it from being committed to your repository.
env_vars:
  # Required by postgres-core
  # These variables are used to configure the PostgreSQL database
  POSTGRES_USER: root               # use a strong username 
  POSTGRES_PASSWORD: rootpassword   # use a strong password
  APP_USER: appuser                 # the remote user for connection_string
  APP_PASS: apppass                 # use a strong password
  # S3 Bucket for backups
  # Uncomment and set the following variables if you want to use S3 for backups
  # S3_BUCKET: your-s3-bucket-name 
  
  # Uncomment and set the following variables if you want to use pgAdmin
  # PGADMIN_DEFAULT_EMAIL: admin@yourdomain.dev
  # PGADMIN_DEFAULT_PASSWORD: securepassword

  # Add any other environment variables your application needs here

services:
  frontend:
    image: ghcr.io/myuser/app-frontend:latest
    domain: app.dev.example.com
    type: static
  api:
    image: ghcr.io/myuser/app-api:latest
    domain: api.dev.example.com
    port: 5000
  pgadmin:
    image: dpage/pgadmin4:latest
    domain: pgadmin.dev.example.com
    port: 80
    type: reverse
```

### Enable DB Backups to S3
- To enable S3 backups, uncomment the `S3_BUCKET` variable and set it to your bucket name.
- You will need to attach an IAM role to your EC2 instance with permissions to write to the S3 bucket.

1. Go to EC2 > Instances.
2. Select your instance.
3. `Actions > Security > Modify IAM Role.`
4. Choose an existing IAM Role (or create a new one with S3 access).
  - If you don’t have a role yet: Create one in `IAM > Roles` 
  - Use EC2 as the trusted entity
  - Attach the `AmazonS3FullAccess` (or a more limited policy if you prefer)

### `templates/env.j2`

Creates a `.env` file used by Compose. If you add any new environment variables to your `vault.yml`, make sure to add them here as well.

### `templates/docker-compose.override.yml.j2`

Allows you to add services like your API. By default, it references the API image and sets the DB connection string.

### `templates/Caddyfile.j2`

Used to automatically configure HTTPS routing for:
- Your frontend (`APP_DOMAIN`)
- Your API (`API_DOMAIN`)
- pgAdmin (`PGADMIN_DOMAIN`)

## 🚀 How to Deploy

1. Clone this repo or use it as a base
2. Create your own `group_vars/overrides/vault.yml` file with your secrets
   - You can use the provided example as a starting point
   - DO NOT commit your actual `vault.yml` file to the repo
3. Use the core `postgres-stack` playbook and pass this repo as the `--extra-vars` source


## Testing without Custom DNS

[nip.io](https://nip.io/) is a free service that resolves any IP address to a domain name. This allows you to test your setup without needing to configure custom DNS records.

When using nip.io, you can set your domain in your service to something like:

```yaml
api:
   image: ghcr.io/myuser/app-api:latest
   # appname.ipaddress.nip.io
   domain: api.123.456.789.10.nip.io
   port: 5000
```


## Optional Services

You can include optional services like Grafana, Prometheus, and pgAdmin by adding them to your `vault.yml`. These services are conditionally rendered in the final `docker-compose.override.yml` and proxied by Caddy **only if a `domain` is defined**.

### Grafana + Prometheus Example

```yaml
services:
  prometheus:
    image: prom/prometheus
    port: 9090
    volumes:
      - ./monitoring/prometheus.yml:/etc/prometheus/prometheus.yml

  grafana:
    image: grafana/grafana
    port: 3000
    environment:
      GF_SECURITY_ADMIN_USER: admin
      GF_SECURITY_ADMIN_PASSWORD: admin
    volumes:
      - grafana_data:/var/lib/grafana
    depends_on:
      - prometheus
    domain: grafana.{{YOUR_IP}}.nip.io
    type: reverse
```

### PG_Admin

To include pgAdmin for managing your PostgreSQL database, add the following to your `vault.yml`:

```yaml
services:
  pgadmin:
    image: dpage/pgadmin4
    port: 80
    domain: pgadmin.{{YOUR_IP}}.nip.io
    type: reverse
```

### Notes

- Services **must include both a `domain` and `type: reverse`** to be exposed through Caddy.
- If `type` is omitted or invalid, the service is ignored in the Caddyfile.
- Internal-only services (like the database, backup, or exporters) are never exposed unless explicitly configured.
- You can use [nip.io](https://nip.io) or similar dynamic DNS providers for easy local development using IP-based hostnames.

### Preview: Caddyfile Output

When using the example above, your generated Caddyfile will look like:

```caddyfile
prometheus.54.214.79.127.nip.io {
  reverse_proxy prometheus:9090
}

grafana.54.214.79.127.nip.io {
  reverse_proxy grafana:3000
}

pgadmin.54.214.79.127.nip.io {
  reverse_proxy pgadmin:80
}
```

## Schema Hint (optional)

To help guide students, you can optionally share this as a sample `vault.schema.yml`:

```yaml
services:
  type: object
  patternProperties:
    "^[a-zA-Z0-9_-]+$":
      type: object
      required: [image, port]
      properties:
        image:
          type: string
        domain:
          type: string
        port:
          type: number
        type:
          type: string
          enum: [reverse]
        env:
          type: object
```



---

Let your vault do the talking 👏
