
__Taint a node__

        $ k taint nodes controlplane node-role.kubernetes.io/master:NoSchedule
        node/controlplane tainted
        $ k describe node controlplane | grep Taints
        Taints:             node-role.kubernetes.io/master:NoSchedule
        $ k taint nodes node01 node-role.kubernetes.io/node:NoSchedule
        node/node01 tainted

__To get the taint details of nodes__

        $ kubectl get nodes -o jsonpath='{.items[*].spec.taints}' 

__To unjoin a code from the cluster__

        $ kubectl drain node01
        $ ssh node01
        node01 $ kubeadm reset
               $ exit
        $ kubectl delete node node01 

__To get the number of clusters in kubeconfig__

        $ kubectl config get-clusters

