<p align="center">
    <img src="https://miro.medium.com/max/1510/1*e4w0j0SUdsfx_U7hdHHpyw.png" alt="Laradock Logo"/>
</p>


# Installation - tạo phân vùng với argo

Điều này sẽ tạo ra một không gian tên mới argocd, nơi các dịch vụ Argo CD và tài nguyên ứng dụng sẽ tồn tại.

```bash
kubectl create namespace argocd
kubectl apply -n argocd -f install.yaml
```

# Lấy quyền access các Argo CD

## 1. Tạo một service LoadBalancer trên k8s

```bash

kubectl patch svc argocd-server -n argocd -p '{"spec": {"type": "LoadBalancer"}}'

```

## 2. Tạo một layer proxy_pass trên k8s

```bash
kubectl port-forward svc/argocd-server -n argocd 443:443
```

# Cấu hình triển khai hệ thống java 

## Deploy Infrastructure

Giả sử quyền đc truy cập hệ thống là quyền cao nhất :

<p align="center">
    <img src="https://d2908q01vomqb2.cloudfront.net/ca3512f4dfa95a03169c5a670a4c91a19b3077b4/2020/03/09/magnum_f2_sm.png"/>
</p>

To create our infrastructure Application, go to Application | New Application and set up the Source to point to the infra/ directory of our repository. The Destination should be set to https://kubernetes.default.svc, meaning we intend for these resources to be created in the same Kubernetes cluster in which we have installed Crossplane and Argo CD. The full configuration for the application should looks as follows:


<p align="center">
    <img src="https://d2908q01vomqb2.cloudfront.net/ca3512f4dfa95a03169c5a670a4c91a19b3077b4/2020/03/09/magnum_f3_sm.png"/>
</p>


<p align="center">
    <img src="https://d2908q01vomqb2.cloudfront.net/ca3512f4dfa95a03169c5a670a4c91a19b3077b4/2020/03/09/magnum_f4_sm.png"/>
</p>


Click Create and you should be able to view each of the resources and their status by clicking on the application. You will notice that we are creating a VPC in each region, subnets and networking components for those VPCs, as well as provisioning an Amazon EKS cluster in each. If you go to the AWS console for the account whose credentials were used in your account credentials Secret, you should see these resources being created in their respective dashboards.

We are specifically interested in the readiness of argo-west-cluster and argo-east-cluster, which are the claims for the Amazon EKS clusters we have created in each region. A claim in Crossplane refers to a Kubernetes object that serves as an abstract request for a concrete implementation of a managed service.
In this case, the claims argo-west-cluster and argo-east-cluster are KubernetesCluster claims, which are being satisfied by an EKSCluster. They could just as easily be satisfied by a managed Kubernetes offering from a different provider. It may take some time, but when they are fully provisioned, Crossplane will be able to schedule our applications on each of those clusters. You will see a corresponding Secret and KubernetesTarget appear for each cluster in the Argo CD UI when they are ready:


<p align="center">
    <img src="https://d2908q01vomqb2.cloudfront.net/ca3512f4dfa95a03169c5a670a4c91a19b3077b4/2020/03/09/magnum_f5_sm.png"/>
</p>


## Deploy application in region

<p align="center">
    <img src="https://d2908q01vomqb2.cloudfront.net/ca3512f4dfa95a03169c5a670a4c91a19b3077b4/2020/03/09/magnum_f6_sm.png"/>
</p>

<p align="center">
    <img src="https://d2908q01vomqb2.cloudfront.net/ca3512f4dfa95a03169c5a670a4c91a19b3077b4/2020/03/09/magnum_f7_sm.png"/>
</p>

<p align="center">
    <img src="https://d2908q01vomqb2.cloudfront.net/ca3512f4dfa95a03169c5a670a4c91a19b3077b4/2020/03/09/magnum_f8_sm.png"/>
</p>

## Clean up

To clean up all of our deployed application and infrastructure components, you can simply delete each of the Argo CD applications we created. All of the AWS infrastructure components, as well as their corresponding Kubernetes resources, will be removed from your cluster.

## Conclusion

The Crossplane project enables infrastructure owners to define their custom cloud resources, including managed services, in a standardized way using the Kubernetes API. That in turn enables application developers to author workloads in an abstract way that can be deployed anywhere, and that can be declaratively managed.

mục tiêu : 
# 1.  Tạo ingress và config nó
## kubernetes/ingress-nginx
```yaml
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  name: argocd-server-ingress
  namespace: argocd
  annotations:
    kubernetes.io/ingress.class: nginx
    nginx.ingress.kubernetes.io/force-ssl-redirect: "true"
    nginx.ingress.kubernetes.io/ssl-passthrough: "true"
spec:
  rules:
  - host: argocd.example.com
    http:
      paths:
      - backend:
          serviceName: argocd-server
          servicePort: https
```
## SSL-Passthrough with cert-manager and Let's Encrypt
```yaml
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  name: argocd-server-ingress
  namespace: argocd
  annotations:
    cert-manager.io/cluster-issuer: letsencrypt-prod
    kubernetes.io/ingress.class: nginx
    kubernetes.io/tls-acme: "true"
    nginx.ingress.kubernetes.io/ssl-passthrough: "true"
    # If you encounter a redirect loop or are getting a 307 response code 
    # then you need to force the nginx ingress to connect to the backend using HTTPS.
    #
    # nginx.ingress.kubernetes.io/backend-protocol: "HTTPS"
spec:
  rules:
  - host: argocd.example.com
    http:
      paths:
      - backend:
          serviceName: argocd-server
          servicePort: https
        path: /
  tls:
  - hosts:
    - argocd.example.com
    secretName: argocd-secret # do not change, this is provided by Argo CD
```
# 2. Multiple Ingress Objects And Hosts

```yaml HTTP/HTTPS Ingress:
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  name: argocd-server-http-ingress
  namespace: argocd
  annotations:
    kubernetes.io/ingress.class: "nginx"
    nginx.ingress.kubernetes.io/force-ssl-redirect: "true"
    nginx.ingress.kubernetes.io/backend-protocol: "HTTP"
spec:
  rules:
  - http:
      paths:
      - backend:
          serviceName: argocd-server
          servicePort: http
    host: argocd.example.com
  tls:
  - hosts:
    - argocd.example.com
    secretName: argocd-secret # do not change, this is provided by Argo CD
```


```yaml grpc ingress
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  name: argocd-server-grpc-ingress
  namespace: argocd
  annotations:
    kubernetes.io/ingress.class: "nginx"
    nginx.ingress.kubernetes.io/backend-protocol: "GRPC"
spec:
  rules:
  - http:
      paths:
      - backend:
          serviceName: argocd-server
          servicePort: https
    host: grpc.argocd.example.com
  tls:
  - hosts:
    - grpc.argocd.example.com
    secretName: argocd-secret # do not change, this is provided by Argo CD
```

# 3. Tích hợp treafik

[Tích hợp treafik](https://viblo.asia/p/tong-quan-ve-traefik-XL6lAA8Dlek)

# 4 . Security trong DevSecOps

# 5 . Cluster Boostrapping

# 6. Resource Health và Git Webhook Configuration

# 7. Secret Management

# 8 . Notifications và Troubleshooting Tools

# 9 . Server Configuration Parameters

# 10. argocd-util Tools


