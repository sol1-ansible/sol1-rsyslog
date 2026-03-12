# sol1-rsyslogd

Configures rsyslog as a remote log collector. Listens for syslog over UDP/TCP (or other supported protocols), writes logs to per-host files, and optionally forwards to an external target such as Vector.

## Requirements

Collection dependencies are declared in [`meta/main.yml`](meta/main.yml).

## Variables

### Packages

| Variable | Default | Description |
|---|---|---|
| `rsyslog_packages` | *(see vars)* | Override the full list of packages to install. Defaults to `_rsyslog_packages` from `vars/default.yml` (`[rsyslog]`). |
| `rsyslog_additional_packages` | `[]` | Extra packages to install alongside the base packages. |

### Listener

| Variable | Default | Description |
|---|---|---|
| `rsyslog_listener_port` | `514` | Port rsyslog listens on for incoming remote logs. |
| `rsyslog_listener_proto` | `[udp, tcp]` | List of protocols to load. Supported values: `udp`, `tcp`, `relp`, `imtls`. Each value maps to the rsyslog module `im<proto>`. |

### Log storage

| Variable | Default | Description |
|---|---|---|
| `rsyslog_remote_root` | `/var/log/remote` | Base directory under which remote logs are written. |

### Templates

| Variable | Default | Description |
|---|---|---|
| `rsyslog_template_remote_log` | undefined | List of rsyslog template definitions. Merged into `rsyslog_templates` at run time. See structure below. |
| `rsyslog_templates` | undefined | Combined list of templates used to render the rsyslog config. Can be pre-populated externally; `rsyslog_template_remote_log` is unioned into it. |

#### Template structure

Each entry in `rsyslog_template_remote_log` (or `rsyslog_templates`) supports two types.

**`type: list`** — builds the log path from rsyslog `property()` and `constant()` statements:

```yaml
rsyslog_template_remote_log:
  - name: "RemoteLogs"
    type: "list"
    properties:
      - 'constant(value="/var/log/remote/")'
      - 'property(name="HOSTNAME")'
      - 'constant(value="/")'
      - 'property(name="PROGRAMNAME" onEmpty="unknown")'
      - 'constant(value="-")'
      - 'property(name="$YEAR")'
      - 'constant(value="-")'
      - 'property(name="$MONTH")'
      - 'constant(value="-")'
      - 'property(name="$DAY")'
      - 'constant(value=".log")'
    action:
      - selector: "*.*"
        type: "omfile"
        options:
          DynaFile: "RemoteLogs"
```

**`type: string`** — defines the log path as a single string using rsyslog `%PROPERTY%` syntax:

```yaml
rsyslog_template_remote_log:
  - name: "RemoteLogs"
    type: "string"
    properties: "/var/log/remote/%HOSTNAME%/%PROGRAMNAME%.log"
    action:
      - selector: "*.*"
        type: "omfile"
        options:
          DynaFile: "RemoteLogs"
```

For both types, `action` is a list of rsyslog actions bound to this template. Each action has:

| Key | Description |
|---|---|
| `selector` | rsyslog facility/severity selector (e.g. `*.*`) |
| `type` | rsyslog output module (e.g. `omfile`, `omfwd`) |
| `options` | Key/value pairs passed as module parameters |

### Forwarding

| Variable | Default | Description |
|---|---|---|
| `rsyslog_forward_enable` | `true` | Enable forwarding of all received logs to an external target. |
| `rsyslog_forward_target` | `127.0.0.1` | Hostname or IP of the forwarding destination. |
| `rsyslog_forward_port` | `1514` | Port of the forwarding destination. |
| `rsyslog_forward_protocol` | `tcp` | Protocol used for forwarding (`tcp` or `udp`). |

Forwarding uses octet-counted TCP framing and keepalive, suitable for targets such as Vector.

### Firewall

| Variable | Default | Description |
|---|---|---|
| `rsyslog_configure_firewall` | `true` | Open `rsyslog_listener_port` for each protocol in `rsyslog_listener_proto` using firewalld. |
