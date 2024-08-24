## [kubebuilder](https://sigs.k8s.io/kubebuilder)

**Kubernetes 使用者**将通过学习 API 是如何设计和实现的，获得对 Kubernetes 更深入的了解。

- 如何构造 Kubernetes API 和 Resources
- 如何进行 API 版本控制
- 如何实现故障自愈
- 如何实现垃圾回收和 Finalizers
- 如何创建声明式和命令式 API
- 如何创建 Level-Based API 和 Edge-Base API
- 如何创建 Resources 和 Subresources

**Kubernetes 开发者**将学习实现标准的 Kubernetes API 的原理和概念，以及用于快速构建 API 的工具和库

- 如何用一个 reconciliation 方法处理多个 events
- 如何定期执行 reconciliation 方法
- 何时使用 lister cache 与 live lookups 实时查找
- 如何垃圾回收和 Finalizers
- 如何使用 Declarative Validation 和 Webhook Validation
- 如何实现 API 版本控制

### [Architecture](https://book.kubebuilder.io/architecture)

- **Process** `main.go`
  - **Manager** per process.
    - **Client** interacts with API Server
    - **Cache** holds recently watched or GET'ed objects. Use Clients
    - **Controller** per Kind. Use Cache & Client & get event by Filter & call Reconciler per event
      - **Predicate** filters events
      - **Reconciler** handles business logic 
    - **Webhook** per Kind; Default & Validator

