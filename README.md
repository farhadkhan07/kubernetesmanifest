# Flask Demo with Docker Compose Jenkins ArgoCD Nginx-Ingress-Controller HaProxy on Kubernetes

### Activity
```
> GitOps Workflow
> Dockerfile & Jenkins File Walkthrough
> HAProxy, ArgoCD, NginX Ingres Controller Install
> Automating Github to Jenkins, ArgoCD using Webhook
```


#### In this guide, we’ll use Docker Compose to containerize a Python Flask simple application for development. When you’re finished, you’ll have a demo Laravel application running on three separate service containers:



# Install Nginx Ingress Controller
```
kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/controller-v1.4.0/deploy/static/provider/baremetal/deploy.yaml
```

```
# kubectl -n ingress-nginx get all
NAME                                            READY   STATUS      RESTARTS   AGE
pod/ingress-nginx-admission-create-l4vbx        0/1     Completed   0          10h
pod/ingress-nginx-admission-patch-sswhz         0/1     Completed   1          10h
pod/ingress-nginx-controller-55cbdb4b89-l5s5x   1/1     Running     0          9h

NAME                                         TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)                      AGE
service/ingress-nginx-controller             NodePort    10.10.104.201   <none>        80:30594/TCP,443:32716/TCP   10h
service/ingress-nginx-controller-admission   ClusterIP   10.10.78.131    <none>        443/TCP                      10h

NAME                                       READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/ingress-nginx-controller   1/1     1            1           10h

NAME                                                  DESIRED   CURRENT   READY   AGE
replicaset.apps/ingress-nginx-controller-55cbdb4b89   1         1         1       9h
replicaset.apps/ingress-nginx-controller-6d68c766c6   0         0         0       10h

NAME                                       COMPLETIONS   DURATION   AGE
job.batch/ingress-nginx-admission-create   1/1           13s        10h
job.batch/ingress-nginx-admission-patch    1/1           12s        10h
```
### Edit Nginx Ingress Controller and add SSL-Passthrough parameter
```
kubectl edit deployment ingress-nginx-controller -n ingress-nginx -o yaml
```
```
...
spec:
      containers:
      - args:
        - /nginx-ingress-controller
        - --default-backend-service=kube-system/nginx-ingress-default-backend
        - --election-id=ingress-controller-leader
        - --ingress-class=nginx
        - --enable-ssl-passthrough # Add this flag
        - --configmap=kube-system/nginx-ingress-controller
...
```

# Install ArgoCD

```
kubectl create namespace argocd
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
```
```
# kubectl get all -n argocd
NAME                                                    READY   STATUS    RESTARTS   AGE
pod/argocd-application-controller-0                     1/1     Running   0          11h
pod/argocd-applicationset-controller-5bff759d68-j2psk   1/1     Running   0          11h
pod/argocd-dex-server-59c59b5d96-kh259                  1/1     Running   0          11h
pod/argocd-notifications-controller-6df97c8577-fh4ql    1/1     Running   0          11h
pod/argocd-redis-684fb8c6dd-f85mt                       1/1     Running   0          11h
pod/argocd-repo-server-79d8c5f7b4-mgwr8                 1/1     Running   0          11h
pod/argocd-server-754766bfc6-tnnxq                      1/1     Running   0          11h
pod/nginx                                               1/1     Running   0          11h

NAME                                              TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)                      AGE
service/argocd-applicationset-controller          ClusterIP   10.10.180.99    <none>        7000/TCP,8080/TCP            11h
service/argocd-dex-server                         ClusterIP   10.10.235.246   <none>        5556/TCP,5557/TCP,5558/TCP   11h
service/argocd-metrics                            ClusterIP   10.10.166.172   <none>        8082/TCP                     11h
service/argocd-notifications-controller-metrics   ClusterIP   10.10.164.132   <none>        9001/TCP                     11h
service/argocd-redis                              ClusterIP   10.10.113.112   <none>        6379/TCP                     11h
service/argocd-repo-server                        ClusterIP   10.10.168.2     <none>        8081/TCP,8084/TCP            11h
service/argocd-server                             ClusterIP   10.10.178.205   <none>        80/TCP,443/TCP               11h
service/argocd-server-metrics                     ClusterIP   10.10.248.164   <none>        8083/TCP                     11h

NAME                                               READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/argocd-applicationset-controller   1/1     1            1           11h
deployment.apps/argocd-dex-server                  1/1     1            1           11h
deployment.apps/argocd-notifications-controller    1/1     1            1           11h
deployment.apps/argocd-redis                       1/1     1            1           11h
deployment.apps/argocd-repo-server                 1/1     1            1           11h
deployment.apps/argocd-server                      1/1     1            1           11h

NAME                                                          DESIRED   CURRENT   READY   AGE
replicaset.apps/argocd-applicationset-controller-5bff759d68   1         1         1       11h
replicaset.apps/argocd-dex-server-59c59b5d96                  1         1         1       11h
replicaset.apps/argocd-notifications-controller-6df97c8577    1         1         1       11h
replicaset.apps/argocd-redis-684fb8c6dd                       1         1         1       11h
replicaset.apps/argocd-repo-server-79d8c5f7b4                 1         1         1       11h
replicaset.apps/argocd-server-754766bfc6                      1         1         1       11h

NAME                                             READY   AGE
statefulset.apps/argocd-application-controller   1/1     11h

```


### ArgoCD ingress rule

```
#   cat <<EOF >> argocd-ingress.yaml
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  name: argocd-server-ingress
  namespace: argocd
  annotations:
    kubernetes.io/ingress.class: "nginx"
    nginx.ingress.kubernetes.io/force-ssl-redirect: "true"
    nginx.ingress.kubernetes.io/ssl-passthrough: "true"
spec:
  tls:
  - secretName: argocd-secret
    hosts:
    - argocd.example.com
  rules:
  - host: argocd.example.com
    http:
      paths:
      - backend:
          serviceName: argocd-server
          servicePort: https
        path: /
EOF

# kubectl apply -f  argocd-ingress.yaml

# kubectl get ingress -n argocd
NAME                    CLASS   HOSTS                   ADDRESS       PORTS   AGE
argocd-server-ingress   nginx   argocd.example.com   172.16.10.7   80      9h
```

### ArgoCD web Login & default Password change
```
# kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d; echo

web login:
https://argocd.example.com
```

# HaProxy Configure
```
# For Ingress-Nginx Http Traffic
frontend for-ingress-nginx-http
    bind 172.16.10.10:80
    option tcplog
    mode tcp
    default_backend ingress-nginx-http
backend ingress-nginx-http
    balance roundrobin
    mode tcp
    server master01 172.16.10.4:30594 check inter 2000 rise 2 fall 5
    server master02 172.16.10.5:30594 check inter 2000 rise 2 fall 5
    server master03 172.16.10.6:30594 check inter 2000 rise 2 fall 5

# For Inggress-nginx Https Traffic
frontend for-ingress-nginx-https
    bind 172.16.10.10:443
    option tcplog
    mode tcp
    default_backend ingress-nginx-https

backend ingress-nginx-https
    balance roundrobin
    mode tcp
    server master01 172.16.10.4:32716 check inter 2000 rise 2 fall 5
    server master02 172.16.10.5:32716 check inter 2000 rise 2 fall 5
    server master03 172.16.10.6:32716 check inter 2000 rise 2 fall 5
```

# Dockerhub & Github credentials setup in Jenkins

After login to Jenkins Click to Manage Jenkins > Manage Credentials > Under global click on System  > Global credentials (unresticted) > Add credentials. 
- Docker Hub: put unique ID, docker user & password
- GitHub: Put unique id & github user name. generate personal token in github for password (In Github: Settings > Developer Settings > Personal access tokens > Tokens (classic) > Generate new token: New personal access token (classic) > put note & select repo, webhook & notifications)

# Create Build Image Job in Jenkins
From Jenkins Dashboard > Create New Item > Put Item Name > Pipeline & click OK > Scroll Down to Pipeline.
Select Pipeline Script from SCM & in SEM select Git. In Repository URL put the Github URL HTTPS link.
Under Branch change it to main and click Save
```
https://github.com/farhadkhan07/kubernetescode.git
```

From Jenkins Dashboard > Create New Item > Put Item Name "updatemanifes"> Pipeline & click OK > Select This project is Parameterized > select String Parameter > Set Name: "DOCKERTAG" Default value: latest > Scroll Down to Pipeline.
Select Pipeline Script from SCM & in SEM select Git. In Repository URL put the Github URL HTTPS link.
Under Branch change it to main and click Save
```
https://github.com/farhadkhan07/kubernetesmanifest.git
```

# ArgoCD

Login to ArgoCD and click on New APP. Application 
name: flaskdemo
Project: default
Sync Policy: Automatic
Repository URL: https://github.com/farhadkhan07/kubernetesmanifest.git
Path: ./
Destination Cluster URL: https://kubernetes.default.svc
Namespace: default

# Configure WebHook:
For Git:
collect Jenkins web URL.
Go to GitHub "https://github.com/farhadkhan07/kubernetescode.git"
Setings > WebHooks > Add Web Hook > Payload URL: http://jenkinsIP/github-webhook/
Under Content type: application/json
Select Just the push event
Click on Add Webhook

For Jenkins:
Clink on you Jenkins_Build_image_job
Clink Configure & select "GitHub hook trigger for GITScm pooling"
Click Save.



Ref. Link:
```
https://www.youtube.com/watch?v=o4QG_kqYvHk&ab_channel=CloudWithRaj
https://github.com/saha-rajdeep/kubernetescode
https://github.com/saha-rajdeep/kubernetesmanifest
```
