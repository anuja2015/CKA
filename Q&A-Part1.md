
### 1. Deploy a pod called nginxpod with image nginx in controlplane, Make sure pod is not scheduled in worker node.

          $ kubectl run nginxpod --image nginx --dry-run=client -o yaml > nginx.yaml

                      apiVersion: v1
                      kind: Pod
                      metadata:
                        creationTimestamp: null
                        labels:
                          run: nginxpod
                        name: nginxpod
                      spec:
                        containers:
                        - image: nginx
                          name: nginxpod
                          resources: {}
                        dnsPolicy: ClusterFirst
                        restartPolicy: Always
                        nodeName: controlplane

          $ kubectl apply -f nginx.yaml
          pod/nginxpod created

          $ kubectl get po
          NAME       READY   STATUS              RESTARTS   AGE
          nginxpod   0/1     ContainerCreating   0          4s

          $ kubectl get po
          NAME       READY   STATUS    RESTARTS   AGE
          nginxpod   1/1     Running   0          30s

          $ kubectl get po -o wide
          NAME       READY   STATUS    RESTARTS   AGE   IP            NODE           NOMINATED NODE   READINESS GATES
          nginxpod   1/1     Running   0          36s   192.168.0.4   controlplane   <none>           <none>

### 2. Expose a existing pod called nginxpod as a service, service name should be nginxsvc. Pod port = 80

          $ alias k=kubectl
          $ k get svc
          NAME         TYPE        CLUSTER-IP   EXTERNAL-IP   PORT(S)   AGE
          kubernetes   ClusterIP   10.96.0.1    <none>        443/TCP   31d

          $ k expose pod nginxpod --name nginxsvc --port 80                         
          service/nginxsvc exposed

### 3. expose a existing pod called nginxpod, service name as nginxnodeportsvc, service should access through Nodeport. Nodeport= 30200

          $ k expose pod nginxpod --name nginxnodeportsvc --type NodePort --port 80 --dry-run=client -o yaml > nginxnodeportsvc.yaml
          $ vi nginxnodeportsvc.yaml 

                  apiVersion: v1
                  kind: Service
                  metadata:
                    creationTimestamp: null
                    labels:
                      run: nginxpod
                    name: nginxnodeportsvc
                  spec:
                    ports:
                    - port: 80
                      protocol: TCP
                      targetPort: 80
                      nodePort: 30200
                    selector:
                      run: nginxpod
                    type: NodePort
                  status:
                    loadBalancer: {}
          $ k apply -f nginxnodeportsvc.yaml 
            service/nginxnodeportsvc created
          $ k get svc -o wide
            NAME               TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)        AGE    SELECTOR
            kubernetes         ClusterIP   10.96.0.1        <none>        443/TCP        31d    <none>
            nginxnodeportsvc   NodePort    10.100.215.230   <none>        80:30200/TCP   7s     run=nginxpod
            nginxsvc           ClusterIP   10.96.165.139    <none>        80/TCP         9m8s   run=nginxpod

### 4. you can find an existing deployment frontend in production namespace, scale down the replicas to 2 and change the image to nginx:1.25

          $ k get deploy -n production
          NAME       READY   UP-TO-DATE   AVAILABLE   AGE
          frontend   3/3     3            3           9s

    ## __1st method__

          $ k edit deploy frontend -n production
          deployment.apps/frontend edited


          $ k get deploy -n production
          NAME       READY   UP-TO-DATE   AVAILABLE   AGE
          frontend   2/2     2            2           96s

          $ k describe deploy frontend -n production

           Replicas:               2 desired | 2 updated | 2 total | 2 available | 0 unavailable
           StrategyType:           RollingUpdate
           MinReadySeconds:        0
           RollingUpdateStrategy:  25% max unavailable, 25% max surge
           Pod Template:
             Labels:  app=frontend
             Containers:
             nginx:
               Image:         nginx:1.25

      ## __2nd method__

          $ k get deploy -n production
          NAME       READY   UP-TO-DATE   AVAILABLE   AGE
          frontend   3/3     3            3           8s

          $ k scale deploy frontend -n production --replicas=2
          deployment.apps/frontend scaled
          $ k set image deploy frontend -n production nginx=nginx:1.25
          deployment.apps/frontend image updated
          $ k describe deploy frontend -n production

          Replicas:               2 desired | 2 updated | 2 total | 2 available | 0 unavailable
          StrategyType:           RollingUpdate
          MinReadySeconds:        0
          RollingUpdateStrategy:  25% max unavailable, 25% max surge
          Pod Template:
            Labels:  app=frontend
            Containers:
            nginx:
              Image:         nginx:1.25
### 5. Auto scale the existing deployment frontend in production namespace at 80% of pod CPU usage, and set Minimum replicas= 3 and Maximum replicas= 5

          $ k get deploy -n production
          NAME       READY   UP-TO-DATE   AVAILABLE   AGE
          frontend   2/2     2            2           9m5s

          $ k autoscale deploy frontend -n production --min=3 --max=5 --cpu-percent=80
          horizontalpodautoscaler.autoscaling/frontend autoscaled

          $ k get hpa -n production
          NAME       REFERENCE             TARGETS              MINPODS   MAXPODS   REPLICAS   AGE
          frontend   Deployment/frontend   cpu: <unknown>/80%   3         5         0          9s

          $ k get deploy -n production
          NAME       READY   UP-TO-DATE   AVAILABLE   AGE
          frontend   3/3     3            3           10m
  
### 6. Expose existing deployment in production namespace named as frontend through Nodeport and Nodeport service name should be frontendsvc

          $ k expose deployment frontend --name frontendsvc -n production --type NodePort --port 80
          service/frontendsvc exposed

          $ k get svc frontendsvc -n production -o wide
          NAME          TYPE       CLUSTER-IP      EXTERNAL-IP   PORT(S)        AGE   SELECTOR
          frontendsvc   NodePort   10.105.12.115   <none>        80:32127/TCP   60s   app=frontend

          $ k get nodes -o wide
          NAME           STATUS   ROLES           AGE   VERSION   INTERNAL-IP   EXTERNAL-IP   OS-IMAGE             KERNEL-VERSION      CONTAINER-RUNTIME
          controlplane   Ready    control-plane   31d   v1.30.0   172.30.1.2    <none>        Ubuntu 20.04.5 LTS   5.4.0-131-generic   containerd://1.7.13
          node01         Ready    <none>          31d   v1.30.0   172.30.2.2    <none>        Ubuntu 20.04.5 LTS   5.4.0-131-generic   containerd://1.7.13
          
          $ curl 172.30.2.2:32127
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

### 7. You can find a pod named task-pv-pod in the default namespace, please check the status of the pod and troubleshoot, you can recreate the pod if you want.

        $ k get po
        NAME          READY   STATUS    RESTARTS   AGE
        task-pv-pod   0/1     Pending   0          8s

        $ k describe po task-pv-pod
          
        
        $ k get pv
        NAME             CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS   CLAIM                   STORAGECLASS   VOLUMEATTRIBUTESCLASS   REASON   AGE
        task-pv-volume   10Gi       RWO            Retain           Bound    default/task-pv-claim   manual         <unset>                          3m9s
        
        $ k get pvc
        NAME            STATUS   VOLUME           CAPACITY   ACCESS MODES   STORAGECLASS   VOLUMEATTRIBUTESCLASS   AGE
        task-pv-claim   Bound    task-pv-volume   10Gi       RWO            manual         <unset>                 2m39s
        

claim name in the pod definition is wrongly spelled as task-pv-claimm. We will have to correct it and recreate the pod.
        $ k get pod -o yaml > pod.yaml
        $ vi pod.yaml
        
        $ k delete po task-pv-pod
        pod "task-pv-pod" deleted
        
        $ k apply -f pod.yaml 
        pod/task-pv-pod created
        
        $ k get po 
        NAME          READY   STATUS              RESTARTS   AGE
        task-pv-pod   0/1     ContainerCreating   0          7s
        
        $ k get po
        NAME          READY   STATUS    RESTARTS   AGE
        task-pv-pod   1/1     Running   0          9s

### 8. Deploy a pod with the following specifications:

### Pod name: web-pod
### Image: httpd
### Node: Node01
### Note: do not modify any settings on master and worker nodes

        $ k run web-pod --image=httpd
        pod/web-pod created

        $ k get po
        NAME      READY   STATUS    RESTARTS   AGE
        web-pod   0/1     Pending   0          3s 

        $ k get po -o yaml > pod1.yaml
        $ vi pod1.yaml

Add tolerations in the pod definition file.
        spec:
          tolerations:
            - effect: NoSchedule
              key: node-role.kubernetes.io/node
              operator: Exists

        $ k delete po web-pod
        pod "web-pod" deleted
        $ k apply -f pod1.yaml 
        pod/web-pod created

        $ k get po
        NAME      READY   STATUS              RESTARTS   AGE
        web-pod   0/1     ContainerCreating   0          4s
        
        $ k get po
        NAME      READY   STATUS    RESTARTS   AGE
        web-pod   1/1     Running   0          12s

        $ k get po -o wide
        NAME      READY   STATUS    RESTARTS   AGE     IP            NODE     NOMINATED NODE   READINESS GATES
        web-pod   1/1     Running   0          4m12s   192.168.1.4   node01   <none>           <none>


### 9. Create a new PersistentVolume named web-pv. It should have a capacity of 2Gi, accessMode ReadWriteOnce hostPath /vol/data and no storageClassName defined. 

### Next create a new PersistentVolumeClaim in Namespace production named web-pvc. It should request 2Gi storage, accessMode ReadWriteOnce and should not define a storageClassName. The PVC should bound to the PV correctly. 

### Finally create a new Deployment web-deploy in Namespace production which mounts that volume at /tmp/web-data. The Pods of that Deployment should be of image nginx:1.14.2

__Create a PV__

        $ cat pv1.yaml 
            apiVersion: v1
            kind: PersistentVolume
            metadata:
              name: web-pv
              labels:
                type: local
            spec:
              capacity:
                storage: 2Gi
              accessModes:
                - ReadWriteOnce
              hostPath:
                path: "/vol/data"
              claimRef:
                namespace: production
                name: web-pvc

        $ k apply -f pv1.yaml 
           persistentvolume/web-pv created
          
        $ k get pv
        NAME     CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS      CLAIM   STORAGECLASS   VOLUMEATTRIBUTESCLASS   REASON   AGE
        web-pv   2Gi        RWO            Retain           Available                          <unset>                          2m16s

__Create a PVC__

        $ k create ns production  
        namespace/production created
        
        $ vi pvc.yaml
              apiVersion: v1
              kind: PersistentVolumeClaim
              metadata:
                name: web-pvc
              spec:
                accessModes:
                  - ReadWriteOnce
                resources:
                  requests:
                    storage: 2Gi

        $ k apply -f pvc.yaml -n production
        persistentvolumeclaim/web-pvc created

        $ k get pvc -n production
        NAME      STATUS   VOLUME   CAPACITY   ACCESS MODES   STORAGECLASS   VOLUMEATTRIBUTESCLASS   AGE
        web-pvc   Bound    web-pv   2Gi        RWO            local-path     <unset>                 9s


__Create deployment__


        $ vi web-deploy.yaml

                apiVersion: apps/v1
                kind: Deployment
                metadata:
                  name: web-deploy
                  namespace: production
                  labels:
                    app: nginx
                spec:
                  replicas: 3
                  selector:
                    matchLabels:
                      app: nginx
                  template:
                    metadata:
                      labels:
                        app: nginx
                    spec:
                      volumes:
                      - name: web-pv-storage
                        persistentVolumeClaim:
                          claimName: web-pvc
                      containers:
                      - name: task-pv-container
                        image: nginx:1.14.2
                        ports:
                          - containerPort: 80
                            name: "http-server"
                        volumeMounts:
                          - mountPath: "/tmp/web-data"
                            name: web-pv-storage
        $ k apply -f web-deploy.yaml 
        deployment.apps/web-deploy created

        $ k get deploy -n production
        NAME         READY   UP-TO-DATE   AVAILABLE   AGE
        web-deploy   3/3     3            3           16s

        $ k get po -n production
        NAME                          READY   STATUS    RESTARTS   AGE
        web-deploy-76cd784444-cblrz   1/1     Running   0          26s
        web-deploy-76cd784444-g665k   1/1     Running   0          26s
        web-deploy-76cd784444-zcgdk   1/1     Running   0          26s
 
### 10. Create a Kubernetes Pod named "my-busybox" with the busybox:1.31.1 image. The Pod should run a sleep command for 4800 seconds. Verify that the Pod is running in Node01.

        $ k run my-busybox --image busybox:1.31.1 --command sleep 4800
        pod/my-busybox created

        $ k get po
        NAME         READY   STATUS    RESTARTS   AGE
        my-busybox   1/1     Running   0          6s

        $ k get po -o wide
        NAME         READY   STATUS    RESTARTS   AGE   IP            NODE           NOMINATED NODE   READINESS GATES
        my-busybox   1/1     Running   0          11s   192.168.0.4   controlplane   <none>           <none>

The pod is scheduled on controlplane and not on node01. There are a few reasons for the pod is not scheduled on worker node

1. Pod may have tolerations
2. The cluster may have only one mode - master node
3. The pod may have nodeSelector that matches the master node.

        $ k get nodes
        NAME           STATUS                     ROLES           AGE   VERSION
        controlplane   Ready                      control-plane   32d   v1.30.0
        node01         Ready,SchedulingDisabled   <none>          32d   v1.30.0

Status for node01 is Ready/Scheduling disabled which means the node is in maintainance mode. 

        $ k uncordon node01
        node/node01 uncordoned

        $ k get nodes
        NAME           STATUS   ROLES           AGE   VERSION
        controlplane   Ready    control-plane   32d   v1.30.0
        node01         Ready    <none>          32d   v1.30.0

        $ k delete po my-busybox
        pod "my-busybox" deleted

        $ k run my-busybox --image busybox:1.31.1 --command sleep 4800
        pod/my-busybox created
        
        $ k get po
        NAME         READY   STATUS    RESTARTS   AGE
        my-busybox   1/1     Running   0          7s
        
        $ k get po -o wide
        NAME         READY   STATUS    RESTARTS   AGE   IP            NODE     NOMINATED NODE   READINESS GATES
        my-busybox   1/1     Running   0          13s   192.168.1.4   node01   <none>           <none>

### 11. You have a Kubernetes cluster that runs a three-tier web application: a frontend tier (port 80), an application tier (port 8080),and a backend tier (3306). The security team has mandated that the backend tier should only be accessible from the application tier.
 
 Normally any container can communicate with any other container in the k8s environment. But as per the question, frontend to backend communication has to be blocked and only communication from app container is to be allowed. So ingress network policy has to be applied on backend pod.

        $ k get deploy
        NAME          READY   UP-TO-DATE   AVAILABLE   AGE
        application   1/1     1            1           5s
        backend       1/1     1            1           26s
        frontend      1/1     1            1           54s

        $ k get po
        NAME                           READY   STATUS    RESTARTS   AGE
        application-7974d7b74f-kzgz5   1/1     Running   0          10s
        backend-6c8f58695c-8m77t       1/1     Running   0          31s
        frontend-76c6d4c997-xlgbs      1/1     Running   0          59s

To check whether frontend pod and app pod has connection to backend pod, we will be using telnet. In the CKA examination telnet will be already installed on the pods.

        $ k exec frontend-76c6d4c997-xlgbs telnet 192.168.1.5 3306
        kubectl exec [POD] [COMMAND] is DEPRECATED and will be removed in a future version. Use kubectl exec [POD] -- [COMMAND] instead.
        Connection closed by foreign host.
        Trying 192.168.1.5...
        Connected to 192.168.1.5.
        Escape character is '^]'.
        J
        5.6.51Nz<Y,pnO`vSz.sy)\sNmysql_native_password$

Similarly do it from app pod also.

        $ k get po --show-labels
        NAME                           READY   STATUS    RESTARTS   AGE   LABELS
        application-7974d7b74f-kzgz5   1/1     Running   0          20m   pod-template-hash=7974d7b74f,tier=application
        backend-6c8f58695c-8m77t       1/1     Running   0          20m   pod-template-hash=6c8f58695c,tier=backend
        frontend-76c6d4c997-xlgbs      1/1     Running   0          20m   pod-template-hash=76c6d4c997,tier=frontend
        
        $ vi network.yaml
              apiVersion: networking.k8s.io/v1
              kind: NetworkPolicy
              metadata:
                name: backend-network-policy
                namespace: default
              spec:
                podSelector:
                  matchLabels:
                    tier: backend
                policyTypes:
                - Ingress
                ingress:
                - from:
                  - podSelector:
                      matchLabels:
                        tier: application
                  ports:
                  - protocol: TCP
                    port: 3306
          $ k apply -f network.yaml 
            networkpolicy.networking.k8s.io/backend-network-policy created
          
           $ k get networkpolicies
            NAME                     POD-SELECTOR   AGE
            backend-network-policy   tier=backend   4m44s
           $ k describe networkpolicies backend-network-policy
            Name:         backend-network-policy
            Namespace:    default
            Created on:   2024-09-03 11:51:14 +0000 UTC
            Labels:       <none>
            Annotations:  <none>
            Spec:
              PodSelector:     tier=backend
              Allowing ingress traffic:
                To Port: 3306/TCP
                From:
                  PodSelector: tier=application
              Not affecting egress traffic
              Policy Types: Ingress

We can retry telnet to backend pod from both frontend and application.
          
          $ k exec frontend-76c6d4c997-xlgbs telnet 192.168.1.5 3306
            kubectl exec [POD] [COMMAND] is DEPRECATED and will be removed in a future version. Use kubectl exec [POD] -- [COMMAND] instead.



            ^C
          $ 

### 12. You have a Kubernetes cluster and running pods in multiple namespaces, The security team has mandated that the db pods on Project-a namespace only accessible from the service pods that are running in Project-b namespace.

          $ k get ns
          NAME                 STATUS   AGE
          default              Active   32d
          kube-node-lease      Active   32d
          kube-public          Active   32d
          kube-system          Active   32d
          local-path-storage   Active   32d
          project-a            Active   9m5s
          project-b            Active   8m56s

          $ k get pods -n project-a
          NAME                 READY   STATUS    RESTARTS   AGE
          db-fcb64695b-87cdz   1/1     Running   0          7m29s

          $ k get pods -n project-b
          NAME                       READY   STATUS    RESTARTS   AGE
          service-76cf458b4f-p6x6k   1/1     Running   0          3m17s
          web-7c56dcdb9b-c2rwl       1/1     Running   0          53s

          $ k get pods -n project-a -o wide
            NAME                 READY   STATUS    RESTARTS   AGE   IP            NODE     NOMINATED NODE   READINESS GATES
            db-fcb64695b-87cdz   1/1     Running   0          13m   192.168.1.5   node01   <none>           <none>

          $ k exec service-76cf458b4f-p6x6k -n project-b telnet 192.168.1.5 3306
            kubectl exec [POD] [COMMAND] is DEPRECATED and will be removed in a future version. Use kubectl exec [POD] -- [COMMAND] instead.
            Connection closed by foreign host.
            Trying 192.168.1.5...
            Connected to 192.168.1.5.
            Escape character is '^]'.
          
          $ k exec web-7c56dcdb9b-c2rwl -n project-b -- telnet 192.168.1.5 3306          
            Trying 192.168.1.5...
            Connected to 192.168.1.5.
            Escape character is '^]'.
            Connection closed by foreign host.

Here ingress network policy has to be applied on namespace project-a to accept communication from only service pods in namespace project-b.

__Step1 : label the namespaces__

          $ k label ns project-a namespace=project-a
          namespace/project-a labeled
          $ k label ns project-b namespace=project-b
          namespace/project-b labeled
          
          $ k get ns --show-labels
          NAME                 STATUS   AGE     LABELS
          default              Active   32d     kubernetes.io/metadata.name=default
          kube-node-lease      Active   32d     kubernetes.io/metadata.name=kube-node-lease
          kube-public          Active   32d     kubernetes.io/metadata.name=kube-public
          kube-system          Active   32d     kubernetes.io/metadata.name=kube-system
          local-path-storage   Active   32d     kubernetes.io/metadata.name=local-path-storage
          project-a            Active   5m5s    kubernetes.io/metadata.name=project-a,namespace=project-a
          project-b            Active   4m57s   kubernetes.io/metadata.name=project-b,namespace=project-b

          $ k get pods -n project-a --show-labels
          NAME                 READY   STATUS    RESTARTS   AGE   LABELS
          db-fcb64695b-98crm   1/1     Running   0          28m   app=db,pod-template-hash=fcb64695b
          
          $ k get pods -n project-b --show-labels
          NAME                       READY   STATUS    RESTARTS   AGE   LABELS
          service-76cf458b4f-4wsmf   1/1     Running   0          26m   app=service,pod-template-hash=76cf458b4f
          web-7c56dcdb9b-mssmf       1/1     Running   0          27m   app=web,pod-template-hash=7c56dcdb9b

__Step2 : Create and apply networkpolicy__

### networkpolicy created is saved here as ingress_np_on_ns.yaml under yaml directory.

          $ k apply -f np.yaml  
          networkpolicy.networking.k8s.io/db-network-policy created
          $ k get networkpolicies
          No resources found in default namespace.
          $ k get networkpolicies -n project-a
          NAME                POD-SELECTOR   AGE
          db-network-policy   app=db         16s
          $ k describe networkpolicies db-network-policy -n project-a
          Name:         db-network-policy
          Namespace:    project-a
          Created on:   2024-09-03 14:15:15 +0000 UTC
          Labels:       <none>
          Annotations:  <none>
          Spec:
            PodSelector:     app=db
            Allowing ingress traffic:
              To Port: <any> (traffic allowed to all ports)
              From:
                NamespaceSelector: namespace=project-b
                PodSelector: app=service
            Not affecting egress traffic
            Policy Types: Ingress

__Verification__

          $ k -n project-b exec service-76cf458b4f-4wsmf telnet 192.168.1.4 3306
          kubectl exec [POD] [COMMAND] is DEPRECATED and will be removed in a future version. Use kubectl exec [POD] -- [COMMAND] instead.
          Trying 192.168.1.4...
          Connected to 192.168.1.4.
          Escape character is '^]'.
          Connection closed by foreign host.
          $ k -n project-b exec web-7c56dcdb9b-mssmf telnet 192.168.1.4 3306
          kubectl exec [POD] [COMMAND] is DEPRECATED and will be removed in a future version. Use kubectl exec [POD] -- [COMMAND] instead.



          ^C
          $ 

### 13. you can find a pod named multi-pod is running and that is logging to a volume. You need to insert a container into the pod that will also read the logs from using this command "tail -f /var/busybox/log/*. log" specifications given below:

### image: busybox:1.28
### Name: sidecar
### volumepath: /var/busybox/log


![Screenshot 2024-09-03 205728](https://github.com/user-attachments/assets/8d8e86e7-3073-4814-9c36-dcd0043834c9)


$ k get po
NAME        READY   STATUS    RESTARTS   AGE
multi-pod   1/1     Running   0          18s

$ k get po multi-pod -o yaml > multi.yaml
$ cp multi.yaml bkup_multi.yaml
$ vi multi.yaml   

Add the below snippet under containers in the definition file

            - image: busybox:1.28
                name: sidecar
                volumeMounts:
                - mountPath: /var/busybox/log
                  name: hostpath-volume

Delete the existing pod and recreate the pod using definition file.

          $ k delete po multi-pod 
          pod "multi-pod" deleted
          $ k apply -f multi.yaml 
          pod/multi-pod created
          $ k get po
          NAME        READY   STATUS    RESTARTS   AGE
          multi-pod   2/2     Running   0          3s

          $ curl 192.168.1.5
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

          $ k logs multi-pod -c sidecar
          ==> /var/busybox/log/access.log <==

          ==> /var/busybox/log/error.log <==
          2024/09/03 15:58:30 [notice] 1#1: signal 17 (SIGCHLD) received from 28
          2024/09/03 15:58:30 [notice] 1#1: worker process 28 exited with code 0
          2024/09/03 15:58:30 [notice] 1#1: exit
          2024/09/03 16:00:10 [notice] 1#1: using the "epoll" event method
          2024/09/03 16:00:10 [notice] 1#1: nginx/1.27.1
          2024/09/03 16:00:10 [notice] 1#1: built by gcc 12.2.0 (Debian 12.2.0-14) 
          2024/09/03 16:00:10 [notice] 1#1: OS: Linux 5.4.0-131-generic
          2024/09/03 16:00:10 [notice] 1#1: getrlimit(RLIMIT_NOFILE): 1048576:1048576
          2024/09/03 16:00:10 [notice] 1#1: start worker processes
          2024/09/03 16:00:10 [notice] 1#1: start worker process 28

          ==> /var/busybox/log/access.log <==
          192.168.0.0 - - [03/Sep/2024:16:00:59 +0000] "GET / HTTP/1.1" 200 615 "-" "curl/7.68.0" "-"

### 14. Create a CronJob for running every 2 minutes with busybox image. The job name should be my-job and it should print the current date and time to the console. After running the job save any one of the pod logs to below path /root/logs.txt.


          $ cat cronjob.yaml 
            apiVersion: batch/v1
            kind: CronJob
            metadata:
              name: my-job
            spec:
              schedule: "*/2 * * * *"
              jobTemplate:
                spec:
                  template:
                    spec:
                      containers:
                      - name: my-job
                        image: busybox:1.28
                        imagePullPolicy: IfNotPresent
                        command:
                        - /bin/sh
                        - -c
                        - date; 
                      restartPolicy: OnFailure

          $ k apply -f cronjob.yaml 
            cronjob.batch/my-job created

          $ k get po --watch
          NAME                    READY   STATUS      RESTARTS   AGE
          my-job-28756942-h75tl   0/1     Completed   0          5s
          my-job-28756944-c8ljj   0/1     Pending     0          0s
          my-job-28756944-c8ljj   0/1     Pending     0          0s
          my-job-28756944-c8ljj   0/1     ContainerCreating   0          0s
          my-job-28756944-c8ljj   0/1     ContainerCreating   0          0s
          my-job-28756944-c8ljj   0/1     Completed           0          1s
          my-job-28756944-c8ljj   0/1     Completed           0          2s
          my-job-28756944-c8ljj   0/1     Completed           0          2s
          my-job-28756944-c8ljj   0/1     Completed           0          3s
          my-job-28756944-c8ljj   0/1     Completed           0          3s

          $ k logs my-job-28756944-c8ljj
          Wed Sep  4 02:24:00 UTC 2024

          $ touch logs.txt
          $ k logs my-job-28756944-c8ljj > logs.txt 
          $ cat logs.txt 
          Wed Sep  4 02:24:00 UTC 2024
          $ 

### 15. Find the schedulable nodes in the cluster and save the name and count in to the below file

### File path : /root/nodes.txt

          $ k get nodes
          NAME           STATUS   ROLES           AGE   VERSION
          controlplane   Ready    control-plane   32d   v1.30.0
          node01         Ready    <none>          32d   v1.30.0

          $ k get nodes -o jsonpath='{.items[*].spec.taints}'
          [{"effect":"NoSchedule","key":"node-role.kubernetes.io/master"}] $ 

          $ vi nodes.txt 
          $ cat nodes.txt 
          Node_Name= [ node01 ]
          Node_Count= [ 1  ]

### 16. Please deploy a pod on node01 as per the below specification

### Pod name: web-pod
### Container name = web
### image: nginx

          $ k run web-pod --image nginx --dry-run=client -o yaml > web-pod.yaml
          
          $ vi web-pod.yaml 
            apiVersion: v1
          kind: Pod
          metadata:
            creationTimestamp: null
            labels:
              run: web-pod
            name: web-pod
          spec:
            containers:
            - image: nginx
              name: web
              resources: {}
            dnsPolicy: ClusterFirst
            restartPolicy: Always
          status: {}

          $ k apply -f web-pod.yaml 
          pod/web-pod created

          $ k get po
          NAME      READY   STATUS    RESTARTS   AGE
          web-pod   0/1     Pending   0          4s


Pod status is in pending due to multiple reasons:

1. Pod is not running on controlplane as it will be tainted to be not schedulable
2. Pod is not running on node01 as it would be tainted or kubelet will not be running
3. kubelet on node01 will not be running as kubelet would be stopped or kubelet configuration would be wrong.

          $ k get nodes
          NAME           STATUS     ROLES           AGE   VERSION
          controlplane   Ready      control-plane   32d   v1.30.0
          node01         NotReady   <none>          32d   v1.30.0

Since the question wants the pod to run on node01, we can ignore controlplane.
Node01 status is NotReady due to kubelet not running or misconfiguration.


          $ ssh node01
          Last login: Wed Sep  4 03:23:14 2024 from 10.244.7.4
          node01 $ systemctl status kubelet
          ● kubelet.service - kubelet: The Kubernetes Node Agent
              Loaded: loaded (/lib/systemd/system/kubelet.service; enabled; vendor preset: enabled)
              Drop-In: /usr/lib/systemd/system/kubelet.service.d
                      └─10-kubeadm.conf
              Active: activating (auto-restart) (Result: exit-code) since Wed 2024-09-04 03:51:04 UTC; 963ms ago
                Docs: https://kubernetes.io/docs/
              Process: 9279 ExecStart=/usr/bin/local/kubelet $KUBELET_KUBECONFIG_ARGS $KUBELET_CONFIG_ARGS $KUBELET_K>
            Main PID: 9279 (code=exited, status=203/EXEC)

          Sep 04 03:51:04 node01 systemd[1]: kubelet.service: Main process exited, code=exited, status=203/EXEC
          Sep 04 03:51:04 node01 systemd[1]: kubelet.service: Failed with result 'exit-code'.
          lines 1-11/11 (END)

First we will try starting the kubelet service.

          node01 $ systemctl start kubelet
          node01 $ systemctl status kubelet
          ● kubelet.service - kubelet: The Kubernetes Node Agent
              Loaded: loaded (/lib/systemd/system/kubelet.service; enabled; vendor preset: enabled)
              Drop-In: /usr/lib/systemd/system/kubelet.service.d
                      └─10-kubeadm.conf
              Active: activating (auto-restart) (Result: exit-code) since Wed 2024-09-04 04:07:38 UTC; 9s ago
                Docs: https://kubernetes.io/docs/
              Process: 12041 ExecStart=/usr/bin/local/kubelet $KUBELET_KUBECONFIG_ARGS $KUBELET_CONFIG_ARGS $KUBELET_>
            Main PID: 12041 (code=exited, status=203/EXEC)

          Sep 04 04:07:38 node01 systemd[1]: kubelet.service: Main process exited, code=exited, status=203/EXEC
          Sep 04 04:07:38 node01 systemd[1]: kubelet.service: Failed with result 'exit-code'.
          lines 1-11/11 (END)

Restart didnt help. Next we will check for kubelet configuration. In the above snippet we see

        ExecStart=/usr/bin/local/kubelet

We check for kubelet in /usr/bin/local

          node01 $ ls -lrt /usr/bin/local/kubelet
          ls: cannot access '/usr/bin/local': No such file or directory

          node01 $ ls -lrt /usr/bin/kubelet
          -rwxr-xr-x 1 root root 100100024 Apr 17 18:28 /usr/bin/kubelet

kubelet is in /usr/bin. So we have to correct the configuration.
ExecStart=/usr/bin/local/kubelet is corrected to ExecStart=/usr/bin/kubelet

          node01 $ cd /usr/lib/systemd/system/kubelet.service.d 
          node01 $ vi 10-kubeadm.conf
          node01 $ systemctl daemon-reload
          node01 $ systemctl start kubelet
          node01 $ systemctl status kubelet
          ● kubelet.service - kubelet: The Kubernetes Node Agent
              Loaded: loaded (/lib/systemd/system/kubelet.service; enabled; vendor preset: enabled)
              Drop-In: /usr/lib/systemd/system/kubelet.service.d
                      └─10-kubeadm.conf
              Active: active (running) since Wed 2024-09-04 04:12:56 UTC; 12s ago
                Docs: https://kubernetes.io/docs/
            Main PID: 12996 (kubelet)
                Tasks: 10 (limit: 2338)
              Memory: 31.7M
              CGroup: /system.slice/kubelet.service
                      └─12996 /usr/bin/kubelet --bootstrap-kubeconfig=/etc/kubernetes/bootstrap-kubelet.conf --kubec>

          node01 $ exit
          logout
          Connection to node01 closed.
          
          $ k get nodes
          NAME           STATUS   ROLES           AGE   VERSION
          controlplane   Ready    control-plane   32d   v1.30.0
          node01         Ready    <none>          32d   v1.30.0
          
          $ k get po
          NAME      READY   STATUS    RESTARTS   AGE
          web-pod   1/1     Running   0          30m
          
          $ k get po -o wide
          NAME      READY   STATUS    RESTARTS   AGE   IP            NODE     NOMINATED NODE   READINESS GATES
          web-pod   1/1     Running   0          30m   192.168.1.4   node01   <none>           <none>
          $ 

### 17. Please join node01 worker node to the cluster, and you have to deploy a pod in the node01, pod name should be web and image should be nginx

          $ k get nodes
          NAME           STATUS   ROLES           AGE   VERSION
          controlplane   Ready    control-plane   32d   v1.30.0
          $ kubeadm token create --print-join-command
          kubeadm join 172.30.1.2:6443 --token zysv9m.1l8oybm3206wtxoj --discovery-token-ca-cert-hash sha256:f5d5f74bd26d6e4311a49a05e4482ddc8cb922dbb08f594c94760fe2e44fc0a5 
          $ ssh node01
          Last login: Wed Sep  4 06:01:01 2024 from 10.244.3.176
          node01 $ systemctl status kubelet
          ● kubelet.service - kubelet: The Kubernetes Node Agent
              Loaded: loaded (/lib/systemd/system/kubelet.service; enabled; vendor preset: enabled)
              Drop-In: /usr/lib/systemd/system/kubelet.service.d
                      └─10-kubeadm.conf
              Active: inactive (dead) since Wed 2024-09-04 06:01:08 UTC; 1min 8s ago

          node01 $ kubeadm join 172.30.1.2:6443 --token zysv9m.1l8oybm3206wtxoj --discovery-token-ca-cert-hash sha256:f5d5f74bd26d6e4311a49a05e4482ddc8cb922dbb08f594c94760fe2e44fc0a5
          [preflight] Running pre-flight checks
          [preflight] Reading configuration from the cluster...
          [preflight] FYI: You can look at this config file with 'kubectl -n kube-system get cm kubeadm-config -o yaml'
          [kubelet-start] Writing kubelet configuration to file "/var/lib/kubelet/config.yaml"
          [kubelet-start] Writing kubelet environment file with flags to file "/var/lib/kubelet/kubeadm-flags.env"
          [kubelet-start] Starting the kubelet
          [kubelet-check] Waiting for a healthy kubelet. This can take up to 4m0s
          [kubelet-check] The kubelet is healthy after 510.123858ms
          [kubelet-start] Waiting for the kubelet to perform the TLS Bootstrap

          This node has joined the cluster:
          * Certificate signing request was sent to apiserver and a response was received.
          * The Kubelet was informed of the new secure connection details.

          Run 'kubectl get nodes' on the control-plane to see this node join the cluster.

          node01 $ exit
          logout
          Connection to node01 closed.
          $ k get nodes
          NAME           STATUS   ROLES           AGE   VERSION
          controlplane   Ready    control-plane   32d   v1.30.0
          node01         Ready    <none>          70s   v1.30.0
          $ 

          $ k run web --image nginx
          pod/web created
         
          $ k get po
          NAME   READY   STATUS              RESTARTS   AGE
          web    0/1     ContainerCreating   0          4s
          $ k get po -o wide
          NAME   READY   STATUS    RESTARTS   AGE   IP            NODE     NOMINATED NODE   READINESS GATES
          web    1/1     Running   0          10s   192.168.1.4   node01   <none>           <none>
          $ 

### 18








 


 
            
 


