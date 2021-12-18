使用多阶段构建加速Go项目的编译打包速度
------------------------------------------------------------

在云原生时代，传统的编译打包部署方式不断被Docker容器化所替代，不仅是因为传统方式存在很多的不确定性，每次的构建都可能因为任何细小的变化而导致程序无法正常按预期工作。而作为替代，我们在Docker中编译打包程序，在Docker中部署运行生产环境。

这篇实践专注于如何在云原生时代用最靓的方式打出最靓的包，本文选用Go作为示例语言，其他语言也半斤八两差不了多少理解思想就好。

**注意：本文省略了很多比较基础的常识，比如科学上网、安装环境、下载依赖、Docker镜像的结构等。**

### 准备测试用例

首先简单的写一个示例程序，其中我们需要引入第三方库（才能知道速度体现在什么地方）

```go
package main

import (
	"log"
	"net/http"

	"github.com/gin-gonic/gin"
)

func main() {
	r := gin.New()
	r.GET("/echo", func(c *gin.Context) {
		c.JSON(http.StatusOK, gin.H{
			"text": c.Query("text"),
		})
	})
	if err := http.ListenAndServe(":8080", r); err != nil {
		log.Fatalf("http failed: %s", err)
	}
}
```

可以将以上内容拷贝到某个目录下的`main.go`文件中，然后使用`go mod init speedup`来初始化一个最简环境。



### 使用宿主机编译后直接COPY

比较常见的一种方式是使用Makefile进行编译和打包工作，比如我们准备一个Dockerfile

```dockerfile
FROM alpine

COPY speedup /speedup

ENTRYPOINT ["/speedup"]
```

然后编写一个Makefile文件组合编译和打包步骤

```makefile
.PHONY: all
.DEFAULT_GOAL: all

all: exe image

exe: main.go
	CGO_ENABLED=0 go build -o speedup main.go

image: speedup
	docker build -t company/speedup .
```

这种方式很好，但也不是那么好。虽然有go.sum可能让依赖不成为问题，但是每个人环境不一样，打出来的包总带有点“个人特色”（比如日志输出里携带的文件路径），这对于强迫症来说不能忍。



### 使用多阶段构建

在Docker 17.05版本之后支持在一个Dockerfile文件中使用多个`FROM`指令进行多阶段构建。



#### 多阶段构建的意义

说个题外话，在多阶段构建之前，如何缩小Docker镜像的体积一直是有趣的问题，因为Docker的镜像都是一层一层的，所以最简单的方式就是尽量减少层的数量，并保证每一层的对上一层的修改尽可能少。所以我们一般是通过`&&`命令拼接所有的命令

```dockerfile
FROM alpine

RUN set -ex \
    && apk add gcc make libssl ... \
    && wget https://github.com/xxx/yyy/latest.tar.gz \
    && tar xf latest.tar.gz \
    && cd xxx \
    && make -j \
    && strip xxx \
    && apk del .build-deps \
    && rm -rf xxx

ENTRYPOINT ["xxx"]
```

这样操作下来虽然我们可以打出理论上最小的镜像，但是随之带来的问题也非常致命

* 首先是无法充分利用Docker构建缓存的优势（因为只有一条语句）
* 其次调试过程简直是个噩梦（其中的任何修改都需要重新执行全部命令）
* 为了减少层的数量使用`&&`拼接所有的命令导致可读性比较差，后续修改也比较困难

而在有了多阶段构建之后，我们可以把安装依赖、下载源码、构建分别放在不同的阶段，最后通过COPY命令有选择性的从前几个阶段复制内容到最终镜像中。

```dockerfile
FROM ubuntu as builder

RUN apt install gcc g++ make ...


FROM builder as compiler

RUN ./configure
RUN make -j


FROM alpine

COPY --from=compiler target.out /target

ENTRYPOINT ["/target"]
```



#### 多阶段构建实践

回到正题，我们可以通过多阶段构建来让统一所有的步骤和环境，以保证生成的镜像是最靓的

```dockerfile
FROM golang:alpine as builder

WORKDIR /workspace
ENV GOPROXY=https://goproxy.cn

COPY . /workspace
RUN CGO_ENABLED=0 go build -o speedup


FROM alpine

COPY --from=builder /workspace/speedup /speedup

ENTRYPOINT ["/speedup"]
```

对比**宿主机编译方案**我们这里把我们能控制变量都尽量控制住了，而且Dockerfile也非常清晰，打出来的镜像也是理论上最小的（甚至还可以使用[scratch][1]镜像打出更小的镜像）

```plain
$ docker build -t company/speedup:multistage .
Sending build context to Docker daemon  9.027MB
Step 1/8 : FROM golang:alpine as builder
 ---> 861f73fe2c7f
Step 2/8 : WORKDIR /workspace
 ---> Using cache
 ---> 0a29ea7cbdfd
Step 3/8 : ENV GOPROXY=https://goproxy.cn
 ---> Running in 06b96b3ef804
Removing intermediate container 06b96b3ef804
 ---> 58144fae404f
Step 4/8 : COPY . /workspace
 ---> be37f1c91e38
Step 5/8 : RUN CGO_ENABLED=0 go build -o speedup
 ---> Running in 64a27b84b5af
go: downloading github.com/gin-gonic/gin v1.7.7
go: downloading github.com/gin-contrib/sse v0.1.0
go: downloading github.com/mattn/go-isatty v0.0.12
go: downloading github.com/ugorji/go/codec v1.1.7
go: downloading github.com/golang/protobuf v1.3.3
go: downloading gopkg.in/yaml.v2 v2.2.8
go: downloading github.com/go-playground/validator/v10 v10.4.1
go: downloading golang.org/x/sys v0.0.0-20200116001909-b77594299b42
go: downloading golang.org/x/crypto v0.0.0-20200622213623-75b288015ac9
go: downloading github.com/leodido/go-urn v1.2.0
go: downloading github.com/go-playground/universal-translator v0.17.0
go: downloading github.com/go-playground/locales v0.13.0
Removing intermediate container 64a27b84b5af
 ---> c2fc71f01ef6
Step 6/8 : FROM alpine
 ---> c059bfaa849c
Step 7/8 : COPY --from=builder /workspace/speedup /speedup
 ---> Using cache
 ---> 05a3cfd7a14a
Step 8/8 : ENTRYPOINT ["/speedup"]
 ---> Using cache
 ---> 194daff60077
Successfully built 194daff60077
Successfully tagged company/speedup:multistage
```

嗯，工作的非常完美。对了，前端刚才说接口字段需要改下名字，之前约定的字段是`message`（还好不是啥大事）。

改完代码，构建，完....淦。观察`docker build`的输出日志，发现了嘛。我们只是修改了字段名称，但是为什么还要重新下载依赖！



### 使用优化的多阶段构建

出现这个问题原因其实是我们偷懒了，但其实是Go的问题（Doge），因为`go build`命令会检查依赖是否存在，当依赖不存在时会下载依赖之后再进行编译。所以我们将每个步骤按逻辑拆开，**充分利用Docker的缓存优化**来构建镜像。

```dockerfile
FROM golang:alpine as builder

WORKDIR /workspace
ENV GOPROXY=https://goproxy.cn

COPY go.mod /workspace/go.mod
COPY go.sum /workspace/go.sum
RUN go mod download

COPY . /workspace
RUN CGO_ENABLED=0 go build -o speedup


FROM alpine

COPY --from=builder /workspace/speedup /speedup

ENTRYPOINT ["/speedup"]
```

仔细看以上Dockerfile文件，有几个值的注意的修改

* 我们单独拷贝了`go.mod`和`go.sum`
* 手动执行`go mod download`下载依赖

这对于我们有什么意义呢？意义就在于只要`go.mod`和`go.sum`文件不发生修改，那我们就可以充分利用Docker的缓存优化构建的速度。再次执行构建试试

```plain
$ docker build -t company/speedup:optimize .
Sending build context to Docker daemon  9.028MB
Step 1/11 : FROM golang:alpine as builder
 ---> 861f73fe2c7f
Step 2/11 : WORKDIR /workspace
 ---> Using cache
 ---> 0a29ea7cbdfd
Step 3/11 : ENV GOPROXY=https://goproxy.cn
 ---> Using cache
 ---> 58144fae404f
Step 4/11 : COPY go.mod /workspace/go.mod
 ---> Using cache
 ---> 13684fc507e7
Step 5/11 : COPY go.sum /workspace/go.sum
 ---> Using cache
 ---> fb4f773c1fc4
Step 6/11 : RUN go mod download
 ---> Using cache
 ---> 4180ef0c48c2
Step 7/11 : COPY . /workspace
 ---> 8581c00c42ca
Step 8/11 : RUN CGO_ENABLED=0 go build -o speedup
 ---> Running in 9d96e90ea875
Removing intermediate container 9d96e90ea875
 ---> 6d7bab1ac2e9
Step 9/11 : FROM alpine
 ---> c059bfaa849c
Step 10/11 : COPY --from=builder /workspace/speedup /speedup
 ---> e86149f148d3
Step 11/11 : ENTRYPOINT ["/speedup"]
 ---> Running in 2d3b0ecb0495
Removing intermediate container 2d3b0ecb0495
 ---> 9650d4ce978f
Successfully built 9650d4ce978f
Successfully tagged company/speedup:optimize
```

我们可以看到由于依赖没有修改，所以下载依赖的部分都是复用的缓存（`Using cache`），把CPU和时间都花在刀刃（编译）上。这下我就可以帮老板娶更多的姨太太了，真好。



### 总结

在Dockerfile中注意以下几点就可以以最靓的方式打出最靓的镜像啦

* **利用多阶段构建拆分不同的逻辑阶段**

* **按照“基本不变动  => 偶尔变动 => 经常变动”的顺序组织Dockerfile，尽可能利用Layer缓存**

其实本篇实践内容很简单也很容易理解（没看懂的。。不管），不在乎镜像大小、代码是否美观的也大有人在。但是我觉得吧程序员也应该是懂得浪漫的，程序员专属的浪漫。



### References

* [Use multi-stage builds][2]
* [FROM scratch][1]



[1]: https://hub.docker.com/_/scratch	"scratch"
[2]: https://docs.docker.com/develop/develop-images/multistage-build/	"multistage"