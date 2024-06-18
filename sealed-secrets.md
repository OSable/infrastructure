# Post-installation procedure
Create sealed secrets certificates secret: `kubectl -n kube-system create secret tls sealed-secrets-certs --cert="<path to public key>" --key="<path to private key>"`
Label the secret so that the controller picks it up: `kubectl -n kube-system label secret sealed-secrets-certs sealedsecrets.bitnami.com/sealed-secrets-key=active`
Restart the controller to use the new certificates: `kubectl rollout restart -n kube-system deployment sealed-secrets-controller`

## Kubeseal incompatibility
By default, the Helm chart deploys sealed-secrets into its own namespace, with pods called `sealed-secrets`. Kubeseal, on the other hand, expects sealed-secrets to be in the `kube-system` namespace, with pods named `sealed-secrets-controller`. You can fix this by either passing `--controller-name sealed-secrets --controller-namespace sealed-secrets` to Kubeseal every time (perhaps with a bash alias?), or modify the helm chart values to deploy sealed-secrets where it is expected to be.

## Secrets permissions issue
Once sealed-secrets had been installed, I was seeing messages similar to `cannot list resource "secrets" in API group "" at the cluster scope` in the logs.
To fix this, I had to manually modify the clusterrole that the helm chart made to add the `list` and `watch` permissions for secrets.
```
kubectl edit clusterrole secrets-unsealer
kubectl rollout restart -n sealed-secrets deployment sealed-secrets
```