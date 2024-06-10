This cluster will use:
- K3s
	- Enable dashboard
	- Disable ServiceLB in favour of MetalLB
	- Disable Flannel in favour of Calico
	- Disable kube-proxy in favour of Calico's eBPF dataplane
	- Secrets encryption
- Calico CNI
	- eBPF dataplane
- MetalLB load balancer
- Traefik ingress controller

Some handy bash functions I wrote that you may want to add to your bashrc:

```bash
kubedebug() {
	kubectl run -i --rm --tty debug --image=busybox --restart=Never -- sh
}

kubeexec() {
        pod=$1
        shift
        kubectl exec --stdin --tty $pod "$@" -- /bin/sh
}

alias kubens='kubectl config set-context --current --namespace'
```

For kubens, use "" or default to reset to default. \
Useful debugging command if deployments create pods but pods are pending and `kubectl describe` is no help: `kubectl get events -n whatever`

# Installing K3s

Place this configuration file at `/etc/rancher/k3s/config.yaml`

```yaml
# Cluster config
cluster-cidr: "10.42.0.0/16"       # Pod address space
service-cidr: "10.43.0.0/16"       # Service address space
cluster-dns: "10.43.0.10"          # CoreDNS address

# Calico required configuration
flannel-backend: none              # Disable Flannel CNI
disable-network-policy: true       # Disable default network policy

# Etc
disable:
  - servicelb # Disable ServiceLB for MetalLB
secrets-encryption: true           # Enable secrets encryption
service-node-port-range: "0-65535" # Allow node ports on any port
```

Creating the cluster: `curl -sfL https://get.k3s.io | sh -`

Since we're not proving a token, one will be generated for us and stored at `/var/lib/rancher/k3s/server/token`. We'll need it later if we want a multi-node cluster.

After setup is complete, edit `~.bashrc` to include:`export KUBECONFIG=/etc/rancher/k3s/k3s.yaml`

At this point, `kubectl get pods -A` should show all pods in a pending state. This is normal, the pods cannot start because there is no CNI as we've disabled Flannel. 

I'm not disabling kube-proxy on install because it handles pod-service communication, which means that when I go to install tigera-operator and it wants to talk to the apiserver, it can't.

# Installing Calico

So, we need a CNI. I'm going to use Calico, so install Tigera Operator:
```bash
kubectl create -f https://raw.githubusercontent.com/projectcalico/calico/v3.27.2/manifests/tigera-operator.yaml
```

Apply the following ConfigMap in the `tigera-operator` namespace:
```yaml
kind: ConfigMap  
apiVersion: v1  
metadata:  
  name: kubernetes-services-endpoint  
  namespace: tigera-operator  
data:  
  KUBERNETES_SERVICE_HOST: "10.43.0.1"  
  KUBERNETES_SERVICE_PORT: "443"  
```

`10.43.0.1` is the ClusterIP service for the API server, which exposes port 443. Seems consistent across k3s installs.

## Modifying Cailco custom resources
Download the Calico custom resources yaml: `curl -o calico-custom-resources.yaml https://raw.githubusercontent.com/projectcalico/calico/v3.27.2/manifests/custom-resources.yaml`

Make sure to change the cluster and service CIDR values to match what we've defined.

Under the `Installation` resource (should be at the top of the file), add a new `calicoNetwork` section inside the `spec`, including the `linuxDataplane` field.

Example below:
```yaml
# This section includes base Calico installation configuration.
# For more information, see: https://docs.tigera.io/calico/latest/reference/installation/api#operator.tigera.io/v1.Installation
apiVersion: operator.tigera.io/v1
kind: Installation
metadata:
  name: default
spec:
  variant: Calico
  # Configures Calico networking.
  calicoNetwork:
    linuxDataplane: BPF
    # Note: The ipPools section cannot be modified post-install.
    ipPools:
    - blockSize: 26
      cidr: 10.42.0.0/16
      encapsulation: VXLANCrossSubnet
      natOutgoing: Enabled
      nodeSelector: all()

---

# This section configures the Calico API server.
# For more information, see: https://docs.tigera.io/calico/latest/reference/installation/api#operator.tigera.io/v1.APIServer
apiVersion: operator.tigera.io/v1
kind: APIServer
metadata:
  name: default
spec: {}
```

Apply the edited file: `kubectl create -f custom-resources.yaml` \
Monitor the installation with `watch kubectl get tigerastatus`. This takes a few minutes and calico itself will come up before the apiserver. \
Run `kubectl rollout status ds/calico-node -n calico-system` to verify that the deployment has finished. Output should be similar to "daemon set "calico-node" successfully rolled out".
## Check Calico has installed correctly
Confirm all pods are running now we have a CNI: `watch kubectl get pods -A`
Confirm there is one node in the cluster: `kubectl get nodes -o wide`

## Stop conflicts with kube-proxy
K3s embeds kube-proxy, which makes it hard to disable, if you don't on instal. Since both kube-proxy and eBPF are trying to interact with the cluster data flow, we must change the Felix configuration parameter `BPFKubeProxyIptablesCleanupEnabled` to false. If left, kube-proxy will write its iptables rules and Felix will try to clean them up, resulting in iptables flipping between the two.

Disable iptables cleanup: `calicoctl patch felixconfiguration default --patch='{"spec": {"bpfKubeProxyIptablesCleanupEnabled": false}}'`

## Sources
[Quickstart for Calico on K3s | Calico Documentation (tigera.io)](https://docs.tigera.io/calico/latest/getting-started/kubernetes/k3s/quickstart)
[Install in eBPF mode | Calico Documentation (tigera.io)](https://docs.tigera.io/calico/latest/operations/ebpf/install#create-the-config-map)
[Enable the eBPF dataplane | Calico Documentation (tigera.io)](https://docs.tigera.io/calico/latest/operations/ebpf/enabling-ebpf)
[How to maximize K3s resource efficiency using Calico's eBPF data plane (tigera.io)](https://www.tigera.io/blog/how-to-maximize-k3s-resource-efficiency-using-calicos-ebpf-data-plane/)

# Installing MetalLB
```bash
kubectl apply -f https://raw.githubusercontent.com/metallb/metallb/v0.14.4/config/manifests/metallb-native.yaml
```

Create the following IPAddressPool:
```yaml
apiVersion: metallb.io/v1beta1
kind: IPAddressPool
metadata:
  name: default
  namespace: metallb-system
spec:
  addresses:
  - xxx.xxx.xxx.xxx/32 # Give single external IP
```

Then, enable L2 mode instead of using BGP, as we have no control over any routers:
```yaml
apiVersion: metallb.io/v1beta1
kind: L2Advertisement
metadata:
  name: empty
  namespace: metallb-system
```

On any LoadBalancer services, add these annotations: \
`metallb.universe.tf/address-pool: default` \
`metallb.universe.tf/allow-shared-ip: "yes"` \
As there is only one IP, we don't have to specify `loadBalancerIP`.
## Sources
[Service via BGP with Metallb and Calico - Evangelista Tragni & Francesco Grimaldi, Desotech SRL (youtube.com)](https://www.youtube.com/watch?v=boOP5EzZUAk)
[Install a Kubernetes cluster on cloud servers | Hetzner Community](https://community.hetzner.com/tutorials/install-kubernetes-cluster)
[metallb/configsamples at v0.14.3 · metallb/metallb · GitHub](https://github.com/metallb/metallb/tree/v0.14.3/configsamples)
https://metallb.universe.tf/usage/#ip-address-sharing
[Kubernetes NodePort vs LoadBalancer vs Ingress? When should I use what? | by Sandeep Dinesh | Google Cloud - Community | Medium](https://medium.com/google-cloud/kubernetes-nodeport-vs-loadbalancer-vs-ingress-when-should-i-use-what-922f010849e0)

# TLS certificates & cert-manager
We need to set up some stuff for automatic TLS certificates via Let's Encrypt. Traefik can handle this for a single node, but since I'm trying to do things properly this time, we'll use cert-manager. Cert-manager can handle TLS certificates cluster-wide.

Install cert-manager: 
```bash
kubectl apply -f https://github.com/cert-manager/cert-manager/releases/download/v1.14.4/cert-manager.yaml
```

Define a new ClusterIssuer to represent Let's Encrypt's ACME-based CA with the following. We will use the http01 challenge because it's easier.
```yaml
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
 name: le-issuer
 namespace: cert-manager
spec:
 acme:
   email: webmaster@osable.net
   # We use the staging server here for testing to avoid rate limits
   # server: https://acme-staging-v02.api.letsencrypt.org/directory
   server: https://acme-v02.api.letsencrypt.org/directory
   privateKeySecretRef:
     # if not existing, it will register a new account and stores it
     name: le-issuer-account-key
   solvers:
     - http01:
         # The ingressClass used to create the necessary ingress routes
         ingress:
           class: traefik
```


# Configuring Traefik

## Static Config & Helm Chart

Create the following file at `/var/lib/rancher/k3s/server/manifests/traefik-config.yaml`.
The static configuration of Traefik on k3s can be done via helm chart config. It can be done via mounting a file at `/etc/traefik/traefik.yaml`, but then you have to manually restart Traefik anyway. K3s watches those server manifests and restarts whatever is changed, so I'm just going to use that. If there's anything in the future that is a pain to implement via the helm chart, I may consider switching. 
This will:
1) Put the required MetalLB annotations on the Traefik service to hopefully allow it to share our single external IP with whatever else will go through MetalLB.
2) Enable the file provider to allow me to create the dynamic config via file.
3) Mount this dynamic configuration (`dynamic-config.yaml`) from the ConfigMap `traefik-dynamic-config` into `/etc/traefik/dynamic-config`.
4) Point Traefik towards using the above configuraton file via additionalArguments, since `providers.file.filename` isn't in the chart's values.
5) Define the entrypoint `web` for HTTP on port 80. Trust all `X-Forwarded-*` headers in the incoming requests (proxied by CF and CF set to remove them on the edge), and redirect to HTTPS.
6) Define the entrypoint `websecure` for HTTPS on port 443. See above regarding `X-Forwarded-*`.  Note that I don't configure TLS here since we're using cert-manager.

Note that I don't configure any persistent storage. This is because the static and dynamic configs are already on the host, as are the TLS certificates as they're handled by cert-manager.


```yaml
apiVersion: helm.cattle.io/v1
kind: HelmChartConfig
metadata:
  name: traefik
  namespace: kube-system
spec:
  valuesContent: |-
    # Begin helm chart configuration
    service:
      annotations:
        metallb.universe.tf/address-pool: default
        metallb.universe.tf/allow-shared-ip: "yes"
    volumes:
    - name: traefik-dynamic-config
      mountPath: "/etc/traefik/"
      type: configMap
    # Begin static configuration
    providers: # Enable dynamic configuration file
      file:
        enabled: true
        watch: true
    additionalArguments:
    - "--providers.file.filename=/etc/traefik/dynamic-config.yaml"
    globalArguments: [] # Disable sending anonymous usage stats
    ports:
      web:
        port: 8000 # port for endpoint inside the cluster. Traefik drops all capabilities so it can't bind to ports < 1024. Default anyway, so leaving.
        exposedPort: 80 # port for outside the cluster, as expected
        protocol: TCP
        redirectTo:
          port: websecure # redirect HTTP to HTTPS
        forwardedHeaders:
          insecure: true # forward X-Forwarded-* headers from any incoming address. Proxied by CF. Maybe specify CF ranges later.
      websecure:
        port: 8443 # See above
        exposedPort: 443
        protocol: TCP
        forwardedHeaders:
          insecure: true # See above
        
```


## Dynamic configuration
The dynamic configuration will be managed by a yaml file within the ConfigMap `traefik-dynamic-config` in the kube-system namespace, as described in the sections above.

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: traefik-dynamic-config
  namespace: kube-system
data:
  dynamic-config.yaml: |
    # Empty for now.
```

To enable TLS, modify the `metadata` and `spec` of any Ingress resources which need it to include the following:
```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: xyz
  namespace: xyz-namespace
  annotations:
    cert-manager.io/cluster-issuer: "le-issuer"
spec:
  tls:
    - hosts:
        - whatever.example.com
      secretName: tls-whatever-ingress-http
```
cert-manager automatically creates a new Certificate resource for the specified domain with the given `secretName`, provisions a CertificateRequest to request a signed certificate from the ClusterIssuer specified in the annotation, and stores the certificate and private key with the same name as the secret. 
## Sources
[Traefik Configuration Documentation - Traefik](https://doc.traefik.io/traefik/getting-started/configuration-overview/#configuration-file)
[Traefik EntryPoints Documentation - Traefik](https://doc.traefik.io/traefik/routing/entrypoints/#configuration-examples)
[Setup Traefik routing in Kubernetes with Helm chart | by Stephen Cow Chau | FAUN — Developer Community 🐾](https://faun.pub/setup-traefik-routing-in-kubernetes-with-helm-chart-34008100b88f)
[Annotated Ingress resource - cert-manager Documentation](https://cert-manager.io/docs/usage/ingress/#supported-annotations)
[Automating Certificate Management in a Kubernetes Environment - NGINX](https://www.nginx.com/blog/automating-certificate-management-in-a-kubernetes-environment/)
[Secure Web Apps: Traefik Proxy, cert-manager & Let’s Encrypt](https://traefik.io/blog/secure-web-applications-with-traefik-proxy-cert-manager-and-lets-encrypt/)


Right then, our cluster should be set up. Time for our stuff.