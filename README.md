# Akeyless Unified Gateway on MicroK8s

This guide demonstrates how to configure the Akeyless unified gateway on a cloud-hosted VM running MicroK8s. It provides a cost-effective way to run Kubernetes for testing or development environments.

## Why MicroK8s?

- **Cost-effective**: Running K8s on a VM is cheaper than a dedicated cluster and provides more resources (CPU, Memory, etc.)
- **Lightweight**: Uses minimal resources and can run as a single-node cluster
- **Feature-rich**: Provides a fully-featured Kubernetes environment with the same APIs, behavior, and features as production environments
- **Production-ready runtime**: Uses containerd, the same runtime commonly used in production Kubernetes environments (e.g., in managed services like GKE or AKS)
- **Flexible add-ons**: DNS, ingress, and storage are bundled into MicroK8s and enabled explicitly
- **Unified gateway support**: Supports Unified gateway in on-prem environments without having to use docker/docker-compose

## Prerequisites

- A cloud provider account (this guide uses GCP)
- A reserved static IP address
- An Ubuntu server VM

## 1. Reserve a Static IP Address

```bash
gcloud compute addresses create example-lab-base-image \
  --project=example-project-123456 \
  --region=us-central1 \
  --description="Static IP for example-lab"
```

Verify the IP address:

```bash
gcloud compute addresses describe example-lab-base-image --region=us-central1 --format="get(address)"
```

## 2. Configure VM Instance

Create an Ubuntu VM with the following ports allowed inbound
- allow inbound http/https/ssh/8000 (adjust network tags)
- adjust your service-account name, YOUR_EXTERNAL_IP

```bash
gcloud compute instances create example-lab-base-image \
    --project=example-project-123456 \
    --zone=us-central1-c \
    --machine-type=e2-standard-8 \
    --network-interface=address=YOUR_EXTERNAL_IP,network-tier=PREMIUM,stack-type=IPV4_ONLY,subnet=example-subnet \
    --maintenance-policy=MIGRATE \
    --provisioning-model=STANDARD \
    --service-account=example-sa@example-project-123456.iam.gserviceaccount.com \
    --scopes=https://www.googleapis.com/auth/devstorage.read_only,https://www.googleapis.com/auth/logging.write,https://www.googleapis.com/auth/monitoring.write,https://www.googleapis.com/auth/service.management.readonly,https://www.googleapis.com/auth/servicecontrol,https://www.googleapis.com/auth/trace.append \
    --tags=my-inbound-rules,allow-all-outbound,http-server,https-server,ssh-server \
    --create-disk=auto-delete=yes,boot=yes,device-name=example-lab-base-image-disk,image=projects/ubuntu-os-cloud/global/images/ubuntu-2404-noble-amd64-v20250130,mode=rw,size=200,type=pd-balanced \
    --no-shielded-secure-boot \
    --shielded-vtpm \
    --shielded-integrity-monitoring \
    --labels=goog-ec-src=vm_add-gcloud,always_up=false,owner=yourname,skip_shutdown=false,tf_created=false \
    --reservation-affinity=any
```

## 3. SSH into Your Instance

```bash
gcloud compute ssh --zone "us-central1-c" "example-lab-base-image" --project "example-project-123456"
```

## 4. Install & Configure MicroK8s

Set your environment variables:

```bash
# Environment Variables
EMAIL="user@example.com"
PUBLIC_IP="YOUR_EXTERNAL_IP"

# Confirm you have these variables set:
echo $EMAIL
echo $PUBLIC_IP
```

Install required packages:

```bash
sudo apt update && sudo apt upgrade -y
sudo snap install microk8s --classic
sudo snap install docker --classic
sudo groupadd docker
sudo usermod -aG docker $USER
sudo snap restart docker
newgrp docker
```

Configure kubectl:

```bash
mkdir ~/.kube
microk8s config > ~/.kube/config
sudo usermod -a -G microk8s $USER
sudo chown -f -R $USER ~/.kube
newgrp microk8s
microk8s status --wait-ready
```

Set up aliases for kubectl and helm:

```bash
sudo snap alias microk8s.kubectl kubectl
sudo snap alias microk8s.helm3 helm
```

Enable MicroK8s addons:

```bash
microk8s enable dns
microk8s enable ingress
kubectl delete ingressclass public
microk8s enable helm3
microk8s enable cert-manager --set crds.enabled=true --set global.leaderElection.namespace=cert-manager
```

Install kubectx and configure:

```bash
sudo snap install kubectx --classic
echo 'export KUBECONFIG=/var/snap/microk8s/current/credentials/client.config' >> ~/.bashrc

# Add kubectl aliases
curl -sSL https://raw.githubusercontent.com/ahmetb/kubectl-aliases/master/.kubectl_aliases -o ~/.kubectl_aliases && grep -qxF '[ -f ~/.kubectl_aliases ] && source ~/.kubectl_aliases' ~/.bashrc || echo '[ -f ~/.kubectl_aliases ] && source ~/.kubectl_aliases' >> ~/.bashrc
export KUBECONFIG=/var/snap/microk8s/current/credentials/client.config
source ~/.bashrc
```

Create namespaces:

```bash
k create namespace my-apps
k create namespace k8sinjector
```

## 5. Install and Configure NGINX & Cert Manager

Configure Ingress with your public IP:

```bash
cat << EOF >| nginx-ingress-service.yaml
apiVersion: v1
kind: Service
metadata:
  name: nginx-ingress-microk8s-controller
  namespace: ingress
spec:
  type: LoadBalancer
  selector:
    name: nginx-ingress-microk8s
  externalIPs:
    - $PUBLIC_IP # Replace with your reserved static external IP
  ports:
    - protocol: TCP
      port: 80
      targetPort: 80
      name: http
    - protocol: TCP
      port: 443
      targetPort: 443
      name: https
EOF

kubectl apply -f nginx-ingress-service.yaml
```

Set up Let's Encrypt certificate issuer:

```bash
cat << EOF >| lets-encrypt-prod-issuer.yml
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: letsencrypt-cluster-issuer
spec:
  acme:
    server: https://acme-v02.api.letsencrypt.org/directory
    email: $EMAIL
    privateKeySecretRef:
      name: letsencrypt-private-key
    solvers:
    - http01:
        ingress:
          ingressClassName: nginx
EOF

kubectl apply -f lets-encrypt-prod-issuer.yml
```

Add Helm repositories:

```bash
helm repo add akeyless https://akeylesslabs.github.io/helm-charts
helm repo add bitnami https://charts.bitnami.com/bitnami
helm repo update
```

Verify the Public IP Assignment:

```bash
kubectl get services -n ingress
```

## 6. Install the Akeyless Unified Gateway Helm Chart

Get the default Helm values:

```bash
helm show values akeyless/akeyless-gateway > gateway-values.yaml
```

After customizing the values, install the Unified Gateway:

```bash
helm install akl-gcp-gw akeyless/akeyless-gateway -n akeyless -f gateway-values.yaml
```

## 7. Sample Unified Gateway Helm Values File with NGINX

Here's a sample `gateway-values.yaml` configuration:

```yaml
############
## Global ##
############
globalConfig:
  gatewayAuth:
    ## https://docs.akeyless.io/docs/gateway-chart#authentication
    gatewayAccessId: p-abc123
    gatewayAccessType: gcp
    gatewayCredentialsExistingSecret:
  ## See docs for examples https://docs.akeyless.io/docs/gateway-chart#gateway-admins
  allowedAccessPermissions:
    - name: Administrators
      access_id: p-samlid
      sub_claims:
        email:
          - user@example.com
      permissions:
        - admin
  ## https://docs.akeyless.io/docs/gateway-chart#access-permissions
  ##
  allowedAccessPermissionsExistingSecret:
  ## will appear. For more information: https://docs.akeyless.io/docs/remote-access-setup-k8s#configuration
  ##
  authorizedAccessIDs:
  ## By default, this secret is named <deployment>-cache-encryption-key, unless a custom name has been specified.
  ##
  serviceAccount:
    create: false
    serviceAccountName:
    annotations:
  ## This is the actual name of the cluster as in account/access-id/clusterName
  ##
  clusterName: my-microk8s-gcp-gw
  ## This is the vanity display name of the cluster
  ##
  initialClusterDisplayName: my-microk8s-gcp-gw
  ## The key which is used to encrypt the Gateway configuration.
  ## If left empty - the account's default key will be used.
  ## This key can be determined on cluster bringup only and cannot be modified afterwards
  ##
  configProtectionKeyName:
  ## Use k8s secret to set the CF, the k8s secret must include the key: customer-fragments
  ## See docs for examples https://docs.akeyless.io/docs/advanced-chart-configuration#customer-fragment
  ##
  customerFragmentsExistingSecret:
  ## See docs for examples https://docs.akeyless.io/docs/advanced-chart-configuration#tls-configuration
  ##
  TLSConf:
    enabled: false
    ## Specifies an existing secret for tls-certificate:
    tlsExistingSecret:
  ## Telemetry Metrics see docs for examples https://docs.akeyless.io/docs/telemetry-metrics-k8s
  ##
  metrics:
    enabled: false
    ## Existing secret for metrics must include:
    ## - otel-config.yaml (base64) secret
    ##
    metricsExistingSecret:
  ## Linux system HTTP Proxy
  httpProxySettings:
    http_proxy: ""
    https_proxy: ""
    no_proxy: ""
  # env: []
  ## https://docs.akeyless.io/docs/advanced-chart-configuration#cache-configuration
  ##
  clusterCache:
    ## In case Cache is enabled in the Gateway, and the encryptionKeyExistingSecret parameter has a value
    ## Akeyless will use this specified encryption key and store it securely within Akeyless Gateway.
    ## If the encryptionKeyExistingSecret parameter is empty or not specified,
    ## Akeyless will automatically generate a new encryption key and a new ServiceAccount for K8s.
    ## for more information: https://docs.akeyless.io/docs/advanced-chart-configuration#cache-configuration
    ##
    encryptionKeyExistingSecret:
    # Enable/Disable TLS  between the Gateway and the cluster cache service
    # using generated certificates and keys
    enableTls: false
    ## The resources limits for the redis cluster cache
    ##
    resources:
      limits:
        # cpu: 500m
        memory: 2Gi
      requests:
        cpu: 250m
        memory: 256Mi
####################################################
##          Default values for Gateway            ##
####################################################
gateway:
  ## Default values for akeyless-gateway.
  deployment:
    annotations: {}
    labels: {}
    replicaCount: 2
    image:
      #  repository: akeyless/base
      #   Alternative mirror registry
      #  repository: docker.registry-2.akeyless.io/base
      pullPolicy: IfNotPresent
    # Place here any pod annotations you may need
    pod:
      annotations: {}
    affinity:
      enabled: false
      data:
    #    nodeAffinity:
    #      requiredDuringSchedulingIgnoredDuringExecution:
    #        nodeSelectorTerms:
    #          - matchExpressions:
    #              - key: kubernetes.io/arch
    #                operator: In
    #                values:
    #                  - amd64
    # ref: https://kubernetes.io/docs/concepts/scheduling-eviction/assign-pod-node/#node-affinity
    nodeSelector:
    #     iam.gke.io/gke-metadata-server-enabled: "true"
    securityContext:
      enabled: false
      fsGroup: 0
      runAsUser: 0
    containerSecurityContext: {}
    ## Remove the {} and add any needed values to your SecurityContext
    ##
    #  runAsUser: 0
    #  seccompProfile:
    #    type: RuntimeDefault
    livenessProbe:
      initialDelaySeconds: 60
      periodSeconds: 30
      failureThreshold: 10
    readinessProbe:
      initialDelaySeconds: 60
      periodSeconds: 10
      timeoutSeconds: 5
  service:
    ## Remove the {} and add any needed annotations regarding your LoadBalancer implementation
    ##
    annotations: {}
    labels: {}
    type: ClusterIP
    ## Gateway service port
    ##
    port: 8000
  ## Configure the ingress resource that allows you to access the
  ## akeyless-api-gateway installation. Set up the URL
  ## ref: https://kubernetes.io/docs/concepts/services-networking/ingress/
  ##
  ingress:
    ## Set to true to enable ingress record generation
    enabled: true
    ## A reference to an IngressClass resource
    ## ref: https://kubernetes.io/docs/concepts/services-networking/ingress/#deprecated-annotation
    ingressClassName: nginx
    labels: {}
    annotations:
      cert-manager.io/cluster-issuer: letsencrypt-cluster-issuer
      nginx.ingress.kubernetes.io/ssl-redirect: "true"
      nginx.ingress.kubernetes.io/proxy-connect-timeout: "3600"
      nginx.ingress.kubernetes.io/proxy-read-timeout: "3600"
      nginx.ingress.kubernetes.io/proxy-send-timeout: "3600"
      nginx.ingress.kubernetes.io/proxy-buffer-size: "8k"
      nginx.ingress.kubernetes.io/proxy-buffers-number: "4"
      # Large Header Support ----=v
      #nginx.ingress.kubernetes.io/client-body-buffer-size: 64k
      #nginx.ingress.kubernetes.io/client-header-buffer-size: 100k
      #nginx.ingress.kubernetes.io/http2-max-header-size: 96k
      #nginx.ingress.kubernetes.io/large-client-header-buffers: 4 100k
      #nginx.ingress.kubernetes.io/server-snippet: | # this is where the magic happens pt2
      #  client_header_buffer_size 100k;
      #  large_client_header_buffers 4 100k;
    rules:
      - servicePort: gateway
        hostname: "example-microk8s.example.com"
    ## Path for the default host
    path: /
    ## Ingress Path type the value can be ImplementationSpecific, Exact or Prefix
    pathType: ImplementationSpecific
    ## Enable TLS configuration for the hostname defined at ingress.hostname parameter
    ## TLS certificates will be retrieved from a TLS secret with name: {{- printf "%s-tls" .Values.gateway.ingress.hostname }}
    ## or a custom one if you use the tls.existingSecret parameter
    ##
    tls: true
    #tls:
    #  - hosts:
    #    - example-microk8s.example.com
    #    secretName: example-microk8s.example.com-tls
    #  existingSecret: name-of-existing-secret
    ## Set this to true in order to add the corresponding annotations for cert-manager and secret name
    certManager: true
  resources: {}
  hpa:
    ## Set the below to false in case you do not want to add Horizontal Pod AutoScaling
    ## Note that metrics server must be installed for this to work:
    ## https://github.com/kubernetes-sigs/metrics-server
    ##
    enabled: false
    minReplicas: 1
    maxReplicas: 10
    cpuAvgUtil: 70
    memAvgUtil: 70
    annotations: {}
######################################################
## Default values for akeyless-secure-remote-access ##
######################################################
## If you are only using Akeyless Gateway, ignore this section
##
sra:
  ## Enable secure-remote-access. Valid values: true/false.
  ## For more information on a Quick Start guide for Remote Access <https://docs.akeyless.io/docs/remote-access-quick-start-guide>
  ## Or setup SRA on K8s <https://docs.akeyless.io/docs/remote-access-setup-k8s>
  enabled: false
  image:
    ##  Default image repository is: akeyless/zero-trust-bastion
    ##
    pullPolicy: IfNotPresent
    #  tag: latest
  env: []
  ## The below section is for the Remote Access Web app
  ##
  webConfig:
    deployment:
      annotations: {}
      labels: {}
    replicaCount: 1
    ## Persistence Volume is used to store RDP recordings when it is configured to save recordings locally
    ## Akeyless requires data persistence to be shared within all pods in the cluster
    ## accessMode: ReadWriteMany
    ## Make sure to change the below values according to your environment except for the hostPath values
    ## see docs for more information <https://docs.akeyless.io/docs/remote-access-setup-k8s#configuration>
    ##
    persistence:
      volumes: {}
      #  volumes:
      #  - name: akeyless-data
      #    storageClassName: efs-zero-trust-bastion-sc
      #   #  storageClassDriver: efs.csi.aws.com
      #    size: 100Mi
      #    annotations:
      #     volume.beta.kubernetes.io/storage-class: ""
    livenessProbe:
      initialDelaySeconds: 15
      periodSeconds: 30
      failureThreshold: 10
    readinessProbe:
      initialDelaySeconds: 15
      periodSeconds: 30
      timeoutSeconds: 5
    resources:
      requests:
        cpu: 1
        memory: 2G
    hpa:
      ## Set the below to false in case you do not want to add Horizontal Pod AutoScaling to the Deployment
      ## If HPA is enabled resources requests must be set
      ##
      enabled: false
      minReplicas: 1
      maxReplicas: 10
      cpuAvgUtil: 70
      memAvgUtil: 70
  ## The below section is for the Remote Access SSH app
  ## For more information: <https://docs.akeyless.io/docs/remote-access-advanced-configuration-k8s#ssh-configuration>
  ##
  sshConfig:
    replicaCount: 1
    ## This is a required RSA Public Key for your Akeyless SSH Cert Issuer
    ## See docs for examples <https://docs.akeyless.io/docs/remote-access-setup-k8s#ssh--config>
    ##
    CAPublicKey:
    # CAPublicKey: |
    sshHostKeysPath:
    annotations: {}
    labels: {}
    nodeSelector:
    #  iam.gke.io/gke-metadata-server-enabled: "true"
    securityContext:
      enabled: false
      fsGroup: 0
      runAsUser: 0
    service:
      ## Remove the {} and add any needed annotations regarding your LoadBalancer implementation
      ##
      annotations: {}
      labels: {}
      type: LoadBalancer
      port: 22
    livenessProbe:
      failureThreshold: 5
      periodSeconds: 30
      timeoutSeconds: 5
    readinessProbe:
      initialDelaySeconds: 20
      periodSeconds: 10
      timeoutSeconds: 5
    resources:
      requests:
        cpu: 1
        memory: 2G
    hpa:
      ## Set the below to true only when using a shared persistent storage (defined at .persistence.volumes)
      ## If HPA is enabled resources requests must be set
      ##
      enabled: false
      minReplicas: 1
      maxReplicas: 10
      cpuAvgUtil: 70
      memAvgUtil: 70
```

## 8. Large Header Support

You may see these errors when implementing K8s auth and other integrations.

```
failed to create k8s auth config: Failed to create k8s auth config. Status 400 Bad Request. Unexpected error reply object, Status: 400 Bad Request. Body: <html>
<head><title>400 Request Header Or Cookie Too Large</title></head>
<body>
<center><h1>400 Bad Request</h1></center>
<center>Request Header Or Cookie Too Large</center>
<hr><center>nginx</center>
</body>
</html>
```

You'll need to modify the NGINX ConfigMap:

```bash
kubectl get configmap -n ingress
kubectl describe configmap nginx-load-balancer-microk8s-conf -n ingress
kubectl edit configmap nginx-load-balancer-microk8s-conf -n ingress

#add this to the very bottom of the config
data:
  large-client-header-buffers: "4 64k"
  client-header-buffer-size: "64k"
  http2-max-header-size: "64k"
```

## 9. Configure Shared Storage Class and Persistent Volume

Switch to the my-apps namespace:

```bash
kubens my-apps
```

Create a storage class:

```bash
cat << EOF > storageclass.yml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: shared-local-storage
provisioner: kubernetes.io/no-provisioner
volumeBindingMode: WaitForFirstConsumer
EOF

kubectl apply -f storageclass.yml
```

Create a persistent volume:

```bash
cat << EOF > pv.yml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: pv-shared
spec:
  capacity:
    storage: 50Gi  # Adjust as needed
  accessModes:
    - ReadWriteOnce
  persistentVolumeReclaimPolicy: Retain
  storageClassName: shared-local-storage
  hostPath:
    path: /mnt/data/shared  # Base directory for shared use
EOF

kubectl apply -f pv.yml
```

Prepare the host path directory:

```bash
sudo mkdir -p /mnt/data/shared
sudo chmod -R 777 /mnt/data/shared
```

When installing applications like a database, set the storage class and size:

```bash
helm install postgres-db oci://registry-1.docker.io/bitnamicharts/postgresql \
  -n my-apps \
  --set primary.persistence.storageClass="shared-local-storage" \
  --set primary.persistence.size="10Gi"
```

## Conclusion

You now have a functional Akeyless Unified Gateway running on MicroK8s. This setup provides a cost-effective way to run Kubernetes for your security gateway needs while maintaining the flexibility and features of a production-grade Kubernetes environment.

For more information and documentation, visit the [Akeyless documentation site](https://docs.akeyless.io/).
