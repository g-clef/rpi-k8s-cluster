# Secrets Management

This playbook uses a separate `secrets.yaml` file to store sensitive configuration that should not be committed to git.

## Setup

1. Copy the example secrets file:
   ```bash
   cp secrets.yaml.example secrets.yaml
   ```

2. Edit `secrets.yaml` with your actual values:
   ```bash
   nano secrets.yaml  # or your preferred editor
   ```

3. The `secrets.yaml` file is automatically ignored by git and will not be committed.

## Required Secrets

You must set these values in `secrets.yaml`:

- `cluster_token`: A random 64+ character string for k3s cluster authentication
- `minio_root_password`: Password for MinIO S3 gateway
- `argocd_admin_password`: Password for ArgoCD admin user

## Optional Secrets

These are only needed if you're using the related features:

- `github_username` and `github_token`: For private GitHub repository access
- `docker_registry_secret`: For private Docker registry access
- `ray_redis_password`: For Ray cluster Redis authentication

## Security Notes

- Never commit `secrets.yaml` to version control
- Use strong, unique passwords for all services
- Consider using ansible-vault for additional encryption:
  ```bash
  ansible-vault encrypt secrets.yaml
  ```
- If using ansible-vault, run playbooks with:
  ```bash
  ansible-playbook -i inventory/hosts.yaml site.yaml -K --ask-vault-pass
  ```

## Example secrets.yaml

See `secrets.yaml.example` for the complete template with all available options.
