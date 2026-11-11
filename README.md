# mongodb test

This repository takes care of deploying a mongodb test instances. The environment is managed via the [AppFlow UI](https://control.vshn.net/appflow/apps/vshn/nunki-mongodb).

## CI/CD

- Every push and PR runs the [Smoke test](.github/workflows/ci.yml) workflow, which snapshot-tests the rendered chart.
- Every push is deployed to the `dev` environment by [Build and deploy to dev](.github/workflows/dev.yml). That is the only deployment target.
- Charts are published to `oci://ghcr.io/bacluc-agent/mongodb-helm` on `v*` tags by [Publish Helm Chart](.github/workflows/helm-release.yml). Label a PR with `Publish Helm Chart` (via [Autorelease](.github/workflows/auto-release.yaml)) to create the tag on merge.
- Renovate keeps the chart and image versions up to date; all docker and helm dependencies of the chart update in a single `renovate/mongodb` branch.

## Setup

The following pipeline variables are required for the AppFlow deployment:

- MONGODB_PVC_SIZE_GI
- OPENBAO_SVC_NAME
- OPENBAO_NAMESPACE

## replication

Currently replication isn't configured automatically.
Once all replicas have successfully started we need to exec into one of the pods and run the following command via mongosh.

The replica set name is not fixed: it defaults to the Helm release name and can be overridden with the `MONGODB_REPLICA_SET_NAME` environment variable.
The members are the pod hostnames `mongodb-<ordinal>.<namespace>.svc.cluster.local`, one per replica:

```mongodb-json
rs.initiate({
   _id: "<replica-set-name>",
   version: 1,
   members: [
      { _id: 0, host: "mongodb-0.<namespace>.svc.cluster.local:27017" },
      { _id: 1, host: "mongodb-1.<namespace>.svc.cluster.local:27017" },
      { _id: 2, host: "mongodb-2.<namespace>.svc.cluster.local:27017" }
   ]
 })
```

If it is broken you can fix it with:

```shell
rs.reconfig({
    _id: "<replica-set-name>",
    version: 1,
    members: [
      { _id: 0, host: "mongodb-0.<namespace>.svc.cluster.local:27017" },
      { _id: 1, host: "mongodb-1.<namespace>.svc.cluster.local:27017" },
      { _id: 2, host: "mongodb-2.<namespace>.svc.cluster.local:27017" }
    ]
  }, {force: true})
```

For a release named `dev` in namespace `fortcontrol-mongodb-dev` that resolves to the replica set name `dev` and the hosts `mongodb-N.fortcontrol-mongodb-dev.svc.cluster.local`.


## LUKS encryption:

See https://docs.appuio.cloud/user/how-to/encrypted-volumes.html

## testing

Chart rendering is snapshot-tested with [helm-chartsnap](https://github.com/jlandowner/helm-chartsnap) against the values in ./test/values.
Run `./scripts/chartsnap.sh` locally to compare, and `./scripts/chartsnap.sh --update` after intentional chart changes.

The snapshot check runs as the `chartsnap` job of the [Smoke test](.github/workflows/ci.yml) workflow on every push and PR.

