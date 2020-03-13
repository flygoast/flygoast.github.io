---
title: HashiCorp Vault介绍
date: 2020-03-13 16:09:56
categories: Network
---
应用服务常常需要存取各种各样的机密信息，比如，数据库的访问凭证，依赖的外部服务的`Token`和`Key`，服务之间的通信凭证等等。在每个这样的应用中都重复实现机密信息的存取、更新与吊销等管理操作相当繁重，而且容易出问题。HashiCorp公司的开源项目[Vault](https://www.vaultproject.io/)就将这部分工作以独立服务方式来实现，需要机密信息操作的应用可以与Vault服务交互完成相应的操作。

Vault的应用场景非常广泛，如:
* 机密信息存取
* 凭证的动态生成，比如数据库凭证、PKI证书、公有云服务的凭证等等
* 加密即服务

Vault架构非常清晰，如下图所示，主要分为三部分:

{% img /images/2020-03-13/1.png %}

* **HTTP/S API**: Vault服务对外提供HTTP API接口
* **Storage Backend**: 负责加密数据的持久化存储, Vault支持多种存储机制。
* **Barrier**: 负责各种机密信息相关的处理逻辑，最核心的组件是`Secrets Engine`和`Auth Method`。`Secrets Engine`负责实际机密信息处理功能的实现，各种不同的`Secrets Engine`有着不同的功能服务，`Auth Method`提供各种不同的身份校验方法。这两个组件都依赖Plugin机制实现，如果官方提供的功能不满足需求，还可以自己写相应Plugin实现特有功能或身份校验方法。

<!--more-->

Vault服务部署启动后，处于”密封(`sealed`)”状态，此时不能提供服务。在该服务能接受请求前，需要初始化和”解封(`unseal`)”两个步骤。
Vault初始化时内部会生成一个加密密钥(`Encryption Key`), 这个密钥用于加密数据，对外是不可见的。Vault会再生成一个`Master key`来加密`Encryption Key`来保护它。Vault会使用[`Shamir密钥分享算法`](https://en.wikipedia.org/wiki/Shamir's_Secret_Sharing)将`MasterKey`分割成若干份`KeyShares`，而根据其中的若干份`KeyShares`可以计算出原来的`MasterKey`。默认配置的`KeyShares`是`5`，还原所需要的份数为`3`。我们需要记录好所生成的`KeyShares`, Vault自身并不保存`MasterKey`。当我们忘记`KeyShares`后，我们就无法得到`MasterKey`, 从而数据也无法解密。

{% img /images/2020-03-13/2.png %}

解封(`Unseal`)过程就是使用若干份`KeyShares`计算得到`MasterKey`, 解封之后，Vault就可以通过HTTP API对外提供服务了。

Vault接收到请求后需要先校验身份请求者身份信息，有多种`Auth method`可以选择，比如用户名/密码、Github帐号等等。支持的方法可以参考[官方文档](https://www.vaultproject.io/docs/auth/)。身份校验通过后，Vault会根据身份所关联的策略检查该资源请求是否合法。`root`策略是内置的策略，它允许访问任何资源。Vault管理用户可以创建更细粒度的资源访问控制策略。除非在策略中明确允许访问某种资源，否则Vault会禁止访问。更深入了解策略可以参考[官方文档](https://www.vaultproject.io/docs/concepts/policies/)

身份校验和策略检测都通过后，Vault会给该客户端分配一个`Token`。后续的请求都需要携带该`token`，类似于WEB访问的`cookie`机制。在Vault中，URL路径会关联一个`Secrets Engine`。Vault的路由组件会将特定的URL请求路由所关联的`Secrets Engine`来处理。更细节的了解可[参考](https://www.vaultproject.io/docs/internals/architecture/)

下边我们实际演示。

我们选择MySQL作为`Storage Backend`。首先创建Vault服务的配置文件`mysql.hcl`:
```plain
storage "mysql" {
    address = "127.0.0.1:3306"
    username = "root"
    password = "123456!"
    database = "vault"
}

ui = true

listener "tcp" {
    address = "0.0.0.0:8200"
    tls_disable = true
}

api_addr = "http://0.0.0.0:8200"
cluster_addr = "http://0.0.0.0:8201"

cluster_name = "flygoast"
pid_file = "/var/run/vault.pid"
```
具体配置说明，参考[链接](https://www.vaultproject.io/docs/configuration/)

启动服务:
```bash
$ vault server -config=./mysql.hcl
```

Vault支持以命令行、UI、HTTP API来访问服务接口。我们这里使用Vault CLI来说明，它也是调用HTTP API实现的相应请求。

打开另一终端，配置环境变量指定Vault服务地址:
```bash
$ export VAULT_ADDR='http://127.0.0.1:8200'
```

然后执行初始化操作:
```plain
[root@fg-t2 vault]# ./vault operator init
Unseal Key 1: DKhxs3QAHqwUVZ5JwmbrHNjFrxT7OGQPpfy7WPwh0hXX
Unseal Key 2: kRM5krXtme5ZLdDNKhK5aA2VTTfBIg29UW751Y2ZoSfp
Unseal Key 3: fR+B1/b2ma8eNoYMfACPF10OrVJ/dMvygC2bQo2Dzk0p
Unseal Key 4: enZsywfA3wZb/h5MgBKaRGSdcvG16rLOzO5ml1NyyGCv
Unseal Key 5: y1Mto6nPy7oRel2q7c/sWA4n0o0OCbCs8VeAWYIiPSwm


Initial Root Token: s.e4hh2uBH83HuAh3voGiEiUfE


Vault initialized with 5 key shares and a key threshold of 3. Please securely
distribute the key shares printed above. When the Vault is re-sealed,
restarted, or stopped, you must supply at least 3 of these keys to unseal it
before it can start servicing requests.


Vault does not store the generated master key. Without at least 3 key to
reconstruct the master key, Vault will remain permanently sealed!


It is possible to generate new unseal keys, provided you have a quorum of
existing unseal keys shares. See "vault operator rekey" for more information.
```
可以看到，Vault返回了5个`KeyShares`和`Root Token`。

接着使用做任意3个`KeyShares`来解封Vault服务, 需要每次输入不同的`KeyShare`:
```plain
[root@fg-t2 vault]# ./vault operator unseal
Unseal Key (will be hidden):
Key                Value
---                -----
Seal Type          shamir
Initialized        true
Sealed             true
Total Shares       5
Threshold          3
Unseal Progress    1/3
Unseal Nonce       c9b57a45-b09d-309d-4925-2a96a97e17af
Version            1.3.2
HA Enabled         false
```

```plain
[root@fg-t2 vault]# ./vault operator unseal
Unseal Key (will be hidden):
Key                Value
---                -----
Seal Type          shamir
Initialized        true
Sealed             true
Total Shares       5
Threshold          3
Unseal Progress    2/3
Unseal Nonce       c9b57a45-b09d-309d-4925-2a96a97e17af
Version            1.3.2
HA Enabled         false
```

```plain
[root@fg-t2 vault]# ./vault operator unseal
Unseal Key (will be hidden):
Key             Value
---             -----
Seal Type       shamir
Initialized     true
Sealed          false
Total Shares    5
Threshold       3
Version         1.3.2
Cluster Name    flygoast
Cluster ID      96883f23-97e2-665a-2e17-07541a8e146b
HA Enabled      false
```
可以看到第三次执行后，`Sealed`值变为`false`，此时Vault服务解封完成。

接着我们使用`RootToken`来登录。
```plain
[root@fg-t2 vault]# ./vault login
Token (will be hidden):
Success! You are now authenticated. The token information displayed below
is already stored in the token helper. You do NOT need to run "vault login"
again. Future Vault requests will automatically use this token.


Key                  Value
---                  -----
token                s.e4hh2uBH83HuAh3voGiEiUfE
token_accessor       ZQVAa5C3FceWS1oOHL0aqRIw
token_duration       ∞
token_renewable      false
token_policies       ["root"]
identity_policies    []
policies             ["root"]
```
从返回结果看到RootToken所关联的策略为`root`, 能够访问所有资源。在生产环境不建议直接使用RootToken, 可以创建更细粒度权限控制的Token。

接下来我们就可以来和Vault服务交互完成各种机密信息相关的操作。我们上边说到，Vault的具体功能实现由Secrets Engine来负责。我们下面展示几个不同的Secrets Engine的功能实例。
首先来看一下[Key/Value引擎](https://www.vaultproject.io/docs/secrets/kv/)

`Key/Value`引擎可以存取K/V形式的数据。它有两个版本，`Version2`支持数据的历史版本存储，默认存储10个版本。
我们开启Version2版本的Key/Value引擎，默认的`path`为kv/, 可以通过`-path`参数指定其他path。
```plain
[root@fg-t2 vault]# ./vault secrets enable -version=2 kv
Success! Enabled the kv secrets engine at: kv/
```
查看已经启用的`Secrets Engine`, 可以看到路径`kv/`上已经开启`kv`引擎:
```plain
[root@fg-t2 vault]# ./vault secrets list
Path          Type         Accessor              Description
----          ----         --------              -----------
cubbyhole/    cubbyhole    cubbyhole_6f838d14    per-token private secret storage
identity/     identity     identity_df2eda00     identity store
kv/           kv           kv_08d6d535           n/a
sys/          system       system_a0682fee       system endpoints used for control, policy and debugging
```

接着，我们在key `kv/hello`中写入一些数据:
```plain
[root@fg-t2 vault]# ./vault kv put kv/hello foo=bar dummy=demo
Key              Value
---              -----
created_time     2020-03-10T14:54:20.451990799Z
deletion_time    n/a
destroyed        false
version          1
```
接着读取数据:
```plain
[root@fg-t2 vault]# ./vault kv get kv/hello
====== Metadata ======
Key              Value
---              -----
created_time     2020-03-10T14:54:20.451990799Z
deletion_time    n/a
destroyed        false
version          1


==== Data ====
Key      Value
---      -----
dummy    demo
foo      bar
```

给key `kv/hello`写入不同的数据:
```plain
[root@fg-t2 vault]# ./vault kv put kv/hello foo=bar dummy=demo2
Key              Value
---              -----
created_time     2020-03-10T14:55:39.294508022Z
deletion_time    n/a
destroyed        false
version          2
```

再次读取:
```plain
[root@fg-t2 vault]# ./vault kv get kv/hello
====== Metadata ======
Key              Value
---              -----
created_time     2020-03-10T14:55:39.294508022Z
deletion_time    n/a
destroyed        false
version          2


==== Data ====
Key      Value
---      -----
dummy    demo2
foo      bar
```

也可以通过指定`-version`来读取历史版本内容:
```plain
[root@fg-t2 vault]# ./vault kv get -version=1 kv/hello
====== Metadata ======
Key              Value
---              -----
created_time     2020-03-10T14:54:20.451990799Z
deletion_time    n/a
destroyed        false
version          1


==== Data ====
Key      Value
---      -----
dummy    demo
foo      bar
```

接着来看`Transit`引擎。`Transit`引擎实现了对数据的加解密操作，使用它可以实现加密即服务。

启用`transit`引擎:
```plain
[root@fg-t2 vault]# ./vault secrets enable transit
Success! Enabled the transit secrets engine at: transit/
```

指定名称生成一个加密密钥, 一般每个应用都会有自己的加密密钥:
```plain
[root@fg-t2 vault]# ./vault write -f transit/keys/flygoast-key
Success! Data written to: transit/keys/flygoast-key
```

这样服务就已经配置完成。我们来调用加密服务。因为Transit引擎支持二进制数据的加解密，所以要求明文数据先以BASE64编码:
```plain
[root@fg-t2 vault]# ./vault write transit/encrypt/flygoast-key plaintext=$(base64 <<< "my secret data")
Key           Value
---           -----
ciphertext    vault:v1:DRItP5PSamzwTgxDPzlczBqH0DH4EeCAY1pQ2x0mp4QzYiret1HuOxqLBw==
```
我们拷贝生成的密文，再次来调用解密服务:
```plain
[root@fg-t2 vault]# ./vault write transit/decrypt/flygoast-key ciphertext=vault:v1:DRItP5PSamzwTgxDPzlczBqH0DH4EeCAY1pQ2x0mp4QzYiret1HuOxqLBw==
Key          Value
---          -----
plaintext    bXkgc2VjcmV0IGRhdGEK
```
将得到的明文BASE64解码还原了原来的数据:
```plain
[root@fg-t2 vault]# base64 --decode <<< "bXkgc2VjcmV0IGRhdGEK"
my secret data
```

再来看一个动态生成数据库凭证的示例。很多应用需要访问数据库，往往需要将凭证硬编码在代码或配置文件中，这样非常不安全。我们可以通过调用Vault服务来动态生成相应的数据库的访问凭证。
这需要用到[database引擎](https://www.vaultproject.io/docs/secrets/databases/)，它是通用的数据库引擎，支持多种数据库，每种数据库以Plugin方法配置，我们以MySQL来说明。
首先启用`database`引擎:
```plain
[root@fg-t2 vault]# ./vault secrets enable database
Success! Enabled the database secrets engine at: database/
```
然后, 配置相应的Plugin和数据库连接信息，database引擎会使用这些信息去连接需要动态生成凭证的数据库:
```bash
$ ./vault write database/config/flygoast-mysql-database plugin_name=mysql-legacy-database-plugin connection_url="{{username}}:{{password}}@tcp(127.0.0.1:3306)/" allowed_roles="flygoast-role" username="root" password=‘123456!’
```

这里需要说明的是针对MySQL有几种不同的plugin:
* mysql-database-plugin
* mysql-aurora-database-plugin
* mysql-rds-database-plugin
* mysql-legacy-database-plugin

它们的不同之处是所生成的MySQL的用户名的长度, 因为不同版本的MySQL对用户名长度有着不同的限制。`mysql-legacy-database-plugin`的用户名长度为16.

接着创建一个角色(`role`)映射到创建用户的SQL语句, 数据库指定我们上边所创建的数据库信息。为了避免被Shell解释SQL语句中引号中的内容，Vault支持以BASE64编码的SQL语句。在这个示例中，我们授与`cloud_express`数据库的`SELECT`权限给动态生成的用户。这里我们先将动态创建用户的SQL语句使用BASE64编码:
```plain
[root@fg-t2 vault]# base64  <<< "CREATE USER '{{name}}'@'%' IDENTIFIED BY '{{password}}'; GRANT SELECT ON cloud_express.* TO '{{name}}'@'%';"
Q1JFQVRFIFVTRVIgJ3t7bmFtZX19J0AnJScgSURFTlRJRklFRCBCWSAne3twYXNzd29yZH19Jzsg
R1JBTlQgU0VMRUNUIE9OIGNsb3VkX2V4cHJlc3MuKiBUTyAne3tuYW1lfX0nQCclJzsK
```
创建角色:
```bash
$ ./vault write database/roles/flygoast-role db_name=flygoast-mysql-database creation_statements="Q1JFQVRFIFVTRVIgJ3t7bmFtZX19J0AnJScgSURFTlRJRklFRCBCWSAne3twYXNzd29yZH19Jzsg
R1JBTlQgU0VMRUNUIE9OIGNsb3VkX2V4cHJlc3MuKiBUTyAne3tuYW1lfX0nQCclJzsK" default_ttl="1h" max_ttl="24h"
```

现在服务已经配置完成。后续我们从该角色读取就会生成相应的数据库用户:
```plain
[root@fg-t2 vault]# ./vault read database/creds/flygoast-role
Key                Value
---                -----
lease_id           database/creds/flygoast-role/AZKwNAH6tYEy0v6OykKOyjyg
lease_duration     1h
lease_renewable    true
password           A1a-BywEBFA3AeUlncdK
username           v-flyg-gvQygnKRB
```

我们使用上边所生成的用户来尝试登录MySQL:
```plain
[root@fg-t2 vault]# mysql -uv-flyg-gvQygnKRB -pA1a-BywEBFA3AeUlncdK
Welcome to the MariaDB monitor.  Commands end with ; or \g.
Your MariaDB connection id is 610
Server version: 5.5.60-MariaDB MariaDB Server


Copyright (c) 2000, 2018, Oracle, MariaDB Corporation Ab and others.


Type 'help;' or '\h' for help. Type '\c' to clear the current input statement.


MariaDB [(none)]>
```

登录成功。

最后我们再来看如何使用PKI引擎来构建私有`CA:Certificate Authority`服务。CA是PKI(Public Key Infrastructure)体系中负责颁发和吊销证书的机构。这些证书可用于通信加密及真实性校验。因为在线CA服务有可能被入侵，往往CA的实际部署是分层的, `Root CA`保持离线，而使用离线的Root CA来签发中间CA证书，中间CA在线提供证书签发服务。当在线的中间CA沦陷后，可以使用离线的RootCA签发新的中间CA证书。Vault本身也可以创建RootCA, 为了模拟RootCA离线存储方式，我们使用[CFSSL](https://github.com/cloudflare/cfssl)来生成RootCA，使用Vault来构建中间CA服务。

首先我们来创建RootCA:
编辑Root CA的配置文件`root_ca.json`:
```json
{
    "CN": "Just4coding Root CA",
    "key": {
        "algo": "rsa",
        "size": 2048
    },
    "names": [{
        "C": "CN",
        "ST": "Beijing",
        "O": "Just4coding",
        "OU": "Just4coding",
        "L": "Beijing"
    }],
    "ca": {
        "expiry": "87600h"
    }
}
```

创建自签名证书:
```bash
$ cfssl gencert -initca root_ca.json |cfssljson -bare root_ca
```

会产生3个文件:
* root_ca.pem: 公钥证书
* root_ca.csr: CSR:Certificate Signing Request
* root_ca-key.pem: 证书私钥，需要离线保存

我们在Vault服务中在路径`pki_int`上启用`PKI`引擎:
```plain
[root@fg-t2 vault]# ./vault secrets enable -path=pki_int pki
Success! Enabled the pki secrets engine at: pki_int/
```
调整`pki_int`最大颁发证书的有效期为`43800`小时:
```plain
[root@fg-t2 vault]# ./vault secrets tune -max-lease-ttl=43800h pki_int
Success! Tuned the secrets engine at: pki_int/
```

生成该中间CA的CSR:
```bash
$ ./vault write -format=json pki_int/intermediate/generate/internal common_name="Just4coding Intermediate CA" | jq -r '.data.csr' > inter_ca.csr
```

我们使用RootCA来签发中间CA证书。
创建cfssl需要使用的配置文件`root_signing.json`:
```json
{
    "signing":{
        "default": {
            "usages": ["digital signature", "cert sign", "crl sign", "signing"],
            "expiry": "43800h",
            "ca_constraint": {"is_ca": true}
        }
    }
}
```
执行签名命令:
```bash
$ cfssl sign -ca root_ca.pem -ca-key root_ca-key.pem  -config root_signing.json inter_ca.csr | cfssljson -bare inter_ca
```
生成的文件`inter_ca.pem`即为中间CA证书。我们将该证书导入Vault的`pki_int`:
```plain
[root@fg-t2 vault]# ./vault write pki_int/intermediate/set-signed certificate=@inter_ca.pem
Success! Data written to: pki_int/intermediate/set-signed
```
接着创建角色。角色是用来表示生成证书的策略，可以通过参数来配置这些策略:
```plain
[root@fg-t2 vault]# ./vault write pki_int/roles/just4coding allowed_domains="just4coding.com" allow_subdomains=true max_ttl="720h"
Success! Data written to: pki_int/roles/just4coding
```

至此服务已经配置完成。我们来请求证书:
```bash
$ ./vault write -format=json pki_int/issue/just4coding common_name="test.just4coding.com" ttl=“24h"
```

响应里包含:私钥、证书、CA链、颁发者证书。CA链里默认只包含签发者的证书。应用程序在使用生成的该证书时，需要把RootCA的证书信息添加到CA链中。

本文为了简单使用vault命令行来说明各实例。HTTP API及各语言的客户端库可以参考官方文档:
* https://www.vaultproject.io/api-docs/index/
* https://www.vaultproject.io/api-docs/libraries/

Vault还自带了WEB UI界面，UI上提供了非常详细的操作指引,使用者可以非常快速上手Vault操作。

{% img /images/2020-03-13/3.png %}

参考链接:
https://learn.hashicorp.com/vault/secrets-management/sm-pki-engine
https://cloudinvent.com/blog/howto-hashicorp-vault-ca-pki-deployment/
http://yet.org/2018/10/vault-pki/
