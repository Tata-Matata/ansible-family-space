# assert_paths

Validation role that asserts required filesystem paths exist with correct type, ownership, and permissions.

## Requirements

None.

## Role Variables

### Required

- `required_paths`: List of path specifications to validate

Each item in `required_paths` should contain:
- `path`: Absolute path to check (required)
- `type`: Expected type - `file` or `folder` (required)
- `owner`: Expected owner username (optional)
- `group`: Expected group name (optional)
- `mode`: Expected octal mode string (optional, e.g., `"0644"`)

## Dependencies

None.

## Example Usage

```yaml
- name: Verify TLS certificates are present
  include_role:
    name: assert_paths
  vars:
    required_paths:
      - path: /etc/tls
        type: folder
        owner: root
        group: root
        mode: "0755"
      - path: /etc/tls/ca.pem
        type: file
        owner: root
        group: root
        mode: "0644"
      - path: /etc/tls/server.key
        type: file
        owner: root
        group: root
        mode: "0600"
```

## Behavior

- Checks each path exists
- Validates type matches (file vs directory)
- Validates ownership if specified
- Validates permissions if specified
- Fails with detailed error message if any check fails
- Does not modify filesystem (read-only validation)

## License

Proprietary
