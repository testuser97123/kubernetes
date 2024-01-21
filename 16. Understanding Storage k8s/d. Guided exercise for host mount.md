Create a new hostpod pv & pvc yaml.

      user@vsphere:~/nfs$ vim 06-hostpod.yaml
      apiVersion: v1 
      kind: PersistentVolume 
      metadata: 
        labels: 
          volume: pv0001
        name: pv0001 
      spec: 
        accessModes:
        - ReadWriteOnce
        capacity: 
          storage: 5Gi 
        claimRef: 
          apiVersion: v1   
          kind: PersistentVolumeClaim
          name: myclaim
          namespace: default
        hostPath: 
          path: /mnt/pv-data/pv0001
          type: ""
        persistentVolumeReclaimPolicy: Recycle 
      status: 
        phase: Bound 
      ---
      apiVersion: v1 
      kind: PersistentVolumeClaim 
      metadata: 
        name: myclaim 
      spec: 
        accessModes: 
        - ReadWriteOnce 
        resources: 
          requests: 
            storage: 1Gi 

Apply hostpod.yaml 

    user@vsphere:~/nfs$ kubectl apply -f 06-hostpod.yaml 
    persistentvolume/pv0001 created
    persistentvolumeclaim/myclaim created

Getting status of physical volume. 

    user@vsphere:~/nfs$ kubectl get pv 
    NAME                                       CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS   CLAIM                    STORAGECLASS   REASON   AGE
    pv0001                                     5Gi        RWO            Recycle          Bound    default/myclaim                                  4s

Getting status of pyhsical volume claim. 

    user@vsphere:~/nfs$ kubectl get pvc
    NAME             STATUS   VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS   AGE
    myclaim          Bound    pv0001                                     5Gi        RWO                           6

Create a new pod.yaml 

    kind: Pod
    apiVersion: v1 
    metadata: 
      name: pod-vol2
    spec: 
      containers: 
      - image: nginx 
        name: web
        ports: 
        - containerPort: 80 
        volumeMounts: 
        - mountPath: "/tmp/container" 
          name: pv0001 
      volumes: 
      - name: pv0001 
        persistentVolumeClaim: 
          claimName: myclaim

Apply pod yaml. 

    user@vsphere:~/nfs$ kubectl apply -f pod.yaml 
    pod/pod-vol2 created

Getting pod status. 

    user@vsphere:~/nfs$ kubectl get pods pod-vol2
    NAME       READY   STATUS    RESTARTS   AGE
    pod-vol2   1/1     Running   0          7s

checking pod status. 

    user@vsphere:~/nfs$ kubectl exec -it pod-vol2 -- bash 
    root@pod-vol2:/# df -h 
    Filesystem      Size  Used Avail Use% Mounted on
    overlay         314G   15G  283G   5% /
    tmpfs            64M     0   64M   0% /dev
    shm              64M     0   64M   0% /dev/shm
    tmpfs           976M  3.3M  972M   1% /etc/hostname
    /dev/nvme0n1p3  314G   15G  283G   5% /tmp/container
    tmpfs           9.5G   12K  9.5G   1% /run/secrets/kubernetes.io/serviceaccount
    tmpfs           4.8G     0  4.8G   0% /proc/asound
    tmpfs           4.8G     0  4.8G   0% /proc/acpi
    tmpfs           4.8G     0  4.8G   0% /proc/scsi
    tmpfs           4.8G     0  4.8G   0% /sys/firmware