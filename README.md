# Description
This role installs `rest-server` ([see here for sourcecode](https://github.com/restic/rest-server)), an HTTP REST server for restic clients with additional authentication and TLS support. All users from the password-store are registered in the `.htpasswd` file (append only so far).

# Requirements
* A trusted SSL certificate is required (we use Let's Encrypt). If it is not trusted (e.g. self-signed), restic clients must be run with the additional flag `--cacert` (not yet implemented).
  * SSL certificates must be accessible by user `rest-server`.
* Installation is currently only supported for x86_64 architecture.
* Installation is currently only supported on Linux distributions using systemd.
* [pass](https://www.passwordstore.org/) installed on the Ansible runner and configured for password storage. ([See example configuration](https://www.fifty2.eu/innovation/how-we-provide-i-t-secrets-through-passwordstore-in-ansible-at-fifty2/)) Backup encryption password and REST authentication password creation are the backup client's concern (done in `restic_client` role).

## Remote backup clients
The server is capable of connecting to restic clients via SSH if they can't reach the backup server directly, and trigger backup jobs there. This feature is described in further detail in the `restic_remote` README file.

To allow the server to login on clients via SSH, it is necessary to specify the SSH public key of the server as variable `restic_remote_server_authorized_key`. The key is shown by the `Print SSH public key - See README file for usage of this key` task when running this role. This variable must be accessible to (at least) all remote backup clients (e.g. via `group_vars/remoteclients.yml`) and is necessary for the `restic_remote` role to work correctly.

# Role Variables
All variables which can be overridden are stored in defaults/main.yml file as well as in table below.

| Name | Default value | Description |
| ------ | ------ | ----- |
| `backup_server` | `"{{ groups['backupservers'] \| first }}"` | Inventory hostname of the machine running the backup server (where the backup clients shall send their backup to). Used to correctly set the SSH key comment |
| `restic_server_user` | rest-server | User to run 'rest-server' as. Will also own `restic_server_backup_path` |
| `restic_server_group` | rest-server | Group to run 'rest-server' as. Will also own `restic_server_backup_path` |
| `restic_server_backup_path` | /opt/restic | The directory to store all encrypted backups and the `.htaccess` file. |
| `restic_server_listen_address` | 0.0.0.0 | The default address for rest-server to listen for incoming clients |
| `restic_server_listen_port` | 3000 | The default port for rest-server to listen for incoming clients |
| `restic_server_tls_enable` | false | Wheather to enable TLS support or not (default: not) |
| `restic_server_tls_cert` | - | Absolute path to TLS certificate |
| `restic_server_tls_key` | - | Absolute path to TLS key |

For all other variables, see the roles `defaults/main.yml` and check `host_vars/` directory for host-specific values.

# Dependencies
Built for `rest-server` version `0.10.0` and above (relies on the `--version` option).

# Example Playbook
```
- hosts: backupservers
  gather_facts: true
  become: true

  roles:
  - role: restic_server
```

```
- hosts: backupservers
  gather_facts: true
  become: true

  vars:
    restic_server_tls_enable: true
    restic_server_tls_cert: "/etc/letsencrypt/live/{{ backup_server }}/fullchain.pem"
    restic_server_tls_key: "/etc/letsencrypt/live/{{ backup_server }}/privkey.pem"

  roles:
  - role: restic_server
```

