# clear_paths

Safely clears all contents from a directory while preserving the directory itself.

## Requirements

None.

## Role Variables

### Required

- `clear_path`: Absolute path to directory to clear (string, minimum 5 characters, cannot be `/`)

## Dependencies

None.

## Example Usage

```yaml
- name: Clear temporary TLS directory
  include_role:
    name: clear_paths
  vars:
    clear_path: /tmp/bootstrap-tls
```

## Behavior

- Validates path is safe to clear (not `/`, not empty, minimum length)
- Ensures directory itself exists
- Removes all files and subdirectories within the path
- Preserves the root directory
- Provides debug output showing what will be deleted
- Non-recursive initial scan (but deletes subdirectories completely)

## Safety Features

- Refuses to clear `/`
- Requires path to be at least 5 characters
- Validates path is a non-empty string
- Lists contents before deletion

## License

Proprietary
