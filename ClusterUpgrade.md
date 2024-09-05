### 1. For this question, please set this context (In exam, diff cluster name)
### kubectl config use-context kubernetes-admin@kubernetes

### Upgrade controlplane node kubeadm , cluster and kubelet to next version. EXAMPLE: If current version is v1.27.1 then upgrade to v1.27.2


         $ kubectl config use-context kubernetes-admin@kubernetes
         Switched to context "kubernetes-admin@kubernetes".
         $ kubeadm version
         kubeadm version: &version.Info{Major:"1", Minor:"30", GitVersion:"v1.30.0", GitCommit:"7c48c2bd72b9bf5c44d21d7338cc7bea77d0ad2a", GitTreeState:"clean", BuildDate:"2024-04-17T17:34:08Z", GoVersion:"go1.22.2", Compiler:"gc", Platform:"linux/amd64"}
         $ 

So we have to upgrade to 1.30.1


         $ k drain controlplane --ignore-daemonsets
         node/controlplane is cordoned
         Warning: ignoring DaemonSet-managed Pods: kube-system/canal-rc7zk, kube-system/kube-proxy-6xsx9
         evicting pod local-path-storage/local-path-provisioner-75655fcf79-s7bbc
         evicting pod kube-system/calico-kube-controllers-75bdb5b75d-kh2wj
         pod/calico-kube-controllers-75bdb5b75d-kh2wj evicted
         pod/local-path-provisioner-75655fcf79-s7bbc evicted
         node/controlplane drained
         $ 

         $ echo "deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.30/deb/ /" | sudo tee /etc/apt/sources.list.d/kubernetes.list
         deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.30/deb/ /

         $ curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.30/deb/Release.key | sudo gpg --dearmor -o /e
         tc/apt/keyrings/kubernetes-apt-keyring.gpg

         $ sudo apt-get update
         Hit:1 http://security.ubuntu.com/ubuntu focal-security InRelease
         Hit:3 http://ppa.launchpad.net/rmescandon/yq/ubuntu focal InRelease 
         Hit:4 http://archive.ubuntu.com/ubuntu focal InRelease
         Hit:2 https://prod-cdn.packages.k8s.io/repositories/isv:/kubernetes:/core:/stable:/v1.30/deb  InRelease
         Hit:5 http://archive.ubuntu.com/ubuntu focal-updates InRelease   
         Hit:6 http://archive.ubuntu.com/ubuntu focal-backports InRelease
         Reading package lists... Done
         $ 

         $ sudo apt-cache madison kubeadm
            kubeadm | 1.30.4-1.1 | https://pkgs.k8s.io/core:/stable:/v1.30/deb  Packages
            kubeadm | 1.30.3-1.1 | https://pkgs.k8s.io/core:/stable:/v1.30/deb  Packages
            kubeadm | 1.30.2-1.1 | https://pkgs.k8s.io/core:/stable:/v1.30/deb  Packages
            kubeadm | 1.30.1-1.1 | https://pkgs.k8s.io/core:/stable:/v1.30/deb  Packages
            kubeadm | 1.30.0-1.1 | https://pkgs.k8s.io/core:/stable:/v1.30/deb  Packages

We will be upgrading to 1.30.1-1.1.

         $ sudo apt-mark unhold kubeadm
            kubeadm was already not hold.

         $ sudo apt-get update && sudo apt-get install -y kubeadm='1.30.1-1.1'
            Hit:1 http://security.ubuntu.com/ubuntu focal-security InRelease
            Hit:2 http://archive.ubuntu.com/ubuntu focal InRelease              
            Hit:4 http://archive.ubuntu.com/ubuntu focal-updates InRelease      
            Hit:3 https://prod-cdn.packages.k8s.io/repositories/isv:/kubernetes:/core:/stable:/v1.30/deb  InRelease
            Hit:5 http://archive.ubuntu.com/ubuntu focal-backports InRelease                       
            Hit:6 http://ppa.launchpad.net/rmescandon/yq/ubuntu focal InRelease
            Reading package lists... Done
            Reading package lists... Done
            Building dependency tree       
            Reading state information... Done
            The following packages will be upgraded:
            kubeadm
            1 upgraded, 0 newly installed, 0 to remove and 181 not upgraded.
            Need to get 10.4 MB of archives.
            After this operation, 0 B of additional disk space will be used.

         $ sudo apt-mark hold kubeadm
         kubeadm set on hold.

         $ kubeadm version
         kubeadm version: &version.Info{Major:"1", Minor:"30", GitVersion:"v1.30.1", GitCommit:"6911225c3f747e1cd9d109c305436d08b668f086", GitTreeState:"clean", BuildDate:"2024-05-14T10:49:05Z", GoVersion:"go1.22.2", Compiler:"gc", Platform:"linux/amd64"}
         $ 

         $ sudo kubeadm upgrade plan
         [preflight] Running pre-flight checks.
         [upgrade/config] Reading configuration from the cluster...
         [upgrade/config] FYI: You can look at this config file with 'kubectl -n kube-system get cm kubeadm-config -o yaml'
         [upgrade] Running cluster health checks
         [upgrade] Fetching available versions to upgrade to
         [upgrade/versions] Cluster version: 1.30.0
         [upgrade/versions] kubeadm version: v1.30.1
         I0905 11:57:37.854225   10582 version.go:256] remote version is much newer: v1.31.0; falling back to: stable-1.30
         [upgrade/versions] Target version: v1.30.4
         [upgrade/versions] Latest version in the v1.30 series: v1.30.4

         Components that must be upgraded manually after you have upgraded the control plane with 'kubeadm upgrade apply':
         COMPONENT   NODE           CURRENT   TARGET
         kubelet     controlplane   v1.30.0   v1.30.4
         kubelet     node01         v1.30.0   v1.30.4

         Upgrade to the latest version in the v1.30 series:

         COMPONENT                 NODE           CURRENT    TARGET
         kube-apiserver            controlplane   v1.30.0    v1.30.4
         kube-controller-manager   controlplane   v1.30.0    v1.30.4
         kube-scheduler            controlplane   v1.30.0    v1.30.4
         kube-proxy                               1.30.0     v1.30.4
         CoreDNS                                  v1.11.1    v1.11.1
         etcd                      controlplane   3.5.12-0   3.5.12-0

         You can now apply the upgrade by executing the following command:

               kubeadm upgrade apply v1.30.4

         $ kubeadm upgrade apply v1.30.1
         [preflight] Running pre-flight checks.
         [upgrade/config] Reading configuration from the cluster...
         [upgrade/config] FYI: You can look at this config file with 'kubectl -n kube-system get cm kubeadm-config -o yaml'
         [upgrade] Running cluster health checks
         [upgrade/version] You have chosen to change the cluster version to "v1.30.1"
         [upgrade/versions] Cluster version: v1.30.0
         [upgrade/versions] kubeadm version: v1.30.1
         [upgrade] Are you sure you want to proceed? [y/N]: y
         [upgrade/prepull] Pulling images required for setting up a Kubernetes cluster


         [upgrade/successful] SUCCESS! Your cluster was upgraded to "v1.30.1". Enjoy!

         [upgrade/kubelet] Now that your control plane is upgraded, please proceed with upgrading your kubelets if you haven't already done so.

         $ sudo apt-mark unhold kubelet kubectl
         kubelet was already not hold.
         kubectl was already not hold.
         controlplane $ sudo apt-get update && sudo apt-get install -y kubelet='1.30.1-1.1' kubectl='1.30.1-1.1'
         Hit:2 http://security.ubuntu.com/ubuntu focal-security InRelease    
         Hit:3 http://archive.ubuntu.com/ubuntu focal InRelease              
         Hit:1 https://prod-cdn.packages.k8s.io/repositories/isv:/kubernetes:/core:/stable:/v1.30/deb  InRelease
         Hit:4 http://archive.ubuntu.com/ubuntu focal-updates InRelease                         
         Hit:5 http://archive.ubuntu.com/ubuntu focal-backports InRelease
         Hit:6 http://ppa.launchpad.net/rmescandon/yq/ubuntu focal InRelease
         Reading package lists... Done
         Reading package lists... Done
         Building dependency tree       
         Reading state information... Done
         The following packages will be upgraded:
         kubectl kubelet
         2 upgraded, 0 newly installed, 0 to remove and 180 not upgraded.

         $ sudo apt-mark hold kubelet kubectl
         kubelet set on hold.
         kubectl set on hold.

         $ sudo systemctl daemon-reload
         $ sudo systemctl restart kubelet
         $ kubectl uncordon controlplane
         node/controlplane uncordoned
         $ 

         $ kubectl get nodes
         NAME           STATUS   ROLES           AGE   VERSION
         controlplane   Ready    control-plane   34d   v1.30.1
         node01         Ready    <none>          34d   v1.30.0
         controlplane $ kubeadm upgrade plan
         [preflight] Running pre-flight checks.
         [upgrade/config] Reading configuration from the cluster...
         [upgrade/config] FYI: You can look at this config file with 'kubectl -n kube-system get cm kubeadm-config -o yaml'
         [upgrade] Running cluster health checks
         [upgrade] Fetching available versions to upgrade to
         [upgrade/versions] Cluster version: 1.30.1
         [upgrade/versions] kubeadm version: v1.30.1
         I0905 12:07:10.858923   15968 version.go:256] remote version is much newer: v1.31.0; falling back to: stable-1.30
         [upgrade/versions] Target version: v1.30.4
         [upgrade/versions] Latest version in the v1.30 series: v1.30.4

         Components that must be upgraded manually after you have upgraded the control plane with 'kubeadm upgrade apply':
         COMPONENT   NODE           CURRENT   TARGET
         kubelet     node01         v1.30.0   v1.30.4
         kubelet     controlplane   v1.30.1   v1.30.4


