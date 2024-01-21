
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

step 2. Create storageClass. 

    user@vsphere:~$ vim 02-class.yaml 
    apiVersion: storage.k8s.io/v1
    kind: StorageClass
    metadata:
      name: nfs
    provisioner: external
    parameters:
      archiveOnDelete: "false"

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

