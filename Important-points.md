
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

__To switch to a cluster context__

        $ kubectl config use-context cluster1

__To know the number of nodes in a etcd cluster__

        $ ETCDCTL_API=3 etcdctl --endpoints=https://127.0.0.1:2379 --cacert=/etc/etcd/pki/ca.pem --cert=/etc/etcd/pki/etcd.pem --key=/etc/etcd/pki/etcd-key.pem  member list

__To list the pods sorted by creation time__
        
        $ kubectl get pods --sort-by=.metadata.creationTimestamp

__To sort the pods in descending order__

        $ kubectl get pods --sort-by=.metadata.creationTimestamp | tac


