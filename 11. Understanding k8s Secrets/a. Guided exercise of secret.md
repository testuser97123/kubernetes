Create a secret.  

    user@vsphere:~$ kubectl create secret generic creds --from-literal=LIC1=2837-8822-0982-7821-2123
    secret/creds created


Getting secret object. 

    user@vsphere:~$ kubectl get secret 
    NAME    TYPE     DATA   AGE
    creds   Opaque   1      3s


Create secret pod using volumeMount.

    user@vsphere:~$ cat pod.yaml 
    apiVersion: v1 
    kind: Pod
    metadata:
      name: web01
    spec: 
      volumes:
      - name: secret-volume
        secret:
          secretName: creds
      containers:
      - name: web
        image: registry.k8s.io/busybox
        #image: rockylinux/rockylinux
        command:
          - '/bin/sh'
          - "-c"
          - "sleep 10000"
        volumeMounts:
          - name: secret-volume
            readOnly: true
            mountPath: "/etc/secret-volume"

Apply, pod-secret file. 

    user@vsphere:~$ kubectl apply -f pod.yaml
    pod/web01 created

Getting pod status. 

    user@vsphere:~$ kubectl get pods
    NAME      READY   STATUS    RESTARTS   AGE
    web01     1/1     Running   0          4s

Getting data inside the pod. 

    user@vsphere:~$ kubectl exec -it web01 -- sh
    / # ls /etc/secret-volume/LIC1
    /etc/secret-volume/LIC1

    / # cat /etc/secret-volume/LIC1
    2837-8822-0982-7821-2123

Create secret pod using environment variable.

    user@vsphere:~$ cat pod-secret-env.yaml 
    apiVersion: v1 
    kind: Pod
    metadata:
      name: pod-env
    spec: 
      containers: 
      - name: web
        image: ubuntu:latest
        command: 
          - '/bin/sh'
          - "-c"
          - "sleep 10000"
        env: 
        - name: LIC1
          valueFrom: 
            secretKeyRef: 
              name: creds
              key: LIC1

Apply, pod-secret-env file. 

    user@vsphere:~$ kubectl apply -f pod-secret-env.yaml
    pod/pod-env created

Getting pod status. 

    user@vsphere:~$ kubectl get pods
    NAME      READY   STATUS    RESTARTS   AGE
    pod-env   1/1     Running   0          17s

login inside a pod, gathering env variable. 

    user@vsphere:~$ kubectl exec -it pod-env -- bash

    root@pod-env:/# env | grep LIC1
    LIC1=2837-8822-0982-7821-2123


Now apply pod-cm.yaml 

    user@vsphere:~$ kubectl apply -f pod-cm.yaml
    pod/pod-configmap created


Getting the status of the pod. 

    user@vsphere:~$ kubectl get pods
    NAME            READY   STATUS    RESTARTS   AGE
    pod-configmap   1/1     Running   0          43s

Getting env variable inside the pod. 

    user@vsphere:~$ kubectl exec -it pod-configmap -- bash
    root@pod-configmap:/# env | grep -i vhost
    vhost=vsphere.vmware.local

    root@pod-configmap:/# ls -l /tmp/special/vhost
    lrwxrwxrwx 1 root root 12 Jan 21 06:08 /tmp/special/vhost -> ..data/vhost

    root@pod-configmap:/# cat /tmp/special/vhost
    vsphere.vmware.local

Delete a pod. 

    user@vsphere:~$ kubectl delete pod pod-configmap
    pod "pod-configmap" deleted

Delete a configmap. 

    user@vsphere:~$ kubectl get secret
    NAME               DATA   AGE
    confighost         1      4m42s
    kube-root-ca.crt   1      69m

    user@vsphere:~$ kubectl delete secret confighost
    configmap "confighost" deleted


