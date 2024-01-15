# Istio Lab

In this lab, our primary steps involve:

1. **Istio Installation:**
   - Initially, we'll install Istio within our k3d cluster, setting up the foundation for service mesh capabilities.

2. **install istioctl**
   - To install istioctl on Linux, you can use the following steps. These instructions assume you are using a Bash shell.
    
    ```bash
    # Download the latest release of istioctl
    curl -L https://istio.io/downloadIstio | sh -

    # Navigate to the Istio directory
    cd istio-<VERSION>

    # Add the istioctl to your PATH
    export PATH=$PWD/bin:$PATH

    # Verify the installation
    istioctl version
    ```
    - Make sure to replace <VERSION> with the specific version of Istio you downloaded.
    - If you want to make the change permanent, you can add the export line to your shell profile configuration file.

    ```bash
    echo 'export PATH=$PWD/bin:$PATH' >> ~/.bashrc
    ``` 

3. **Web Application Deployment:**
   - Following Istio installation, we'll deploy our web application into the cluster, preparing it for enhanced networking and traffic management.

4. **Istio Configuration:**
   - With the application in place, we'll proceed to configure Istio components, including sidecar injection, gateway setup, virtual service definition, destination rule establishment, and fine-tuning traffic behavior through strategies like splitting and shifting.

These steps will enable us to leverage Istio's powerful features for improved service communication and traffic control within our Kubernetes environment.

## install K3D
Install K3D with the following command:

```bash
curl -s https://raw.githubusercontent.com/rancher/k3d/main/install.sh | bash
```

### Create k3d cluster with one master and 2 worker nodes:

```bash
k3d cluster create my-cluster --agents 2
```

## install istio using helm:

Before installing Istio using Helm, make sure to add the official Istio Helm chart repository. Use the following command:

```bash
helm repo add istio https://istio.io/latest/docs/ops/distribution/helm/
```

After adding the repository, you can proceed with installing Istio:

```bash
helm install istio-lab istio/istio
```

Certainly! I'll integrate the provided YAML code into the updated instructions:

## Enable Automatic Sidecar Injection

For automatic sidecar injection, label the default namespace:

```bash
kubectl label namespace default istio-injection=enabled
```

Verify that the label has been applied:

```bash
kubectl get ns -Listio-injection
```

## Deploy Web Application

The web application consists of two parts: `web frontend` and `customers backend`.

### Kubernetes YAML Files for `Web Frontend`:

Create a file named `web-frontend.yaml` with the following content:

```yaml
---
# Service Account for the web-frontend application
apiVersion: v1
kind: ServiceAccount
metadata:
  name: web-frontend

---
# Deployment configuration for the web-frontend application
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web-frontend
  labels:
    app: web-frontend
spec:
  replicas: 1
  selector:
    matchLabels:
      app: web-frontend
  template:
    metadata:
      labels:
        app: web-frontend
        version: v1
    spec:
      # Assign the ServiceAccount to the Pod
      serviceAccountName: web-frontend
      containers:
      - name: web
        # Docker image for the web-frontend application
        image: gcr.io/tetratelabs/web-frontend:1.0.0
        # Always pull the latest image
        imagePullPolicy: Always
        ports:
        - containerPort: 8080
        env:
        - name: CUSTOMER_SERVICE_URL
          # URL for the customer service within the Kubernetes cluster
          value: "http://customers.default.svc.cluster.local"

---
# Service configuration for the web-frontend application
kind: Service
apiVersion: v1
metadata:
  name: web-frontend
  labels:
    app: web-frontend
spec:
  selector:
    app: web-frontend
  ports:
  - port: 80
    name: http
    # Expose the containerPort 8080 of the Pods
    targetPort: 8080
```

Apply the YAML file to your Kubernetes cluster:

```bash
kubectl apply -f web-frontend.yaml
```

### Kubernetes YAML Files for `Customers Backend`:

Create a file named `customers.yaml` with the following content:

```yaml
---
# Service Account for the customers application
apiVersion: v1
kind: ServiceAccount
metadata:
  name: customers

---
# Deployment configuration for the customers application (version: v1)
apiVersion: apps/v1
kind: Deployment
metadata:
  name: customers-v1
  labels:
    app: customers
    version: v1
spec:
  replicas: 1
  selector:
    matchLabels:
      app: customers
      version: v1
  template:
    metadata:
      labels:
        app: customers
        version: v1
    spec:
      # Assign the ServiceAccount to the Pod
      serviceAccountName: customers
      containers:
      - image: gcr.io/tetratelabs/customers:1.0.0
        # Always pull the latest image
        imagePullPolicy: Always
        name: svc
        ports:
        - containerPort: 3000

---
# Service configuration for the customers application
kind: Service
apiVersion: v1
metadata:
  name: customers
  labels:
    app: customers
spec:
  selector:
    app: customers
  ports:
  - port: 80
    name: http
    # Expose the containerPort 3000 of the Pods
    targetPort: 3000
```

Apply the YAML file to your Kubernetes cluster:

```bash
kubectl apply -f customers.yaml
```

Validate your pods:

```bash
kubectl get pod
```

## Istio Ingress Gateway

View the Istio Ingress Gateway pod in the `istio-system` namespace:

```bash
kubectl get pod -n istio-system
```

Note the external IP address for the load balancer:

```bash
kubectl get svc -n istio-system
```

Assign it to an environment variable:

```bash
export GATEWAY_IP=$(kubectl get svc -n istio-system istio-ingressgateway -ojsonpath='{.status.loadBalancer.ingress[0].ip}')
```

## Configuring Ingress

### Create a Gateway Resource

Create a `Gateway` resource using the following specification:

```yaml
---
# This YAML defines an Istio Gateway named frontend-gateway for handling incoming HTTP traffic.

apiVersion: networking.istio.io/v1alpha3
kind: Gateway
metadata:
  name: frontend-gateway
spec:
  # Selector for the Istio Ingress Gateway
  selector:
    istio: ingressgateway
  # List of servers and their configurations
  servers:
  - 
    # Port configuration for HTTP
    port:
      number: 80
      name: http
      protocol: HTTP
    # Hosts to which this gateway is applicable
    hosts:
    - "*"
```

Apply the gateway resource to your cluster:

```bash
kubectl apply -f gateway.yaml
```

Attempt an HTTP request:

```bash
curl -v http://$GATEWAY_IP/
```

It should return a **404: not found**.

### Create a VirtualService Resource For `web-frontend`

Create a `VirtualService` resource using the following specification:

```yaml
---
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: web-frontend
spec:
  hosts:
    - "*"  # Matches all hosts
  gateways:
    - frontend-gateway  # Associates with the frontend-gateway
  http:
    - route:
        - destination:
            host: web-frontend.default.svc.cluster.local
            port:
              number: 80  # Routes to port 80 of the specified host
```

Apply the VirtualService resource to your cluster:

```bash
kubectl apply -f web-frontend-virtualservice.yaml
```

List virtual services in the default namespace:

```bash
kubectl get virtualservice
```

# Traffic Management
Here we have two section traffic splitting, and traffic shifting.

## Traffic Splitting

**Version 2** of the customers service has been developed, and it's time to deploy it to production. Whereas **Version 1** returned a list of customer names, **Version 2** also includes each customer's city.

## Deploying customers (V1)

We are ready to deploy the new service, but we're holding off on directing traffic to it for now.

For a structured approach, let's distinctly handle the deployment of the new service and the traffic directing tasks.

The customers service is labeled with `app=customers`.

To validate the labels in the `customers.yaml` file, use the following command:

```bash
kubectl get pod -Lapp,version
```

Note the selector on the `customers` service in the output of the following command:

```bash
kubectl get svc customers -o wide
```

If we were to deploy only v2, the selector would match both versions.


## DestinationRules for `customers backend`

We can inform Istio that two distinct `subsets` of the `customers` service exist, and we can use the `version` label as the discriminator.

create a file named `customers-destinationrule.yaml` and add the following yaml code there:

```yaml
---
apiVersion: networking.istio.io/v1alpha3
kind: DestinationRule
metadata:
  name: customers
spec:
  host: customers.default.svc.cluster.local
  subsets:
    - name: v1
      labels:
        version: v1
    - name: v2
      labels:
        version: v2
```
1. Apply the above destination rule to the cluster.
2. Verify that it's been applied.
   ```bash
    kubectl get destinationrule
   ```

## VirtualServices for `customers backend`

Armed with two distinct destinations, the `VirtualService` Custom Resource allows us to define a routing rule that sends all traffic to the v1 subset.

create a file named `customers-virtualservice-v1.yaml`

and add the following yaml code there:
```yaml
---
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: customers
spec:
  hosts:
  - customers.default.svc.cluster.local
  http:
  - route:
    - destination:
        host: customers.default.svc.cluster.local
        subset: v1
```
Above, note how the route specifies subset v1.

1. Apply the virtual service to the cluster.
2. Verify that it's been applied.
   ```bash
   kubectl get virtualservice 
   ```
We can now safely proceed to deploy v2, without having to worry about the new workload receiving traffic.

## Finally deploy customers (v2)
create a file named `customers-v2.yaml`.

Apply the following Kubernetes deployment to the cluster for creating `version-2` of our `customer-backend` deployment:

```yaml
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: customers-v2
  labels:
    app: customers
    version: v2
spec:
  replicas: 1
  selector:
    matchLabels:
      app: customers
      version: v2
  template:
    metadata:
      labels:
        app: customers
        version: v2
    spec:
      serviceAccountName: customers
      containers:
        - image: gcr.io/tetratelabs/customers:2.0.0
          imagePullPolicy: Always
          name: svc
          ports:
            - containerPort: 3000
```
### Note:
Check that traffic routes strictly to v1.
1. Generate some traffic, by refreshing the web page few times.
2. Open a separate terminal and launch the Kiali dashboard.

    ```bash
    istioctl dashboard kiali
    ```
    - Take a look at the graph, and select the `default` namespace. The graph should show `all` traffic going to `v1`.

## Route to customers, (v2)
We wish to proceed with caution. Before customers can see version 2, we want to make sure that the service functions properly.

### Expose "debug" traffic to v2
create a file called `customers-vs-debug.yaml`, and add the following yaml code into it. 

```yaml
---
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: customers
spec:
  hosts:
  - customers.default.svc.cluster.local
  http:
  - match:
    - headers:
        user-agent:
          exact: debug
    route:
    - destination:
        host: customers.default.svc.cluster.local
        subset: v2
  - route:
    - destination:
        host: customers.default.svc.cluster.local
        subset: v1
```
- We are telling Istio to check an HTTP header: if the `user-agent` is set to `debug`, route to `v2`, otherwise route to `v1`.

-  Open a new terminal and apply the above resource to the cluster; it will overwrite the currently defined VirtualService as both yaml files use the same resource name.
    ```bash
    kubectl apply -f customers-vs-debug.yaml
    ```

#### Test the debug VS
Open a browser and visit the application.

We can tell v1 and v2 apart in that v2 displays not only customer names but also their city (in two columns).

The `user-agent` header can be included in a request by the following command:

```bash
curl -H "user-agent: debug" http://$GATEWAY_IP
```
**Note:** If you refresh the page a good dozen times and then wait ~15-30 seconds, you should see some of that v2 traffic appear in **Kiali**.

## Now Canary Deployment for our (V2) customers backend

Well, v2 looks good; we decide to expose the new version to the public, but we're still prudent.

Start by siphoning 10% of traffic over to v2.

create a file named `customers-vs-canary.yaml` and add following yaml code there.

```yaml
---
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: customers
spec:
  hosts:
  - customers.default.svc.cluster.local
  http:
  - route:
    - destination:
        host: customers.default.svc.cluster.local
        subset: v2
      weight: 10
    - destination:
        host: customers.default.svc.cluster.local
        subset: v1
      weight: 90
```
Above, note the weight field specifying 10 percent of traffic to subset v2. Kiali should now show traffic going to both v1 and v2.

- Apply the above resource.
    ```bash
    kubectl apply -f customers-vs-canary.yaml
    ```
- In your browser: undo the injection of the user-agent header, and refresh the page a bunch of times.

- In Kiali, under the Display pulldown menu, you can turn on "Traffic Distribution", to view the relative percentage of traffic sent to each subset.

- Most of the requests still go to v1, but some (10%) are directed to v2.

If all looks good, up the percentage from 90/10 to, say 50/50.

If everything goes well , then we should be switch all traffic over to v2.

## Traffic Shifting

Finally, switch all traffic over to v2.

create a file called `customers-virtualservice-final.yaml`, and add following yaml code there.

```yaml
---
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: customers
spec:
  hosts:
  - customers.default.svc.cluster.local
  http:
  - route:
    - destination:
        host: customers.default.svc.cluster.local
        subset: v2
```
- now apply the above file:
    ```bash
    kubectl apply -f customers-virtualservice-final.yaml
    ```

After applying the above resource, go to your browser and make sure all requests land on v2 (two-column output). Within a minute or so, the Kiali dashboard should also reflect the fact that all traffic is going to the customers v2 service.

## Acknowledgment
I extend my sincere gratitude to all the readers who have dedicated their valuable time and exhibited patience in exploring this content. Your commitment to learning and understanding is truly appreciated.
