# Install-k3s-cluster

How I started

I’ve selected 3 HP ProDesk 600 G2 computers as the nodes. One of the nodes is the control plane node, the other two are worker nodes.

I’ve selected mini computers instead of Raspberry PI’s because of the processor architecture (x86 instead of ARM) and the possibility to expand resources. I was inspired by the Tiny Mini Micro project.

Initially, I installed Ubuntu 22.04 LTS as operating system and Kubernetes 1.26 on it (using Kubeadm). The purpose was to prepare for my CKA exam (which I’ve passed).

After passing the exam, I wanted a different setup, existing of the following components:

    K3S as Kubernetes distribution
    MetalLB as load balancer
    Rancher as cluster management
    Traefik as ingress controller
    Cert-Manager as certificate manager

During the process, I found a lot of articles, but a lot of them weren’t very clear (especially for K3S). I hope to provide a clear guide for other engineers with the same challenge.
K3S installation

To install k3s I chose to use shell scripting, because I don’t need a lot of options to install (only exclude Traefik installation).
As mentioned earlier, I have a 3 node clusters with one control plane and 2 workers. During the following steps, the node token needs to be retrieved from the control plane node, in order to join the worker nodes.

The used scripts are shown below.
Install k3s server

SSH into the control plane server and execute the script beneath.


curl -sfL https://get.k3s.io | INSTALL_K3S_EXEC=”server --disable traefik 
--disable servicelb” sh -


Retrieve token

Retrieve the node token from the control plane, which is needed further in the process to join the worker nodes.


cat /var/lib/rancher/k3s/server/node-token

 
Retrieve certificate

Retrieve the certificate in order to communicate with the cluster.
Save this on your local machine in ~/.kube/<your config file>.


cat /etc/rancher/k3s/k3s.yaml


Add worker node

SSH into your worker node(s) and apply the script below. This will add the node as worker node to the cluster.


curl -sfL https://get.k3s.io | K3S_URL=https://<Contol Plane IP>:6443 
K3S_TOKEN=<YourToken> sh -

 
Load certificate

Once the steps before are applied, you can execute the following script to load the Kubernetes config.


export KUBECONFIG=/.kube/<your config file>


Now you should be able to use the cluster.

Check this by executing the following command.


kubectl get nodes


The result is a list of the available nodes.

k3s

MetalLB installation

Because I wanted to access all applications, running on the cluster, via one endpoint, I’ve added MetalLB to the cluster. To add it I’ve used the default manifests for version 0.11. To do this, I’ve used the default install scripts. To configure, I’ve used the layer 2 configuration scripts. Select a range of IP-addresses which isn’t already in use.

MetalLB
Rancher installation

The rancher installation was straight forward. I’ve used Helm templates to install Rancher. If you would like to use the options in the helm values file, please have a look at this website.
Helm install


Add repository


helm repo add rancher-latest https://releases.rancher.com/
server- charts/latest

Install helm chart


helm install rancher rancher-latest/rancher \
  --namespace cattle-system \
  --set hostname=<Your Domain Name>
  --set bootstrapPassword=<Your password>

When done, you should see that all deployments in the cattle-system namespace (which is the Rancher default namespace) are running. After that you can access the dashboard, with the earlier provided hostname. If you didn’t the provide a hostname, you can access it by setting up a port forward using the following command.


kubectl port-forward svc/rancher 8443:443 -n cattle-system


rancher-cluster-dashboard

 

Traefik installation

 

The Traefik installation was also pretty straight forward. I’ve used the helm template with an additional values file. These values are necessary to access the dashboard.
Helm install

Add repository

helm repo add traefik https://helm.traefik.io/traefik

 

Create the values file with the following content.


dashboard:
 enabled: true
 domain: traefik.<Your Domain Name>
rbac:
 enabled: true


Install the helm chart, in the traefik namespace, with the configuration from the values file. If the namespace doesn’t exist, create it.


helm install traefik traefik/traefik -n traefik -f values.yaml

When done, you should see that all deployments in the traefik namespace (which I’ve created as the Traefik namespace) are running. After that you can access the dashboard, with the earlier provided hostname. If you didn’t the provide a hostname, you can access it by setting up a port forward using the following command.


kubectl port-forward deploy/traefik 9000 -n traefik



treafik

 
Cert-Manager

Because I want automatic certificate renewal I needed a certificate manager.Cert-Manager is the default certificate manager for Kubernetes. I’ve installed Cert-Manager using helm templates. To install it I’ve used the commands you’ll find below.
Install Cert-Manager

Add helm repository


helm repo add jetstack https://charts.jetstack.io

Install cert manager with the following script.


helm install cert-manager jetstack/cert-manager \
  --namespace cert-manager \
  --create-namespace \
  --version v1.5.1


When Cert-Manager is installed, a ClusterIssuer or Issuer is needed to help with the provisioning of certificates. To do this with self signed certificates, a simple yaml file needs to be applied (which is shown below).


apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: selfsigned
spec:
  selfSigned: {}

 

If you wish to use CA signed certificates, the ClusterIssuer or Issuer should be configured differently.

cert-manager
Test application

To test if the environment works, I’ve created a simple test deployment. This test deployment consist of:
- Namespace
- Deployment
- Service
- Ingress
- Certificate

Beneath the yaml files are shown which I’ve used.

Namespace


apiVersion: v1
kind: Namespace
metadata:
  name: test

 

Deployment


apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx
  namespace: test
spec:
  replicas: 1
  selector:
    matchLabels:
      # manage pods with the label app: nginx
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx
        ports:
        - containerPort: 80


Service


apiVersion: v1
kind: Service
metadata:
  name: nginx-svc
  namespace: test
spec:
  ports:
  - name: http
    port: 80
    targetPort: 80
  - name: https
    port: 443
    targetPort: https
  selector:
    app: nginx

 

Ingress


apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: nginx-ingress
  namespace: test
  annotations:
    kubernetes.io/ingress.class: traefik
    cert-manager.io/cluster-issuer: selfsigned
    traefik.ingress.kubernetes.io/router.tls: "true"
spec:
  rules:
  - host: test.<Your Domain Name>
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: nginx-svc
            port:
              number: 80
  tls:
  - hosts:
    - test.<Your Domain Name>
    secretName: test.<Your Domain Name>

 

Certificate


apiVersion: cert-manager.io/v1
kind: Certificate
metadata:
  name: test.<Your Domain Name>
  namespace: test
spec:
  commonName: test.<Your Domain Name>
  secretName: test.<Your Domain Name>
  dnsNames:
    - test.<Your Domain Name>
  issuerRef:
    name: selfsigned
    kind: ClusterIssuer

 

When deployed, you should be able to reach the website in your browser. To do this at my local system, I’ve made an addition to my hosts file, which looks like the example below. In the example replace the IP with the IP from your MetalLB and <Your Domain Name> with your domain name.


10.0.0.1 test.<Your Domain Name>

 

You should see a website with the following content:
Note! because a self signed certificate is used, the certificate should be trusted.
 
Summary

When the steps are followed correctly, you should have your K3S cluster running.

Thanks for reading, and I wish you a lot of fun while creating your own cluster!

Share your experience here below!
