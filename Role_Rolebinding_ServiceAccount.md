### Create a new ServiceAccount __gitops__ in Namespace project-1. Create a Role and RoleBinding, both named __gitops-role__ and __gitops-rolebinding__ as well. These should allow the new SA to only create Secrets and ConfigMaps in that Namespace.

__Note:__ No need to create namespace during exam as it would be already present.

          $ k create ns project-1
          namespace/project-1 created
          $ k get ns
          NAME                 STATUS   AGE
          default              Active   33d
          kube-node-lease      Active   33d
          kube-public          Active   33d
          kube-system          Active   33d
          local-path-storage   Active   33d
          project-1            Active   11m
          $ 

          $ k create sa gitops -n project-1
          serviceaccount/gitops created
          $ k get sa -n project-1
          NAME      SECRETS   AGE
          default   0         12m
          gitops    0         26s

          $ k create role gitops-role --verb=create --resource=secrets,configmaps -n project-1
          role.rbac.authorization.k8s.io/gitops-role created
          $ k get role -n project-1
          NAME          CREATED AT
          gitops-role   2024-09-04T11:45:09Z

          $ k describe role gitops-role -n project-1
          Name:         gitops-role
          Labels:       <none>
          Annotations:  <none>
          PolicyRule:
            Resources   Non-Resource URLs  Resource Names  Verbs
            ---------   -----------------  --------------  -----
            configmaps  []                 []              [create]
            secrets     []                 []              [create]

          $ k create rolebinding gitops-rolebinding --role gitops-role --serviceaccount project-1:gitops -n project-1
          rolebinding.rbac.authorization.k8s.io/gitops-rolebinding created
          $ k get rolebinding -n project-1
          NAME                 ROLE               AGE
          gitops-rolebinding   Role/gitops-role   5s

          $ k describe rolebinding gitops-rolebinding -n project-1
          Name:         gitops-rolebinding
          Labels:       <none>
          Annotations:  <none>
          Role:
            Kind:  Role
            Name:  gitops-role
          Subjects:
            Kind            Name    Namespace
            ----            ----    ---------
            ServiceAccount  gitops  project-1

          $ k -n project-1 auth can-i create pod --as system:serviceaccount:project-1:gitops
          no
          $ k -n project-1 auth can-i create secret --as system:serviceaccount:project-1:gitops
          yes
          $ k -n project-1 auth can-i create configmaps --as system:serviceaccount:project-1:gitops
          yes
          $ k -n project-1 auth can-i create service --as system:serviceaccount:project-1:gitops
          no
          $ k -n project-1 auth can-i create deployment --as system:serviceaccount:project-1:gitops
          no


