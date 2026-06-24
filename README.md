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
| `telegraf_influxdb_v3_insecure_skip_verify` | `true` | Whether Telegraf skips TLS certificate verification for the InfluxDB v3 endpoint. Keep `true` for self-signed/dev certs; set `false` in production with valid certs. |
| `telegraf_influxdb_v3_token`    | `""`         | InfluxDB v3 auth token. Leave empty when the receiving instance has authentication disabled (its default state) — the role then omits the `token =` line entirely.                               |
| `telegraf_influxdb_v3_retention_enabled` | `true` | Enable post-start retention configuration workflow for the InfluxDB v3 database. |
| `telegraf_influxdb_v3_retention_duration` | `"7d"` | Retention period applied to the target database. |
| `telegraf_influxdb_v3_retention_wait_extra_seconds` | `3` | Extra seconds added to `telegraf_agent_interval` before API checks begin. |
| `telegraf_influxdb_v3_retention_check_database_presence` | `true` | Poll the database list API and wait until the target database appears before setting retention. |
| `telegraf_influxdb_v3_retention_check_retries` | `10` | Maximum database list polling attempts. |
| `telegraf_influxdb_v3_retention_check_delay` | `2` | Seconds between database list polling attempts. |
| `telegraf_influxdb_v3_retention_http_timeout` | `10` | HTTP timeout in seconds for retention workflow API calls. |
| `telegraf_influxdb_v3_retention_best_effort` | `true` | When true, warns and continues if database-check or retention API calls fail; when false, fails the play. |
| `telegraf_outputs`              | `[]`         | List of additional output plugin configuration dicts, populated by the caller (see example below). Rendered in addition to any first-class InfluxDB v3 output.                                   |
| `telegraf_inputs_extra`         | `[]`         | Additional input plugin dicts the caller can append to the default inputs.                                                                                                                       |
| `telegraf_win_version`          | `""`         | (Windows only) Pin a specific Telegraf version, e.g. `"1.38.3"`. When empty, the role queries `api.github.com` for the latest release tag. Pin this to remove the implicit dependency on GitHub. |
| `telegraf_win_cleanup`          | `true`       | (Windows only) Remove the downloaded ZIP and extract directory after a successful install. Set `false` to keep them for debugging.                                                               |

### Automatic Database Retention Configuration

When all of the following are true, the role applies retention automatically:

- `telegraf_service_enabled: true`
- `telegraf_influxdb_v3_enabled: true`
- `telegraf_influxdb_v3_retention_enabled: true`
- `telegraf_influxdb_v3_urls` contains at least one URL

Execution order:

1. Start/restart Telegraf and flush pending handlers.
2. Wait for `telegraf_agent_interval + telegraf_influxdb_v3_retention_wait_extra_seconds`.
3. Query `GET <first_url>/api/v3/configure/database?format=json` until the target database name appears.
4. Call `POST <first_url>/api/v3/configure/database/retention_period?db=<db>&duration=<duration>`.

Notes:

- The API base URL is the first entry in `telegraf_influxdb_v3_urls`.
- `telegraf_agent_interval` must be seconds-based (for example `"10s"`).
- If `telegraf_influxdb_v3_retention_best_effort` is true, failures in the check/update path log warnings and do not fail the play.

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
        telegraf_influxdb_v3_insecure_skip_verify: true
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

Test to force CI run
