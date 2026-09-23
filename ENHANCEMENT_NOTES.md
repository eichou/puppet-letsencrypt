# SUSE Let's Encrypt Enhancements Summary

## Overview
This document summarizes three major enhancements to `puppet-letsencrypt`:
1. **Containerized Certbot** via Podman (new `letsencrypt::suse::podman` class)
2. **Packman Repository** mirrors (version-specific URLs in Hiera)
3. **Documentation** and usage examples

---

## Architecture Decisions

### 1. Podman vs Native: Separate Classes
- **Native**: `letsencrypt::suse` - uses zypper to install certbot directly
- **Containerized**: `letsencrypt::suse::podman` - runs certbot inside container via Podman
- **Philosophy**: Clean separation of concerns; users choose one approach per node

**Rationale**: Mixing container and native package management in the same class creates coupling and complexity. Separate classes are simpler to test, document, and reason about.

### 2. Packman Mirrors: Hiera-Driven Config
- Added `letsencrypt::suse::packman_mirrors` hash to `data/Suse-family.yaml`
- Maps SLES version (e.g., "15.5") → official Packman mirror URL
- Users can override in their own Hiera data if they have preferred mirrors
- Supports SLES 15.4, 15.5, 15.6, 15.7 (current support range)

**Rationale**: Version-specific mirrors prevent URL drift over time. Hiera allows enterprise customization without code changes.

---

## Implementation Details

### New Class: `letsencrypt::suse::podman`

**File**: `manifests/suse/podman.pp`

**Parameters**:
- `email` - ACME registration email (required)
- `container_image` - Docker image URI (default: `certbot/certbot`)
- `container_tag` - Image tag (default: `latest`)
- `manage_podman` - Install/enable podman (default: `true`)
- `manage_config` - Manage certbot CLI config (default: `true`)
- `renew_cron_ensure` - Enable renewal cron (default: `present`)
- `azure_config_file` - Path to Azure plugin config (default: `/etc/letsencrypt/azure.ini`)
- `manage_azure_config` - Manage Azure config file (default: `false`)
- `azure_config_content` - Azure config file content (INI format, optional)
- `letsencrypt_dir` - Host-side `/etc/letsencrypt` path (default: `/etc/letsencrypt`)
- `certbot_wrapper_dir` - Wrapper script location (default: `/usr/local/lib/letsencrypt`)

**Behavior**:
1. Installs `podman` package and enables service (if `manage_podman => true`)
2. Creates `/etc/letsencrypt` directory with proper permissions (0755)
3. Optionally manages Azure plugin config file (if `manage_azure_config => true`)
4. Generates `/usr/local/lib/letsencrypt/certbot-podman.sh` wrapper script
5. Creates cron job for renewal (if `renew_cron_ensure => present`)

**Wrapper Script** (`templates/suse/certbot-podman.sh.epp`):
- Pulls container image if not present
- Mounts `/etc/letsencrypt` as persistent volume (SELinux context: `:Z`)
- Optionally mounts Azure config file as read-only
- Runs certbot with `--cap-drop=all` for security
- Exit code reflects certbot command success/failure

**Usage**:
```puppet
# Minimal
class { 'letsencrypt::suse::podman':
  email => 'admin@example.com',
}

# With Azure plugin config management
class { 'letsencrypt::suse::podman':
  email                => 'admin@example.com',
  manage_azure_config  => true,
  azure_config_content => "dns_azure_client_id = ...\n",
}

# Custom container image
class { 'letsencrypt::suse::podman':
  email           => 'admin@example.com',
  container_image => 'registry.example.com/certbot',
  container_tag   => 'v2.0.0',
}

# Disable renewal cron
class { 'letsencrypt::suse::podman':
  email               => 'admin@example.com',
  renew_cron_ensure   => 'absent',
}
```

**Manual Certificate Request**:
```bash
/usr/local/lib/letsencrypt/certbot-podman.sh certonly \
  --dns-azure \
  --dns-azure-config /etc/letsencrypt/azure.ini \
  -d example.com -d '*.example.com'
```

---

## Hiera Configuration

### New Data: Packman Mirrors
**File**: `data/Suse-family.yaml`

```yaml
letsencrypt::suse::packman_mirrors:
  '15.4': 'https://mirrors.opensuse.org/opensuse/repositories/home:/dcai/SLE_15_SP4/'
  '15.5': 'https://mirrors.opensuse.org/opensuse/repositories/home:/dcai/SLE_15_SP5/'
  '15.6': 'https://mirrors.opensuse.org/opensuse/repositories/home:/dcai/SLE_15_SP6/'
  '15.7': 'https://mirrors.opensuse.org/opensuse/repositories/home:/dcai/SLE_15_SP7/'
```

**Usage** (in your Control Repo Hiera):
```yaml
---
letsencrypt::suse::manage_suseconnect: true
letsencrypt::suse::suseconnect_products:
  - 'sle-module-python3/15.5/x86_64'
letsencrypt::suse::packman_repo: "%{lookup('letsencrypt::suse::packman_mirrors')[${facts[os][release][full]}]}"
```

---

## Test Coverage

### New Spec File
**File**: `spec/classes/letsencrypt_suse_podman_spec.rb`

**Coverage**:
- Compilation on supported SLES versions (15.4-15.7)
- Package/service management
- File/directory creation
- Cron job configuration (present/absent states)
- Custom container image/tag handling
- Azure config file management
- OS validation (rejects non-SLES)
- Version range enforcement (enforces 15.4-15.7 by default)
- `enforce_sles_range => false` bypass

**Running the Spec**:
```bash
bundle exec rspec spec/classes/letsencrypt_suse_podman_spec.rb
```

---

## Documentation Updates

### README.md
Added a section describing SUSE with Podman (containerized certbot).

**Includes**:
- Basic setup example
- Custom container image example
- Azure DNS plugin integration example
- Manual certificate request command
- Explanation of benefits and isolation model

---

## Deployment Workflow (Control Repo Integration)

### Option 1: Native SUSE Path
```yaml
# In Control Repo profile/le_suse.pp (or via Hiera):
class { 'letsencrypt::suse':
  email                     => 'admin@example.com',
  manage_suseconnect        => true,
  suseconnect_products      => ['sle-module-python3/15.5/x86_64'],
  renew_disable_distro_cron => true,
}

# Request certificates:
letsencrypt::suse::certificate { 'www.example.com':
  domains => ['www.example.com', 'api.example.com'],
  plugin  => 'dns-01',  # default, uses dns-azure
}
```

### Option 2: Containerized Podman Path
```yaml
# In Control Repo profile/le_suse_podman.pp:
class { 'letsencrypt::suse::podman':
  email                => 'admin@example.com',
  manage_podman        => true,
  manage_azure_config  => true,
  azure_config_content => lookup('sensitive_data::azure_config'),
}

# Request certificates:
exec { 'request-certificate':
  command => '/usr/local/lib/letsencrypt/certbot-podman.sh certonly --dns-azure -d www.example.com',
  unless  => 'test -f /etc/letsencrypt/live/www.example.com/cert.pem',
  path    => ['/usr/sbin', '/usr/bin', '/bin'],
}
```

**Decision factors**:
- **Native**: Simpler, uses system packages, tighter OS integration
- **Podman**: Isolation, consistent across environments, self-contained dependencies

---

## Future Enhancements

### Potential Next Steps
1. **Podman Pod Management**: Create a dedicated pod for certbot with shared volumes
2. **Systemd Service Unit**: Run certbot renewal as `certbot-podman.service` instead of cron
3. **Multi-DNS-Plugin Support**: Define enum for supported DNS plugins (dns-azure, dns-cloudflare, etc.)
4. **Certificate Define for Podman**: Create `letsencrypt::suse::podman::certificate` define similar to native wrapper
5. **Prometheus Metrics Export**: Add sidecar container for cert expiry monitoring

### Testing Roadmap
- Run the full module RSpec suite in the supported development environment.
- Add integration coverage for supported SLES releases when suitable test infrastructure is available.
- Add acceptance coverage when supported SUSE test images are available.

---

## Verification Scope

The implementation is covered by Puppet manifest syntax checks, Hiera YAML
validation, EPP template validation, and focused RSpec catalog tests. Full
module, acceptance, Podman runtime, and ACME issuance validation depend on the
target platform and test environment. These checks should be run before a
release or deployment.
