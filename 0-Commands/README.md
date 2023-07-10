# Commands


**Create an NGINX Pod**

`kubectl run nginx --image=nginx`

**Generate POD Manifest YAML file (-o yaml). Don't create it(--dry-run)**

`kubectl run nginx --image=nginx --dry-run=client -o yaml`

**Create a deployment**

`kubectl create deployment --image=nginx nginx`

**Generate Deployment YAML file (-o yaml). Don't create it(--dry-run)**

`kubectl create deployment --image=nginx nginx --dry-run=client -o yaml`

**Generate Deployment YAML file (-o yaml). Don’t create it(–dry-run) and save it to a file.**

`kubectl create deployment --image=nginx nginx --dry-run=client -o yaml > nginx-deployment.yaml`

**Make necessary changes to the file (for example, adding more replicas) and then create the deployment.**

`kubectl create -f nginx-deployment.yaml`

  
kubectl get pod webapp -o yaml > my-new-pod.yaml

**OR**

**In k8s version 1.19+, we can specify the --replicas option to create a deployment with 4 replicas.**

`kubectl create deployment --image=nginx nginx --replicas=4 --dry-run=client -o yaml > nginx-deployment.yaml`

#### Service

**Create a Service named redis-service of type ClusterIP to expose pod redis on port 6379**

`kubectl expose pod redis --port=6379 --name redis-service --dry-run=client -o yaml`

(This will automatically use the pod's labels as selectors)

Or

`kubectl create service clusterip redis --tcp=6379:6379 --dry-run=client -o yaml` (This will not use the pods labels as selectors, instead it will assume selectors as **app=redis.** [You cannot pass in selectors as an option.](https://github.com/kubernetes/kubernetes/issues/46191) So it does not work very well if your pod has a different label set. So generate the file and modify the selectors before creating the service)

  

**Create a Service named nginx of type NodePort to expose pod nginx's port 80 on port 30080 on the nodes:**

`kubectl expose pod nginx --type=NodePort --port=80 --name=nginx-service --dry-run=client -o yaml`

(This will automatically use the pod's labels as selectors, [but you cannot specify the node port](https://github.com/kubernetes/kubernetes/issues/25478). You have to generate a definition file and then add the node port in manually before creating the service with the pod.)

Or

`kubectl create service nodeport nginx --tcp=80:80 --node-port=30080 --dry-run=client -o yaml`

(This will not use the pods labels as selectors)

Both the above commands have their own challenges. While one of it cannot accept a selector the other cannot accept a node port. I would recommend going with the `kubectl expose` command. If you need to specify a node port, generate a definition file using the same command and manually input the nodeport before creating the service.
```


## Get number of pods per node
```
kubectl get pods -A -o jsonpath='{range .items[?(@.spec.nodeName)]}{.spec.nodeName}{"\n"}{end}' | sort | uniq -c | sort -rn
```
### Define default limits
```
apiVersion: v1
kind: LimitRange
metadata:
  name: mem-limit-range
spec:
  limits:
  - default:
      memory: 512Mi
    defaultRequest:
      memory: 256Mi
    type: Container

apiVersion: v1
kind: LimitRange
metadata:
  name: cpu-limit-range
spec:
  limits:
  - default:
      cpu: 1
    defaultRequest:
      cpu: 0.5
    type: Container
```

### Logging

```
git clone https://github.com/kodekloudhub/kubernetes-metrics-server.git
kubectl top nodes
kubectl top pods
```

### Rolling Updates
```
kubectl rollout status deployment my-app
kubectl rollout history deployment my-app
kubectl rollout undo deployment my-app
```

### Testing
```
for i in {1..35}; do
   kubectl exec --namespace=kube-public curl -- sh -c 'test=`wget -qO- -T 2  http://webapp-service.default.svc.cluster.local:8080/info 2>&1` && echo "$test OK" || echo "Failed"';
   echo ""
done
```

### Initcontainers
```
apiVersion: v1
kind: Pod
metadata:
  name: myapp-pod
  labels:
    app: myapp
spec:
  containers:
  - name: myapp-container
    image: busybox:1.28
    command: ['sh', '-c', 'echo The app is running! && sleep 3600']
  initContainers:
  - name: init-myservice
    image: busybox:1.28
    command: ['sh', '-c', 'until nslookup myservice; do echo waiting for myservice; sleep 2; done;']
  - name: init-mydb
    image: busybox:1.28
    command: ['sh', '-c', 'until nslookup mydb; do echo waiting for mydb; sleep 2; done;']
```

### Cluster maintenance
```
##  gracefully terminates pods on node-1, no pods can be schedule until you perform your maintenance

kubectl drain node-1 

## No more pods will be schedule on node 2 but pods which were running will not be evicted from node 2 unlike drain

kubectl cordon node-2

### Once maintenance done, mark node as ready
kubectl uncordon node-1

```

### Cluster Upgrade
```
https://v1-26.docs.kubernetes.io/docs/tasks/administer-cluster/kubeadm/kubeadm-upgrade/
##################
kubeadm upgrade plan
apt-get upgrade -y kubeadm=1.X+1.0-00
kubeadm upgrade apply v1.X+1.0
apt-get upgrade -y kubelet=1.X+1.0-00
sudo systemctl restart kubelet
kubectl get nodes

kubectl drain node-1
ssh node-1
apt-get upgrade -y kubeadm=1.X+1.0-00
apt-get upgrade -y kubelet=1.X+1.0-00
kubeadm upgrade node config --kubelet-version v1.12.0
sudo systemctl restart kubelet
ssh master
kubectl uncordon node-1

```

### Backup and Restore
```
### Save
ETCDCTL_API=3 etcdctl --endpoints=https://[127.0.0.1]:2379 \
--cacert=/etc/kubernetes/pki/etcd/ca.crt \
--cert=/etc/kubernetes/pki/etcd/server.crt \
--key=/etc/kubernetes/pki/etcd/server.key \
snapshot save /opt/snapshot-pre-boot.db

### Restore
ETCDCTL_API=3 etcdctl --data-dir /var/lib/etcd-from-backup \ snapshot restore /opt/snapshot-pre-boot.db

vi /etc/kubernetes/manifests/etcd.yaml (point volumes hostpath to /var/lib/etcd-from-backup)

### list members of the cluster
ETCDCTL_API=3 etcdctl \
 --endpoints=https://127.0.0.1:2379 \
 --cacert=/etc/etcd/pki/ca.pem \
 --cert=/etc/etcd/pki/etcd.pem \
 --key=/etc/etcd/pki/etcd-key.pem \
  member list
  ```

### Switch to Cluster
```
kubectl config view
kubectl config use-context cluster1

```

### Openssl
```

### CA certificates
# Generate keys
openssl genrsa -out ca.key 2048

# Certificate Signing Request
openssl req -new -key ca.key -subj "/CN=KUBERNETES-CA" -out ca.csr

# Sign Certificates
openssl x509 -req -in ca.csr -signkey ca.key -out ca.crt

### Client certificates

# Admin
openssl genrsa -out admin.key 2048
openssl req -new -key admin.key -subj "/CN=kube-admin" -out admin.csr
openssl x509 -req -in admin.csr --CA ca.crt -CAkey ca.key -out admin.crt

### CHeck certs health
openssl x509 -in /etc/kubernetes/pki/apiserver.crt -text -noout



### Create a dev certificate for handling kubernetes
openssl genrsa -out josh.key 2048
openssl req -new -key josh.key -subj "/CN=josh" -out josh.csr
cat josh.csr | base64 -w 0
cat <<EOF | kubectl create -f -
---
apiVersion: certificates.k8s.io/v1
kind: CertificateSigningRequest
metadata:
  name: akshay
spec:
  groups:
  - system:authenticated
  request: <Paste the base64 encoded value of the CSR file>
  signerName: kubernetes.io/kube-apiserver-client
  usages:
  - client auth
EOF
kubectl get csr
kubectl certificate approve josh
kubectl get csr josh -o yaml
echo "XXXXX" | base64 --decode

```

### Check Accesses
```
kubectl auth can-i create deployments
kubectl auth can-i delete nodes
kubectl auth can-i create deployments --as dev-user
```
