#### 机制说明

**kubernetes作为一个分布式集群的管理工具，保证集群的安全性是其一个中药的任务。API Server是建群内部各个组件通信的中介，也是外部控制的入口。所以kubernetes的安全机制基本就是围绕保护API Server来设计的。kubernetes使用了认证(Authentication)、鉴权(Authorization)、准入控制(Admission Control)三步来保证API Server的安全**

















