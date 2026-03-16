# ansible-role-haproxy
![GitHub tag (latest SemVer)](https://img.shields.io/github/v/tag/chrecht/ansible-role-haproxy?sort=semver&label=version)

Ansible role to install and configure [HAProxy](https://www.haproxy.org/) on Ubuntu, including optional installation of the [HAProxy Data Plane API](https://www.haproxy.com/documentation/dataplaneapi/latest/).

## Features

- Installs HAProxy from the official [vbernat PPA](https://launchpad.net/~vbernat) (supports versions 2.3 through 3.2)
- Deploys a fully templated `haproxy.cfg`
- Generates a 2048-bit Diffie-Hellman parameters file (`dhparam.pem`) if not already present
- Raises the systemd `LimitNOFILE` via a drop-in override (configurable max open files)
- Optionally installs and configures the HAProxy Data Plane API
- Validates the final HAProxy configuration before finishing

## Requirements

- **OS:** Ubuntu (Xenial, Bionic, Focal, Jammy, Noble)
- **Ansible:** ≥ 2.14
- **Collections:**
  - `ansible.posix` (for the `sysctl` module)
  - `community.general` (if managing apt repositories on older Ansible versions)

Install required collections:

```bash
ansible-galaxy collection install ansible.posix
```

## Role Variables

All variables are defined in `defaults/main.yml` and can be overridden in your playbook or inventory.

### HAProxy

| Variable | Default | Description |
|---|---|---|
| `haproxy_version` | `"3.2"` | HAProxy version to install from the vbernat PPA. Supported: `2.3`, `2.4`, `2.6`, `3.0`, `3.2`. |
| `haproxy_config` | `"default.haproxy.conf.j2"` | Jinja2 template (filename only) to use as `haproxy.cfg`. |
| `haproxy_allow_bind_non_local_ip` | `true` | Set `net.ipv4.ip_nonlocal_bind=1` via sysctl. Required for VIP/floating-IP setups (e.g. keepalived). |
| `haproxy_max_open_files` | `"5000000"` | Value for `LimitNOFILE` in the systemd service drop-in override. |

### HAProxy Data Plane API

| Variable | Default | Description |
|---|---|---|
| `haproxy_dataplaneapi.version` | `"3.2.9"` | Data Plane API version to download and install. |
| `haproxy_dataplaneapi.user` | `"admin"` | API basic-auth username. **Change this.** |
| `haproxy_dataplaneapi.password` | `"adminpwd"` | API basic-auth password. **Change this — use a vault-encrypted value.** |
| `haproxy_dataplaneapi.api.host` | `"0.0.0.0"` | Address the API listens on. |
| `haproxy_dataplaneapi.api.port` | `5555` | Port the API listens on. |
| `haproxy_dataplaneapi.api.haproxy_bin` | `"/usr/sbin/haproxy"` | Path to the HAProxy binary. |
| `haproxy_dataplaneapi.api.config_file` | `"/usr/local/etc/haproxy/haproxy.cfg"` | HAProxy config file path used by the API. |
| `haproxy_dataplaneapi.api.reload_cmd` | `"kill -SIGUSR2 1"` | Command used to reload HAProxy. |
| `haproxy_dataplaneapi.api.restart_cmd` | `"kill -SIGUSR2 1"` | Command used to restart HAProxy. |
| `haproxy_dataplaneapi.api.reload_delay` | `5` | Minimum delay (seconds) between two reloads. |

## Dependencies

None.

## Example Playbook

Minimal usage with defaults:

```yaml
- hosts: loadbalancers
  become: true
  roles:
    - role: ansible-role-haproxy
```

Customised example:

```yaml
- hosts: loadbalancers
  become: true
  vars:
    haproxy_version: "3.2"
    haproxy_allow_bind_non_local_ip: true
    haproxy_max_open_files: "1000000"
    haproxy_dataplaneapi:
      version: "3.2.9"
      user: ops
      password: "{{ vault_dataplaneapi_password }}"
      api:
        command: /usr/bin/dataplaneapi
        host: 127.0.0.1
        port: 5555
        haproxy_bin: /usr/sbin/haproxy
        config_file: /etc/haproxy/haproxy.cfg
        reload_cmd: "systemctl reload haproxy"
        restart_cmd: "systemctl restart haproxy"
        reload_delay: 5
  roles:
    - role: ansible-role-haproxy
```

## Security Notes

- **Change the default Data Plane API credentials** (`haproxy_dataplaneapi.user` / `haproxy_dataplaneapi.password`) and store the password in an Ansible Vault-encrypted variable.
- Consider binding the Data Plane API to `127.0.0.1` instead of `0.0.0.0` if it does not need to be accessible remotely.
- The default HAProxy config template enables TLS with strong cipher suites and generates a dedicated `dhparam.pem` file.

## License

GPLv2

## Author

Christian WALDBILLIG — CBW
