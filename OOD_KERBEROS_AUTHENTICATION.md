# Converting OOD from Shibboleth to Kerberos Authentication

## Overview

This document describes how to replace Shibboleth authentication in Open OnDemand with native Kerberos authentication using Apache's mod_auth_gssapi. This approach is suitable for environments with existing Kerberos infrastructure and provides seamless Single Sign-On (SSO) for users with valid Kerberos tickets.

## Background: Kerberos Authentication

### What is Kerberos?

Kerberos is a network authentication protocol designed to provide strong authentication using secret-key cryptography. It uses tickets to allow nodes communicating over a non-secure network to prove their identity to one another in a secure manner.

### Kerberos Architecture Components

#### 1. Key Distribution Center (KDC)

The KDC is the central authentication server consisting of two services:

- **Authentication Server (AS)** - Verifies user credentials and issues Ticket Granting Tickets (TGT)
- **Ticket Granting Server (TGS)** - Issues service tickets using valid TGTs

**Example KDC Configuration:** `/etc/krb5kdc/kdc.conf`
```ini
[kdcdefaults]
    kdc_ports = 88
    kdc_tcp_ports = 88

[realms]
    EXAMPLE.COM = {              # ← SITE-SPECIFIC: Your Kerberos realm
        database_name = /var/lib/krb5kdc/principal
        admin_keytab = FILE:/etc/krb5kdc/kadm5.keytab
        acl_file = /etc/krb5kdc/kadm5.acl
        key_stash_file = /etc/krb5kdc/stash
        kdc_ports = 88
        kdc_tcp_ports = 88
        max_life = 10h 0m 0s
        max_renewable_life = 7d 0h 0m 0s
        master_key_type = aes256-cts
        supported_enctypes = aes256-cts:normal aes128-cts:normal
        default_principal_flags = +preauth
    }
```

#### 2. Realm

A Kerberos realm is an administrative domain. By convention, realm names are typically the uppercase version of your DNS domain.

**Common Examples:**
- `MIT.EDU` - MIT's Kerberos realm
- `ATHENA.MIT.EDU` - MIT Athena realm (historical)
- `AD.EXAMPLE.COM` - Active Directory realm
- `EXAMPLE.COM` - Generic organization realm

#### 3. Principals

A principal is a unique identity in Kerberos. Format: `primary/instance@REALM`

**User Principals:**
```
alice@EXAMPLE.COM
bob@MIT.EDU
jdoe@AD.EXAMPLE.COM
```

**Service Principals:**
```
host/server.example.com@EXAMPLE.COM        # Host principal
HTTP/ood.example.com@EXAMPLE.COM           # Web service principal
slurm/headnode.example.com@EXAMPLE.COM     # Slurm service
```

#### 4. Keytab Files

Keytabs are files containing principal names and their encrypted keys. They allow services to authenticate without passwords.

**Location Conventions:**
- System keytabs: `/etc/krb5.keytab`
- Service keytabs: `/etc/krb5.keytab.HTTP` or `/etc/httpd/conf/http.keytab`
- Application keytabs: `/var/lib/app/service.keytab`

**Keytab Contents:**
```bash
$ klist -kt /etc/httpd/conf/http.keytab
Keytab name: FILE:/etc/httpd/conf/http.keytab
KVNO Timestamp           Principal
---- ------------------- ----------------------------------------------------
   2 01/15/2025 10:30:00 HTTP/ood.example.com@EXAMPLE.COM
   2 01/15/2025 10:30:00 HTTP/ood.example.com@EXAMPLE.COM
```

### Kerberos Client Configuration

The Kerberos client configuration file defines how clients locate KDCs and map domains to realms.

**Generic Configuration:** `/etc/krb5.conf`
```ini
[libdefaults]
    default_realm = EXAMPLE.COM              # ← SITE-SPECIFIC: Your default realm
    dns_lookup_realm = false
    dns_lookup_kdc = false                   # Set to true for DNS-based KDC discovery
    ticket_lifetime = 24h
    renew_lifetime = 7d
    forwardable = true
    rdns = false
    default_ccache_name = KEYRING:persistent:%{uid}

[realms]
    EXAMPLE.COM = {                          # ← SITE-SPECIFIC: Your realm configuration
        kdc = kdc1.example.com               # ← SITE-SPECIFIC: Primary KDC
        kdc = kdc2.example.com               # ← SITE-SPECIFIC: Secondary KDC (optional)
        admin_server = kdc1.example.com      # ← SITE-SPECIFIC: Admin server
        default_domain = example.com         # ← SITE-SPECIFIC: Your domain
    }
    
    # Additional realms for cross-realm trust (if needed)
    PARTNER.ORG = {                          # ← SITE-SPECIFIC: Partner realm (optional)
        kdc = kerberos.partner.org
    }

[domain_realm]
    .example.com = EXAMPLE.COM               # ← SITE-SPECIFIC: Domain to realm mapping
    example.com = EXAMPLE.COM
    .cluster.example.com = EXAMPLE.COM       # ← SITE-SPECIFIC: Subdomain mapping
    cluster.example.com = EXAMPLE.COM

[logging]
    default = FILE:/var/log/krb5libs.log
    kdc = FILE:/var/log/krb5kdc.log
    admin_server = FILE:/var/log/kadmind.log
```

**Configuration Breakdown:**

| Section | Purpose | Site-Specific? |
|---------|---------|----------------|
| `[libdefaults]` | Client behavior defaults | Partially |
| `default_realm` | Default Kerberos realm | **YES** |
| `ticket_lifetime` | How long tickets are valid | Policy decision |
| `forwardable` | Allow ticket forwarding | Policy decision |
| `[realms]` | KDC locations per realm | **YES** |
| `kdc` | KDC server addresses | **YES** |
| `admin_server` | Admin server address | **YES** |
| `[domain_realm]` | DNS to realm mapping | **YES** |
| Domain entries | Your DNS domains | **YES** |
| `[logging]` | Log file locations | Generic (paths) |

### Authentication Flow

1. **User Login:**
   ```
   User → kinit → AS → TGT → Credential Cache
   ```

2. **Service Access:**
   ```
   User → Service Request → TGS → Service Ticket → Service
   Service verifies ticket with keytab
   ```

3. **Web SSO with Kerberos:**
   ```
   Browser → HTTP/OOD Server
   ↓ 401 + WWW-Authenticate: Negotiate
   Browser (with Kerberos ticket) → SPNEGO token
   ↓
   Apache mod_auth_gssapi → Verify with keytab
   ↓
   Grant access with authenticated principal
   ```

## Testing Kerberos Authentication

Before configuring OOD, verify your Kerberos environment is working correctly.

### Prerequisites

Install Kerberos client utilities:

```bash
# RHEL/Rocky/CentOS
dnf install krb5-workstation

# Ubuntu/Debian
apt-get install krb5-user
```

### Test 1: Basic Ticket Acquisition

```bash
# Obtain a ticket for a user
kinit username@EXAMPLE.COM     # ← SITE-SPECIFIC: Replace with your realm
# Enter password when prompted

# Verify ticket was obtained
klist
```

**Expected Output:**
```
Ticket cache: KEYRING:persistent:1000:1000
Default principal: username@EXAMPLE.COM

Valid starting       Expires              Service principal
01/15/2025 10:00:00  01/16/2025 10:00:00  krbtgt/EXAMPLE.COM@EXAMPLE.COM
    renew until 01/22/2025 10:00:00
```

**What to Check:**
- Ticket cache type (KEYRING, FILE, etc.)
- Principal name matches your user@REALM
- Service principal is `krbtgt/REALM@REALM` (TGT)
- Ticket is valid (not expired)
- Renewable flag if needed

### Test 2: Service Principal Creation

Create a service principal for OOD on your KDC:

```bash
# On KDC server (requires admin privileges)
kadmin.local

# In kadmin.local prompt:
addprinc -randkey HTTP/ood.example.com@EXAMPLE.COM    # ← SITE-SPECIFIC: Your OOD hostname and realm
ktadd -k /tmp/http.keytab HTTP/ood.example.com@EXAMPLE.COM
quit
```

**Or using remote kadmin:**
```bash
kadmin -p admin/admin@EXAMPLE.COM    # ← SITE-SPECIFIC: Your admin principal
# In kadmin prompt:
addprinc -randkey HTTP/ood.example.com@EXAMPLE.COM    # ← SITE-SPECIFIC
ktadd -k /tmp/http.keytab HTTP/ood.example.com@EXAMPLE.COM
quit
```

**Important Notes:**
- Service principal MUST be `HTTP/fully.qualified.domain.name@REALM`
- Hostname must match the DNS name users access
- Use `-randkey` for service principals (no password)
- Store keytab securely and transfer to OOD server

### Test 3: Keytab Verification

Verify the keytab on the OOD server:

```bash
# Copy keytab to OOD server
scp /tmp/http.keytab ood-server:/etc/httpd/conf/http.keytab

# On OOD server, set permissions
chmod 640 /etc/httpd/conf/http.keytab
chown root:apache /etc/httpd/conf/http.keytab

# List keytab contents
klist -kt /etc/httpd/conf/http.keytab

# Test authentication with keytab
kinit -kt /etc/httpd/conf/http.keytab HTTP/ood.example.com@EXAMPLE.COM
klist
```

**Expected Output:**
```
Ticket cache: KEYRING:persistent:0:0
Default principal: HTTP/ood.example.com@EXAMPLE.COM

Valid starting       Expires              Service principal
01/15/2025 11:00:00  01/16/2025 11:00:00  krbtgt/EXAMPLE.COM@EXAMPLE.COM
```

### Test 4: Web Authentication Test

Create a minimal Apache configuration to test Kerberos authentication:

**Test Configuration:** `/etc/httpd/conf.d/krb-test.conf`
```apache
<Location "/krb-test">
    AuthType GSSAPI
    AuthName "Kerberos Test"
    GssapiCredStore keytab:/etc/httpd/conf/http.keytab
    Require valid-user
    
    # Debugging (remove in production)
    GssapiBasicAuth On
    GssapiLocalName On
</Location>
```

**Test Page:** `/var/www/html/krb-test/index.html`
```html
<!DOCTYPE html>
<html>
<head><title>Kerberos Test</title></head>
<body>
    <h1>Kerberos Authentication Test</h1>
    <p>User: <?= $_SERVER['REMOTE_USER'] ?></p>
    <p>Auth Type: <?= $_SERVER['AUTH_TYPE'] ?></p>
</body>
</html>
```

```bash
# Create test directory
mkdir -p /var/www/html/krb-test
# Create index file above

# Restart Apache
systemctl restart httpd

# Test with curl (from a client machine with valid ticket)
kinit testuser@EXAMPLE.COM    # ← SITE-SPECIFIC
curl --negotiate -u : https://ood.example.com/krb-test/
```

**Expected Output:**
```html
<h1>Kerberos Authentication Test</h1>
<p>User: testuser@EXAMPLE.COM</p>
<p>Auth Type: GSSAPI</p>
```

**Or test with browser:**
1. Obtain Kerberos ticket on workstation: `kinit testuser@EXAMPLE.COM`
2. Configure browser for Kerberos (see Browser Configuration section)
3. Navigate to `https://ood.example.com/krb-test/`
4. Should authenticate automatically without password prompt

### Test 5: SSH with Kerberos

Test that Kerberos authentication works for SSH (helpful for OOD shell access):

```bash
# On client with valid ticket
klist  # Verify you have a ticket

# SSH to OOD server using Kerberos
ssh -o GSSAPIAuthentication=yes -o PreferredAuthentications=gssapi-with-mic ood.example.com

# Should login without password
```

**SSH Server Configuration:** `/etc/ssh/sshd_config`
```
GSSAPIAuthentication yes
GSSAPICleanupCredentials yes
GSSAPIStrictAcceptorCheck no    # ← May be needed if hostname mismatches
```

### Troubleshooting Common Issues

#### Issue: "Server not found in Kerberos database"
```
kinit: Server not found in Kerberos database while getting initial credentials
```

**Solution:**
- Check `/etc/krb5.conf` has correct KDC addresses
- Verify KDC is running: `systemctl status krb5-kdc`
- Check network connectivity: `telnet kdc.example.com 88`
- Verify DNS resolution: `nslookup kdc.example.com`

#### Issue: "Clock skew too great"
```
kinit: Clock skew too great while getting initial credentials
```

**Solution:**
- Synchronize time with NTP:
  ```bash
  systemctl enable chronyd --now
  chronyc sources
  ```
- Maximum allowed skew is typically 5 minutes

#### Issue: "Cannot find KDC for realm"
```
Cannot find KDC for realm "EXAMPLE.COM"
```

**Solution:**
- Verify `[realms]` section in `/etc/krb5.conf`
- Check KDC hostname resolves correctly
- Enable DNS discovery if appropriate: `dns_lookup_kdc = true`

#### Issue: Keytab authentication fails
```
kinit: Keytab contains no suitable keys for HTTP/ood.example.com@EXAMPLE.COM
```

**Solution:**
- Verify principal name matches exactly
- Check keytab has correct encryption types: `klist -ket /path/to/keytab`
- Recreate keytab with compatible enctypes

## Browser Configuration for Kerberos

Users must configure their browsers to support Kerberos/SPNEGO authentication.

### Firefox Configuration

1. Navigate to `about:config`
2. Accept the warning
3. Set the following preferences:

```
network.negotiate-auth.trusted-uris = .example.com    # ← SITE-SPECIFIC: Your domain
network.negotiate-auth.delegation-uris = .example.com # ← SITE-SPECIFIC: For ticket delegation
network.automatic-ntlm-auth.trusted-uris = .example.com
```

### Chrome/Edge Configuration

**Linux/macOS:**
```bash
# Launch with Kerberos support
google-chrome --auth-server-whitelist="*.example.com" --auth-negotiate-delegate-whitelist="*.example.com"
```

**Windows:**
Chrome uses Windows Integrated Authentication automatically.

### Safari Configuration

Safari uses the system Kerberos settings on macOS automatically.

### User Documentation Template

Provide this to your users:

```markdown
# OOD Kerberos Access Instructions

## Before Accessing OOD

1. Obtain a Kerberos ticket:
   ```
   kinit username@EXAMPLE.COM
   ```
   Enter your Kerberos password when prompted.

2. Verify your ticket:
   ```
   klist
   ```

3. Configure your browser (one-time setup):
   - **Firefox:** See [Firefox Setup Guide]
   - **Chrome:** See [Chrome Setup Guide]

4. Access OOD:
   https://ood.example.com

Your browser should authenticate automatically.

## Troubleshooting

- If prompted for password, your ticket may have expired. Run `kinit` again.
- Tickets expire after 24 hours by default. Run `kinit -r` to renew.
- For help, contact: support@example.com    # ← SITE-SPECIFIC
```

## Implementing Kerberos in OOD Ansible

### Architecture Overview

The Kerberos-based authentication will replace Shibboleth with:

1. **Apache mod_auth_gssapi** - Handles GSSAPI/Kerberos authentication
2. **HTTP Service Keytab** - Allows Apache to verify Kerberos tickets
3. **User Mapping** - Maps Kerberos principals to local usernames
4. **Credential Delegation** - Forwards user tickets for job submission

### Modified Ansible Role Structure

Create a new authentication option in the OOD role:

```
roles/ood/
├── tasks/
│   ├── main.yml                    # Main entry point
│   ├── common.yml                  # Common OOD setup
│   ├── auth-shibboleth.yml         # Shibboleth auth (existing)
│   └── auth-kerberos.yml           # Kerberos auth (new)
├── templates/
│   ├── ood_portal-shibboleth.yml.j2    # Shibboleth config (existing)
│   └── ood_portal-kerberos.yml.j2      # Kerberos config (new)
├── files/
│   └── ood_kerb_user_handler.sh    # Kerberos user mapping (new)
└── vars/
    └── main.yml
```

### Task Files

#### Main Task File

**File:** `roles/ood/tasks/main.yml`
```yaml
---
# Common OOD setup tasks
- name: Include common OOD tasks
  include_tasks: common.yml

# Conditional authentication setup
- name: Setup Shibboleth authentication
  include_tasks: auth-shibboleth.yml
  when: ood_auth_method == "shibboleth"

- name: Setup Kerberos authentication
  include_tasks: auth-kerberos.yml
  when: ood_auth_method == "kerberos"

# Generate OOD portal configuration
- name: Generate ood_portal.yml
  template:
    src: "ood_portal-{{ ood_auth_method }}.yml.j2"
    dest: "/etc/ood/config/ood_portal.yml"

# Continue with common tasks
- name: Create folder for desktop env
  ansible.builtin.file:
    path: /etc/ood/config/apps/dashboard/
    state: directory
    mode: '0755'
    owner: root
    group: root

# ... rest of common tasks ...

- name: Run ood-portal-generator to update OOD portal
  ansible.builtin.command: /opt/ood/ood-portal-generator/sbin/update_ood_portal

- name: Starting httpd
  ansible.builtin.service:
    name: httpd
    enabled: true
    state: started
```

#### Common Setup Tasks

**File:** `roles/ood/tasks/common.yml`
```yaml
---
- name: "Write SSL cert"
  ansible.builtin.copy:
    dest: /etc/pki/tls/certs/localhost.crt
    content: "{{ ood_ssl_cert | b64decode }}"
    mode: '0644'
  tags: secret

- name: "Write SSL key"
  ansible.builtin.copy:
    dest: /etc/pki/tls/private/localhost.key
    content: "{{ ood_ssl_key | b64decode }}"
    mode: '0600'
  tags: secret

- name: Copy application folders
  ansible.builtin.synchronize:
    src: "."
    dest: "/"
    recursive: yes
    delete: no
    archive: no
    owner: yes
    group: yes

- name: write decoded munge key to /etc/munge/munge.key
  ansible.builtin.copy:
    dest: /etc/munge/munge.key
    content: "{{ MUNGE_KEY | b64decode }}"
    mode: '0600'
    owner: munge
    group: munge
  register: munge_key
  tags: secret

- name: Starting munge
  ansible.builtin.service:
    name: munge
    enabled: true
    state: started
```

#### Kerberos Authentication Tasks

**File:** `roles/ood/tasks/auth-kerberos.yml`
```yaml
---
- name: Install Kerberos and GSSAPI packages
  ansible.builtin.yum:
    name:
      - krb5-workstation
      - mod_auth_gssapi
      - mod_session
    state: present

- name: Create Kerberos configuration directory
  ansible.builtin.file:
    path: /etc/krb5.conf.d
    state: directory
    mode: '0755'

- name: Deploy Kerberos client configuration
  ansible.builtin.template:
    src: krb5.conf.j2
    dest: /etc/krb5.conf
    mode: '0644'
  when: ood_krb5_config is defined

- name: Write HTTP service keytab
  ansible.builtin.copy:
    dest: "{{ ood_keytab_path }}"
    content: "{{ ood_http_keytab | b64decode }}"
    mode: '0640'
    owner: root
    group: apache
  tags: secret

- name: Verify keytab is valid
  ansible.builtin.command:
    cmd: "klist -kt {{ ood_keytab_path }}"
  register: keytab_check
  changed_when: false

- name: Display keytab contents
  debug:
    var: keytab_check.stdout_lines

- name: Deploy Kerberos user mapping script
  ansible.builtin.copy:
    src: ood_kerb_user_handler.sh
    dest: /etc/ood/config/ood_kerb_user_handler.sh
    mode: '0755'
    owner: root
    group: root

- name: Create credential cache directory for Apache
  ansible.builtin.file:
    path: /var/run/httpd/ood-krb-cache
    state: directory
    mode: '0700'
    owner: apache
    group: apache

- name: Configure SELinux for Kerberos (if enabled)
  ansible.builtin.seboolean:
    name: "{{ item }}"
    state: yes
    persistent: yes
  loop:
    - httpd_can_connect_ldap
    - httpd_use_cifs
    - httpd_use_nfs
  when: ansible_selinux.status == "enabled"
  ignore_errors: yes
```

### Configuration Templates

#### Kerberos OOD Portal Configuration

**File:** `roles/ood/templates/ood_portal-kerberos.yml.j2`
```yaml
---
servername: {{ ansible_host }}                    # GENERIC
server_aliases: ['{{ ansible_host }}']             # GENERIC
proxy_server: {{ ood_servername }}                 # ← SITE-SPECIFIC: Public OOD hostname

ssl:                                               # GENERIC
  - 'SSLCertificateFile "/etc/pki/tls/certs/localhost.crt"'
  - 'SSLCertificateKeyFile "/etc/pki/tls/private/localhost.key"'

# Kerberos authentication configuration
auth:
  - "AuthType GSSAPI"                              # GENERIC: Use GSSAPI/Kerberos
  - "AuthName \"{{ ood_auth_name }}\""             # ← SITE-SPECIFIC: Login prompt text
  - "GssapiCredStore keytab:{{ ood_keytab_path }}" # GENERIC: Keytab location
  - "GssapiCredStore ccache:FILE:/var/run/httpd/ood-krb-cache/%{REMOTE_USER}"  # GENERIC
  - "GssapiLocalName On"                           # GENERIC: Strip @REALM from username
  - "GssapiNameAttributes json"                    # GENERIC: Provide user attributes
{% if ood_krb_allow_fallback | default(false) %}
  - "GssapiBasicAuth On"                           # ← SITE-SPECIFIC: Allow password fallback
  - "GssapiBasicAuthMech krb5"                     # GENERIC
{% endif %}
{% if ood_krb_delegation | default(true) %}
  - "GssapiDelegCcacheDir /var/run/httpd/ood-krb-cache"  # ← SITE-SPECIFIC: Enable ticket delegation
  - "GssapiDelegCcachePerms mode:0600"             # GENERIC
  - "GssapiUseS4U2Proxy on"                        # ← SITE-SPECIFIC: Policy decision
{% endif %}
  - "Require valid-user"                           # GENERIC

# User mapping handler (optional, for custom logic)
user_map_cmd: '/etc/ood/config/ood_kerb_user_handler.sh'  # GENERIC

# Logout redirect (no Shibboleth logout needed)
logout_redirect: '/logged_out.html'                # GENERIC

# Node proxy configuration
host_regex: '[\w.-]+\.{{ ood_cluster_domain }}'    # ← SITE-SPECIFIC: Your cluster domain
node_uri: '/node'                                   # GENERIC
rnode_uri: '/rnode'                                 # GENERIC
```

**Configuration Breakdown:**

| Setting | Purpose | Site-Specific? |
|---------|---------|----------------|
| `AuthType GSSAPI` | Enable Kerberos auth | Generic |
| `AuthName` | Login prompt text | **YES** - Your org name |
| `GssapiCredStore keytab` | Path to keytab | Generic (standard path) |
| `GssapiLocalName On` | Strip @REALM | Generic |
| `GssapiBasicAuth` | Password fallback | **YES** - Policy decision |
| `GssapiDelegCcacheDir` | Ticket forwarding | **YES** - Policy decision |
| `GssapiUseS4U2Proxy` | Constrained delegation | **YES** - Policy decision |
| `user_map_cmd` | Custom user mapping | Generic (script path) |
| `host_regex` | Node proxy pattern | **YES** - Your domain |

#### Kerberos Client Configuration Template

**File:** `roles/ood/templates/krb5.conf.j2`
```ini
# Kerberos client configuration for OOD
# This file is managed by Ansible

[libdefaults]
    default_realm = {{ ood_krb_realm }}              # ← SITE-SPECIFIC: Your Kerberos realm
    dns_lookup_realm = false                         # GENERIC
    dns_lookup_kdc = {{ ood_krb_dns_lookup_kdc | default('false') }}  # ← SITE-SPECIFIC: Policy
    ticket_lifetime = {{ ood_krb_ticket_lifetime | default('24h') }}  # ← SITE-SPECIFIC: Policy
    renew_lifetime = {{ ood_krb_renew_lifetime | default('7d') }}     # ← SITE-SPECIFIC: Policy
    forwardable = {{ ood_krb_forwardable | default('true') }}         # ← SITE-SPECIFIC: Policy
    rdns = false                                     # GENERIC (usually false)
    default_ccache_name = KEYRING:persistent:%{uid}  # GENERIC

[realms]
{% for realm in ood_krb_realms %}                    # ← SITE-SPECIFIC: All realms
    {{ realm.name }} = {
{% for kdc in realm.kdcs %}                          # ← SITE-SPECIFIC: KDC servers
        kdc = {{ kdc }}
{% endfor %}
{% if realm.admin_server is defined %}              # ← SITE-SPECIFIC: Admin server
        admin_server = {{ realm.admin_server }}
{% endif %}
{% if realm.default_domain is defined %}            # ← SITE-SPECIFIC: Default domain
        default_domain = {{ realm.default_domain }}
{% endif %}
    }
{% endfor %}

[domain_realm]
{% for mapping in ood_krb_domain_mappings %}        # ← SITE-SPECIFIC: Domain mappings
    {{ mapping.domain }} = {{ mapping.realm }}
{% endfor %}

[logging]
    default = FILE:/var/log/krb5libs.log             # GENERIC
    kdc = FILE:/var/log/krb5kdc.log                  # GENERIC
    admin_server = FILE:/var/log/kadmind.log         # GENERIC
```

#### Cluster Configuration for Kerberos

**File:** `roles/ood/templates/engaging-kerberos.yml.j2`
```yaml
---
v2:
  metadata:
    title: "{{ ood_cluster_name }}"                  # ← SITE-SPECIFIC: Cluster name
  login:
    host: "{{ ood_login_host }}"                     # ← SITE-SPECIFIC: Login node
  job:
    adapter: "slurm"                                 # GENERIC
    bin: "/usr/bin"                                  # GENERIC
    conf: "/etc/slurm/slurm.conf"                    # GENERIC
    copy_environment: false                          # GENERIC
    
    # Kerberos ticket passing configuration
    set_host: "export KRB5CCNAME=/tmp/krb5cc_$(id -u)_ood_${SLURM_JOB_ID}"  # GENERIC
    
  batch_connect:
    basic:
      script_wrapper: |
        source /etc/profile.d/modules.sh
        module purge
        
        # Setup Kerberos credential cache
        export KRB5CCNAME=/tmp/krb5cc_$(id -u)_ood_${SLURM_JOB_ID}
        
        %s
      set_host: "host=$(hostname -A | awk '{print $1}')"  # GENERIC
    vnc:
      script_wrapper: |
        source /etc/profile.d/modules.sh
        module purge
        export PATH="/opt/TurboVNC/bin:${PATH}"
        export WEBSOCKIFY_CMD="/usr/bin/websockify"
        
        # Setup Kerberos credential cache
        export KRB5CCNAME=/tmp/krb5cc_$(id -u)_ood_${SLURM_JOB_ID}
        
        %s
      set_host: "host=$(hostname -A | awk '{print $1}')"  # GENERIC
```

### User Mapping Script

**File:** `roles/ood/files/ood_kerb_user_handler.sh`
```bash
#!/bin/bash
# Kerberos user mapping handler for OOD
# Maps Kerberos principals to local usernames

log() {
  logger -p local0.info -t ood_kerb_mapping "$@"
}

# Input: Kerberos principal (e.g., alice@EXAMPLE.COM)
INPUT="$1"
log "Mapping Kerberos principal: $INPUT"

# Extract username from principal (before @)
USERNAME=$(echo "$INPUT" | sed 's/@.*//')

# Optional: Transform username (e.g., lowercase)
USERNAME=$(echo "$USERNAME" | tr '[:upper:]' '[:lower:]')

# Check if user exists locally
if getent passwd "$USERNAME" > /dev/null 2>&1; then
    log "Mapped $INPUT to local user: $USERNAME"
    echo "$USERNAME"
    exit 0
else
    log "ERROR: No local user found for principal: $INPUT (username: $USERNAME)"
    
    # ← SITE-SPECIFIC: Custom logic here
    # Option 1: Create user on-the-fly (if using central user management)
    # Option 2: Map to a default user
    # Option 3: Reject the login
    
    exit 1
fi
```

**Advanced User Mapping (Optional):**

For organizations with non-trivial mapping (e.g., `alice@EXAMPLE.COM` → `alice123`):

```bash
#!/bin/bash
# Advanced Kerberos user mapping

log() {
  logger -p local0.info -t ood_kerb_mapping "$@"
}

INPUT="$1"
PRINCIPAL=$(echo "$INPUT" | sed 's/@.*//')
REALM=$(echo "$INPUT" | sed 's/.*@//')

log "Mapping principal=$PRINCIPAL realm=$REALM"

# ← SITE-SPECIFIC: Lookup user in directory service
# Example: LDAP lookup
if command -v ldapsearch > /dev/null; then
    # Query LDAP for Kerberos principal
    # ← SITE-SPECIFIC: Adjust LDAP query for your schema
    LDAP_USER=$(ldapsearch -x -LLL \
        -h ldap.example.com \
        -b "dc=example,dc=com" \
        "(krbPrincipalName=$INPUT)" \
        uid | awk '/^uid:/ {print $2}')
    
    if [ -n "$LDAP_USER" ]; then
        log "LDAP mapped $INPUT to $LDAP_USER"
        echo "$LDAP_USER"
        exit 0
    fi
fi

# ← SITE-SPECIFIC: Fallback to simple mapping
USERNAME=$(echo "$PRINCIPAL" | tr '[:upper:]' '[:lower:]')

if getent passwd "$USERNAME" > /dev/null 2>&1; then
    log "Fallback mapped $INPUT to $USERNAME"
    echo "$USERNAME"
    exit 0
fi

log "ERROR: Cannot map $INPUT to local user"
exit 1
```

### Inventory Variables

#### Host Variables for Kerberos OOD

**File:** `inventory/prod/host_vars/ood001.yml`
```yaml
# Service configuration (same as before)
ansible_host: ood001.inband
servicename: ood001
container_url: oras://ghcr.io/mit-orcd/ood-4.0:main_latest

# ← SITE-SPECIFIC: Authentication method selection
ood_auth_method: kerberos

# ← SITE-SPECIFIC: Kerberos realm and KDC configuration
ood_krb_realm: EXAMPLE.COM
ood_krb_realms:
  - name: EXAMPLE.COM
    kdcs:
      - kdc1.example.com
      - kdc2.example.com
    admin_server: kdc1.example.com
    default_domain: example.com

# ← SITE-SPECIFIC: Domain to realm mappings
ood_krb_domain_mappings:
  - domain: .example.com
    realm: EXAMPLE.COM
  - domain: example.com
    realm: EXAMPLE.COM
  - domain: .cluster.example.com
    realm: EXAMPLE.COM

# ← SITE-SPECIFIC: Kerberos policy settings
ood_krb_ticket_lifetime: 24h
ood_krb_renew_lifetime: 7d
ood_krb_forwardable: true
ood_krb_dns_lookup_kdc: false

# ← SITE-SPECIFIC: Authentication text
ood_auth_name: "Kerberos Login - Example Cluster"

# ← SITE-SPECIFIC: HTTP keytab (base64 encoded)
# Generate with: cat http.keytab | base64 -w0
ood_http_keytab: "{{ PROD_OOD_HTTP_KEYTAB }}"

# GENERIC: Keytab path (standard location)
ood_keytab_path: /etc/httpd/conf/http.keytab

# ← SITE-SPECIFIC: Policy decisions
ood_krb_allow_fallback: false        # Allow password auth as fallback?
ood_krb_delegation: true              # Forward user tickets to compute nodes?

# ← SITE-SPECIFIC: Cluster configuration
ood_servername: orcd-ood.example.com
ood_cluster_name: "Example HPC Cluster"
ood_cluster_domain: inband
ood_login_host: login.inband

# Secrets (same as before)
ood_ssl_cert: "{{ PROD_OOD_SSL_CERT }}"
ood_ssl_key: "{{ PROD_OOD_SSL_KEY }}"
MUNGE_KEY: "{{ PROD_MUNGE_KEY }}"
```

#### Group Variables

**File:** `inventory/prod/group_vars/ood.yml`
```yaml
# ← SITE-SPECIFIC: Common Kerberos settings for all OOD instances
ood_krb_realm: EXAMPLE.COM
ood_krb_dns_lookup_kdc: false
ood_krb_ticket_lifetime: 24h
ood_krb_renew_lifetime: 7d
ood_krb_forwardable: true

# GENERIC: Standard paths
ood_keytab_path: /etc/httpd/conf/http.keytab

# ← SITE-SPECIFIC: Default policy
ood_krb_allow_fallback: false
ood_krb_delegation: true
```

### Deployment Playbook

No changes needed to `ood.yml` - it works with the new authentication method:

```yaml
# ood.yml (unchanged)
- name: Iterate over ood hosts and run roles
  hosts: ood
  gather_facts: false
  vars:
    install_path: "/shared/services/ood"
  tasks:
    - name: Execute master role
      delegate_to: "{{ master }}"
      include_role:
        name: master
    
    - name: Execute ood role
      include_role:
        name: ood
```

The `ood_auth_method` variable in the inventory controls which authentication is configured.

## Ticket Delegation and Job Submission

### Why Ticket Delegation Matters

When users submit jobs through OOD, they need authentication to:
1. Submit jobs to Slurm (requires Munge or Kerberos)
2. Access their files on NFS (may require Kerberos)
3. Access remote data sources (may require Kerberos)

Ticket delegation allows OOD to forward the user's Kerberos ticket to the job.

### Configuration for Delegation

**Apache Configuration (in ood_portal.yml):**
```apache
GssapiDelegCcacheDir /var/run/httpd/ood-krb-cache
GssapiDelegCcachePerms mode:0600
GssapiUseS4U2Proxy on
```

**OOD Configuration:**

OOD needs to pass the credential cache to jobs:

```ruby
# /etc/ood/config/clusters.d/cluster.yml
---
v2:
  job:
    adapter: slurm
    # Set KRB5CCNAME before running Slurm commands
    set_host: "export KRB5CCNAME=/var/run/httpd/ood-krb-cache/krb5cc_%{uid}"
```

### User Credential Cache Management

**Automatic Cleanup Script:**

Create a cron job to clean up expired credential caches:

```bash
#!/bin/bash
# /usr/local/bin/ood-krb-cache-cleanup.sh

CACHE_DIR="/var/run/httpd/ood-krb-cache"
MAX_AGE=1  # days

find "$CACHE_DIR" -type f -mtime +${MAX_AGE} -delete

# Also clean up based on ticket expiration
for ccache in "$CACHE_DIR"/krb5cc_*; do
    if [ -f "$ccache" ]; then
        if ! KRB5CCNAME="$ccache" klist -s 2>/dev/null; then
            rm -f "$ccache"
        fi
    fi
done
```

**Cron Entry:**
```cron
# Clean up expired Kerberos caches daily
0 2 * * * /usr/local/bin/ood-krb-cache-cleanup.sh
```

## Security Considerations

### Keytab Security

**Critical Security Measures:**

1. **File Permissions:**
   ```bash
   chmod 640 /etc/httpd/conf/http.keytab
   chown root:apache /etc/httpd/conf/http.keytab
   ```

2. **Restrict Principal:**
   - Use service-specific principal: `HTTP/ood.example.com@REALM`
   - Do NOT use a user principal
   - Do NOT use a generic service account

3. **Key Rotation:**
   ```bash
   # Periodically rotate keytab (e.g., annually)
   kadmin
   change_password HTTP/ood.example.com@EXAMPLE.COM
   ktadd -k /tmp/new_http.keytab HTTP/ood.example.com@EXAMPLE.COM
   ```

4. **Monitoring:**
   - Monitor keytab access: `auditctl -w /etc/httpd/conf/http.keytab -p ra`
   - Alert on unauthorized access

### Credential Cache Security

1. **Separate Directory per User:**
   ```apache
   GssapiDelegCcacheDir /var/run/httpd/ood-krb-cache/%{uid}
   GssapiDelegCcachePerms mode:0600
   ```

2. **Filesystem Isolation:**
   ```bash
   # Mount cache directory with noexec, nosuid
   mount -t tmpfs -o noexec,nosuid,mode=0700 tmpfs /var/run/httpd/ood-krb-cache
   ```

3. **SELinux Context:**
   ```bash
   semanage fcontext -a -t httpd_var_run_t "/var/run/httpd/ood-krb-cache(/.*)?"
   restorecon -R /var/run/httpd/ood-krb-cache
   ```

### Network Security

1. **Encrypt Kerberos Traffic:**
   - Ensure KDC uses encrypted protocols
   - Check firewall allows Kerberos ports (88/tcp, 88/udp)

2. **HTTPS Required:**
   - Kerberos tickets sent over HTTPS
   - Use valid SSL certificates
   - Enforce HTTPS redirects

3. **IP Restrictions (optional):**
   ```apache
   <Location />
       AuthType GSSAPI
       Require valid-user
       Require ip 10.0.0.0/8        # ← SITE-SPECIFIC: Your network
   </Location>
   ```

## Comparison: Shibboleth vs. Kerberos

| Aspect | Shibboleth | Kerberos |
|--------|------------|----------|
| **Identity Source** | Federated (Okta, SAML IdP) | Local KDC or AD |
| **SSO Scope** | Web-only, federation-wide | OS-level, enterprise-wide |
| **User Experience** | Web redirect to IdP | Transparent with ticket |
| **Setup Complexity** | High (SP, IdP, metadata) | Medium (KDC, keytabs) |
| **Ticket Management** | Session cookies | Kerberos tickets (kinit) |
| **Delegation** | Not native | Built-in (S4U2Proxy) |
| **Infrastructure** | Requires IdP | Requires KDC |
| **Cross-Organization** | Excellent (federation) | Limited (cross-realm) |
| **Password Entry** | At IdP (web form) | At kinit (command line) |
| **Mobile/Remote** | Easy | Requires VPN or gateway |
| **Attribute Release** | Extensive (SAML attributes) | Limited (principal only) |

### When to Use Kerberos

✅ Existing Kerberos infrastructure (AD, FreeIPA, MIT Kerberos)
✅ Users already authenticate with Kerberos for other services
✅ Need OS-level SSO (SSH, NFS, etc.)
✅ Ticket delegation required for job submission
✅ Internal-only deployment (not public-facing)
✅ Want transparent authentication without web redirects

### When to Keep Shibboleth

✅ Federating with external organizations
✅ Using institutional IdP (MIT Touchstone, InCommon, eduGAIN)
✅ Need rich attribute release (groups, roles, affiliations)
✅ Public-facing portal for external collaborators
✅ No existing Kerberos infrastructure
✅ Mobile/remote access without VPN

## Troubleshooting Kerberos Authentication

### Apache Logs

**Enable debug logging:**

```apache
# /etc/httpd/conf.d/ood-debug.conf
LogLevel auth_gssapi:trace8
ErrorLog /var/log/httpd/ood_kerb_error.log
CustomLog /var/log/httpd/ood_kerb_access.log combined
```

**Common Log Messages:**

```
[auth_gssapi:error] Cannot find key for HTTP/ood.example.com@EXAMPLE.COM
→ Keytab missing or incorrect principal

[auth_gssapi:error] Unspecified GSS failure
→ Check clock skew, network connectivity

[auth_gssapi:info] Delegating credentials for alice@EXAMPLE.COM
→ Delegation successful
```

### Testing Checklist

1. **Verify keytab:**
   ```bash
   klist -kt /etc/httpd/conf/http.keytab
   kinit -kt /etc/httpd/conf/http.keytab HTTP/ood.example.com
   ```

2. **Test authentication:**
   ```bash
   curl -v --negotiate -u : https://ood.example.com/
   ```

3. **Check credential cache:**
   ```bash
   ls -la /var/run/httpd/ood-krb-cache/
   KRB5CCNAME=/var/run/httpd/ood-krb-cache/krb5cc_1000 klist
   ```

4. **Verify ticket delegation:**
   ```bash
   # Submit a test job through OOD
   # On compute node:
   echo $KRB5CCNAME
   klist
   ```

### Common Issues and Solutions

#### Issue: "Cannot find KDC for realm"

**Symptoms:**
- Browser shows "Authentication failed"
- Apache log: "Cannot contact KDC"

**Solution:**
```bash
# Check DNS resolution of KDC
nslookup kdc.example.com

# Check network connectivity
telnet kdc.example.com 88

# Verify /etc/krb5.conf has correct KDC addresses
```

#### Issue: "Clock skew too great"

**Symptoms:**
- Authentication works intermittently
- Log: "Clock skew too great"

**Solution:**
```bash
# Ensure NTP is running
systemctl status chronyd
chronyc sources

# Maximum skew is 5 minutes
timedatectl
```

#### Issue: "No credentials cache found"

**Symptoms:**
- Job submission fails
- Can't access NFS shares in jobs

**Solution:**
```bash
# Verify delegation is enabled
grep GssapiDelegCcache /etc/ood/config/ood_portal.yml

# Check cache directory exists and is writable
ls -ld /var/run/httpd/ood-krb-cache
ps aux | grep httpd | head -1  # Check Apache runs as 'apache' user

# Verify browser sends delegated ticket
# In Firefox about:config:
# network.negotiate-auth.delegation-uris = .example.com
```

#### Issue: User mapping fails

**Symptoms:**
- Authentication succeeds but OOD shows error
- Log: "No local user found"

**Solution:**
```bash
# Check user exists
getent passwd alice

# Test mapping script manually
/etc/ood/config/ood_kerb_user_handler.sh "alice@EXAMPLE.COM"

# Check script permissions
ls -l /etc/ood/config/ood_kerb_user_handler.sh
```

## Migration from Shibboleth to Kerberos

### Phased Migration Approach

#### Phase 1: Parallel Authentication (Testing)

Run both authentication methods simultaneously:

```apache
# Allow both Shibboleth and Kerberos
<Location "/krb">
    AuthType GSSAPI
    # ... Kerberos config
</Location>

<Location "/shib">
    AuthType shibboleth
    # ... Shibboleth config
</Location>
```

Users can test Kerberos at `/krb` while production uses `/shib`.

#### Phase 2: User Migration

1. **Educate users** on obtaining Kerberos tickets
2. **Configure browsers** for Kerberos
3. **Provide documentation** on kinit usage
4. **Set up helpdesk** procedures

#### Phase 3: Gradual Switchover

1. **Week 1:** Announce Kerberos availability
2. **Week 2-4:** Users test and report issues
3. **Week 5:** Set Kerberos as default
4. **Week 6+:** Monitor and support

#### Phase 4: Shibboleth Removal

Once all users migrated:

1. Change `ood_auth_method: kerberos` in inventory
2. Re-run Ansible playbook
3. Remove Shibboleth configuration
4. Decommission Shibboleth service principal

### Rollback Plan

To rollback to Shibboleth:

```bash
# Change inventory
ood_auth_method: shibboleth

# Re-run playbook
ansible-playbook -i inventory/prod ood.yml

# Restart services
systemctl restart httpd
```

## Summary

Kerberos authentication for OOD provides:

✅ **Seamless SSO** - Transparent authentication with existing tickets
✅ **Ticket Delegation** - Native support for credential forwarding
✅ **OS Integration** - Works with SSH, NFS, and other services
✅ **Simplified Architecture** - No federated IdP required
✅ **Enterprise Integration** - Works with Active Directory

**Key Implementation Points:**

1. **Site-Specific Configuration:**
   - Kerberos realm and KDC addresses
   - Domain to realm mappings
   - Keytab for HTTP service principal
   - User mapping logic

2. **Generic Configuration:**
   - mod_auth_gssapi Apache module
   - Standard keytab location
   - Credential cache management
   - OOD portal configuration structure

3. **Testing is Critical:**
   - Verify Kerberos independently first
   - Test keytab authentication
   - Confirm ticket delegation
   - Validate user mapping

4. **Security Best Practices:**
   - Protect keytab files (640 root:apache)
   - Isolate credential caches
   - Monitor for unauthorized access
   - Regular key rotation

By following this guide, you can successfully deploy OOD with Kerberos authentication in any environment with existing Kerberos infrastructure.




