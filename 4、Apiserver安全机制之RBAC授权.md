### 一、k8s-Apiserver安全机制之RBAC授权-笔记

#### 1·k8s·认证、授权、准入控制机制

Kubernetes (K8s) 在处理用户或服务对集群的操作请求时，会经历三个关键的安全阶段：**认证 (Authentication)**、**授权 (Authorization)** 和**准入控制 (Admission Control)**。

- **认证 (Authentication):** 这是请求进入集群的第一道防线。它的目标是**验证请求者的身份**。Kubernetes 支持多种认证方式，包括<span style="color:red">客户端证书</span>、用户名和密码、Bearer Token（例如服务账户的Token）、OpenID Connect (OIDC) 等。只有通过认证的请求，才会被认为是合法的，并进入下一个阶段。

- **授权 (Authorization):** 在请求者的身份被确认后，授权阶段会**判断该请求者是否有权限执行其请求的操作**。例如，用户“Alice”是否被允许在“default”命名空间中创建 Pod。如果请求者被授权，请求会继续；否则，请求会被拒绝。RBAC (Role-Based Access Control) 是 Kubernetes 最常用和推荐的授权机制。
- **准入控制 (Admission Control):** 这是请求进入集群的最后一道关卡。准入控制器是一组在请求被持久化到 Etcd 之前拦截请求的插件。它们可以用于**修改请求**（如注入 sidecar）、**验证请求**（如确保所有 Pod 都有资源限制）、甚至**拒绝请求**（如不允许创建特权容器）。准入控制提供了一种强大的方式来实施集群级别的策略。

#### 2·认证和授权基本介绍

##### 	2.1 Useraccount介绍

​	Kubernetes 本身**不管理用户账号**，它依赖外部系统（如证书、OAuth、令牌等）来识别用户身份

| 类型                           | 说明                                                         |
| ------------------------------ | ------------------------------------------------------------ |
| **UserAccount（普通用户）**    | UserAccounts 是为Kubernetes 集群外部的用户设计的，通常用于运维或集群管理人员 |
| **ServiceAccount（服务账号）** | 用于 Pod 或 Controller 内部访问 API 的账号（自动创建）       |

​	在使用kubeadm 安装Kubernetes时，默认的用户账号是**kubernetes-admin**。用户通过 Kubernetes 客户端(通常使用 kubectl)与 APIServer 进行交互用的就是kubernetes-admin。kubernetes-admin这个用户就是 useraccount，可通过以下命令查看

```bash
kubectl config view --kubeconfig ./.kube/config 
```

​	当 Pod 中的容器进程需要访问 APIServer 时，使用的就是 ServiceAccount 账户。ServiceAccount 仅在其所在的命名空间中有效。在每个命名空间创建时，都会自动创建一个名为"default"的 ServiceAccount。如果在创建 Pod 时没有指定 ServiceAccount， Pod 将使用默认的 ServiceAccount

##### 	2.2 授权

RBAC<span style="color:red">(基于角色的访问控制)</span>是Kubernetes中的一种授权机制，它允许用户扮演特定角色并获得相应的权限。在 RBAC中，权限被定义在角色(Role)中，用户被绑定到这些角色上，从而获得相应的权限。

| 项目               |                **Role**                 |                    **ClusterRole**                    |                   **RoleBinding**                    |       **ClusterRoleBinding**       |
| ------------------ | :-------------------------------------: | :---------------------------------------------------: | :--------------------------------------------------: | :--------------------------------: |
| **作用**           |         定义命名空间内权限规则          |                 定义集群范围权限规则                  | 把 Role/ClusterRole 授权给主体（用户、组或服务账号） |  同RoleBinding，但作用于整个集群   |
| **作用范围**       |              某个命名空间               |               所有命名空间或集群级资源                |                     某个命名空间                     |               全集群               |
| **资源类型支持**   |    命名空间资源（Pods、Services 等）    | 命名空间资源 + 集群资源（Nodes、Namespaces、CRDs 等） |              把权限授予命名空间内某主体              | 把权限授予所有命名空间或集群级主体 |
| **绑定的角色类型** |                    \                    |                           \                           |                 Role 或 ClusterRole                  |        只能绑定 ClusterRole        |
| **典型用途**       | 限制某用户只能读写某个命名空间中的资源  |   给管理员、监控程序或控制器设置跨命名空间/集群权限   | 授权命名空间内某个服务账号使用 Role/ClusterRole 权限 |    授权全局权限给用户或服务账号    |
| **常用对象类型**   | Pods、ConfigMaps、Secrets（命名空间内） |     Nodes、Namespaces、PersistentVolumes、CRD 等      |          ServiceAccount、User（绑定 Role）           |   监控服务、系统组件、管理员账号   |



###### 2.2.1 如果用户通过 RoleBinding 绑定到 Role，那么用户只会在该 Role 所在的名称空间中拥有权

![image-20250625172226983](./k8s/image-20250625172226983.png)

```bash
kubectl create rolebinding read-pods-binding \      # read-pods-binding：新建的 RoleBinding 名称
  --role=pod-reader \							    # 绑定的角色（Role 或 ClusterRole）
  --serviceaccount=dev:my-sa \					    # 授予 dev 命名空间下的 my-sa 服务账号
  --namespace=dev									# 作用范围是 dev 命名空间
```



###### 	2.2.2 如果用户通过 RoleBinding 绑定到 ClusterRole，用户仅在 RoleBinding 所在的名称空间中拥有相应的权限。

![image-20250625172417504](./k8s/image-20250625172417504.png)

###### 	2.2.3 通过 ClusterRoleBinding 绑定ClusterRole

通过 ClusterRoleBinding 将用户绑定到 ClusterRole，用户将在整个集群中拥有相应的权限。

![image-20250625172711038](./k8s/image-20250625172711038.png)

kubernetes-admin（User）可以通过ClusterRoleBinding绑定到ClusterRole上的，根据下面命令查看

```bash
kubectl get clusterrole                         
#NAME                                                                   CREATED AT
#admin                                                                  2025-06-19T06:58:43Z
kubectl get clusterrole cluster-admin -o yaml
#kind: ClusterRole
#metadata:
#  annotations:
#    rbac.authorization.kubernetes.io/autoupdate: "true"
#  creationTimestamp: "2025-06-19T06:58:43Z"
#  labels:
#   kubernetes.io/bootstrapping: rbac-defaults
#  name: cluster-admin
#  resourceVersion: "72"
#  uid: 595a8c69-577a-484b-976d-d30a8a0d10a7
#rules:
#- apiGroups:
#  - '*'
#  resources:
#  - '*'
#  verbs:
#  - '*'
#- nonResourceURLs:
#  - '*'
#  verbs:
#  - '*'
```



#### 3·RBAC 认证授权策略

##### 1.RBAC 的核心概念包括：

- **Role (角色):** Role 是**命名空间内**的权限集合。它定义了一组可以在特定命名空间中执行的操作。例如，一个 Role 可以定义为“允许在 `dev` 命名空间中查看 Pod 和日志”。

- **ClusterRole (集群角色):** ClusterRole 是**集群范围**的权限集合。它定义了可以在整个集群中执行的操作，或者跨所有命名空间的操作（如查看所有命名空间中的节点），或者非资源型操作（如 `/healthz`）。例如，一个 ClusterRole 可以定义为“允许在整个集群中管理节点”。

- **RoleBinding (角色绑定):** RoleBinding 将一个 **Role** 绑定到一个或多个 **User (用户)**、**Group (用户组)** 或 **ServiceAccount (服务账户)**。这个绑定是**命名空间内**的。这意味着被绑定的用户/组/服务账户将在该 Role 所在的命名空间内拥有该 Role 定义的权限。

- **ClusterRoleBinding (集群角色绑定):** ClusterRoleBinding 将一个 **ClusterRole** 绑定到一个或多个 **User (用户)**、**Group (用户组)** 或 **ServiceAccount (服务账户)**。这个绑定是**集群范围**的。这意味着被绑定的用户/组/服务账户将在整个集群中拥有该 ClusterRole 定义的权限。

  **创建role：**

  ```yaml
  apiVersion: rbac.authorization.k8s.io/v1
  kind: Role
  metadata:
    name: pod-reader
    namespace: dev
  rules:
  - apiGroups: [""]
    resources: ["pods"]
    verbs: ["get", "list", "watch"]
  ```

  **创建 ClusterRole：**
  
  ```yaml
  apiVersion: rbac.authorization.k8s.io/v1
  kind: ClusterRole
  metadata:
    name: cluster-readonly
  rules:
  - apiGroups: [""]
    resources: ["pods", "services", "configmaps"]
    verbs: ["get", "list", "watch"]
  - apiGroups: ["apps"]
    resources: ["deployments"]
    verbs: ["get", "list", "watch"]
  - nonResourceURLs: ["/healthz","/healthz/*"]  #允许对非资源端点“/healthz”及其所有子路径进行 GET和 POST 操作
    verbs: ["get","post"]						  #必须使用ClusterRole 和ClusterRoleBinding)
  ```
  
  **附加说明：常见的 API Group 对应资源**
  
  | `apiGroups` 值                | 包含的资源示例                                     |
  | :---------------------------- | :------------------------------------------------- |
  | `""`（空字符串）              | pods, services, configmaps, secrets, namespaces    |
  | `"apps"`                      | deployments, daemonsets, statefulsets, replicasets |
  | `"batch"`                     | jobs, cronjobs                                     |
  | `"rbac.authorization.k8s.io"` | roles, clusterroles, rolebindings                  |
  | `"networking.k8s.io"`         | ingress, networkpolicies                           |
  | `"apiextensions.k8s.io"`      | customresourcedefinitions（CRD）                   |
  
  **RBAC 授权流程：**
  
  1. 用户/ServiceAccount 发送请求到 API Server。
  2. API Server 认证请求者身份。
  3. API Server 检查请求者所属的 RoleBinding 或 ClusterRoleBinding。
  4. 根据绑定，查找对应的 Role 或 ClusterRole 定义的权限。
  5. 如果请求的操作匹配到权限定义，则授权成功；否则，授权失败。
  
  **示例：**
  
  如果想让用户“dev-user”在“development”命名空间中管理 Pods，你可以这样做：
  
  1. 定义一个 Role：
  
     ```yaml
     apiVersion: rbac.authorization.k8s.io/v1
     kind: Role
     metadata:
       namespace: development
       name: pod-manager
     rules:
     - apiGroups: [""] # "" 表示核心 API 组
       resources: ["pods", "pods/log"]
       verbs: ["get", "list", "watch", "create", "update", "patch", "delete"]
      
     ```
  
  2. 定义一个 RoleBinding：
  
     ```yaml
     apiVersion: rbac.authorization.k8s.io/v1
     kind: RoleBinding
     metadata:
       name: dev-user-pod-manager
       namespace: development
     subjects:
     - kind: User
       name: dev-user # 替换为你的用户名称
       apiGroup: rbac.authorization.k8s.io
     roleRef:
       kind: Role
       name: pod-manager
       apiGroup: rbac.authorization.k8s.io
     ```
  
     或者
  
     ```bash
     kubectl create rolebinding dev-user-pod-manager \      
       --role=pod-manager \							    
       --serviceaccount=dev:dev-user \					    
       --namespace=development									
     ```

#### 4·对 Service·Account 的授权管理

**ServiceAccount** 是 Kubernetes 中特殊的账户，它们为在 Pod 中运行的进程提供身份。当 Pod 需要与 Kubernetes API Server 交互时，它们通常使用 ServiceAccount 关联的 Token 进行认证。

对 ServiceAccount 的授权管理是 RBAC 的一个重要应用场景，因为许多应用程序都需要通过 ServiceAccount 来访问集群资源。

- **默认 ServiceAccount:** 每个命名空间都会自动创建一个名为 `default` 的 ServiceAccount。Pod 如果没有明确指定 ServiceAccount，就会使用该默认 ServiceAccount。

- **创建自定义 ServiceAccount:** 最佳实践是为每个需要与 API Server 交互的应用程序创建专门的 ServiceAccount，并赋予其最小所需权限。

- **绑定权限:**

  - **命名空间级别:** 通过 `RoleBinding` 将 `Role` 绑定到 ServiceAccount，使其在特定命名空间内拥有权限。
  - **集群级别:** 通过 `ClusterRoleBinding` 将 `ClusterRole` 绑定到 ServiceAccount，使其在整个集群中拥有权限。
  
  **示例：为部署在 `my-app` 命名空间中的应用程序赋予查看 Pod 的权限：**
  
  1. 创建 ServiceAccount：
  
     ```yaml
     apiVersion: v1
     kind: ServiceAccount
     metadata:
       name: my-app-sa
       namespace: my-app
     ```
  
     或
  
     ```bash
     kubectl create sa my-app-sa -n my-app
     ```
  
  2. 定义一个 Role：
  
     ```yaml
     apiVersion: rbac.authorization.k8s.io/v1
     kind: Role
     metadata:
       namespace: my-app
       name: pod-reader
     rules:
     - apiGroups: [""]
       resources: ["pods"]
       verbs: ["get", "list"]
     ```
  
  3. 创建 RoleBinding 将 Role 绑定到 ServiceAccount：
  
     ```yaml
     apiVersion: rbac.authorization.k8s.io/v1
     kind: RoleBinding
     metadata:
       name: my-app-sa-pod-reader
       namespace: my-app
     subjects:
     - kind: ServiceAccount
       name: my-app-sa
       namespace: my-app
     roleRef:
       kind: Role
       name: pod-reader
       apiGroup: rbac.authorization.k8s.io
     ```
  
     或者
  
     ```sh
     kubectl create rolebinding my-app-sa-pod-reader \      
       --role=pod-reader \							    
       --serviceaccount=my-app:my-app-sa \					    
       --namespace=my-app	
     ```
  
     
  
  4. 在 Pod 中指定使用该 ServiceAccount：
  
     ```yaml
     apiVersion: v1
     kind: Pod
     metadata:
       name: my-app-pod
       namespace: my-app
     spec:
       serviceAccountName: my-app-sa
       containers:
       - name: my-app-container
         image: your-app-image
     ```

#### 5·限制不同的用户操作 k8s 集群

##### 1.ssl证书

生成一个证书
(1)生成一个私钥：

```sh
cd /etc/kubernetes/pki/
(umask 077; openssl genrsa -out lucky.key 2048)
```

(2)生成一个证书请求：

```sh
openssl reg -new -key lucky.key -out lucky.csr -subj "/CN=lucky"   #证书将标识其所有者为 "lucky 定义在config文件中																	  的user （替换掉kubernetes-admin）
```

(3)生成一个证书

```sh
openssl x509 -reg -in lucky.csr -CA ca.crt -CAkey ca.key -CAcreateserial -out lucky.crt.days 3650
```

(4)指定要访问的 k8s 集群

```sh
kubectl conig set-cluster kubernetes --server=https://IP:6443 --insecure-skip-tls-verify=true --kubeconfig=/root/lucky
```

(5)把lucky这个用户添加到 lucky 文文件，可以用来认证 apiserver 的连接

```sh
kubectl config set-credentials lucky --client-certificate=./lucky.crt --client-key=./lucky.key --embed-certs=true --kubeconfig=/root/lucky
```

(6)创建上下文

```sh
kubectl config set-context lucky@kubernetes --cluster=kubernetes --user=luckykubeconfig=/root/lucky
```

(7)切换账号到lucky，默认没有任何权限

```sh
kubectl config use-context lucky@kubernetes --kubeconfig=/root/lucky
```

把 lucky 这个用户通过 rolebinding 绑定到 clusterrole 上，授予权限,权限只是在lucky 这个名称

```sh
kubectl create ns lucky
```

```sh
kubectl create rolebinding lucky -n lucky --clusterrole=cluster-acmin --user=lucKy
```

##### 2.kubernetes-admin默认配置

```yaml
apiVersion: v1
clusters:
- cluster:
    certificate-authority-data: DATA+OMITTED
    server: https://10.1.20.3:6443
  name: kubernetes
contexts:
- context:
    cluster: kubernetes
    user: kubernetes-admin
  name: kubernetes-admin@kubernetes
current-context: kubernetes-admin@kubernetes
kind: Config
preferences: {}
users:
- name: kubernetes-admin
  user:
    client-certificate-data: DATA+OMITTED
    client-key-data: DATA+OMITTED
```

##### 3.常见的实践和策略：

- **最小权限原则 (Principle of Least Privilege):** 始终只授予用户或服务账户执行其任务所需的最小权限。避免授予不必要的管理权限，特别是集群管理员权限。

- **命名空间隔离:** 尽可能地利用命名空间进行资源隔离和权限划分。为不同的团队或应用程序创建独立的命名空间，并为每个命名空间定义独立的 Role 和 RoleBinding。

- 区分用户和管理员:

  - **普通用户:** 通常只授予其在特定命名空间内操作特定资源的权限，例如开发者只能在自己的开发命名空间中创建和管理应用。
  - **集群管理员:** 谨慎授予 `cluster-admin` ClusterRole，它拥有集群的所有权限。通常只为少数核心运维人员分配此角色。

- 使用预定义的 ClusterRole:

   Kubernetes 提供了一些预定义的 ClusterRole，例如  `view`、 `edit`、`admin`等，可以简化权限配置。

   - `view`: 允许查看大多数资源。
  - `edit`: 允许读写大多数资源，但不允许修改角色或角色绑定。
  - `admin`: 允许对命名空间内的几乎所有资源进行读写，包括修改角色和角色绑定。
  
- **限制 Secret 访问:** Secret 包含敏感信息，应严格控制对其的访问权限。确保只有少数特权用户和必要的 ServiceAccount 才能访问 Secret。

- **审计日志:** 启用 Kubernetes 审计日志，记录所有对 API Server 的请求。这有助于监控和审查用户行为，及时发现异常操作。

- **定期审查权限:** 定期审查用户的 RBAC 权限配置，确保其仍然符合最小权限原则，并移除不再需要的权限。