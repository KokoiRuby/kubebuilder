### Versioning

kubebuilder `vx.y.z`

project `"x..."`

plugin `vx...` → `v(x+1)-alpha`

Project versions should only be increased if a **breaking change** is introduced in the `PROJECT` file scheme itself.

Changes to the Go **scaffolding** or the Kubebuilder **CLI** *do not* affect project version.

Introduction of a new plugin version might only lead to a new minor version release of Kubebuilder, tho no breaking change to CLI itself.