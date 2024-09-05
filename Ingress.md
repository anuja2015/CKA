### There are two existing Deployments in Namespace _world_ which should be made accessible via an Ingress.

### First: create ClusterIP Services for both Deployments for port 80 . The Services should have the same name as the Deployments.

### The Nginx Ingress Controller has been installed. Create a new Ingress resource called world for domain name _world. universe.mine_ . The domain points to the K8s Node IP via /etc/hosts .
### The Ingress resource should have two routes pointing to the existing Services:

### _http://world.universe.mine:30080/europe/_
### and
### _http://world.universe.mine:30080/asia/_


            $ k get ns
            NAME                 STATUS   AGE
            default              Active   33d
            ingress-nginx        Active   4m11s
            kube-node-lease      Active   33d
            kube-public          Active   33d
            kube-system          Active   33d
            local-path-storage   Active   33d
            world                Active   4m11s

            $ k get deploy -n world
            NAME     READY   UP-TO-DATE   AVAILABLE   AGE
            asia     2/2     2            2           4m18s
            europe   2/2     2            2           4m18s

            $ k expose deployment asia --port 80 -n world
            service/asia exposed

            $ k expose deployment europe --port 80 -n world
            service/europe exposed

            $ k get svc -n world
            NAME     TYPE        CLUSTER-IP     EXTERNAL-IP   PORT(S)   AGE
            asia     ClusterIP   10.101.61.10   <none>        80/TCP    17s
            europe   ClusterIP   10.100.42.12   <none>        80/TCP    6s
            $ 


            $ k get svc -n ingress-nginx
            NAME                                 TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)                      AGE
            ingress-nginx-controller             NodePort    10.107.47.18    <none>        80:30080/TCP,443:30443/TCP   7m35s
            ingress-nginx-controller-admission   ClusterIP   10.109.81.136   <none>        443/TCP                      7m35s
            $ cat /etc/hosts
            127.0.0.1 localhost

            # The following lines are desirable for IPv6 capable hosts
            ::1 ip6-localhost ip6-loopback
            fe00::0 ip6-localnet
            ff00::0 ip6-mcastprefix
            ff02::1 ip6-allnodes
            ff02::2 ip6-allrouters
            ff02::3 ip6-allhosts
            127.0.0.1 ubuntu
            127.0.0.1 host01
            127.0.0.1 controlplane
            172.30.1.2 world.universe.mine
            $ 

            $ k get ingressclass          
            NAME    CONTROLLER             PARAMETERS   AGE
            nginx   k8s.io/ingress-nginx   <none>       12m

            $ vi ingress_world.yaml

                apiVersion: networking.k8s.io/v1
                kind: Ingress
                metadata:
                name: world
                namespace: world
                annotations:
                    nginx.ingress.kubernetes.io/rewrite-target: /
                spec:
                ingressClassName: nginx
                rules:
                - host: "world.universe.mine"
                    http:
                    paths:
                    - path: /europe
                        pathType: Prefix
                        backend:
                        service:
                            name: europe
                            port:
                            number: 80
                    - path: /asia
                        pathType: Prefix
                        backend:
                        service:
                            name: asia
                            port:
                            number: 80

            $ k apply -f ingress_world.yaml 
            ingress.networking.k8s.io/world created

            $ k get ingress -n world
            NAME    CLASS   HOSTS                 ADDRESS      PORTS   AGE
            world   nginx   world.universe.mine   172.30.1.2   80      63s

            $ curl http://world.universe.mine:30080/europe/
            hello, you reached EUROPE
            $ curl http://world.universe.mine:30080/asia/
            hello, you reached ASIA
