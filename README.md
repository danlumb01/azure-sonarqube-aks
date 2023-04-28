# sonar-k8s

- Built with Azure Kubernetes Service
- Kubernetes manifest to provision a Sonarqube instance with pgsql database
- Database has it's own pod and permanent storage
- Deploys ingress and service on to Private network inside VNET


## Caveats and pre-reqs (read before deployment!)

### init container

Pods are deployed with an init container to manipulate the kernel param for `vm.max_map_count` and increase it to allow the elasticsearch component of Sonarqube to start correctly. This isn't ideal and it may be a good idea to mark only specific nodes to run Sonar containers and taint them for other pods.

### namespace
Create a sonar namespace:
```
kubectl create namespace sonar
```

### tls
Ingress is deployed with TLS, for testing purposes a self signed certificate can be used (but this will cause cert validation issues later when sonar scanners try and connect!). Creating some example certificates: 

```
# Generate cert for sonarqube ingress
openssl req -x509 -nodes -days 365 -newkey rsa:2048 \
    -out sonar-ingress-tls.crt \
    -keyout sonar-ingress-tls.key \
    -subj "/CN=sonar.example.com/O=sonar-ingress-tls"
```

### secrets
Relies on some secrets to be created in the 'sonar' namespace before deploy that define database credentials and TLS certificates to be used in the deployment. The secrets are as follows:

```
sonar-postgres-secrets # postgres admin password and postgres sonarqube user password
sonar-ingress-tls # TLS cert & key for ingress
```

Example creating the secrets above:
```
kubectl create secret generic sonar-postgres-secrets \
        --from-file=postgres-root-password=postgres-root-password.txt \
         --from-file=postgres-sonar-password=postgres-sonar-password.txt \
         --namespace sonar

kubectl create secret tls sonar-ingress-tls \
        --namespace sonar \
        --key sonar-ingress-tls.key \
        --cert sonar-ingress-tls.crt
```
### ingress
The manifest also relies on an nginx ingress, which can be installed via helm as follows:

Create helm options override file `helm-options-ingress-internal.yaml` with contents:

```
controller:
  service:
    loadBalancerIP: 10.0.0.100
    annotations:
      service.beta.kubernetes.io/azure-load-balancer-internal: "true"
```

Substitute the `loadBalancerIP` with an appropriate address from the subnet you wish to present the Ingress on.

Now install the ingress controller, passing in the file: 

```
helm install nginx-ingress-int ingress-nginx/ingress-nginx \
    --namespace sonar \
    -f helm-options-ingress-internal.yaml \
    --set controller.replicaCount=2 \
    --set controller.nodeSelector."beta\.kubernetes\.io/os"=linux \
    --set defaultBackend.nodeSelector."beta\.kubernetes\.io/os"=linux \
    --set controller.admissionWebhooks.patch.nodeSelector."beta\.kubernetes\.io/os"=linux

```

Subsequent changes to the configuration of the ingress or contens of the `helm-options-ingress-internal.yam` file can be applied with helm upgrade:

```
helm upgrade nginx-ingress-int ingress-nginx/ingress-nginx \
    --namespace sonar \
    -f helm-options-ingress-internal.yaml \
    --set controller.replicaCount=2 \
    --set controller.nodeSelector."beta\.kubernetes\.io/os"=linux \
    --set defaultBackend.nodeSelector."beta\.kubernetes\.io/os"=linux \
    --set controller.admissionWebhooks.patch.nodeSelector."beta\.kubernetes\.io/os"=linux

```

### service

Edit the service definition inside `sonar-k8s.yml` to match subnet resources in your azure environment. Config to change is indicated below:

```
apiVersion: v1
kind: Service
metadata:
  name: sonar-web-service
  namespace: sonar
  annotations: 
    service.beta.kubernetes.io/azure-load-balancer-internal: "true"
    service.beta.kubernetes.io/azure-load-balancer-internal-subnet: subnet-app-dev-001  <--- # CHANGE ME #
  labels:
    run: sonar-web
spec:
  selector:
    app: sonar-web
  type: LoadBalancer
  loadBalancerIP: 10.0.10.60 # dev app subnet      <--- # CHANGE ME #
  ports:
  - port: 9000
    protocol: TCP
```

### disable tls

To deploy the ingress without TLS, replace the sonar ingress definition with: 

```
# No TLS
apiVersion: networking.k8s.io/v1beta1
kind: Ingress
metadata:
  name: sonar-web-ingress
  namespace: sonar
  annotations:
    kubernetes.io/ingress.class: nginx
    nginx.ingress.kubernetes.io/ssl-redirect: "false"
    nginx.ingress.kubernetes.io/use-regex: "true"
    nginx.ingress.kubernetes.io/rewrite-target: /$1
spec:
  rules:
  - http:
      paths:
      - backend:
          serviceName: sonar-web-service
          servicePort: 9000
        path: /(.*)
```
