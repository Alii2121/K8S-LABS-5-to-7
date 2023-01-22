# K8S-LABS-5-to-7

## K8S Lab 5 
* NOTE: Deployment will take place in a namespace called **lab3**

### Create basic resources
- Create the specified Namespace
```bash
k create ns lab3
```
- Create a deployment nginx with 3 replicas
```bash
k create deployment nginx --replicas=3 --image nginx -n lab3 --port 80 -oyaml --dry-run=client > dep1.yaml
k apply -f dep1.yml
```

- Expose the deployment on port 80

```bash
k expose deployment nginx -n lab3  --port 80 --target-port 80 -oyaml --dry-run=client > svc.yaml
k apply -f svc.yaml 
```

![Screenshot from 2023-01-22 09-34-17](https://user-images.githubusercontent.com/103090890/213906846-a640e392-81f1-4bd3-9204-18ba75c4d76a.png)

### 1. Create a **serviceaccount** cronjob-sa

```bash 
k create sa cronjob-sa -n lab3
```

![Screenshot from 2023-01-22 09-35-26](https://user-images.githubusercontent.com/103090890/213907085-956e0286-3af9-4a62-9f61-13d50f87be58.png)




### 2. Create a Role that allows listing all the services and endpoints 

![Screenshot from 2023-01-22 09-35-26](https://user-images.githubusercontent.com/103090890/213907088-03356ce9-16ef-4c5c-8a59-65d2b2a26f29.png)

```bash 
vim role.yml
``` 

```YAML


apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: pod-endpoints-lister
rules:
- apiGroups: [""]
  resources: ["endpoints"]
  verbs: ["get","list", "watch"]
```

```bash
k apply -f role.yml -n lab3 
```


### 3. Link the Role with the created SA 

```bash 
vim rolebind.yaml
```

```YAML
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: pod-endpoints-lister-binding
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: Role
  name: pod-endpoints-lister
subjects:
- kind: ServiceAccount
  name: cronjob-sa
  namespace: lab3
```
```bash 
k apply -f rolebind.yml
```



### 4. Create a CronJob that lists the endpoints in that namespace every minute and paste the output for the first pod created
![Screenshot from 2023-01-22 09-35-47](https://user-images.githubusercontent.com/103090890/213907097-da23a147-b61a-49be-8abb-488e1f8a721d.png)

```bash
vim cronjob.yml
```
```YAML
apiVersion: batch/v1
kind: CronJob
metadata:
  namespace: lab3
  name: list-endpoints
spec:
  schedule: "* * * * *"
  jobTemplate:
    spec:
      template:
        spec:
          serviceAccountName: cronjob-sa
          containers:
          - name: list
            image: bitnami/kubectl
            command:
            - kubectl
            - get
            - endpoints

          restartPolicy: OnFailure
```

```bash
k apply -f cronjob.yml
```
![Screenshot from 2023-01-22 09-36-04](https://user-images.githubusercontent.com/103090890/213907113-7d3e077e-42c7-44e3-9f4c-a7349432431d.png)


### 5. After listing try to delete the 3 nginx pods ? again try to view the logs for the newly created pod for that cronJob what do you think happened ?

![Screenshot from 2023-01-22 09-38-01](https://user-images.githubusercontent.com/103090890/213907102-12859fd6-3bb5-4b3e-9f56-486b5e29a999.png)

![Screenshot from 2023-01-22 10-00-33](https://user-images.githubusercontent.com/103090890/213907116-fd4f22b9-7432-4334-9d46-7ead29322499.png)

![Screenshot from 2023-01-22 10-05-54](https://user-images.githubusercontent.com/103090890/213907119-a91185f2-d97f-4ccc-896d-86bc73c8b2ed.png)

- The Endpoint should've changed, but after some research found that, In Kubernetes, when a pod is deleted, its endpoints will not immediately be removed. This is because the Kubernetes control plane does not have real-time visibility into the state of pods running on worker nodes. Instead, it relies on the Kubernetes API server's cache of pod status information, which is updated periodically by the kubelet.

When a pod is deleted, the kubelet on the worker node where the pod was running sends a deletion event to the API server. The API server then updates its cache to reflect that the pod is no longer running, but the endpoints associated with the pod will not be immediately deleted.

Instead, Kubernetes uses a process called endpoint garbage collection to periodically check for and remove stale endpoints. The endpoint garbage collection process runs at a configurable interval (the default is 30 minutes) and compares the list of endpoints in the API server's cache to the list of endpoints reported by the kubelets. If an endpoint is no longer reported by any kubelet, it is considered stale and is removed from the API server's cache.













# K8S LAB 6


### 1-create pod from the below yaml file
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: webapp
spec:
  containers:
  - env:
    - name: LOG_HANDLERS
      value: file
    image: kodekloud/event-simulator
    imagePullPolicy: Always
    name: event-simulator
```

```bash
vim pod.yml
k create -f pod.yml
```



### 2-Configure a volume to store these logs at /var/log/webapp on the host.

- Use the spec provided below.

- Name: webapp

- Image Name: kodekloud/event-simulator

- Volume HostPath: /var/log/webapp

- Volume Mount: /log


- This Yaml has the pod and volume specified :
```YAML
apiVersion: v1
kind: Pod
metadata:
  name: webapp
spec:
  containers:
  - env:
    - name: LOG_HANDLERS
      value: file
    image: kodekloud/event-simulator
    imagePullPolicy: Always
    name: event-simulator
    volumeMounts:
    - mountPath: /log
      name: webapp-logs
  volumes:
  - name: webapp-logs
    hostPath:
      path: /var/log/webapp
```


 ### 3-Create a Persistent Volume with the given specification.


- Volume Name: pv-log

- Storage: 100Mi

- Access Modes: ReadWriteMany

- Host Path: /pv/log

-  Reclaim Policy: Retain

```YAML 
apiVersion: v1
kind: PersistentVolume
metadata:
  name: pv-log
spec:
  capacity:
    storage: 100Mi
  accessModes:
    - ReadWriteMany
  hostPath:
    path: /pv/log
  persistentVolumeReclaimPolicy: Retain
```






### 4-Let us claim some of that storage for our application. Create a Persistent Volume Claim with the given specification.

- Volume Name: claim-log-1

- Storage Request: 50Mi

- Access Modes: ReadWriteOnce

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: claim-log-1
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 50Mi
  volumeName: pv-log
```
![Screenshot from 2023-01-22 20-20-09](https://user-images.githubusercontent.com/103090890/213936294-35a590f2-8fee-4dc1-8166-56f973a86ecd.png)



------
### 5-What is the state of the Persistent Volume Claim?
 

![Screenshot from 2023-01-22 20-23-39](https://user-images.githubusercontent.com/103090890/213936375-1e88d99c-ab92-482e-954a-d5c5eb371eb8.png)



-------
### 6-What is the state of the Persistent Volume?

![Screenshot from 2023-01-22 20-20-09](https://user-images.githubusercontent.com/103090890/213936408-689b974c-0650-4bb0-88b1-7f459de11343.png)


-------
### 7-Why is the claim not bound to the available Persistent Volume?

- Because the access modes of the pvc 


------
### 8-Update the Access Mode on the claim to bind it to the PV?


- Make the Access Modes : ReadWriteMany
- Made some tweaks on the pv and pvc because there were a problem with the paths and another problem causing the claim to stay pending waiting for consumer ( tried to create a consumer, an error appeard in the local storage type) so I tweaked the pv and pvc
- (Reffered to Documentation)


![Screenshot from 2023-01-22 21-25-28](https://user-images.githubusercontent.com/103090890/213937002-78ed1b25-0711-43a8-99bd-0dd27a2d058c.png)









---------




# THANK YOU !

