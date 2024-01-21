user@vsphere:~/rsc-quota$ kubectl create -f quota-demo-ns.yaml
namespace/quota-demo created

user@vsphere:~/rsc-quota$ kubectl config set-context --current --namespace=quota-demo
Context "kubernetes-admin@kubernetes" modified.


user@vsphere:~/rsc-quota$ kubectl get pods
No resources found in quota-demo namespace.

user@vsphere:~/rsc-quota$ vim cpu-quota.yaml
apiVersion: v1
kind: ResourceQuota
metadata:
  name: test-cpu-quota
  namespace: quota-demo
spec:
  hard:
    requests.cpu: "200m"
    limits.cpu: "300m"


user@vsphere:~/rsc-quota$ kubectl apply -f cpu-quota.yaml 
resourcequota/test-cpu-quota created

user@vsphere:~/rsc-quota$ kubectl describe resourcequota/test-cpu-quota
Name:         test-cpu-quota
Namespace:    quota-demo
Resource      Used  Hard
--------      ----  ----
limits.cpu    0     300m
requests.cpu  0     200m

user@vsphere:~/rsc-quota$ kubectl describe ns quota-demo
Name:         quota-demo
Labels:       kubernetes.io/metadata.name=quota-demo
Annotations:  <none>
Status:       Active

Resource Quotas
  Name:         test-cpu-quota
  Resource      Used  Hard
  --------      ---   ---
  limits.cpu    0     300m
  requests.cpu  0     200m

No LimitRange resource.

user@vsphere:~/rsc-quota$ vim testpod1.yaml
apiVersion: v1
kind: Pod
metadata:
  name: testpod1
spec:
  containers:
  - name: quota-test
    image: busybox
    imagePullPolicy: IfNotPresent
    command: ['sh', '-c', 'echo Pod is Running ; sleep 5000']
    resources:
      requests:
        cpu: "100m"
      limits:
        cpu: "200m"
  restartPolicy: Never

user@vsphere:~/rsc-quota$ kubectl apply -f testpod1.yaml
pod/testpod1 created

user@vsphere:~/rsc-quota$ kubectl get pods
NAME       READY   STATUS    RESTARTS   AGE
testpod1   1/1     Running   0          38s

user@vsphere:~/rsc-quota$ kubectl describe resourcequota/test-cpu-quota
Name:         test-cpu-quota
Namespace:    quota-demo
Resource      Used  Hard
--------      ----  ----
limits.cpu    200m  300m
requests.cpu  100m  200m

user@vsphere:~/rsc-quota$ vim testpod2.yaml
apiVersion: v1
kind: Pod
metadata:
  name: testpod2
spec:
  containers:
  - name: quota-test
    image: busybox
    imagePullPolicy: IfNotPresent
    command: ['sh', '-c', 'echo Pod is Running ; sleep 5000']
    resources:
      requests:
        cpu: "200m"
      limits:
        cpu: "200m"
  restartPolicy: Never


user@vsphere:~/rsc-quota$ kubectl apply -f testpod2.yaml 
Error from server (Forbidden): error when creating "testpod2.yaml": pods "testpod2" is forbidden: exceeded quota: test-cpu-quota, requested: limits.cpu=200m,requests.cpu=200m, used: limits.cpu=200m,requests.cpu=100m, limited: limits.cpu=300m,requests.cpu=200m