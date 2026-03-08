# systemd_service

Reusable role for installing and managing systemd services with configuration-aware restart logic.

## Requirements

- systemd-based Linux distribution
- Service-specific systemd unit template in calling role

## Role Variables

### Required

- `service_name`: Name of the systemd service (e.g., `consul`, `vault`)
- `service_unit_template`: Path to Jinja2 template for systemd unit file (e.g., `consul.service.j2`)
- `service_config_changed`: Registered result from configuration task containing `.changed` attribute

## Dependencies

None.

## Example Usage

```yaml
- name: Deploy service configuration
  template:
    src: myservice.conf.j2
    dest: /etc/myservice/myservice.conf
  register: myservice_config

- name: Install and manage myservice systemd service
  include_role:
    name: systemd_service
  vars:
    service_name: myservice
    service_unit_template: myservice.service.j2
    service_config_changed: "{{ myservice_config }}"
```

## Behavior

- **First install**: Service is started
- **Config changed**: Service is restarted
- **No changes**: Service state is checked (idempotent)

## License

Proprietary
