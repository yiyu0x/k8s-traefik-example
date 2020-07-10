###### tags: `blog`
# 在 K8S 中使用 Traefik 作為 Ingress Controllers

## 前言

K8S 中的 Ingress Controllers 並沒有實作，可以自己使用各種版本。在 Minikube 中的 Ingress Controllers 預設是用 [NGINX Ingress Controller](https://kubernetes.github.io/ingress-nginx/)，然而 NGINX Ingress Controller 的功能非常陽春，官網有提供一系列的 [Ingress Controller](https://kubernetes.io/docs/concepts/services-networking/ingress-controllers/) 給各位參考，該文章紀錄如何使用 [Traefik](https://docs.traefik.io/) 這個 Edge Router 作為 Ingress Controller。


## 正文

在操作 K8S 之前請先確定你正在操作的叢集是練習環境：

`kubectl cluster-info`

Traefik 這套工具可以將外部的流量吸收，並且透過自己定義的路由來將流量往內部的 Service 打。我們希望可以透過 URL 的 Path 來定義不同服務的路由，將流量打向不同的服務。

在 NGINX Ingress Controller 中要做這件事情，要在 `annotations` 中加上 `nginx.ingress.kubernetes.io/rewrite-target: /`

```yaml
apiVersion: networking.k8s.io/v1beta1
kind: Ingress
metadata:
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
  name: rewrite
  namespace: default
spec:
...
...
...
```

避免不同 App 吃到不同的路由位置無法產生正確的回應（假設我們的服物入口都在根路徑 Ex: `app-foo/` ）。

但 Traefik 要達成這件事情必須透過 [Middleware](https://docs.traefik.io/middlewares/overview/) 這個組件（Traefik 定義的功能），簡單來說我們要把讀進來的 /path，strip 掉並且將導向到指定的 Service 上。

![](https://docs.traefik.io/assets/img/middleware/overview.png)

要使用 Traefik Middleware 有兩種方式：

- IngressRoute (CustomResourceDefinitions)
- 在 Ingress 中使用 annotation 定義

個人覺得第二點比較直覺，所以選擇第二種方式。

首先，先建立 Middleware 這個 CRD，並且做 ClusterRoleBinding：

```yaml
apiVersion: apiextensions.k8s.io/v1beta1
kind: CustomResourceDefinition
metadata:
  name: middlewares.traefik.containo.us

spec:
  group: traefik.containo.us
  version: v1alpha1
  names:
    kind: Middleware
    plural: middlewares
    singular: middleware
  scope: Namespaced

---
kind: ClusterRole
apiVersion: rbac.authorization.k8s.io/v1beta1
metadata:
  name: traefik-ingress-controller

rules:
  - apiGroups:
      - ""
    resources:
      - services
      - endpoints
      - secrets
    verbs:
      - get
      - list
      - watch
  - apiGroups:
      - extensions
    resources:
      - ingresses
    verbs:
      - get
      - list
      - watch
  - apiGroups:
      - extensions
    resources:
      - ingresses/status
    verbs:
      - update
  - apiGroups:
      - traefik.containo.us
    resources:
      - middlewares
      - ingressroutes
      - traefikservices
      - ingressroutetcps
      - ingressrouteudps
      - tlsoptions
      - tlsstores
    verbs:
      - get
      - list
      - watch

---
kind: ClusterRoleBinding
apiVersion: rbac.authorization.k8s.io/v1beta1
metadata:
  name: traefik-ingress-controller

roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: traefik-ingress-controller
subjects:
  - kind: ServiceAccount
    name: traefik-ingress-controller
    namespace: default
```

接下來將 Traefik 本身跑起來：

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  namespace: default
  name: traefik-ingress-controller

---
kind: Deployment
apiVersion: apps/v1
metadata:
  namespace: default
  name: traefik
  labels:
    app: traefik

spec:
  replicas: 1
  selector:
    matchLabels:
      app: traefik
  template:
    metadata:
      labels:
        app: traefik
    spec:
      serviceAccountName: traefik-ingress-controller
      containers:
        - name: traefik
          image: traefik:v2.2
          args:
            - --api.insecure
            - --accesslog
            - --entrypoints.web.Address=:8000
            - --providers.kubernetesingress
            - --providers.kubernetescrd
          ports:
            - name: web
              containerPort: 8000
            - name: admin
              containerPort: 8080
```

為 Traefik 設定 Service：

```yaml
apiVersion: v1
kind: Service
metadata:
  name: traefik

spec:
  ports:
    - protocol: TCP
      name: web
      port: 8000
    - protocol: TCP
      name: admin
      port: 8080
    - protocol: TCP
      name: websecure
      port: 4443
  selector:
    app: traefik
```

建立 Ingress 與 Middleware(stripprefix)：

```yaml
apiVersion: networking.k8s.io/v1beta1
kind: Ingress
metadata:
  name: yy-ingress
  annotations:
    traefik.ingress.kubernetes.io/router.entrypoints: web
    traefik.ingress.kubernetes.io/router.middlewares: default-stripprefix@kubernetescrd
spec:
  rules:
  - host: yy.k8s                           
    http:
      paths:
      - path: /
        backend:
          serviceName: whales
          servicePort: 80
      - path: /app
        backend:
          serviceName: whales
          servicePort: 80
      - path: /whoami
        backend:
          serviceName: whoami
          servicePort: 80
  - host: admin.yy.k8s
    http:
      paths:
        - path: /
          backend:
            serviceName: traefik
            servicePort: 8080

---
apiVersion: traefik.containo.us/v1alpha1
kind: Middleware
metadata:
  name: stripprefix
spec:
  stripPrefix:
    prefixes:
      - /app
      - /whoami
```

要注意的重點有：
- CRD 中的權限要記得 middwares 有被加進去。
- Traefik 跑起來時要加上 `--providers.kubernetesingress`
- 如果用 IngressRoute 跑，要加上 `--providers.kubernetescrd`

更多詳細設定可以參考 [user-guides/crd-acme](https://docs.traefik.io/user-guides/crd-acme/#ingressroute-definition)，以上的設定比較精簡，此文章只有留必要部分而已。

完整的設定可以到 [GitHub Repo](https://github.com/yiyu0x/k8s-traefik-example) 上面看。

