Switch kubectl context and namespace to a known MVTS environment.

**Arguments:** `<env-name>`

Example: `/setup-context qa3-devrelayonm`

---

Parse `$ARGUMENTS` to get `env_name`, then look it up in the table below and run the two kubectl commands.

If `env_name` is not in the table, list the known environments and ask the user to confirm the cluster and namespace manually.

## Known Environments

| env-name | cluster | namespace |
|---|---|---|
| `qa3-devrelayonm` | `gke_greymatter-qa_us-east1_qa3-cluster` | `qa3-cluster-devrelayonm-greymatter` |
| `stg001-walmartpotstg` | `gke_gm-prod-sams-atl_us-east1_stg001-cluster` | `stg001-cluster-walmartpotstg-greymatter` |

## Commands to Run

```bash
kubectl config use-context <cluster>
kubectl config set-context --current --namespace=<namespace>
```

Then confirm success by running:
```bash
kubectl config current-context
kubectl config view --minify | grep namespace
```

Print the active context and namespace so the user can verify.

## Adding New Environments

When a new environment is encountered during debugging, add it to the table above, update `CLAUDE.md`'s skills index if needed, commit, and push to origin.
