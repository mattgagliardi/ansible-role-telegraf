# Ansible Role: telegraf

[![CI](https://github.com/mattgagliardi/telegraf-config/actions/workflows/ci.yml/badge.svg)](https://github.com/mattgagliardi/telegraf-config/actions/workflows/ci.yml)

Installs and configures the [Telegraf](https://www.influxdata.com/time-series-platform/telegraf/) metrics collection agent on:

- **Linux** – Ubuntu 22.04 / 24.04 and Red Hat Enterprise Linux / Rocky Linux 8 / 9
- **Windows** – Windows Server 2019 / 2022

---

## Requirements

- Ansible ≥ 2.12
- Target hosts must have internet access to reach the InfluxData package repositories (or you can mirror them internally).
- No additional Ansible collections are required beyond `ansible.builtin`.

---

## Role Variables

All variables are defined in `defaults/main.yml` and can be overridden in the calling playbook or inventory.

| Variable | Default | Description |
|---|---|---|
| `telegraf_service_enabled` | `true` | Whether the Telegraf service is enabled and started at boot. Set to `false` to install without enabling. |
| `telegraf_agent_interval` | `"10s"` | How often Telegraf collects metrics. |
| `telegraf_agent_flush_interval` | `"10s"` | How often Telegraf flushes metrics to outputs. |
| `telegraf_outputs` | `[]` | List of output plugin configuration dicts, populated by the caller (see example below). |
| `telegraf_inputs_extra` | `[]` | Additional input plugin dicts the caller can append to the default inputs. |

### OS-family variables (not normally overridden)

These are set automatically by including `vars/Debian.yml` or `vars/RedHat.yml`:

**Debian/Ubuntu (`vars/Debian.yml`)**

| Variable | Description |
|---|---|
| `telegraf_repo_url` | InfluxData apt repository URL (e.g. `https://repos.influxdata.com/debian`). |
| `telegraf_repo_key_url` | URL to the InfluxData GPG signing key. |
| `telegraf_package` | Package name to install (`telegraf`). |

**RedHat/Rocky/RHEL (`vars/RedHat.yml`)**

| Variable | Description |
|---|---|
| `telegraf_repo_baseurl` | InfluxData yum/dnf repository base URL. |
| `telegraf_repo_gpgkey` | URL to the InfluxData GPG signing key. |
| `telegraf_package` | Package name to install (`telegraf`). |

---

## Example Playbook

```yaml
- hosts: monitoring_targets
  become: true
  roles:
    - role: mattgagliardi.telegraf
      vars:
        telegraf_agent_interval: "30s"
        telegraf_agent_flush_interval: "30s"
        telegraf_outputs:
          - plugin: influxdb_v2
            config: |
              urls = ["http://influxdb.example.com:8086"]
              token = "{{ '{{' }} vault_influxdb_token {{ '}}' }}"
              organization = "myorg"
              bucket = "telegraf"
        telegraf_inputs_extra:
          - |
            [[inputs.exec]]
              commands = ["/usr/local/bin/my-collector.sh"]
              data_format = "influx"
```

---

## License

MIT

---

## Author

[mattgagliardi](https://github.com/mattgagliardi)
