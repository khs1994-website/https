# cfssl 生成自签名网站证书

[使用 openssl 生成网站证书](https://www.khs1994.com/linux/openssl/https/self-signed-ssl-certificates.html)

相比 `openssl` `cfssl` 可能更简单一点。

GitHub：https://github.com/cloudflare/cfssl

## 安装

```bash
$ brew install cfssl
```

其他系统参考 GitHub 自行安装。

## CA 私钥（key）和证书

**此步骤只需执行一次，后续直接使用此步骤生成的文件，签发多个网站证书。**

```bash
$ cfssl print-defaults config > ca-config.json

$ cfssl print-defaults csr > ca-csr.json
```

以上命令生成了两个示例文件，我们在此基础上自行调整配置（主要修改过期时间 `expiry`，我们可以把过期时间设置的长一点）

### ca-config.json


`client` certificate is used to authenticate client by server. For example etcdctl, etcd proxy, fleetctl or docker clients.

`server` certificate is used by server and verified by client for server identity. For example docker server or kube-apiserver. (网站证书)

`peer` certificate is used by etcd cluster members as they communicate with each other in both ways.


执行以下命令。

```bash
$ cfssl gencert -initca ca-csr.json | cfssljson -bare ca
```

得到以下三个文件

```bash
ca-key.pem  # CA 私钥，妥善保管
ca.csr      # 请求文件，用不到
ca.pem      # CA 证书文件，公钥
```

# 服务端密钥（生成网站证书）

使用上一步生成的 `CA 私钥和证书` 生成服务端公钥和私钥。

```bash
$ cfssl print-defaults csr > server.json
```

**该 JSON 文件中务必自行修改 `hosts` 字段**

`hosts` 字段即为网站的 `域名` 或 `IP`，可以包含多个，可以包含 `*`(通配符证书)。

比如你要为 `www.khs1994.com` 签发证书，就在 `hosts` 字段增加 `www.khs1994.com`

示例请到本项目 `ssl/server.json` 文件查看。

```bash
$ cfssl gencert -ca=ca.pem -ca-key=ca-key.pem -config=ca-config.json -profile=server server.json | cfssljson -bare server
```

得到以下三个文件

```bash
server-key.pem  # 私钥
server.csr
server.pem      # 公钥
```

配置 NGINX 并将 `ca.pem` （CA 公钥）导入浏览器（自签名证书不被浏览器信任，必须手动导入，导入方法自行搜索）。

# 客户端密钥 (网站用不到)

`k8s`、`Docker`、`etcd` 会用到。

使用最开始一步生成的 `CA 私钥和证书` 生成客户端公钥和私钥。

```bash
$ cfssl print-defaults csr > client.json
```

**该 JSON 文件不要包含 `hosts` 字段。**

```bash
$ cfssl gencert -ca=ca.pem -ca-key=ca-key.pem -config=ca-config.json -profile=client client.json | cfssljson -bare client
```

```bash
client-key.pem  # 私钥
client.csr
client.pem      # 公钥
```

# peer (网站用不到)

暂时不知道有什么用，先列出来。

```bash
$ cfssl print-defaults csr > member1.json
```

**该 JSON 文件中务必自行修改 `hosts` 字段**

```bash
$ cfssl gencert -ca=ca.pem -ca-key=ca-key.pem -config=ca-config.json -profile=peer member1.json | cfssljson -bare member1
```

得到三个文件

```bash
member1-key.pem  # 私钥
member1.csr
member1.pem      # 公钥
```

`member1` 为 `etcd` 集群节点 `hostname`，使用时自行替换为实际 `hostname`。

# 参考链接

* https://coreos.com/os/docs/latest/generate-self-signed-certificates.html
