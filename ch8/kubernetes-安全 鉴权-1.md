#### Authorization

上面认证过程，只是确认通信双方是可信的，可以相互通信。而鉴权是确定请求方有哪些资源的权限。API Server目前支持以下几种授权策略(通过API Server的启动参数"--authorization-mode"设置)

* AlwaysDeny： 拒绝所有请求，一般用于测试
* AlwaysAllow：允许所有请求，如果集群不需要授权流程，则可以采用该策略
* ABAC： 基于属性的访问控制，表示使用用户配置的授权规则对用户请求进行匹配和控制
* Webhook:通过调用外部REST服务对用户进行授权
* RBAC：基于角色的访问控制，现行默认规则

#### RBAC授权模式

**RBAC(Role-Based Access Control)基于角色的访问控制，在kubernetes .15中引入，现行版本为默认标准。相对于其它的访问控制方式，拥有以下优势**

* 对集群中的资源和非资源均拥有完整的覆盖
* 整个RBAC完全由几个API对象组成，同其它API对象一样，可以用kubectl或API进行操作
* 可以在运行时进行调整，无序重启API Server

1、RBAC的API资源对象说明

RBAC引入了4个新的顶级资源对象：**Role、ClusterRole、RoleBinding、ClusterRoleBinding**，4种对象类型均可以通过kubectl与API操作







