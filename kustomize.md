# kustomize

> Kustomize实现不同环境的配置派生。
>
> Kustomize 允许用户以一个应用描述文件 （YAML 文件）为基础（Base YAML），然后通过 Overlay 的方式生成最终部署应用所需的描述文件。

## 解决了什么问题？

一般应用都会存在多套部署环境：开发环境、测试环境、生产环境，多套环境意味着存在多套 K8S 应用资源 YAML。而这么多套 YAML 之间只存在微小配置差异，比如镜像版本不同、Label 不同等，而这些不同环境下的YAML 经常会因为人为疏忽导致配置错误。再者，多套环境的 YAML 维护通常是通过把一个环境下的 YAML 拷贝出来然后对差异的地方进行修改。

Kustomize 通过以下几种方式解决了上述问题：

- kustomize 通过 Base & Overlays 方式(下文会说明)方式维护不同环境的应用配置
- kustomize 使用 patch 方式复用 Base 配置，并在 Overlay 描述与 Base 应用配置的差异部分来实现资源复用
- kustomize 管理的都是 Kubernetes 原生 YAML 文件，不需要学习额外的 DSL 语法

## 术语

### base

base 指的是一个 kustomization , 任何的 kustomization 包括 overlay (后面提到)，都可以作为另一个 kustomization 的 base (简单理解为基础目录)。base 中描述了共享的内容，如资源和常见的资源配置。

### overlay

overlay 是一个 kustomization, 它修改(并因此依赖于)另外一个 kustomization. overlay 中的 kustomization指的是一些其它的 kustomization, 称为其 base. 没有 base, overlay 无法使用，并且一个 overlay 可以用作 另一个 overlay 的 base(基础)。简而言之，overlay 声明了与 base 之间的差异。通过 overlay 来维护基于 base 的不同 variants(变体)，例如开发、QA 和生产环境的不同 variants。

### variant

variant 是在集群中将 overlay 应用于 base 的结果。例如开发和生产环境都修改了一些共同 base 以创建不同的 variant。这些 variant 使用相同的总体资源，并与简单的方式变化，例如 deployment 的副本数、ConfigMap使用的数据源等。简而言之，variant 是含有同一组 base 的不同 kustomization

 ### patch

修改文件的一般说明。文件路径，指向一个声明了 kubernetes API patch 的 YAML 文件



## kustomize官方示例

文件结构

```tex
demo
├── base
│   ├── configMap.yaml
│   ├── deployment.yaml
│   ├── kustomization.yaml
│   └── service.yaml
└── overlays
    ├── production
    │   ├── deployment.yaml
    │   └── kustomization.yaml
    └── staging
        ├── kustomization.yaml
        └── map.yaml
```

```tex
├── base
│   ├── deployment.yaml
│   ├── kustomization.yaml
│   └── service.yaml
└── overlays
    ├── dev
    │   ├── deployment.yaml
    │   ├── kustomization.yaml
    │   └── service.yaml
    ├── prd
    │   ├── deployment.yaml
    │   ├── kustomization.yaml
    │   └── service.yaml
    └── pre
        ├── deployment.yaml
        ├── kustomization.yaml
        └── service.yaml
```



![base](/Users/liyang/Documents/k8s/kustomize.assets/base.jpeg)

该目录下的 Yaml 是不完整的，可以被补充、覆盖，因此该 Yaml 被称为 Base，而补充 Base 的 Yaml 被称作 Overlay，可以补充适用不同环境、场景的 overlay yaml 到该 app 中，从而使得 K8s 应用描述更完整。

![overlay](/Users/liyang/Documents/k8s/kustomize.assets/overlay.jpeg)

kustomize 通过描述文件的叠加，来生成完整的部署用 Yaml。

## 工作原理

kustomize 将对 K8s 应用定义为由 Yaml 表示的资源描述，对 K8s 应用的变更可以映射到对 Yaml 的变更上。而这些变更操作可以利用 Git 等版本控制程序来管理，因此用户得以使用 Git 风格的流程对 K8s 应用进行管理。

kustomize 将对 Kubernetes 应用的管理转换成对 Kubernetes manifests YAML 文件的管理，而对应用的修改也通过 YAML 文件来修改。这种修改变更操作可以通过 Git 版本控制工具进行管理维护, 因此用户可以使用 Git 风格的流程来管理应用。 workflows 是使用并配置应用所使用的一系列 Git 风格流程步骤。

![operating-2](/Users/liyang/Documents/k8s/kustomize.assets/operating-2.png)

​	

![operating](/Users/liyang/Documents/k8s/kustomize.assets/operating.jpeg)

> 当你从GitHub上 clone 一个 repo 到本地时，除非你已明确声明是这个repo的contributor，否则你是不能向其pull request的，此时，该远程的repo对于本地repo来说，就是upstream。
> 当你从GitHub上 fork 一个 repo 之后，再 clone forked repo 到本地，你就可以任意向其pull request，此时，远程的 repo 就是 origin。

