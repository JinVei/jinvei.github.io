---
layout: post
title: "K8S 多集群混合部署"
date: 2025-12-31
tags:
  - k8s
  - openshift
  - cloudnative
---

## K8S 多集群混合部署架构

本文将介绍一种多集群混合部署方案，旨在实现应用在多个 Kubernetes 集群间的统一调度和透明访问，主要由两大核心模块构成：
- **多集群网络**：解决跨集群的服务发现与通信问题。
- **多集群调度**：实现应用在多个集群间的灵活编排与部署。

### 多集群网络互连

多集群网络互连基于 **Submarine** 套件实现底层网络联通。安装 Submarine 后，集群间可通过 Pod IP、Service IP 直接通信，也可通过统一的多集群域名格式 `{app}.{ns}.svc.clusterset.local` 进行访问。

在应用层面，服务间通常使用标准的 Kubernetes Service 域名 `{app}.{ns}.svc` 进行通信。为了尽可能减少应用对底层网络架构的感知（例如，应用被调度到其他集群后无需更改网络配置），并支持应用分散部署在多个集群时仍能通过原生 `{app}.{ns}.svc` 域名进行访问，我们在以下层面进行了增强开发：
- **Istio VirtualService**：用于服务网格内的流量导向。
- **Kubernetes 原生资源**：包括 Service、EndpointSlice 及 ServiceExport。

#### 方案一：基于 Service/EndpointSlice 的跨集群访问

此方案通过扩展 Kubernetes 原生机制，实现跨集群服务发现。

**场景示例**：
- 应用 A 部署于集群 C1。
- 应用 B 部署于集群 C2。
- 目标：使应用 B 在 C2 集群中，仍能通过 `{app-a}.{ns}.svc` 域名访问部署在 C1 集群的应用 A。

**配置步骤**：

1.  **创建基础服务**：在 C1 和 C2 集群中，为应用 A 创建同名的 Service 配置。
2.  **导出服务**：在 C1 集群中创建 `ServiceExport` 资源，声明将应用 A 的服务暴露给其他集群。
    ```yaml
    apiVersion: multicluster.x-k8s.io/v1alpha1
    kind: ServiceExport
    metadata:
      name: app-a
      namespace: ns
    ```
3.  **自动生成端点**：Submarine 的 Lighthouse 组件会自动在 C2 集群的对应命名空间下，创建一个指向 C1 集群服务 IP 的 `EndpointSlice`。
4.  **关联服务与端点**：在 C2 集群中，为此 `EndpointSlice` 添加labels `kubernetes.io/service-name: app-a`，将其关联到本地同名的 Service 上。关键示例如下：
    ```yaml
    metadata:
      labels:
        kubernetes.io/service-name: app-a # 将此 EndpointSlice 关联到本地名为 app-a 的 Service
        multicluster.kubernetes.io/service-name: app-a
        multicluster.kubernetes.io/source-cluster: cluster-c1
    ```
5.  **完成访问**：至此，应用 B 在 C2 集群中即可通过 `{app-a}.{ns}.svc` 域名，透明地访问到部署在 C1 集群的应用 A。

#### 方案二：基于 Istio VirtualService 的跨集群访问

如果应用部署在 Istio 服务网格内，可直接通过配置 VirtualService 和 ServiceEntry 实现更精细的流量控制。

**场景示例**：同上，需在 C2 集群中通过 `{app-a}.{ns}.svc` 访问 C1 集群的应用 A。

**配置步骤**：

1.  **导出服务**：在 C1 集群创建应用 A 的 `ServiceExport`（同方案一）。
2.  **在 C2 集群创建 Istio 资源配置**：
    - **VirtualService**：将发往原生域名的流量重定向到多集群域名。
      ```yaml
      apiVersion: networking.istio.io/v1beta1
      kind: VirtualService
      metadata:
        name: app-a-cross-cluster
        namespace: ns
      spec:
        hosts:
        - app-a.ns.svc.cluster.local # 应用原生的集群内域名
        http:
        - route:
          - destination:
              host: app-a.ns.svc.clusterset.local # 重定向至多集群域名
            weight: 100
      ```
    - **ServiceEntry**：让 Istio 网格能够识别并解析多集群域名 `svc.clusterset.local`。
      ```yaml
      apiVersion: networking.istio.io/v1beta1
      kind: ServiceEntry
      metadata:
        name: external-app-a
        namespace: ns
      spec:
        hosts:
        - app-a.ns.svc.clusterset.local # 声明外部多集群域名
        location: MESH_EXTERNAL
        ports:
        - name: http-7070
          number: 7070
          protocol: HTTP
        - name: http-8080
          number: 8080
          protocol: HTTP
        resolution: DNS # 通过DNS解析
      ```

### 多集群调度

调度部分基于 **KubeVela** 的多集群应用交付能力实现，通过声明式配置定义应用在多个集群间的部署策略。

**核心流程**：
1.  **集群接入**：使用 `vela cluster join` 命令将目标集群纳入统一管理。
    ```bash
    vela cluster join prodc-cluster.kubeconfig --name prodc
    vela cluster join prod-cluster.kubeconfig --name prod
    vela cluster join localproda-cluster.kubeconfig --name localproda
    ```
2.  **应用编排**：在 Application 资源中，通过 `policies` 定义部署策略（如目标集群、副本数、配置差异），通过 `workflow` 定义多集群部署的顺序与步骤。

**配置示例**：
```yaml
kind: Application
apiVersion: core.oam.dev/v1beta1
metadata:
  name: app-a
  namespace: ns
spec:
  components: [...]
  policies:
    # 定义多种策略：缩容、占位符、活跃部署、集群选择、镜像仓库覆盖等
    - name: scaledown
      type: override
      properties: { components: [{ type: fcwebservice, properties: { replicas: 0 } }] }
    - name: multicluster-placeholder
      type: override
      properties: { components: [{ type: fcwebservice, properties: { multiClusterPlaceholder: 'true' } }] }
    - name: multicluster-active
      type: override
      properties: { components: [{ type: fcwebservice, properties: { multiClusterPlaceholder: 'false' } }] }
    - name: select-cluster-prodc
      type: topology
      properties: { clusters: ['prodc'] }
    - name: select-cluster-localproda
      type: topology
      properties: { clusters: ['localproda'] }
    - name: external-registry
      type: override
      properties: { components: [{ type: fcwebservice, properties: { externalRegistry: 'registry.example.com' } }] }

  workflow:
    steps:
      # 第一步：部署到 prodc 集群，作为活跃实例
      - name: deploy-prodc
        type: deploy
        properties:
          policies: [select-cluster-prodc, multicluster-active]
      # 第二步：部署到 localproda 集群，作为占位符实例（副本数为0），并使用特定镜像仓库
      - name: deploy-localproda
        type: deploy
        properties:
          policies: [select-cluster-localproda, external-registry, multicluster-placeholder, scaledown]
```

**策略说明**：
- `multicluster-active`：标识应用的实际部署位置，会在该集群自动初始化网络配置（如创建 `ServiceExport`）。
- `multicluster-placeholder`：标识应用的“占位”部署，用于在其他集群注册服务信息并初始化网络配置（如创建 `VirtualService` 和 `ServiceEntry`），从而实现服务的全局可发现性。

通过以上网络与调度机制的配合，实现了应用在多集群环境下的统一管理、灵活调度与透明访问。