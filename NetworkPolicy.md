### There was a security incident where an intruder was able to access the whole cluster from a single hacked web Pod. To prevent this create a NetworkPolicy in default Namespace. It should allow the web -* Pods only to: connect to service -* Pods on port 8080. After implementation, connections from web -* Pods to application -* Pods on port 80 should also be blocked.

![Screenshot 2024-09-04 152200](https://github.com/user-attachments/assets/22e0b4f4-a993-4f74-8a74-c4fc988f8f8b)


          $ k get pods -o wide
          NAME                       READY   STATUS    RESTARTS   AGE     IP            NODE     NOMINATED NODE   READINESS GATES
          app-76cf458b4f-dqk88       1/1     Running   0          8m6s    192.168.1.4   node01   <none>           <none>
          db-fcb64695b-7rhtw         1/1     Running   0          7m53s   192.168.1.7   node01   <none>           <none>
          service-565ccd4747-8thjb   1/1     Running   0          8m1s    192.168.1.5   node01   <none>           <none>
          web-7c56dcdb9b-m8qkv       1/1     Running   0          7m56s   192.168.1.6   node01   <none>           <none>
          $ k exec web-7c56dcdb9b-m8qkv -- telnet 192.168.1.5 8080
          Trying 192.168.1.5...
          Connected to 192.168.1.5.
          Escape character is '^]'.
          Connection closed by foreign host.
          $ k exec web-7c56dcdb9b-m8qkv -- telnet 192.168.1.7 3306
          Trying 192.168.1.7...
          Connected to 192.168.1.7.
          Escape character is '^]'.
          Connection closed by foreign host.
          $ k exec web-7c56dcdb9b-m8qkv -- telnet 192.168.1.4 80
          Trying 192.168.1.4...
          Connected to 192.168.1.4.
          Escape character is '^]'.
          Connection closed by foreign host.

          $ k get pods --show-labels
          NAME                       READY   STATUS    RESTARTS   AGE   LABELS
          app-76cf458b4f-dqk88       1/1     Running   0          17m   app=service,pod-template-hash=76cf458b4f
          db-fcb64695b-7rhtw         1/1     Running   0          16m   app=db,pod-template-hash=fcb64695b
          service-565ccd4747-8thjb   1/1     Running   0          16m   app=service,pod-template-hash=565ccd4747
          web-7c56dcdb9b-m8qkv       1/1     Running   0          16m   app=web,pod-template-hash=7c56dcdb9b

Notice the labels for app pod and service pod are the same -> app: service. 
So the network policy will dependent on labels and ports in this case. We have to block communication from web pod to all other pods except service pod. So it is egress network policy to be applied on web pod.

The policy would look like:

        egress:
          - to:
            - podSelector:
                matchLabels:
                  app: service
            ports:
            - protocol: TCP
              port: 8080

          $ k apply -f egress_np.yaml 
          networkpolicy.networking.k8s.io/egress-network-policy created

          $ k get networkpolicy
          NAME                    POD-SELECTOR   AGE
          egress-network-policy   app=web        96s
          $ 

__Verification__

          $ k exec web-7c56dcdb9b-m8qkv -- telnet 192.168.1.4 80

          ^C
          $ k exec web-7c56dcdb9b-m8qkv -- telnet 192.168.1.7 3306

          ^C
          $ k exec web-7c56dcdb9b-m8qkv -- telnet 192.168.1.5 8080
          Trying 192.168.1.5...
          Connected to 192.168.1.5.
          Escape character is '^]'.
          Connection closed by foreign host.
          $ 
### You have a Kubernetes cluster and running pods in multiple namespaces, The security team has mandated that the db pods on Project-a namespace only accessible from the service pods that are running in Project-b namespace.

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

### You have a Kubernetes cluster that runs a three-tier web application: a frontend tier (port 80), an application tier (port 8080),and a backend tier (3306). The security team has mandated that the backend tier should only be accessible from the application tier.
 
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


