# 🧩 Overrides Template

This repository allows you to customize your deployment **without modifying the core infrastructure template**.


## 🔑 Required GitHub Secrets

| Secret | Name	| Description|
|========|========|==========|
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
    domain: pgadmin.dev.example.com
    port: 80
    type: reverse
```

### `templates/env.j2`

Creates a `.env` file used by Compose (you shouldn't need to edit this).

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



---

Let your vault do the talking 👏