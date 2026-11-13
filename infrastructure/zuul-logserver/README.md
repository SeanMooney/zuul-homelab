# Temporary Zuul log server

This component adds a web server and SSH upload sidecar alongside MinIO. It does
not change the current Zuul upload playbooks or remove any MinIO resources.

- Public URL: `https://zuul-logs.teim.app/`
- Internal SSH endpoint: `zuul-logserver-ssh.zuul-system.svc.cluster.local:22`
- SSH username: `zuul`
- Upload directory: `/srv/static/logs`
- Storage: `zuul-logserver-data`, a 10 GiB `local-path` PVC

nginx exposes directory listings and serves the log directory read-only. SSH
accepts the executor public key copied from
`apps/zuul/executor-pubkey-secret.yaml` into the SOPS-encrypted
`authorized-keys.enc.yaml` Secret. The Secret is mounted read-only and sshd reads
its `authorized_keys` file directly at `/etc/ssh/zuul_authorized_keys`. Mounting
the individual file beneath the image's root-owned `/etc/ssh` directory keeps
OpenSSH's strict ownership checks enabled. Restart the pod after authorized-key
changes because the file uses a `subPath` mount. There is no public SSH Service.

The init container creates `logs` and `ssh-host-keys` directories on the PVC and
generates an SSH host key only if one is absent. Only the `logs` subdirectory is
mounted in nginx. Pod replacement preserves both logs and the SSH host identity;
deleting or replacing the PVC may change that identity. Before configuring Zuul
uploads, obtain and verify the server's public host key for known-hosts pinning:

```bash
kubectl exec -n zuul-system deployment/zuul-logserver -c ssh -- \
  cat /var/lib/zuul-logserver/ssh-host-keys/ssh_host_ed25519_key.pub
```

The PVC is protected from Flux pruning. Its request does not enforce a disk
quota with local-path provisioning. Check node free space before deployment;
this uses the existing provisioner and does not change its disk placement.

DNS for `zuul-logs.teim.app` must point to the current ingress endpoint. TLS uses
the existing wildcard Secret in `zuul-system`. The Flux Kustomization decrypts
the authorized-keys Secret using the existing `sops-age` Secret.

Image references are pinned by digest. The SSH sidecar bypasses the CI image's
entrypoint and uses the mounted sshd configuration. It needs root to start sshd
and switch authenticated sessions to the `zuul` user; privilege escalation is
disabled and its root filesystem is read-only. The nginx container runs as an
unprivileged user.

Nightly retention and the Zuul SSH upload playbook are subsequent changes.
