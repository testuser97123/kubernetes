## NodePort example

1.a. Let’s start with creating a deployment using the YAML file below. Some key things to note, each container is using the port 80 and has a label called app:nginx

    user@vsphere:~/svc-pod$ vim 01-deploy.yaml
    apiVersion: apps/v1
    kind: Deployment
    metadata:
      name: my-nginx-deploy
    labels:
      app: nginx
    spec:
      replicas: 2
      selector:
        matchLabels:
          app: nginx
      template:
        metadata:
          labels:
            app: nginx
        spec:
          containers:
          - name: test-nginx
            image: nginx:alpine
            ports:
            - containerPort: 80

b. After applying the YAML file, we see that two PODs are up and running. Each POD is running nginx application service on port 80

    user@vsphere:~/svc-pod$ kubectl apply -f 01-pod.yaml 
    deployment.apps/my-nginx-deploy created

    user@vsphere:~/svc-pod$ kubectl get pods 
    NAME                              READY   STATUS    RESTARTS   AGE
    my-nginx-deploy-ff84bd58d-8xkp6   1/1     Running   0          50s
    my-nginx-deploy-ff84bd58d-nc696   1/1     Running   0          50s

c. The IPs assigned to the pods are

    user@vsphere:~/svc-pod$ kubectl describe pod my-nginx-deploy-ff84bd58d-8xkp6 | grep -i IP: | head -1
    IP:               10.85.0.48
    user@vsphere:~/svc-pod$ kubectl describe pod my-nginx-deploy-ff84bd58d-nc696 | grep -i IP: | head -1
    IP:               10.85.0.49

d. Can Pods above talk to each other? Let’s find out. We will try to connect to a pod and try reaching the other pod from there

    user@vsphere:~/svc-pod$ kubectl get pods 
    NAME                              READY   STATUS    RESTARTS   AGE
    my-nginx-deploy-ff84bd58d-8xkp6   1/1     Running   0          50s
    my-nginx-deploy-ff84bd58d-nc696   1/1     Running   0          50s

    user@vsphere:~/svc-pod$ kubectl exec my-nginx-deploy-ff84bd58d-8xkp6  -it sh
    kubectl exec [POD] [COMMAND] is DEPRECATED and will be removed in a future version. Use kubectl exec [POD] -- [COMMAND] instead.
    / # apk add curl
    fetch https://dl-cdn.alpinelinux.org/alpine/v3.18/main/x86_64/APKINDEX.tar.gz
    fetch https://dl-cdn.alpinelinux.org/alpine/v3.18/community/x86_64/APKINDEX.tar.gz
    OK: 45 MiB in 64 packages
    / # curl http://10.85.0.48
    <!DOCTYPE html>
    <html>
    <head>
    <title>Welcome to nginx!</title>
    <style>
    html { color-scheme: light dark; }
    body { width: 35em; margin: 0 auto;
    font-family: Tahoma, Verdana, Arial, sans-serif; }
    </style>
    </head>
    <body>
    <h1>Welcome to nginx!</h1>
    <p>If you see this page, the nginx web server is successfully installed and
    working. Further configuration is required.</p>

    <p>For online documentation and support please refer to
    <a href="http://nginx.org/">nginx.org</a>.<br/>
    Commercial support is available at
    <a href="http://nginx.com/">nginx.com</a>.</p>

    <p><em>Thank you for using nginx.</em></p>
    </body>
    </html>

e. Let’s create a NodePort service.

    user@vsphere:~/svc-pod$ vim svc-np.yaml 
    apiVersion: v1
    kind: Service
    metadata:
      name: my-service
    spec:
      type: NodePort
      selector:
        app: nginx
      ports:
      - port: 80
        targetPort: 80
        nodePort: 30007

f. Let’s apply the YAML file and validate the service gets created

    user@vsphere:~/svc-pod$ kubectl apply -f svc-np.yaml 
    service/pod-svc created

    user@vsphere:~/svc-pod$ kubectl get svc
    NAME         TYPE        CLUSTER-IP     EXTERNAL-IP   PORT(S)        AGE
    kubernetes   ClusterIP   10.96.0.1      <none>        443/TCP        4h16m
    pod-svc      NodePort    10.111.8.230   <none>        80:30007/TCP   3s

g. And voila!! We can now reach the Nginx application hosted in the PODs from outside of the K8s cluster.

    user@vsphere:~/svc-pod$ curl http://172.25.128.2:30007
    <!DOCTYPE html>
    <html>
    <head>
    <title>Welcome to nginx!</title>
    <style>
    html { color-scheme: light dark; }
    body { width: 35em; margin: 0 auto;
    font-family: Tahoma, Verdana, Arial, sans-serif; }
    </style>
    </head>
    <body>
    <h1>Welcome to nginx!</h1>
    <p>If you see this page, the nginx web server is successfully installed and
    working. Further configuration is required.</p>

    <p>For online documentation and support please refer to
    <a href="http://nginx.org/">nginx.org</a>.<br/>
    Commercial support is available at
    <a href="http://nginx.com/">nginx.com</a>.</p>

    <p><em>Thank you for using nginx.</em></p>
    </body>
    </html>
    
# External IPs

2.a. Create second pod yaml. 

    user@vsphere:~/svc-pod$ cat 02-pod.yaml
    apiVersion: apps/v1
    kind: Deployment
    metadata:
      name: nginx-deployment
      labels: 
        app: nginx 
    spec:
      replicas: 1 
      selector: 
        matchLabels: 
          app: nginx
      template:
        metadata:
          labels:
            app: nginx
        spec:
          containers:
          - name: nginx
            image: nginx:1.7.9
            ports:
            - containerPort: 80
    ---
    kind: Service
    apiVersion: v1
    metadata:
      name: nginx-service
    spec:
      type: ClusterIP
      selector:
        app: nginx
      ports:
        - name: http
          protocol: TCP
          port: 80
          targetPort: 80
      externalIPs: 
        - 192.168.0.129

b. Apply kubernetes 02-pod.yaml. 

    user@vsphere:~/svc-pod$ kubectl apply -f 02-pod.yaml
    deployment.apps/nginx-deployment created
    service/nginx-service created

c. Getting service status. 

    user@vsphere:~/svc-pod$ kubectl get svc
    NAME            TYPE        CLUSTER-IP     EXTERNAL-IP     PORT(S)        AGE
    kubernetes      ClusterIP   10.96.0.1      <none>          443/TCP        4h34m
    nginx-service   ClusterIP   10.98.35.236   192.168.0.129   80/TCP         3m14s

d. Getting curl status using externalIPs.

    user@vsphere:~/svc-pod$ curl http://192.168.0.129
    <!DOCTYPE html>
    <html>
    <head>
    <title>Welcome to nginx!</title>
    <style>
    html { color-scheme: light dark; }
    body { width: 35em; margin: 0 auto;
    font-family: Tahoma, Verdana, Arial, sans-serif; }
    </style>
    </head>
    <body>
    <h1>Welcome to nginx!</h1>
    <p>If you see this page, the nginx web server is successfully installed and
    working. Further configuration is required.</p>

    <p>For online documentation and support please refer to
    <a href="http://nginx.org/">nginx.org</a>.<br/>
    Commercial support is available at
    <a href="http://nginx.com/">nginx.com</a>.</p>

    <p><em>Thank you for using nginx.</em></p>
    </body>
    </html>