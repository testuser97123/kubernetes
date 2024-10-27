## Deploy Pod Network

To enable networking between pods across the cluster, deploy a pod network. For example, deploy the Tigera Operator for Calico:

    [root@test-vm-01 ~]# kubectl create -f https://raw.githubusercontent.com/projectcalico/calico/v3.26.1/manifests/tigera-operator.yaml
    
    namespace/tigera-operator created
    customresourcedefinition.apiextensions.k8s.io/bgpconfigurations.crd.projectcalico.org created
    customresourcedefinition.apiextensions.k8s.io/bgpfilters.crd.projectcalico.org created
    customresourcedefinition.apiextensions.k8s.io/bgppeers.crd.projectcalico.org created
    customresourcedefinition.apiextensions.k8s.io/blockaffinities.crd.projectcalico.org created
    customresourcedefinition.apiextensions.k8s.io/caliconodestatuses.crd.projectcalico.org created
    customresourcedefinition.apiextensions.k8s.io/clusterinformations.crd.projectcalico.org created
    customresourcedefinition.apiextensions.k8s.io/felixconfigurations.crd.projectcalico.org created
    customresourcedefinition.apiextensions.k8s.io/globalnetworkpolicies.crd.projectcalico.org created
    customresourcedefinition.apiextensions.k8s.io/globalnetworksets.crd.projectcalico.org created
    customresourcedefinition.apiextensions.k8s.io/hostendpoints.crd.projectcalico.org created
    customresourcedefinition.apiextensions.k8s.io/ipamblocks.crd.projectcalico.org created
    customresourcedefinition.apiextensions.k8s.io/ipamconfigs.crd.projectcalico.org created
    customresourcedefinition.apiextensions.k8s.io/ipamhandles.crd.projectcalico.org created
    customresourcedefinition.apiextensions.k8s.io/ippools.crd.projectcalico.org created
    customresourcedefinition.apiextensions.k8s.io/ipreservations.crd.projectcalico.org created
    customresourcedefinition.apiextensions.k8s.io/kubecontrollersconfigurations.crd.projectcalico.org created
    customresourcedefinition.apiextensions.k8s.io/networkpolicies.crd.projectcalico.org created
    customresourcedefinition.apiextensions.k8s.io/networksets.crd.projectcalico.org created
    customresourcedefinition.apiextensions.k8s.io/apiservers.operator.tigera.io created
    customresourcedefinition.apiextensions.k8s.io/imagesets.operator.tigera.io created
    customresourcedefinition.apiextensions.k8s.io/installations.operator.tigera.io created
    customresourcedefinition.apiextensions.k8s.io/tigerastatuses.operator.tigera.io created
    serviceaccount/tigera-operator created
    clusterrole.rbac.authorization.k8s.io/tigera-operator created
    clusterrolebinding.rbac.authorization.k8s.io/tigera-operator created
    deployment.apps/tigera-operator created

To download the custom Calico resources manifest, you can use the curl or wget command to fetch the YAML file from the Calico project’s GitHub repository. Here’s how you can do it using curl:

    [root@test-vm-01 ~]# curl -O https://raw.githubusercontent.com/projectcalico/calico/v3.26.1/manifests/custom-resources.yaml

Using wget:

    [root@test-vm-01 ~]# wget https://raw.githubusercontent.com/projectcalico/calico/v3.26.1/manifests/custom-resources.yaml

Adjust the CIDR setting in the custom resources file:

    [root@test-vm-01 ~]# sed -i 's/cidr: 192\.168\.0\.0\/16/cidr: 10.244.0.0\/16/g' custom-resources.yaml

Finally, create the Calico custom resources:

    [root@test-vm-01 ~]# kubectl create -f custom-resources.yaml
    installation.operator.tigera.io/default created
    apiserver.operator.tigera.io/default created