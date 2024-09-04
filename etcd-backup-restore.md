### 1. For this question, please set this context (In exam, diff cluster name)

            kubectl config use-context kubernetes-admin@kubernetes

### etcd-controlplane pod is running in kube-system environment, take backup and store it in /opt/cluster_backup.db file.

### ETCD backup is stored at the path /opt/cluster_backup.db on the controlplane node. For -- data-dir use /root/default.etcd , restore it on the controlplane node itself and , and also store restore console output store it in restore.txt.

            $ k get pods -n kube-system
            NAME                                       READY   STATUS    RESTARTS        AGE
            calico-kube-controllers-75bdb5b75d-kh2wj   1/1     Running   2 (8m37s ago)   33d
            canal-rc7zk                                2/2     Running   2 (8m36s ago)   33d
            canal-rnmhk                                2/2     Running   2 (8m37s ago)   33d
            coredns-5c69dbb7bd-nzg9k                   1/1     Running   1 (8m37s ago)   33d
            coredns-5c69dbb7bd-pk85q                   1/1     Running   1 (8m37s ago)   33d
            etcd-controlplane                          1/1     Running   2 (8m36s ago)   33d
            kube-apiserver-controlplane                1/1     Running   2 (8m37s ago)   33d
            kube-controller-manager-controlplane       1/1     Running   2 (8m36s ago)   33d
            kube-proxy-6xsx9                           1/1     Running   2 (8m36s ago)   33d
            kube-proxy-vrwqx                           1/1     Running   1 (8m37s ago)   33d
            kube-scheduler-controlplane                1/1     Running   2 (8m36s ago)   33d

            $ k describe pod etcd-controlplane -n kube-system
            Name:                 etcd-controlplane
            Namespace:            kube-system
            Priority:             2000001000
            Priority Class Name:  system-node-critical
            Node:                 controlplane/172.30.1.2
            Start Time:           Wed, 04 Sep 2024 13:41:47 +0000
            Labels:               component=etcd
                                tier=control-plane
            Annotations:          kubeadm.kubernetes.io/etcd.advertise-client-urls: https://172.30.1.2:2379
                                kubernetes.io/config.hash: c34b114b31567febb45d725bc3b61442
                                kubernetes.io/config.mirror: c34b114b31567febb45d725bc3b61442
                                kubernetes.io/config.seen: 2024-08-02T07:25:56.078698403Z
                                kubernetes.io/config.source: file
            Status:               Running
            SeccompProfile:       RuntimeDefault
            IP:                   172.30.1.2
            IPs:
            IP:           172.30.1.2
            Controlled By:  Node/controlplane
            Containers:
            etcd:
                Container ID:  containerd://cd57ac21d95c044772831ec5166c1e96c0f071474d369ba52af19ce2e9eb3834
                Image:         registry.k8s.io/etcd:3.5.12-0
                Image ID:      registry.k8s.io/etcd@sha256:44a8e24dcbba3470ee1fee21d5e88d128c936e9b55d4bc51fbef8086f8ed123b
                Port:          <none>
                Host Port:     <none>
                Command:
                etcd
                --advertise-client-urls=https://172.30.1.2:2379
                --cert-file=/etc/kubernetes/pki/etcd/server.crt
                --client-cert-auth=true
                --data-dir=/var/lib/etcd
                --experimental-initial-corrupt-check=true
                --experimental-watch-progress-notify-interval=5s
                --initial-advertise-peer-urls=https://172.30.1.2:2380
                --initial-cluster=controlplane=https://172.30.1.2:2380
                --key-file=/etc/kubernetes/pki/etcd/server.key
                --listen-client-urls=https://127.0.0.1:2379,https://172.30.1.2:2379
                --listen-metrics-urls=http://127.0.0.1:2381
                --listen-peer-urls=https://172.30.1.2:2380
                --name=controlplane
                --peer-cert-file=/etc/kubernetes/pki/etcd/peer.crt
                --peer-client-cert-auth=true
                --peer-key-file=/etc/kubernetes/pki/etcd/peer.key
                --peer-trusted-ca-file=/etc/kubernetes/pki/etcd/ca.crt
                --snapshot-count=10000
                --trusted-ca-file=/etc/kubernetes/pki/etcd/ca.crt
                State:          Running

            $ ETCDCTL_API=3 etcdctl snapshot save --cacert /etc/kubernetes/pki/etcd/ca.crt --cert /etc/kubernetes/pki/etcd/server.crt --key /etc/kubernetes/pki/etcd/server.key /opt/cluster_backup.db
            {"level":"info","ts":1725457952.8177009,"caller":"snapshot/v3_snapshot.go:68","msg":"created temporary db file","path":"/opt/cluster_backup.db.part"}
            {"level":"info","ts":1725457952.8518095,"logger":"client","caller":"v3/maintenance.go:211","msg":"opened snapshot stream; downloading"}
            {"level":"info","ts":1725457952.852539,"caller":"snapshot/v3_snapshot.go:76","msg":"fetching snapshot","endpoint":"127.0.0.1:2379"}
            {"level":"info","ts":1725457953.0353334,"logger":"client","caller":"v3/maintenance.go:219","msg":"completed snapshot read; closing"}
            {"level":"info","ts":1725457953.053361,"caller":"snapshot/v3_snapshot.go:91","msg":"fetched snapshot","endpoint":"127.0.0.1:2379","size":"4.4 MB","took":"now"}
            {"level":"info","ts":1725457953.054401,"caller":"snapshot/v3_snapshot.go:100","msg":"saved","path":"/opt/cluster_backup.db"}
            Snapshot saved at /opt/cluster_backup.db

            $ ETCDCTL_API=3 etcdctl snapshot restore /opt/cluster_backup.db --cacert /etc/kubernetes/pki/etcd/ca.crt --cert /etc/kubernetes/pki/etcd/server.crt --key /etc/kubernetes/pki/etcd/server.key --data-dir /root/default.etcd > restore.txt 2>&1

### 2. The master node in our cluster is planned for a regular maintenance reboot tonight. While we do not anticipate anything to go wrong, we are required to take the necessary backups. Take a snapshot of the ETCD database using the built-in snapshot functionality. Store the backup file at location /opt/snapshot-pre-boot.db.

### After the maintenance you done find any pods, deployments or services running. Restore the original state of the cluster using the backup file.

            $ ETCDCTL_API=3 etcdctl snapshot save /opt/snapshot-pre-boot.db --cacert /etc/kubernetes/pki/etcd/ca.crt --cert /etc/kubernetes/pki/etcd/server.crt --key /etc/kubernetes/pki/etcd/server.key
            Snapshot saved at /opt/snapshot-pre-boot.db
            $

            $ ETCDCTL_API=3 etcdctl snapshot restore /opt/snapshot-pre-boot.db --cacert /etc/kubernetes/pki/etcd/ca.crt --cert /etc/kubernetes/pki/etcd/server.crt --key /etc/kubernetes/pki/etcd/server.key --data-dir /var/etcd-from-backup
            2024-09-04 14:52:17.371135 I | etcdserver/membership: added member 8e9e05c52164694d [http://localhost:2380] to cluster cdf818194e3a8c32


Update yaml file for etcd in the volumes part.

            $ cd /etc/kubernetes/manifests/
            /etc/kubernetes/manifests $  vi etcd.yaml 
                            volumes:
                            - hostPath:
                                path: /var/etcd-from-backup
                                type: DirectoryOrCreate
                                name: etcd-data


### 3. An ETCD backup for cluster2 is stored at /opt/cluster2.db . Use this snapshot file to carryout a restore on cluster2 to a new path /var/lib/etcd-data-new . Once the restore is complete, ensure that the controlplane components on cluster2 are running.

            $ kubectl config get-clusters
            NAME
            cluster1
            cluster2

            $ kubectl config use-context cluster2
            Switched to context "cluster2".

            $ ssh cluster2-controlplane
            Welcome to Ubuntu 23.10 (GNU/Linux 5.4.0-1106-gcp x86_64)

            * Documentation:  https://help.ubuntu.com
            * Management:     https://landscape.canonical.com
            * Support:        https://ubuntu.com/pro

            This system has been minimized by removing packages and content that are
            not required on a system that users do not log into.

            To restore this content, you can run the 'unminimize' command.
            Last login: Wed Sep  4 15:50:16 2024 from 192.34.197.15

            cluster2-controlplane $  ps -ef | grep etcd
            root        2822    2465  0 15:01 ?        00:01:55 kube-apiserver --advertise-address=192.34.197.21 --allow-privileged=true --authorization-mode=Node,RBAC --client-ca-file=/etc/kubernetes/pki/ca.crt --enable-admission-plugins=NodeRestriction --enable-bootstrap-token-auth=true --etcd-cafile=/etc/kubernetes/pki/etcd/ca.pem --etcd-certfile=/etc/kubernetes/pki/etcd/etcd.pem --etcd-keyfile=/etc/kubernetes/pki/etcd/etcd-key.pem --etcd-servers=https://192.34.197.9:2379 --kubelet-client-certificate=/etc/kubernetes/pki/apiserver-kubelet-client.crt --kubelet-client-key=/etc/kubernetes/pki/apiserver-kubelet-client.key --kubelet-preferred-address-types=InternalIP,ExternalIP,Hostname --proxy-client-cert-file=/etc/kubernetes/pki/front-proxy-client.crt --proxy-client-key-file=/etc/kubernetes/pki/front-proxy-client.key --requestheader-allowed-names=front-proxy-client --requestheader-client-ca-file=/etc/kubernetes/pki/front-proxy-ca.crt --requestheader-extra-headers-prefix=X-Remote-Extra- --requestheader-group-headers=X-Remote-Group --requestheader-username-headers=X-Remote-User --secure-port=6443 --service-account-issuer=https://kubernetes.default.svc.cluster.local --service-account-key-file=/etc/kubernetes/pki/sa.pub --service-account-signing-key-file=/etc/kubernetes/pki/sa.key --service-cluster-ip-range=10.96.0.0/12 --tls-cert-file=/etc/kubernetes/pki/apiserver.crt --tls-private-key-file=/etc/kubernetes/pki/apiserver.key
            root        9713    9629  0 15:56 pts/0    00:00:00 grep etcd

            cluster2-controlplane $ exit
            $ scp /opt/cluster2.db 192.34.197.9:/opt
            cluster2.db                                100% 2188KB  99.4MB/s   00:00  

            $ ssh 192.34.197.9
            Welcome to Ubuntu 18.04.6 LTS (GNU/Linux 5.4.0-1106-gcp x86_64)

            * Documentation:  https://help.ubuntu.com
            * Management:     https://landscape.canonical.com
            * Support:        https://ubuntu.com/advantage
            This system has been minimized by removing packages and content that are
            not required on a system that users do not log into.

            To restore this content, you can run the 'unminimize' command.
            Last login: Wed Sep  4 15:50:49 2024 from 192.34.197.16

            etcd-server $ ETCDCTL_API=3 etcdctl snapshot restore /opt/cluster2.db --data-dir /var/lib/etcd-data-new
            {"level":"info","ts":1725465584.5727837,"caller":"snapshot/v3_snapshot.go:296","msg":"restoring snapshot","path":"/opt/cluster2.db","wal-dir":"/var/lib/etcd-data-new/member/wal","data-dir":"/var/lib/etcd-data-new","snap-dir":"/var/lib/etcd-data-new/member/snap"}
            {"level":"info","ts":1725465584.58964,"caller":"mvcc/kvstore.go:388","msg":"restored last compact revision","meta-bucket-name":"meta","meta-bucket-name-key":"finishedCompactRev","restored-compact-revision":4281}
            {"level":"info","ts":1725465584.5962706,"caller":"membership/cluster.go:392","msg":"added member","cluster-id":"cdf818194e3a8c32","local-member-id":"0","added-peer-id":"8e9e05c52164694d","added-peer-peer-urls":["http://localhost:2380"]}
            {"level":"info","ts":1725465584.617922,"caller":"snapshot/v3_snapshot.go:309","msg":"restored snapshot","path":"/opt/cluster2.db","wal-dir":"/var/lib/etcd-data-new/member/wal","data-dir":"/var/lib/etcd-data-new","snap-dir":"/var/lib/etcd-data-new/member/snap"}
            etcd-server $ 


            drwx------ 3 root root 4096 Sep  4 15:59 etcd-data-new

            etcd-server /var/lib $  pwd
            /var/lib

            etcd-server $ chown -R etcd:etcd etcd-data-new

            etcd-server $ ls -lrt /var/lib/etcd-data-new
            total 4
            drwx------ 4 etcd etcd 4096 Sep  4 15:59 member

            etcd-server $  systemctl status etcd
            ● etcd.service - etcd key-value store
            Loaded: loaded (/etc/systemd/system/etcd.service; enabled; vendor preset: 
            enabled)
            Active: active (running) since Wed 2024-09-04 15:00:46 UTC; 1h 
            etcd-server $ vi /etc/systemd/system/etcd.service
                [Unit]
                Description=etcd key-value store
                Documentation=https://github.com/etcd-io/etcd
                After=network.target

                [Service]
                User=etcd
                Type=notify
                ExecStart=/usr/local/bin/etcd \
                --name etcd-server \
                --data-dir=/var/lib/etcd-data-new \

            etcd-server $ systemctl daemon-reload

            etcd-server $  systemctl restart etcd

            etcd-server $ ps -ef | grep etcd
            etcd        2234       1  0 16:05 ?        00:00:01 /usr/local/bin/etcd --name etcd-server --data-dir=/var/lib/etcd-data-new --cert-file=/etc/etcd/pki/etcd.pem --key-file=/etc/etcd/pki/etcd-key.pem --peer-cert-file=/etc/etcd/pki/etcd.pem --peer-key-file=/etc/etcd/pki/etcd-key.pem --trusted-ca-file=/etc/etcd/pki/ca.pem --peer-trusted-ca-file=/etc/etcd/pki/ca.pem --peer-client-cert-auth --client-cert-auth --initial-advertise-peer-urls https://192.34.197.9:2380 --listen-peer-urls https://192.34.197.9:2380 --advertise-client-urls https://192.34.197.9:2379 --listen-client-urls https://192.34.197.9:2379,https://127.0.0.1:2379 --initial-cluster-token etcd-cluster-1 --initial-cluster etcd-server=https://192.34.197.9:2380 --initial-cluster-state new
            root        2324    1340  0 16:06 pts/1    00:00:00 grep etcd

            etcd-server $  date
            Wed Sep  4 16:06:34 UTC 2024  



