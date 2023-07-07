# simple-gitlab-in-kuberentes
An easy way to install GitLab in Kuberentes.
This repository contains manifests for installing GitLab in Kubernetes with one Pod, in fact, the same approach is used as in docker-compose.
This installation is not intended for production loads.
Use it for tests.

## Working ssh in Gitlab using NodePort

To access SSH Github via NodePort, uncomment the following lines in the service.yaml file:

```yaml
---
kind: Service
apiVersion: v1
metadata:
  name: gitlab-gitlab-shell
  namespace: gitlab
spec:
  type: NodePort
  selector:
    app: gitlab
  ports:
    - port: 2222
      targetPort: 22
      protocol: TCP
      name: ssh
      nodePort: 32222
```

## Working ssh in Gitlab Using Nginx Ingress

edit ~/.ssh/config

```config
Host kube-gitlab.local #You Host
    Port 2222 #NodePort
    User git
```

git clone kube-gitlab.local:[you repo]

kubectl taint nodes worker1.local kube-app=gitlab:NoSchedule

### Ingress patch to use ssh in port 2222

Edit you nginx Ingress controller

```yaml
        - --tcp-services-configmap=$(POD_NAMESPACE)/tcp-services
        - --udp-services-configmap=$(POD_NAMESPACE)/udp-services
```

```yaml
#kubectl edit deployments -n kube-system ingress-nginx-controller
    spec:
      containers:
      - args:
        - /nginx-ingress-controller
        - --publish-service=kube-system/ingress-nginx-controller
        - --election-id=ingress-controller-leader
        - --ingress-class=nginx
        - --configmap=kube-system/ingress-nginx-controller
        - --tcp-services-configmap=$(POD_NAMESPACE)/tcp-services
        - --udp-services-configmap=$(POD_NAMESPACE)/udp-services
        - --validating-webhook=:8443
        - --validating-webhook-certificate=/usr/local/certificates/cert
        - --validating-webhook-key=/usr/local/certificates/key
```

Use patch Service Ingress Controller

```yaml
spec:
  ports:
    - name: proxied-tcp-ssh
      port: 2222
      targetPort: 2222
      protocol: TCP
```

`kubectl -n ingress-nginx patch service ingress-nginx-controller --patch "$(cat service-patch-ssh.yaml)"`

Add Configmap:

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: tcp-services
  namespace: ingress-nginx
data:
  "2222": "gitlab/gitlab-gitlab-shell:2222"
```
