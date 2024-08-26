**Reconciliation in Operators?**

`cmd/main.go` initializes a **Manager** which manages **Controllers** that offer a reconcile func that **drives resource until desired state is achived**. 驱动资源到期望的状态。

Reconciliation 必须幂等。

构建 Operator 涉及扩展 K8s API，CRD 是一种扩展的手段。

理解 [Operator patterns](https://kubernetes.io/docs/concepts/extend-kubernetes/operator/)。

遵循 [Kubernetes API conventions and standards](https://github.com/kubernetes/community/blob/master/contributors/devel/sig-architecture/api-conventions.md)

- 保证和 K8s 生态的互用性 interoperability
- 提高可维护性和易排错
- 提升平台的健壮性

**Why not a single controller for multiple CRDs?**

- 违反 controller 的设计原则
- 违反封装和单一职责原则
- 复杂度 ↑
- 扩容效率 ↓
- 破坏错误隔离
- 影响并发同步

**Why not multiple controllers for a CRD?**

- 竞争条件
- 并发问题
- 不易维护
- 状态追踪和一致性
- 性能问题

**使用 Status Conditions**

- 标准化
- 可读性
- 扩展性
- 可观察性
- 兼容性







