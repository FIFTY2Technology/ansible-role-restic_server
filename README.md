# Description
This role installs `rest-server` ([see here for sourcecode](https://github.com/restic/rest-server)), an HTTP REST server for restic clients with additional authentication and TLS support. All users from the password-store are registered in the `.htpasswd` file (append only so far).

# Requirements
* If using SSL certificates, it must exist before this role is applied (otherwise, starting the server will fail). If it is a self-signed certificate, restic clients must be run with the additional flag `--cacert` (not yet implemented).
  * SSL certificates must be accessible by user `rest-server`.
  * If no SSL certificate is used, make sure to template clients with `rest:http://`.
* Installation is currently only supported on Linux distributions using systemd.
* Depending on how you store client REST passwords, you might want to deploy those automatically.
  * REST authentication passwords are not deployed automatically to the server by this role by default.
  * Example: Install [pass](https://www.passwordstore.org/) on the Ansible runner and configure for password storage. See `tasks/add_rest_accounts.yml` for example, and ([see example configuration](https://www.fifty2.eu/innovation/how-we-provide-i-t-secrets-through-passwordstore-in-ansible-at-fifty2/)).
  * Other example: See "default" molecule scenario in `restic_remote` role. It stores REST passwords in `host_vars` and deploys them in a similar way as the example above.
  * Backup encryption password and REST authentication password (through custom REST repository template) creation are the backup client's concern (done in `restic_client` role).

## Remote backup clients
The server is capable of connecting to restic clients via SSH, if they can't reach the backup server directly, and trigger backup jobs there. This feature is described in further detail in the [`restic_remote` README](https://github.com/FIFTY2Technology/ansible-role-restic_remote/blob/main/README.md). This role configures basic requirements for this functionality, but this doesn't influence "normal" operation in case you choose to not deploy any remote backup clients.

# Role Variables
All variables which can be overridden are stored in defaults/main.yml file as well as in table below.

| Name | Default value | Description |
| ------ | ------ | ----- |
| `restic_server_user` | rest-server | User to run 'rest-server' as. Will also own `restic_server_backup_path` |
| `restic_server_group` | rest-server | Group to run 'rest-server' as. Will also own `restic_server_backup_path` |
| `restic_server_backup_path` | /opt/restic | The directory to store all encrypted backups and the `.htaccess` file. |
| `restic_server_version` | latest | Specify a release version to download from GitHub. Release versions must not include the leading `v` as tagged on GitHub. E.g. `0.10.0`. See: https://github.com/restic/rest-server/releases Default: 'latest' |
| `restic_server_from_local_dir` | "" | Provide a full path to a local directory (on the Ansible controller) where the `rest-server` binary is available, e.g. for deploying self-compiled versions. Binary must be named 'rest-server' inside this directory. Will be ignored if empty. |
| `restic_server_install_dir` | "/usr/local/bin" | Directory to install 'rest-server' binary into, on target host(s). |
| `restic_server_listen_address` | 0.0.0.0 | The default address for rest-server to listen for incoming clients |
| `restic_server_listen_port` | 3000 | The default port for rest-server to listen for incoming clients. If you plan to deploy `fifty2technology.restic_remote` to clients, `restic_server_listen_port` must be defined in a place where clients can read it, e.g. group_vars/all. |
| `restic_server_tls_enable` | false | Wheather to enable TLS support or not (default: not) |
| `restic_server_tls_cert` | - | Absolute path to TLS certificate |
| `restic_server_tls_key` | - | Absolute path to TLS key |

For all other variables, see the roles `defaults/main.yml` and check `host_vars/` directory for host-specific values.

# Dependencies
Built for `rest-server` version `0.10.0` and above (relies on the `--version` option).

# Example Playbook
Deploy server with HTTP and no users:
```
- hosts: backupservers
  gather_facts: true

  roles:
    - role: fifty2technology.restic_server
```

Deploy server with HTTPS (certificates must exist):
```
- hosts: backupservers
  gather_facts: true

  vars:
    restic_server_tls_enable: true
    restic_server_tls_cert: "/etc/letsencrypt/live/{{ inventory_hostname }}/fullchain.pem"
    restic_server_tls_key: "/etc/letsencrypt/live/{{ inventory_hostname }}/privkey.pem"

  roles:
    - role: fifty2technology.restic_server
```

Deploy server with HTTP and add clients based on Ansible group membership, get REST passwords from somewhere (e.g. `host_vars` pointing to Ansible Vault):
```
- hosts: backupservers
  gather_facts: true

  roles:
    - role: fifty2technology.restic_server

  post_tasks:
    - name: Add rest-server accounts
      community.general.htpasswd:
        path: "{{ restic_server_backup_path }}/.htpasswd"
        name: "{{ item.inventory_hostname | regex_replace('\\.', '_') }}"
        password: "{{ item.our_rest_password }}"
        crypt_scheme: bcrypt
        state: present
        owner: "{{ restic_server_user }}"
        group: "{{ restic_server_group }}"
        mode: '0600'
      become: true
      loop: "{{ groups['backupclients'] }}"
```
See `tasks/add_rest_accounts.yml` for another, similar example.
