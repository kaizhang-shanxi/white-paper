# lvault 维护文档

lvault 有两个重要的概念，分别是 key 和 root_token.
- key: 默认为 1 个，可以指定为 n(n>0) 个；若 n>1, 则由其中的一部分生成主密钥。key 的作用是解密存在 vault 存储后端的数据。
- root_token: ACL 控制手段。 

lvault 的一个基本原则是 key 和 root_token 只存在集群的内存中，若是发生了集群重启等重大事故，lvault 不能自己恢复，则需要人为解锁.

刚刚 bootstrap 的集群，lvault 并没有初始化，原因在于这一步骤需要管理员的参与，
我们不建议将 key 和 root_token 落地到集群内部. 下面已域名为 lain.local 为例，
简单说明一下初始化和解锁的工作流.

## 初始化和解锁

当集群刚刚 bootstrap 后，需要先初始化 lvault, 再解锁，步骤如下：
1. 初始化
	* 初始化操作一般来说对一个集群只需要一次，除非忘记 key，可以删除 etcd 的数据后再次初始化.
	* curl -XPUT http://lvault.lain.local/v2/init -d '{}'
	* 这里传入空值，是因为一般只需要用默认值即可
	* 注意，一定要将反馈的 key 和 root_token 记下来，今后解锁等操作需要用到, 主要用于 /v2/reset 操作.
1. 解锁
	* 刚刚初始化后，可以直接执行 curl -XPUT http://lvault.lain.local/v2/unsealall -d '{}'
	* 目前，为了减少误操作，unsealall 不需要传入参数，lvault 会用内存中记录的 key 进行解锁操作。 

## 故障修复

一般建议 lvault.web.lvault 应该 scale 为两个，这样丢失 key 和 root_token 的概率会大大降低. 下面给出目前一般的工作流, 前提是集群的网络问题已经解决。分为三步：

1. 尝试 reset key 和 root_token, 请勿直接粘贴，需要用自己的 key 和 root_token
```
curl -v -X PUT http://lvault.lain.local/v2/reset -d '{"keys":["193e7e1452679fcec7d06e96438752f4be7434abdc0d8632ac79c6cdb6911792"],"root_token":"4408bbd6-bc3f-1cba-08aa-6c0a12ff4e3d"}'
```
2. 解锁
```
curl -XPUT http://lvault.lain.local/v2/unsealall -d '{}'
```
3. 再次 reset key
```
curl -v -X PUT http://lvault.lain.local/v2/reset -d '{"keys":["193e7e1452679fcec7d06e96438752f4be7434abdc0d8632ac79c6cdb6911792"],"root_token":"4408bbd6-bc3f-1cba-08aa-6c0a12ff4e3d"}'
```

之所以需要两次 reset key，是因为在没有 unseal 时，reset 其实并不会完全生效。


# API
以 /v1/ 为前缀会访问 vault 集群，只有需要更新密钥，锁某个节点等操作会用到。
以 /v2/ 为前缀会访问 lvault，一般来说，我们都利用 lvault 来方便处理 vault 的初始化，集群解锁等功能。

需要说明的是，lvault 两个 proc 类型均为 web. 其中，lvault.web.web proc 为 vault 集群，每个节点是有状态的；另外，
根据 vault 的架构，当 vault 的节点超过一个时，由于有 standby 节点，
访问 vault 集群有可能出现内部重定向，而重定向的地址有可能是集群内 container 地址，
所以，此时需要在集群内操作。lvault.web.lvault 为 lvault，即是对 lain 的秘密配置管理定制的一些功能，主要包括两个功能，
第一个功能是对应用的秘密文件的增删查改，第二个是作为 SSO 的 Client，利用 SSO 的身份认证和 Console 的权限管里，
来进行 ACL 控制。对 lvault 的 API(见下文) 操作可以在集群外进行。

## /v2/init

### PUT

TODO：加入参数
当前默认的两个参数都是1，分别代表分发的 key 数目，和解锁至少需要的 key 的数目，感觉暂时没有需求去改变这两个参数 

对于一个集群只能执行一次，返回 key 和 root token，也是唯一的获得这两项的机会，对于系统管理员，一定要将其记在安全的地方。

**权限：**不需要认证

eg:
```
curl -X PUT http://lvault.lain.local/v2/init -d '{}'
curl -X PUT http://lvualt.lain.local/v2/init -d "{\"secret_shares\":1, \"secret_threshold\":1}"
```

## /v2/vaultstatus

### GET

查询 vault 集群的状态，返回值例如

```json
{
    "172.20.0.7:8200":{
        "Info":{
            "container_ip":"172.20.0.7",
            "container_port":8200
        },
        "Status":{
            "sealed":true,
            "t":1,
            "n":1,
            "progress":0
        }
    },
    "172.20.0.9:8200":{
        "Info":{
            "container_ip":"172.20.0.9",
            "container_port":8200
        },
        "Status":{
            "sealed":false,
            "t":1,
            "n":1,
            "progress":0
        }
    }
}
```

**权限：**不需要认证

## /v2/status

查询 lvault 的状态。

**权限：**不需要认证

## /v2/unsealall

### PUT

从 vault 集群只能对 instances 逐个解锁，这里为了方便，用该 api 将所有当前的 vault 实例解锁; 若 lvault 已经丢掉了 key 和 root_token，则必须先 reset.

eg:
```
curl -XPUT http://lvault.lain.local/v2/unsealall -d "{}"
```



**权限：**不需要认证

## /v2/reset

### PUT

```
curl -v -X PUT http://lvault.lain.local/v2/reset -d '{"keys":["193e7e1452679fcec7d06e96438752f4be7434abdc0d8632ac79c6cdb6911792"],"root_token":"4408bbd6-bc3f-1cba-08aa-6c0a12ff4e3d"}'
```

用传递过来的 key 来解锁，lvault 的 instances 记录 token. 为了安全，仅当该 lvault instance 的 token 丢掉或无效时，才会更新 token, 并且，还会对 token 进一步检查;
对于传递过来的 key，直接替换掉内存里的 key，这是由于这样可以保证 lvault 内存中的 key 已经失效，增强安全性。

reset 是为了当所有 lvault instaces 都同时重启，导致丢掉 root token 的情况。正常情况下，如果一个 lvault 节点重启， lvault 会在一段时间内自动更新该节点的 root token.

注意：若 lvault.web.web 和 lvault.web.lvault 同时不可用，当前的解锁流程是 reset -> unsealall -> reset, 即需要执行两次 reset 操作。lvault 的管理 api 尚未完善。
