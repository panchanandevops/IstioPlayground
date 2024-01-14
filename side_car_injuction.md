## Automatic sidecar injection

For our simplicity we will use Automatic sidecar injection.

When you set the `istio-injection=enabled` label on a namespace and the injection webhook is enabled, any new pods that are created in that namespace will automatically have a sidecar added to them.


for labeling namespace to `istio-injection=enabled` , use need this command:

```bash
kubectl label namespace <your_namespace> istio-injection=enabled
```

to check namespaces labels with istio-injection:

```bash
kubectl get namespace -L istio-injection
```