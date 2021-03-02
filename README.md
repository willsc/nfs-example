# nfs-example

Create a persistent volume with 

```
gcloud compute disks create --size=20GB --zone=europe-west2-a gce-nfs-disk

```

Create NFS server

```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nfs-server
spec:
  replicas: 1
  selector:
    matchLabels:
      role: nfs-server
  template:
    metadata:
      labels:
        role: nfs-server
    spec:
      containers:
      - name: nfs-server
        image: gcr.io/google_containers/volume-nfs:0.8
        ports:
          - name: nfs
            containerPort: 2049
          - name: mountd
            containerPort: 20048
          - name: rpcbind
            containerPort: 111
        securityContext:
          privileged: true
        volumeMounts:
          - mountPath: /exports
            name: mypvc
      volumes:
        - name: mypvc
          gcePersistentDisk:
            pdName: gce-nfs-disk
            fsType: ext4
```

Create a service 

```
apiVersion: v1
kind: Service
metadata:
  name: nfs-server
  namespace: nfs
spec:
  # clusterIP: 10.3.240.20
  ports:
    - name: nfs
      port: 2049
    - name: mountd
      port: 20048
    - name: rpcbind
      port: 111
  selector:
    role: nfs-server

```
Get the IP of the nfs server 
```
get svc -n nfs
NAME         TYPE        CLUSTER-IP     EXTERNAL-IP   PORT(S)                      AGE
nfs-server   ClusterIP   10.108.9.230   <none>        2049/TCP,20048/TCP,111/TCP   116m
```

Create a pv 

```
apiVersion: v1
kind: PersistentVolume
metadata:
  name: nfs
  namespace: nfs
spec:
  capacity:
    storage: 20Gi
  accessModes:
    - ReadWriteMany
  nfs:
    server: 10.108.9.230 << IP address of server
    path: "/"
```

Create PVC


```
kind: PersistentVolumeClaim
apiVersion: v1
metadata:
  name: nfs
  namespace: nfs
spec:
  accessModes:
    - ReadWriteMany
  storageClassName: ""
  resources:
    requests:
      storage: 20Gi
```


Start pod and mount nfs share

```
apiVersion: v1
kind: Pod
metadata:
  name: busybox
  namespace: nfs
spec:
  containers:
    - name: busybox
      image: busybox
      command:
        - sleep
        - "3600"
      volumeMounts:
      - mountPath: "/mnt"
        name: test
  volumes:
    - name: test
      persistentVolumeClaim:
        claimName: nfs


```
