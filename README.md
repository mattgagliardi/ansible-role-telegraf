# Ansible Role: telegraf

[![CI](https://github.com/mattgagliardi/telegraf-config/actions/workflows/ci.yml/badge.svg)](https://github.com/mattgagliardi/telegraf-config/actions/workflows/ci.yml)

Installs and configures the [Telegraf](https://www.influxdata.com/time-series-platform/telegraf/) metrics collection agent on:

- **Linux** – Ubuntu 22.04 / 24.04 and Red Hat Enterprise Linux / Rocky Linux 8 / 9
- **Windows** – Windows Server 2019 / 2022

---

## Requirements

- Ansible ≥ 2.12
- Target hosts must have internet access to reach the InfluxData package repositories (or you can mirror them internally).
- For Windows targets:
  - `ansible.windows` collection on the control node (`ansible-galaxy collection install ansible.windows`).
  - `pywinrm` Python package on the control node when running over WinRM (`pip install pywinrm`). Without it Ansible cannot connect and fails before the role runs.
  - The control node must be able to reach `api.github.com` _unless_ you pin a version with `telegraf_win_version` (see below).

---

## Role Variables

All variables are defined in `defaults/main.yml` and can be overridden in the calling playbook or inventory.

| Variable                        | Default      | Description                                                                                                                                                                                      |
| ------------------------------- | ------------ | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| `telegraf_service_enabled`      | `true`       | Whether the Telegraf service is enabled and started at boot. Set to `false` to install without enabling.                                                                                         |
| `telegraf_agent_interval`       | `"10s"`      | How often Telegraf collects metrics.                                                                                                                                                             |
| `telegraf_agent_flush_interval` | `"10s"`      | How often Telegraf flushes metrics to outputs.                                                                                                                                                   |
| `telegraf_influxdb_v3_enabled`  | `false`      | When `true`, render an `[[outputs.influxdb_v3]]` block in the generated config using the variables below.                                                                                        |
| `telegraf_influxdb_v3_urls`     | `[]`         | List of InfluxDB v3 endpoint URLs (e.g. `["http://influxdb.example.com:8181"]`).                                                                                                                 |
| `telegraf_influxdb_v3_database` | `"telegraf"` | InfluxDB v3 database name (replaces the v2 `bucket` concept).                                                                                                                                    |
| `telegraf_influxdb_v3_token`    | `""`         | InfluxDB v3 auth token. Leave empty when the receiving instance has authentication disabled (its default state) — the role then omits the `token =` line entirely.                               |
| `telegraf_outputs`              | `[]`         | List of additional output plugin configuration dicts, populated by the caller (see example below). Rendered in addition to any first-class InfluxDB v3 output.                                   |
| `telegraf_inputs_extra`         | `[]`         | Additional input plugin dicts the caller can append to the default inputs.                                                                                                                       |
| `telegraf_win_version`          | `""`         | (Windows only) Pin a specific Telegraf version, e.g. `"1.38.3"`. When empty, the role queries `api.github.com` for the latest release tag. Pin this to remove the implicit dependency on GitHub. |
| `telegraf_win_cleanup`          | `true`       | (Windows only) Remove the downloaded ZIP and extract directory after a successful install. Set `false` to keep them for debugging.                                                               |

### OS-family variables (not normally overridden)

These are set automatically by including `vars/Debian.yml` or `vars/RedHat.yml`:

**Debian/Ubuntu (`vars/Debian.yml`)**

| Variable                | Description                                                                 |
| ----------------------- | --------------------------------------------------------------------------- |
| `telegraf_repo_url`     | InfluxData apt repository URL (e.g. `https://repos.influxdata.com/debian`). |
| `telegraf_repo_key_url` | URL to the InfluxData GPG signing key.                                      |
| `telegraf_package`      | Package name to install (`telegraf`).                                       |

**RedHat/Rocky/RHEL (`vars/RedHat.yml`)**

| Variable                | Description                             |
| ----------------------- | --------------------------------------- |
| `telegraf_repo_baseurl` | InfluxData yum/dnf repository base URL. |
| `telegraf_repo_gpgkey`  | URL to the InfluxData GPG signing key.  |
| `telegraf_package`      | Package name to install (`telegraf`).   |

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
        telegraf_influxdb_v3_enabled: true
        telegraf_influxdb_v3_urls:
          - "http://influxdb.example.com:8181"
        telegraf_influxdb_v3_database: "telegraf"
        # Omit telegraf_influxdb_v3_token (or leave it as "") when the
        # receiving InfluxDB v3 instance has authentication disabled.
        telegraf_influxdb_v3_token: "{{ '{{' }} vault_influxdb_token {{ '}}' }}"
        telegraf_inputs_extra:
          - |
            [[inputs.exec]]
              commands = ["/usr/local/bin/my-collector.sh"]
              data_format = "influx"
```

> **Auth-disabled receivers:** InfluxDB v3 ships with authentication disabled by default. In that scenario, leave `telegraf_influxdb_v3_token` unset (or empty) and the role will omit the `token =` line from the rendered config.

---

## License

MIT

---

## Author

[mattgagliardi](https://github.com/mattgagliardi)
