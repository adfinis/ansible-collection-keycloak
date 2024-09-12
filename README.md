# Ansible Collection - adfinis.keycloak

![License](https://img.shields.io/github/license/adfinis/ansible-collection-keycloak)
![GitHub Workflow Status](https://img.shields.io/github/actions/workflow/status/adfinis/ansible-collection-keycloak/ansible-lint.yml)
[![adfinis.keycloak on Ansible Galaxy](https://img.shields.io/badge/collection-adfinis.keycloak-blue)](https://galaxy.ansible.com/ui/repo/published/adfinis/keyclaok/)


Install and configure Keycloak Quarkus as single node or HA cluster.

## Roles

### `adfinis.keycloak.keycloak`

Install and configure Keycloak.

Tags:
- `role::keycloak:install`: Download and install Keycloak.  Not idempotent.
- `role::keycloak:config`: Render Keycloak configuration files.
- `role::keycloak:init`: Create the initial admin user in the `master` realm.  Relevant tasks are tagged with `never`, so this tag must be used explicitly.
- `role::keycloak:realms`: Upload existing Keycloak realm configuration, e.g. exported from another Keycloak instance.  Relevant tasks are tagged with `never`, so this tag must be used explicitly.
- `role::keycloak`: All of the above except `role::keycloak:init` and `role::keycloak:realms`.

### `adfinis.keycloak.reverseproxy`

Install and configure a TLS-terminating reverse proxy using Apache2.

Tags:
- `role::reverseproxy:install`: Install Apache2.
- `role::reverseproxy:config`: Configure Apache2 as a reverse-proxy for Keycloak.
- `role::reverseproxy`: All of the above.


## Usage

### Minimal Example

In this example we deploy a single node Keycloak instance on Debian using the builtin H2 database backend.

First, we create our inventory:

```ini
[keycloak]
kc01.example.org
```

The playbook simply applies the two roles:

```yaml
---

- hosts: keycloak
  roles:
    - adfinis.keycloak.keycloak
    - adfinis.keycloak.reverseproxy
```

We prepare the following hostvars in `group_vars/keycloak/main.yml`:

```yaml
---

# Use the local H2 database backend - NOT SUITED FOR PRODUCTION
keycloak_db: dev-file

keycloak_hostname: "https://keycloak.example.org"

keycloak_admin_user: admin
keycloak_admin_password: changeme  # Choose your own and store securely, e.g. in ansible-vault

# To achieve session persistence in a single-node setup, the persistent-user-sessions feature can be enabled.
# This is a PREVIEW feature in Keycloak 25.
keycloak_features:
  - persistent-user-sessions

reverseproxy_hostname: keycloak.example.org
reverseproxy_backend: "http://127.0.0.1:8080"
# The certificates already need to be in place
# If you omit these two lines, your system's default snakeoil certs will be used
reverseproxy_cert_file: /etc/ssl/certs/keycloak.example.org.pem
reverseproxy_key_file: /etc/ssl/private/keycloak.example.org.key

# Optional: To enable the Apache2 status page on http://localhost/server-status
+reverseproxy_statuspage_path: "/server-status"

# This exposes the Keycloak admin interface - restrict this according to your requirements
reverseproxy_additional_config: |
  <Location /admin>
    <RequireAny>
      Require ip 2001:db8::/64
      Require ip 192.0.2.0/24
    </RequireAny>
    ProxyPass {{ reverseproxy_backend }}/admin
    ProxyPassReverse {{ reverseproxy_backend }}/admin
  </Location>
```

When running the playbook the first time, the `role::keycloak:init` tag must be used to run the one-off Keycloak initialization, which creates the inital admin user and sets its password:

```shell-session
$ ansible-playbook -i inventory playbook.yml -t all,role::keycloak:init --diff
```

On subsequent runs, this tag should not be used anymore, as it won't do anything except cause additional downtime.

If everything went right, you should now be able to access the Keycloak admin interface at [https://keycloak.example.org/admin/],
and sign in with the credentials you put into your hostvars.


### HA Cluster with External Database

Let's extend our inventory with a second node:

```ini
[keycloak]
kc01.example.org
kc02.example.org
```

The playbook remains the same, so let's focus on our hostvars instead.  Again, let's take a look into `group_vars/keycloak/main.yml`:

```yaml
---

keycloak_hostname: keycloak.example.org

keycloak_admin_user: admin
keycloak_admin_password: changeme  # Choose your own and store securely, e.g. in ansible-vault

# All Keycloak nodes need a common database, so we can't use h2 anymore
# Instead, we assume that there is a PostgreSQL database available on the network.
keycloak_db: postgres
keycloak_db_url: "jdbc:postgresql://pgsql01.example.org/keycloak"
keycloak_db_user: keycloak
keycloak_db_pass: changemetoo  # Choose your own and store securely, e.g. in ansible-vault

# The easiest clustering method is to use an UDP multicast mechanism to discover other nodes on the same network:
keycloak_stack: udp
# If this is undesired or not possible (e.g. on a non-multicast-enabled network), you can provide a static peer list instead:
keycloak_stack: tcp-static
keycloak_members:  # This is a host: port mapping.
  kc01.example.org: 7800  # Using the hostnames here requires DNS resolution to work.  IP addresses can be used as well.
  kc02.example.org: 7800


reverseproxy_hostname: keycloak.example.org
reverseproxy_backend: "http://127.0.0.1:8080"
# The certificates already need to be in place
# If you omit these two lines, your system's default snakeoil certs will be used
reverseproxy_cert_file: /etc/ssl/certs/keycloak.example.org.pem
reverseproxy_key_file: /etc/ssl/private/keycloak.example.org.key

# This exposes the Keycloak admin interface - restrict this according to your requirements
reverseproxy_additional_config: |
  <Location /admin>
    <RequireAny>
      Require ip 2001:db8::/64
      Require ip 192.0.2.0/24
    </RequireAny>
    ProxyPass {{ reverseproxy_backend }}/admin
    ProxyPassReverse {{ reverseproxy_backend }}/admin
  </Location>
```

This setup needs another component to be useful:  A load balancer.
The `keycloak.example.org` DNS record should point to the load balancer, which has both Keycloak nodes set as its backends.

If TLS is terminated on the load balancer, and you can make sure that only [certain URL paths](https://www.keycloak.org/server/reverseproxy#_exposed_path_recommendations)
are forwarded to Keycloak, the Apache2 reverse proxy could be omitted entirely.


## Uploading Realm Configuration

1. Place your realm JSON files in a `files` folder relative to the playbook.
1. Update your hostvars to configure the names of files to upload, e.g.:

        keycloak_realm_config:
          - realm-prod.json
          - realm-dev.json
1. Run the playbook with the `role::keycloak:realms` tag: `ansible-playbook -i inventory playbook.yml -t role::keycloak:realms --diff`

## License

[GPL-3.0-or-later](https://github.com/adfinis/ansible-collection-keycloak/blob/main/LICENSE)

## Author Information

The Ansible collection `adfinis.keycloak` was written by:

* Adfinis AG | [Website](https://www.adfinis.com/) | [GitHub](https://github.com/adfinis)

