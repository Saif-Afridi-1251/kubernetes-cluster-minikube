sudo su
apt update && apt -y install docker.io
apt install -y curl wget apt-transport-https
#########minikube
curl -LO https://storage.googleapis.com/minikube/releases/latest/minikube-linux-amd64
install minikube-linux-amd64 /usr/local/bin/minikube
minikube version
########kubectl
curl -LO https://storage.googleapis.com/kubernetes-release/release/`curl -s https://storage.googleapis.com/kubernetes-release/release/stable.txt`/bin/linux/amd64/kubectl
chmod +x ./kubectl
mv ./kubectl /usr/local/bin/kubectl

****start minikube create cluster ************
minikube start --force --driver=docker
kubectl get nodes
For a normal development cluster, I recommend:

minikube addons enable metrics-server

and:

minikube addons enable ingress

Then:

kubectl get pods -A

#####################################################
to create pod by using yaml we have to 

mkdir /first-k8s
cd /first-k8s
touch pod.yaml
***************
apiVersion: v1
kind: Pod
metadata:
  name: first-pod
spec:
  containers:
    - name: web1
      image: nginx
    - name : container
      image : redis 
***********************
kubectl apply -f pod.yaml
kubectl get pods
kubectl logs pod-name
(inspection)
kubectl describe pods pod-name

**********deploy.yaml******
The below yaml file create 3 pods runing with nginx server demonstrate labels , selector concept and auto healing by deleting one pod.
In Kubernetes, labels are key-value tags attached to resources, while a selector is used to find resources with specific labels.
##################
kind: Deployment
apiVersion: apps/v1
metadata:
  name: webdeployment
spec:
  replicas: 3
  selector:
    matchLabels:
      name: deployment
  template:
    metadata:
      name: testpod
      labels:
        name: deployment
    spec:
      containers:
      - name: web1
        image: nginx

######################################################
A **Pod** is the basic unit in Kubernetes that runs one or more containers, but it does not provide built-in scaling, rolling updates, or self-healing. A **Deployment** manages Pods through ReplicaSets and automatically maintains the desired number of Pods, supports **scaling, rolling updates, and rollbacks**, making Deployments more suitable for running applications in production.
######################################################
commands 
kubectl get deployments
kubectl delete pod webdeployment-75d8b75c6b-8tpjw
kubectl get pods --show-labels
kubectl get rs
kubectl edit deployment webdeployment
kubectl edit  deployment webdeployment
kubectl scale deployment nginx-deployment --replicas=5 (Scaling )
kubectl scale deployment nginx-deployment --replicas=2

kubectl rollout history deployment webdeployment
kubectl rollout undo deploy/webdeployment
#########################
Pod is the worker. Deployment is the manager.

##########LAB 02 SERVICES (clusterIp & Nodeport ) #####################
ClusterIP and NodePort are Kubernetes Service types used to expose applications.
ClusterIP = inside the cluster
NodePort = outside access through the Node's IP and port
$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$
vim deployment-service.yml
__________________________________________________________________
kind: Deployment
apiVersion: apps/v1
metadata:
  name: webdeploy
spec:
  replicas: 1
  selector:
    matchLabels:
      name: web
  template:
    metadata:
      name: testpod1
      labels:
        name: web
    spec:
      containers:
      - name: web
        image: httpd
        ports:
        - containerPort: 80
######################################################################
vim service.yml
________________________________________________________________________
kind: Service
apiVersion: v1
metadata:
  name: depservice
  namespace: prod
spec:
  ports:
    - port: 80
      targetPort: 80
  selector:
    name: web
  type: ClusterIP
______________________________________________________________________
kubectl apply -f deployment-service.yml
kubectl aply -f service.yml
kubectl get svc
kubectl get pods -o wide
kubectl describe svc depservice -n prod

 
minikube ip
______________________________________________________________________
Note : - for Nodeport just change clusterIp to Nodeport and acces the app/pod through minikube ip with port
both end point and service must be in same namespace as prod 
####################################FOR NODE PORT ############################################
minikube ip
kubectl get pods -n prod
kubectl describe svc depservice -n prod
kubectl get pods -n prod --show-labels

kubectl get endpoints depservice -n prod
minikube service depservice -n prod --url


###########Node Port Lab Completed######################################################################

#######################STORAGE Host Path Volume#########################################################
###############LAB MANUAL WITH COMMAND ###################################################################
### HostPath Storage Lab — Brief

A **HostPath volume** connects a directory on the Kubernetes/Minikube node to a directory inside a Pod. Data written inside the Pod is stored on the node, so it remains available even after the Pod is deleted.

**Flow:**

```text
Minikube Node
/data/hostpath
      ↓
   HostPath
      ↓
Pod /mnt/data
```

### Commands used

**1. Create namespace**

```bash
kubectl create namespace storage-lab
```

**2. Create storage directory on Minikube node**

```bash
minikube ssh -- sudo mkdir -p /data/hostpath
```

**3. Create Pod YAML**

```bash
vim hostpath-pod.yaml
```

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: hostpath-pod
  namespace: storage-lab
spec:
  containers:
    - name: nginx
      image: nginx
      volumeMounts:
        - name: host-storage
          mountPath: /mnt/data
  volumes:
    - name: host-storage
      hostPath:
        path: /data/hostpath
        type: DirectoryOrCreate
```

**4. Create the Pod**

```bash
kubectl apply -f hostpath-pod.yaml
```

**5. Check Pod**

```bash
kubectl get pods -n storage-lab
```

**6. Enter the Pod**

```bash
kubectl exec -it hostpath-pod -n storage-lab -- /bin/bash
```

**7. Create a file inside the mounted volume**

```bash
echo "Hello from Kubernetes storage" > /mnt/data/test.txt
```

Check it:

```bash
cat /mnt/data/test.txt
```

Exit:

```bash
exit
```

**8. Verify data on Minikube node**

```bash
minikube ssh -- sudo cat /data/hostpath/test.txt
```

**9. Delete the Pod**

```bash
kubectl delete pod hostpath-pod -n storage-lab
```

**10. Verify data still exists**

```bash
minikube ssh -- sudo cat /data/hostpath/test.txt
```

**11. Recreate the Pod**

```bash
kubectl apply -f hostpath-pod.yaml
```

**12. Verify data from the new Pod**

```bash
kubectl exec -it hostpath-pod -n storage-lab -- cat /mnt/data/test.txt
```

### Key point

**Pod storage → `/mnt/data`**
**Node storage → `/data/hostpath`**

HostPath is useful for learning and certain node-specific workloads, but for production Kubernetes storage, **PersistentVolumes (PV) and PersistentVolumeClaims (PVC)** are generally preferred.

______________________________________________________________________________________
vim host-path.yml
-------------host-path.yml------------------------
apiVersion: v1
kind: Pod
metadata:
  name: podtest
spec:
  containers:
    - image: nginx
      name: web
    volumeMounts:
        - mountPath: /tmp/yes
          name: newvol
  volumes:
    - name: newvol
    hostPath:
      path: /tmp/dir

-------------------------------------
kubectl apply -f host-path.yml
kubectl get pods
kubectl exec -it podtest --bash 
ls /tmp/yes
#############################################################################################################
PV & PVC 
###############################
step 1 : Launch EC2 instance 
Step 2 : intialize Minikube 
step 3 : create EBS 5Gi
Step 4 : create file pv.yml
-----------pv.ymml--------------------
apiVersion: v1
kind: PersistentVolume
metadata:
  name: vol1
spec:
  capacity:
    storage: 5Gi
  accessModes:
    - ReadWriteOnce
  persistentVolumeReclaimPolicy: Recycle
  awsElasticBlockStore:
    volumeID: vol-01db20916c03d5722
    fsType: ext4
________________________________________________________
Step 5 : create pvc.yml file volume claim from pv
________________pvc.yml_______________________________
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: myvc
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 5Gi
________________________________________________________
Step6 : Creat dep.yml 
------------dep.yml-------------------------------------
apiVersion: apps/v1
kind: Deployment
metadata:
  name: storage-test
spec:
  replicas: 1
  selector:
    matchLabels:
      app: pv
  template:
    metadata:
      labels:
        app: pv
    spec:
      containers:
      - name: os
        image: nginx
        volumeMounts:
        - name: stg
          mountPath: "/tmp/data"
      volumes:
      - name: stg
        persistentVolumeClaim:
          claimName: myvc
_____________________________________________________________
Step7 : execute the commands to complete lab
kubectl apply -f pv.yml
kubectl apply -f pvc.yml
kubectl apply -f dep.yml
kubectl get pv
kubectl get pvc
kubectl get pods
kubectl describe podsId
____________________________________________________________
###################STORAGE LAB COMPLETED #########################################################

A namespace in Kubernetes is a way to logically divide one cluster into separate environments or groups. Think of it like separate folders inside the same Kubernetes cluster. Resources such as Pods, Deployments, and Services belong to a namespace, and resources in one namespace are generally isolated from resources with the same names in another namespace.

Namespace tells Kubernetes where a resource belongs.

kubectl get ns

kubectlkubectl get svc -n prod
get pods -n prod


########$$$$$$$$$ NAME SPACE LAB #######################################################
vim deploy.yaml
------------------------------------------------------
kind: Deployment
apiVersion: apps/v1
metadata:
  name: webdeployment
  namespace: dev
spec:
  replicas: 3
  selector:
    matchLabels:
      name: deployment
  template:
    metadata:
      labels:
        name: deployment
    spec:
      containers:
      - name: web1
        image: ngtnx
----------------------------------------------------------------------
vim service.yml
--------------------------------------
kind: Service
apiVersion: v1
metadata:
  name: depservice
  namespace: dev
spec:
  ports:
    - port: 80
    targetPort: 80
  selector:
    name: web
  type: NodePort
___________________________________________________________________________
Both Above are same file of deployment and service changeses only namespace 
commmand for namespace 
kubectl apply -f dep.yml
kubectl apply -f service.yml
kubectl get pods 
kubectl get pods -n dev
kubectl get ns (NameSpaces)
kubectl create namespace dev
kubectl get ns
kubectl get pods --all-namespaces
kubectl delete namespace dev
#########################################################Completion of NAME SPACE #############################################
$$$$$$$$$$$$$$$$$$$$$$ SECRETS AND CONFIG MAPS $$$$$$$$$$$$$$$$$$$$$$$$$$$$$&&&&&&&&&&&&&&&&&&&&&&&&&&&&&&&&&&&&&&&&&&&&&&&&&&$
vim secrete.yaml
---------------------
apiVersion: v1
kind: Secret
metadata:
  name: mysecret
stringData:
  SECRETE_PASSWORD: myPassword
___________________________________
vim secret-pod.yaml
----------------------------------
kind: Pod
metadata:
  name: nginx-pod
spec:
  containers:
    - name: nginx
    image: nginx:latest
    env:
        - name: SECRET_PASSWORD
      valueFrom:
        secretKeyRef:
          name: mysecret
          key: SECRET_PASSWORD
---------------------------------------------
kubectl apply -f secrete.yaml
kubectl apply -f secrete-pod.yaml
kubectl get pods (if runing lab succcessful secrete config)
###############################SECRETE COMPLETE #########################################
$$$$$$$$$$$$$$$$$$$$$$$$$ CONFIG MAP $$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$


vim new.conf (WE ARE STUDYING DEVOPS)
kubectl get cm
kubectl create config map mymap -- from-file=new.conf
kubectl get cm
kubectl get cm mymap -o yaml



vim cm-env-pod.yaml
---------------------------
apiVersion: v1
kind: Pod
metadata:
  name: pod-configenv
spec:
  containers:
    - name: web
    image: nginx
    env:
        - name: class
      valueFrom:
        configMapKeyRef:
          name: mymap
          key: new.conf
__________________________________________________________
Kubectl apply -f cm-env-pod.yaml
kubectl get pods
kubectl exec -it pods-configenv -- bash
*******************************************************************
#########CONFIG MAP FROM OUTSIDE OF CONTAINER MOSTLY USED CONCEPT#################################
vim cm-config-pod.yaml 
----------------------------------
apiVersion: v1
kind: Pod
metadata:
  name: configmap-pod
spec:
  containers:
    - name: web
    image: nginx
    volumeMounts:
        - name: configvol
      mountPath: "/tmp/map"
  volumes:
    - name: configvol
    configMap:
      name: mymap
--------------------------------------------------
kubectl apply -f
kubectl exec -it configmap-pod -- bash
ls /tmp/map/
cat /tmp/map/new.conf
#####################################################################################################################################


