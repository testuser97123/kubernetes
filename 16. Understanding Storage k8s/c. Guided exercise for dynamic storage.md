
step 1. Create rbac for nfs-client provisioner. 

    user@vsphere:~$ vim 01-rbac.yaml 
    kind: ServiceAccount
    apiVersion: v1
    metadata:
      name: nfs-client-provisioner
    ---
    kind: ClusterRole
    apiVersion: rbac.authorization.k8s.io/v1
    metadata:
      name: nfs-client-provisioner-runner
    rules:
      - apiGroups: [""]
        resources: ["persistentvolumes"]
        verbs: ["get", "list", "watch", "create", "delete"]
      - apiGroups: [""]
        resources: ["persistentvolumeclaims"]
        verbs: ["get", "list", "watch", "update"]
      - apiGroups: ["storage.k8s.io"]
        resources: ["storageclasses"]
        verbs: ["get", "list", "watch"]
      - apiGroups: [""]
        resources: ["events"]
        verbs: ["create", "update", "patch"]
    ---
    kind: ClusterRoleBinding
    apiVersion: rbac.authorization.k8s.io/v1
    metadata:
      name: run-nfs-client-provisioner
    subjects:
      - kind: ServiceAccount
        name: nfs-client-provisioner
        namespace: default
    roleRef:
      kind: ClusterRole
      name: nfs-client-provisioner-runner
      apiGroup: rbac.authorization.k8s.io
    ---
    kind: Role
    apiVersion: rbac.authorization.k8s.io/v1
    metadata:
      name: leader-locking-nfs-client-provisioner
    rules:
      - apiGroups: [""]
        resources: ["endpoints"]
        verbs: ["get", "list", "watch", "create", "update", "patch"]
    ---
    kind: RoleBinding
    apiVersion: rbac.authorization.k8s.io/v1
    metadata:
      name: leader-locking-nfs-client-provisioner
    subjects:
      - kind: ServiceAccount
        name: nfs-client-provisioner
        # replace with namespace where provisioner is deployed
        namespace: default
    roleRef:
      kind: Role
      name: leader-locking-nfs-client-provisioner
      apiGroup: rbac.authorization.k8s.io

Apply rbac policy. 

    user@vsphere:~/nfs$ kubectl apply -f 01-rbac.yaml 
    serviceaccount/nfs-client-provisioner created
    clusterrole.rbac.authorization.k8s.io/nfs-client-provisioner-runner created
    clusterrolebinding.rbac.authorization.k8s.io/run-nfs-client-provisioner created
    role.rbac.authorization.k8s.io/leader-locking-nfs-client-provisioner created
    rolebinding.rbac.authorization.k8s.io/leader-locking-nfs-client-provisioner created

step 2. Create storageClass. 

    user@vsphere:~$ vim 02-class.yaml 
    apiVersion: storage.k8s.io/v1
    kind: StorageClass
    metadata:
      name: nfs
    provisioner: external
    parameters:
      archiveOnDelete: "false"

Apply storageClass. 

    user@vsphere:~$ kubectl apply -f 02-class.yaml 
    storageclass.storage.k8s.io/nfs created

Getting status of storageClass. 

    user@vsphere:~/nfs$ kubectl get sc 
    NAME   PROVISIONER   RECLAIMPOLICY   VOLUMEBINDINGMODE   ALLOWVOLUMEEXPANSION   AGE
    nfs    external      Delete          Immediate           false                  4s

step 3. Create nfs-provisioner. 

    user@vsphere:~$ vim 03-provisioner.yaml
    kind: Deployment
    apiVersion: apps/v1
    metadata:
      name: redhat-utility
    spec:
      selector:
        matchLabels:
          app: redhat-utility
      replicas: 1
      strategy:
        type: Recreate
      template:
        metadata:
          labels:
            app: redhat-utility
        spec:
          serviceAccountName: nfs-client-provisioner
          containers:
            - name: redhat-utility
              image: rkevin/nfs-subdir-external-provisioner:fix-k8s-1.20
              volumeMounts:
                - name: nfs-client-root
                  mountPath: /persistentvolumes
              env:
                - name: PROVISIONER_NAME
                  value: external
                - name: NFS_SERVER
                  value: 172.25.128.2
                - name: NFS_PATH
                  value: /nfsVsphereVMDK
          volumes:
            - name: nfs-client-root
              nfs:
                server: 172.25.128.2
                path: /nfsVsphereVMDK

Apply deployment provisioner. 

    user@vsphere:~/nfs$ kubectl apply -f 03-provisioner.yaml
    deployment.apps/redhat-utility created

Wait for pod become ready.

    user@vsphere:~/nfs$ kubectl get pods
    NAME                              READY   STATUS              RESTARTS   AGE
    redhat-utility-84f54db69c-9fnxq   0/1     ContainerCreating   0          4s

Wait for sometime until pod become running. 

    user@vsphere:~/nfs$ kubectl get pods
    NAME                              READY   STATUS    RESTARTS   AGE
    redhat-utility-84f54db69c-9fnxq   1/1     Running   0          20s



step 4. Create dynamic pvc. 

    user@vsphere:~$ vim 04-pvc-claim.yaml 
    apiVersion: v1
    kind: PersistentVolumeClaim
    metadata:
      name: redhat-utility
    spec:
      storageClassName: nfs 
      accessModes:
        - ReadWriteMany
      resources:
        requests:
        storage: 100Gi

Apply nfs pvc. 

    user@vsphere:~/nfs$ kubectl apply -f 04-pvc.yaml 
    persistentvolumeclaim/redhat-utility created

Getting pvc status, it should be bound status.

    user@vsphere:~/nfs$ kubectl get pvc
    NAME             STATUS   VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS   AGE
    redhat-utility   Bound    pvc-83a1d13c-77e8-406b-b3f9-8bdec1e4f809   100Gi      RWX            nfs            4s

step 5. Create dynamic pod.

    user@vsphere:~$ vim 05-pod-test.yaml
    apiVersion: v1
    kind: Pod
    metadata:
      labels:
        run: pod1
      name: pod1
    spec:
      volumes:
      - name: host-volume
        persistentVolumeClaim:
          claimName: redhat-utility 
      containers:
      - image: ubuntu:latest
        name: pod1
        command: 
          - "/bin/sh" 
          - "-c"
          - "sleep 6000"
        volumeMounts:
        - name: host-volume
          mountPath: /mydata

Apply test pod. 

    user@vsphere:~/nfs$ kubectl apply -f 05-pod.yaml
    pod/pod1 created

Getting pod status. 

    user@vsphere:~/nfs$ kubectl get pods 
    NAME                              READY   STATUS    RESTARTS   AGE
    pod1                              1/1     Running   0          8s

Check mount point status. 

    user@vsphere:~/nfs$ kubectl exec -it pod1 -- bash 
    root@pod1:/# df -h 
    Filesystem                                                                                    Size  Used Avail Use% Mounted on
    ...
    172.25.128.2:/nfsVsphereVMDK/default-redhat-utility-pvc-83a1d13c-77e8-406b-b3f9-8bdec1e4f809  314G   15G  283G   5% /mydata


