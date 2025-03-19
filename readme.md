# Goal 

Create a basic app that generate data on a pvc every 10 s so that you can test your backups.

## Prepare

Check your available storage class
```
oc get sc
NAME                      PROVISIONER                RECLAIMPOLICY   VOLUMEBINDINGMODE      ALLOWVOLUMEEXPANSION   AGE
azuredisk-legacy          kubernetes.io/azure-disk   Delete          WaitForFirstConsumer   true                   13d
azurefile-csi (default)   file.csi.azure.com         Delete          Immediate              true                   61d
managed-csi               disk.csi.azure.com         Delete          WaitForFirstConsumer   true                   61d
```


## install

Define 2 variables 
```
STORAGE_CLASS=azuredisk-legacy
SIZE="1Gi"
IMAGE=docker.io/busybox:latest
```

`IMAGE` should be an image that you can pull from your cluster.

Now create this manifest by copy and paste the command
```
cat<<EOF | oc create -f -
# create the namespace basic-app
apiVersion: v1
kind: Namespace
metadata:
  name: basic-app
spec: {}
--- 
# Create a pvc in the namespace basic-app whit the name basic-app-pvc and storage class azuredisk-legacy 
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: basic-app-pvc
  namespace: basic-app
spec:
  accessModes:
  - ReadWriteOnce
  resources:
    requests:
      storage: $SIZE
  storageClassName: $STORAGE_CLASS
---
# Create a deployment in the namespace basic-app with the name basic-app-deployment using this pvc and writing every 10 seconds the date in the file /data/date.txt
apiVersion: apps/v1
kind: Deployment
metadata:
  name: basic-app-deployment
  namespace: basic-app
spec:
  replicas: 1
  selector:
    matchLabels:
      app: basic-app
  template:
    metadata:
      labels:
        app: basic-app
    spec:
      containers:
      - name: basic-app
        image: $IMAGE
        command: ["/bin/sh", "-c", "while true; do date >> /data/date.txt; sleep 10; done"]
        volumeMounts:
        - name: basic-app-volume
          mountPath: /data
      volumes:
      - name: basic-app-volume
        persistentVolumeClaim:
          claimName: basic-app-pvc
EOF
```

## check your deployemnt 
Check the pod deploy 
```
oc get po -n basic-app
```

You should see something like this 
```
NAME                                    READY   STATUS    RESTARTS   AGE
basic-app-deployment-7bf477cdbc-5qbcl   1/1     Running   0          63s
```

Check the pvc
```
oc get pvc -n basic-app
```

Check data is generated 
```
oc exec -n basic-app deploy/basic-app-deployment -- cat /data/date.txt
```

You should see something like this 
```
Tue Oct  1 14:11:38 UTC 2024
Tue Oct  1 14:11:48 UTC 2024
Tue Oct  1 14:11:58 UTC 2024
Tue Oct  1 14:12:08 UTC 2024
Tue Oct  1 14:12:18 UTC 2024
Tue Oct  1 14:12:28 UTC 2024
Tue Oct  1 14:12:38 UTC 2024
Tue Oct  1 14:12:48 UTC 2024
Tue Oct  1 14:12:58 UTC 2024
Tue Oct  1 14:13:08 UTC 2024
Tue Oct  1 14:13:18 UTC 2024
Tue Oct  1 14:13:28 UTC 2024
Tue Oct  1 14:13:38 UTC 2024
Tue Oct  1 14:13:48 UTC 2024
Tue Oct  1 14:13:58 UTC 2024
Tue Oct  1 14:14:08 UTC 2024
Tue Oct  1 14:14:18 UTC 2024
Tue Oct  1 14:14:28 UTC 2024
Tue Oct  1 14:14:38 UTC 2024
Tue Oct  1 14:14:48 UTC 2024
```

# Filled up the volumes to measure the snapshot and export time

```
oc exec -n basic-app -it deploy/basic-app-deployment -- sh
cd /data
for i in $(seq 1 10); do
  dd if=/dev/urandom of=file$i bs=1M count=90
done
```

Check your /data is full now 
```
df -h | grep /data
```

You should see something like that 
```
/dev/sdb                973.4M    957.4M         0 100% /data
```

# Test backup 

If you have created the necessary infra profile or annotated the volumesnapshotclass you can create a policy on basic-app namespace and run it.

The backup should be successful.