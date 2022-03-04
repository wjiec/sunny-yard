自签名一个标准可信任的SSL证书
---------------------------------------------

在日常开发中如果需要在本地或者测试环境部署HTTPS服务就需要一张SSL证书。虽然也有免费的可用，但是为了方便测试或者想学习下相关技术就需要完整的自签名一个证书出来。网上大多数的文章签出的证书都是v1版本而且大多步骤也都含糊不清，在查找了大量文档之后整理了个~~正规且标准~~的自签名证书流程。



## 为什么需要根证书和中间证书

我们所购买的用户证书一般是由一个中间证书在线签发出来的，而中间证书是由根证书签发的。而根证书、中间证书、用户证书组成一条信任链，只有当信任链上的所有证书都有效时，我们才认为用户证书是有效且被信任的。

中间证书的出现是为了保护根证书和限制出问题后的影响范围（即当出现安全问题时只需要吊销中间证书而不会影响到根证书）同时可能还有额外的比如区分产品线、交叉验证等作用。



## 准备工作

在开始自签名证书之前，我们需要做一点准备工作，首先新建一个空目录（**除非特殊声明，否则是否所有的命令都在这个目录下执行**）

```bash
# 文件夹的名字随意，自己记得就好
mkdir -p ~/self-signed-v3

# 进入工作目录
cd ~/self-signed-v3

# 创建openssl文件夹
mkdir -p openssl
```

随后我们进入这个目录并将以下内容保存到`openssl/openssl.cnf`文件中，方便我们后续生成证书

```openssl
[ ca ]
default_ca	        = CA_default

[ CA_default ]
dir                 = ./openssl
new_certs_dir       = $dir/newcerts
database            = $dir/index.txt
serial              = $dir/serial
policy              = policy_match
default_md          = sha256

[ policy_match ]
organizationName    = match
commonName          = supplied

[ v3_root_ca ]
basicConstraints        = critical,CA:true,pathlen:65536
keyUsage                = critical,digitalSignature,keyCertSign,cRLSign
subjectKeyIdentifier    = hash
authorityKeyIdentifier  = keyid:always,issuer

[ v3_intermediate_ca ]
basicConstraints        = critical,CA:true,pathlen:0
keyUsage                = critical,digitalSignature,keyCertSign,cRLSign
subjectKeyIdentifier    = hash
authorityKeyIdentifier  = keyid:always,issuer

[ v3_user_cert ]
basicConstraints        = critical,CA:false
keyUsage                = critical,digitalSignature,keyEncipherment
extendedKeyUsage        = serverAuth
subjectKeyIdentifier    = hash
subjectAltName          = @alt_names

[ alt_names ]
```

### 关于x509v3

以上配置的详细信息可以参考[x509v3_config](https://www.openssl.org/docs/manmaster/man5/x509v3_config.html)和[keyUsage](https://datatracker.ietf.org/doc/html/rfc5280#section-4.2.1.3)的描述。这里简单介绍下

- `basicConstraints`：基本约束用于声明当前证书是否是一个CA证书（`CA:true`）

- - `pathlen`：用于指定当前证书可签发的下级证书的数量，为0则只能用于签发用户证书

- `keyUsage`：表示证书的用途，这里可以多选，我们的选项有

- - `digitalSignature`：表示这个证书可用于验证数字签名
  - `keyCertSign`：表示这个证书可用于验证一个公钥证书

- - `cRLSign`：表示这个证书可用于验证证书吊销列表上的签名
  - `keyEncipherment`：表示这个证书可用于加密私钥和公钥

- `extendedKeyUsage`：表示证书一些扩展的用途

- - `serverAuth`：用于WWW服务的的认证

- `subjectKeyIdentifier`和`authorityKeyIdentifier`：是证书的标识符，应该只是标识作用
- `subjectAltName`：主要用于浏览器对证书进行校验

> 这里不直接使用`/etc/ssl/openssl.cnf`的原因是在不同环境下（比如Linux和Mac）可能会得到不同的结果，甚至可能直接报错。所以这里我们采用自己声明所有x509v3属性的方式保证所有环境下都有一样的行为和结果。



## 创建根证书

首先我们生成根证书的私钥，私钥的长度最好选择2048位及其以上以保证安全

```bash
# 最后的4096表示私钥的长度
openssl genrsa -out root-ca.key 4096

# 可以使用以下命令查看私钥的详细信息
openssl rsa -text -noout -in root-ca.key
```

接下来我们通过私钥生成一个证书签名请求文件（Certificate Signing Request, CSR）

```bash
# 我们可以通过-subj来定义证书所证明的实际内容，比如组织名称, 域名等
openssl req -new -key root-ca.key -out root-ca.csr -sha256 \
    -subj "/O=LocalSecurity/CN=LocalSecurity Root CA"

# 也可以不增加-subj参数让openssl进入交互式填写信息流程
openssl req -new -key root-ca.key -out root-ca.csr -sha256

# 我们也可以直接用一条命令直接生成私钥和证书请求文件
openssl req -newkey rsa:4096 -nodes -keyout root-ca.key \
    -subj "/O=LocalSecurity/CN=LocalSecurity Root CA" \
    -sha256 -out root-ca.csr
# -nodes 表示不需要对私钥进行加密

# 同样可以使用以下命令查看证书签名请求文件的详细信息
openssl req -text -noout -in root-ca.csr
```

对以上命令进行我们稍微展开说一下，顾名思义「证书签名请求文件」就是请大家公认的人/机构对自己进行担保，大家都信任它，而它给「我」做了担保并且签名作证了，所以大家都可以相信我。但是担保也是有范围的，不可能「我」说啥大家都信，所以会有个`-subj`参数用于说明大家可以相信「我」的哪些信息。而参数`-sha256`表示使用`sha256`摘要算法，这主要是为了防止浏览器出现警告⚠️，如果使用`sha1`进行签名，浏览器就会认为这是过时的而提示这个证书不安全。

接下来我们就可以开始自签名根证书（Certificate）

```bash
# 通过root-ca.csr和root-ca.key文件生成一个带root-ca.ext属性的证书
openssl x509 -req -extfile openssl/openssl.cnf -extensions v3_root_ca \
    -in root-ca.csr -out root-ca.crt -signkey root-ca.key \
    -days 3650 -sha256

# 我们可以通过以下命令查看证书的信息
openssl x509 -text -noout -in root-ca.crt
```

至此，我们已经根证书的生成和签发（`root-ca.csr`已经没用了，可以删除）。



## 创建中间证书

与根证书一样，我们也需要为中间证书生成私钥和证书签名请求文件，这里我们直接一步生成所有文件

```bash
# -nodes表示不需要对私钥进行加密
openssl req -newkey rsa:3072 -nodes -keyout intermediate-ca.key \
    -subj "/O=LocalSecurity/CN=LocalSecurity Intermediate CA" \
    -sha256 -out intermediate-ca.csr

# 同样可以使用以下命令查看证书签名请求文件的详细信息
openssl req -text -noout -in intermediate-ca.csr
```

在准备开始使用根证书对中间证书进行签名之前，我们还需要准备一点东西

```bash
# 签发证书的数据库文件, 用于记录根证书已签发次数、吊销数据等
touch openssl/index.txt

# 新签发证书的保存路径
mkdir -p openssl/newcerts

# 新签发证书的序列号
echo "01" > openssl/serial
```

接下来我们使用根证书对中间证书的请求进行签名

```bash
# 使用根证书对中间证书进行签名, -batch表示无需交互式确认
openssl ca -config openssl/openssl.cnf -extensions v3_intermediate_ca \
    -in intermediate-ca.csr -out intermediate-ca.crt \
    -cert root-ca.crt -keyfile root-ca.key -days 3650 -batch

# 同样可以通过以下命令查看证书的信息
openssl x509 -text -noout -in intermediate-ca.crt
```

至此，我们已经中间证书的生成和签发（`intermediate-ca.csr`已经没用了，可以删除）。



## 创建用户证书

与其他证书类似，我们同样需要为用户证书生成私钥和证书请求文件。在生成之前我们需要确定为哪个域名或者IP地址生成证书，这个信息将会在后续命令中用到（以域名`hello.hasakk.com`和IP`172.16.0.1`为例）

```bash
# 创建一个基于域名的用户证书私钥和证书签名请求
openssl req -newkey rsa:2048 -nodes -keyout hello-hasakk-com.key \
    -subj "/CN=hello.hasakk.com" -sha256 -out hello-hasakk-com.csr

# 创建一个基于IP的用户证书私钥和证书签名请求
openssl req -newkey rsa:2048 -nodes -keyout 172-16-0-1.key \
    -subj "/CN=hello.hasakk.com" -sha256 -out 172-16-0-1.csr

# 可以使用以下命令查看证书签名请求文件的详细信息
openssl req -text -noout -in hello-hasakk-com.csr
openssl req -text -noout -in 172-16-0-1.csr
```

接下来我们直接使用中间证书（**也可以使用根证书**）对用户证书进行签名

```bash
# 生成基于域名的用户证书（注意替换其中的域名）
openssl x509 -req -in hello-hasakk-com.csr -out hello-hasakk-com.crt \
    -CA intermediate-ca.crt -CAkey intermediate-ca.key \
    -days 397 -sha256 -CAcreateserial \
    -extfile <(cat openssl/openssl.cnf <(printf "\nDNS=hello.hasakk.com")) \
    -extensions v3_user_cert

# 生成基于IP的用户证书（注意替换其中的IP地址）
openssl x509 -req -in 172-16-0-1.csr -out 172-16-0-1.crt \
    -CA intermediate-ca.crt -CAkey intermediate-ca.key \
    -days 397 -sha256 -CAcreateserial \
    -extfile <(cat openssl/openssl.cnf <(printf "\nIP=172.16.0.1")) \
    -extensions v3_user_cert

# 同样可以通过以下命令查看证书的信息
openssl x509 -text -noout -in hello-hasakk-com.crt
openssl x509 -text -noout -in 172-16-0-1.crt
```

至此，我们所需要的所有文件均已生成完毕，真不容易呀～



## 使用用户证书

由于根证书是我们自签名得到的，并没有内置在系统中，所以我们需要手动将`root-ca.crt`添加到系统信任根证书列表中（具体操作这里我就略过了，一般双击证书都会打开导入向导）。

在导入根证书之后接下来将证书配置到nginx中，如下配置（需要根据配置修改证书位置）

```nginx
worker_processes auto;
worker_cpu_affinity auto;
worker_rlimit_nofile 65536;

events {
    multi_accept on;
    worker_connections  1024;
}

http {
    charset utf-8;
    include mime.types;
    default_type application/octet-stream;

    server {
        listen 80;
        server_name hello.hasakk.com;

        return 301 https://$host$request_uri;
    }

    server {
        listen 443 ssl;
        server_name hello.hasakk.com;

        root /usr/share/nginx/html;
        index index.html index.htm;

        ssl_certificate /ssl/hello-hasakk-com.crt;
        ssl_certificate_key /ssl/hello-hasakk-com.key;
    }
}
```

当我们打开浏览器访问这个页面，就会发现Chrome狠狠的打了我们一巴掌

![AUTHORITY INVALID](../../assets/self-signed-ssl/authority-invalid.png)

那么问题出在哪儿呢？我们可以点击查看nginx下发的证书信息就可以发现一些端倪了

![INVALID SSL CHAIN](../../assets/self-signed-ssl/invalid-ssl-chain.png)

啊～原来是我们的用户证书是使用中间证书`Intermediate CA`签发的，而浏览器并没有找到这个中间证书自然就认为我们是无效的啦。



### 解决中间证书问题

解决这个问题有两种方法，第一种显而易见的方式，我们直接再把中间证书添加到系统信任证书列表里不就得了。第二种比较优雅一点，我们可以直接在用户证书中将中间证书附带上，这样可以避免每次都要添加中间证书到系统信任列表里。

```bash
cat hello-hasaskk-com.crt intermediate-ca.crt > hello-hasaskk-com.chain.crt
```

然后修改nginx配置使用带有完整证书链的证书

```nginx
http {
    server {
        listen 443 ssl;
        server_name hello.hasakk.com;

        root /usr/share/nginx/html;
        index index.html index.htm;

        ssl_certificate /ssl/hello-hasakk-com.chain.crt;
        ssl_certificate_key /ssl/hello-hasakk-com.key;
    }
}
```

现在终于可以休息了～



## 还是太麻烦？使用mkcert吧

> 项目地址：[FiloSottile/mkcert](https://github.com/FiloSottile/mkcert)
>
> 预编译二进制：[Releases](https://github.com/FiloSottile/mkcert/releases)

如果在Mac环境下可以使用`brew install mkcert`来安装，其他环境可以参考项目的README文档。接下来就可以愉快的生成用户证书了

```bash
# 生成并安装根证书，一步到位
mkcert -install

# 直接生成对应的域名或者IP证书
mkcert hello.hasakk.com
mkcert *.hasakk.com
mkcert 172.16.0.1
```



## 什么？你一定要中间证书装*？

好家伙，我这辈子没见过这么奇怪的需求！那来试试我的mkcert？

> 项目地址：...还没写...