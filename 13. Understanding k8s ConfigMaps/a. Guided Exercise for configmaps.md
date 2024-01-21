Create a configmap yaml file. 

    user@vsphere:~$ cat configmap.yaml 
    apiVersion: v1
    kind: ConfigMap
    metadata:
      namespace: default
      name: confighost
    data:
      vhost: vsphere.vmware.local

Apply configmap yaml

    user@vsphere:~$ kubectl apply -f configmap.yaml 
    configmap/confighost created

Getting configmap object. 

    user@vsphere:~$ kubectl get cm 
    NAME               DATA   AGE
    confighost         1      24s
    kube-root-ca.crt   1      64m

Now, Create configmap pod yaml.

    user@vsphere:~$ cat pod-cm.yaml 
    apiVersion: v1
    kind: Pod
    metadata:
      name: pod-configmap
    spec:
      restartPolicy: Never
      volumes:
      - name: vol
        configMap:
          name: confighost
      containers:
      - name: test
        image: ubuntu:latest
        command:
          - "/bin/bash"
          - "-c"
          - "sleep 22999"
        env:
        - name: vhost
          valueFrom:
            configMapKeyRef:
              name: confighost
              key: vhost
              optional: true
        volumeMounts:
          - name: vol
            mountPath: /tmp/special

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

    user@vsphere:~$ kubectl get cm
    NAME               DATA   AGE
    confighost         1      4m42s
    kube-root-ca.crt   1      69m

    user@vsphere:~$ kubectl delete cm confighost
    configmap "confighost" deleted


