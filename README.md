# Przeniesione do k8s-homelab (Dzień 83, 12-11-2026): role rclone_mount, postgres_backup.

# home-infra


1. Add ansible ssh Key to servers:

```
ssh-copy-id -i ~/.ssh/ansible.pub 10.0.0.110
```

```
ssh-copy-id -i ~/.ssh/ansible.pub 10.0.0.107
```


2. Encrypt vault file:

```
ansible-vault encrypt group_vars/fujitsu/vault.yml
```
3. Decrypt vault file:
```
ansible-vault decrypt group_vars/fujitsu/vault.yml
```