# Temporary Zuul log server

This component serves Zuul logs over HTTPS with an SSH upload sidecar and a
nightly retention job. The base job in `ci-sean-mooney` uploads through the
executor's SSH agent and reports the manifest and bulk-download script.

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

nginx allows cross-origin reads from `https://zuul.teim.app`, so the Zuul UI can
load the manifest, console output and individual logs from this separate domain.
Kustomize generates the ConfigMap from `nginx.conf` and `sshd_config` with a
content hash in its name. Configuration changes therefore update the pod
template and trigger a rollout automatically.

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

## Nightly retention

`zuul-logserver-prune` runs every night at **03:00 UTC**. It mounts only the
`logs` PVC subdirectory and removes whole build directories whose **modification
time is older than 30 days**. It cannot access the SSH host-key directory. Concurrent cleanup
runs are forbidden, and the local PVC's node affinity places the job on the
node holding the logs.

GNU `find -P` matches only the upstream `<three-character shard>/<tenant>/<build
UUID>` directory layout at depth three. `-mmin +43200` selects directories more
than 30 days old, and `rm -rf` removes each selected build directory. Directory
symlinks are not followed; unrelated directories and root-level files are kept.

Viewing or downloading logs can change access time (`atime`), which this command
does not examine. It does not extend retention. Changes to the build directory's
own modification time can extend retention; modifying a nested file alone does
not necessarily change that directory timestamp.

For a read-only preview, run the same `find` command with only `-print`, omitting
the `-exec rm -rf -- {} +` action. Job logs record the selected directories and
any errors.

## MinIO retirement

MinIO is removed from the Flux tree, including its namespace, ingresses,
credentials and certificate reflection configuration. Historical logs are
discarded as agreed; the new log-server PVC is preserved.

The retired claim is `minio-system/data-minio-0`, bound to
`pvc-172f3b47-29bd-4bb9-b65d-62b5c2f83bea`. Its live reclaim policy is `Delete`,
and the provisioner's teardown script removes its backing directory:

```text
/var/mnt/local-path-provisioner/pvc-172f3b47-29bd-4bb9-b65d-62b5c2f83bea_minio-system_data-minio-0
```

After reconciliation, verify that the MinIO namespace, claim, PV and backing
directory are gone, then check free space with `df` in the log-server pod.
Allow normal PVC protection and provisioner cleanup to finish; do not remove
finalizers merely to hide a stalled cleanup. Both old and new volumes use the
same system filesystem, so deleting the old data releases capacity for new logs.
