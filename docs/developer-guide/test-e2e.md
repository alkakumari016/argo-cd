# E2E Tests

The test [directory](https://github.com/argoproj/argo-cd/tree/master/test) contains E2E tests and test applications. The tests assume that Argo CD services are installed into `argocd-e2e` namespace or cluster in current context. A throw-away
namespace `argocd-e2e***` is created prior to the execution of the tests. The throw-away namespace is used as a target namespace for test applications.

The [/test/e2e/testdata](https://github.com/argoproj/argo-cd/tree/master/test/e2e/testdata) directory contains various Argo CD applications. Before test execution, the directory is copied into `/tmp/argo-e2e***` temp directory and used in tests as a
Git repository via file url: `file:///tmp/argo-e2e***`.

!!! note "Rancher Desktop Volume Sharing"
    The e2e git server runs in a container. If you are using Rancher Desktop, you will need to enable volume sharing for
    the e2e container to access the testdata directory. To do this, add the following to 
    `~/Library/Application\ Support/rancher-desktop/lima/_config/override.yaml` and restart Rancher Desktop:

    ```yaml
    mounts:
    - location: /private/tmp
      writable: true
    ```

## Running Tests Locally

### With virtualized chain
1. Start the e2e version `make start-e2e`
2. Run the tests: `make test-e2e`

### With local chain
1. Start the e2e version `make start-e2e-local`
2. Run the tests: `make test-e2e-local`

## Observing tests results

You can observe the tests by using the UI [http://localhost:4000/applications](http://localhost:4000/applications) with username `"admin"` and password `"password"`.

## Configuration of E2E Tests execution

The Makefile's `start-e2e` target starts instances of ArgoCD on your local machine, of which the most will require a network listener. If, for any reason, your machine already has network services listening on the same ports, then the e2e tests will not run. You can derive from the defaults by setting the following environment variables before you run `make start-e2e`:

* `ARGOCD_E2E_APISERVER_PORT`: Listener port for `argocd-server` (default: `8080`)
* `ARGOCD_E2E_REPOSERVER_PORT`: Listener port for `argocd-reposerver` (default: `8081`)
* `ARGOCD_E2E_DEX_PORT`: Listener port for `dex` (default: `5556`)
* `ARGOCD_E2E_REDIS_PORT`: Listener port for `redis` (default: `6379`)
* `ARGOCD_E2E_YARN_CMD`: Command to use for starting the UI via Yarn (default: `yarn`)

If you have changed the port for `argocd-server`, be sure to also set `ARGOCD_SERVER` environment variable to point to that port, e.g. `export ARGOCD_SERVER=localhost:8888` before running `make test-e2e` so that the test will communicate to the correct server component.


## Test Isolation

Some effort has been made to balance test isolation with speed. Tests are isolated as follows as each test gets:

* A random 5 character ID.
* A unique Git repository containing the `testdata` in `/tmp/argo-e2e/${id}`.
* A namespace `argocd-e2e-ns-${id}`.
* A primary name for the app `argocd-e2e-${id}`.

## Troubleshooting

**Tests fails to delete `argocd-e2e-ns-*` namespaces.**

This maybe due to the metrics server, run this:

```bash
kubectl api-resources
```

If it exits with status code 1, run:

```bash
kubectl delete apiservice v1beta1.metrics.k8s.io
```

Remove `/spec/finalizers` from the namespace

```bash
kubectl edit ns argocd-e2e-ns-*
```
