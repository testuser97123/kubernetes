NFS Setup for Ubuntu. 

Installing nfs-kernel-server package. 

    user@vsphere:~$ sudo apt install -y nfs-kernel-server

Create a new directory for nfs share. 

    user@vsphere:~$ sudo mkdir /nfsVsphereVMDK

Make entry in exports. 

    user@vsphere:~$ sudo vim /etc/exports
    /nfsVsphereVMDK  *(rw,sync,no_subtree_check)

Reload export configuration. 

    user@vsphere:~$ sudo exportfs -arv
    exporting *:/nfsVsphereVMDK

Check mount ready. 

    user@vsphere:~$ sudo showmount -e
    Export list for vsphere.vmware.local:
    /nfsVsphereVMDK *

Change ownership for the nfs share directory. 

    user@vsphere:~$ sudo chown nobody:nogroup /nfsVsphereVMDK/

Create physical volume vsphere-nfs-pv.yaml. 

    user@vsphere:~$ vim  vsphere-nfs-pv.yaml
    apiVersion: v1
    kind: PersistentVolume
    metadata:
    name: vsphere
    spec:
    capacity:
      storage: 5Gi
    volumeMode: Filesystem
    accessModes:
      - ReadWriteMany
    nfs:
      path: /nfsVsphereVMDK
      server: 172.25.128.2

Apply vsphere-nfs-pv.yaml.

    user@vsphere:~$ kubectl apply -f vsphere-nfs-pv.yaml
    persistentvolume/vsphere created

Getting nfs physical volume status. 

    user@vsphere:~$ kubectl get pv
    NAME      CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS      CLAIM   STORAGECLASS   REASON   AGE
    vsphere   5Gi        RWX            Retain           Available                                   25s

Create vsphere physical volume claim. 

    user@vsphere:~$ vim vsphere-nfs-pvc.yaml
    apiVersion: v1
    kind: PersistentVolumeClaim
    metadata:
    name: nfsclaim
    spec:
    accessModes:
      - ReadWriteMany
    volumeMode: Filesystem
    resources:
      requests:
        storage: 2Gi

Apply vsphere physical volume claim. 

    user@vsphere:~$ kubectl apply -f vsphere-nfs-pvc.yaml
    persistentvolumeclaim/nfsclaim created


Getting physical volume claim status it should be bound. 

    user@vsphere:~$ kubectl get pvc
    NAME       STATUS   VOLUME    CAPACITY   ACCESS MODES   STORAGECLASS   AGE
    nfsclaim   Bound    vsphere   5Gi        RWX                           20s

Create pod using volume mount. 

    user@vsphere:~$ vim nfs-pv.yaml
    apiVersion: v1
    kind: Pod
    metadata:
    name: vpod
    spec:
    containers:
    - name: test
      image: nginx:latest
      volumeMounts:
        - mountPath: "/var/www/html"
          name: mypd
    volumes:
      - name: mypd
        persistentVolumeClaim:
          claimName: nfsclaim

Apply nfs-pv.yaml 

    user@vsphere:~$ kubectl apply -f nfs-pv.yaml
    pod/vpod created

Inspect volume is mounted. 

    user@vsphere:~$ kubectl exec -it vpod -- bash
    root@vpod:/# df -h
    Filesystem                    Size  Used Avail Use% Mounted on
    overlay                       314G   15G  283G   5% /
    tmpfs                          64M     0   64M   0% /dev
    ...
    172.25.128.2:/nfsVsphereVMDK  314G   15G  283G   5% /var/www/html

Undo changes or Delete pod, pvc & pv. 


    user@vsphere:~$ kubectl delete pod vpod
    pod "vpod" deleted

    user@vsphere:~$ kubectl delete pvc nfsclaim
    persistentvolumeclaim "nfsclaim" deleted

    user@vsphere:~$ kubectl delete pv vsphere
    persistentvolume "vsphere" deleted

