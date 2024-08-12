# Docker 镜像代理方案

## 方案一：Docker Images Pusher

使用Github Action将国外的Docker镜像转存到阿里云私有仓库，供国内服务器使用，免费易用<br>
- 支持DockerHub, gcr.io, k8s.io, ghcr.io等任意仓库<br>
- 支持最大40GB的大型镜像<br>
- 使用阿里云的官方线路，速度快<br>

视频教程：https://www.bilibili.com/video/BV1Zn4y19743/

作者：**[技术爬爬虾](https://github.com/tech-shrimp/me)**<br>
B站，抖音，Youtube全网同名，转载请注明作者<br>

## 使用方式


### 配置阿里云
登录阿里云容器镜像服务<br>
https://cr.console.aliyun.com/<br>
启用个人实例，创建一个命名空间（**ALIYUN_NAME_SPACE**）
![](/doc/命名空间.png)

访问凭证–>获取环境变量<br>
用户名（**ALIYUN_REGISTRY_USER**)<br>
密码（**ALIYUN_REGISTRY_PASSWORD**)<br>
仓库地址（**ALIYUN_REGISTRY**）<br>

![](/doc/用户名密码.png)


### Fork本项目
Fork本项目<br>
#### 启动Action
进入您自己的项目，点击Action，启用Github Action功能<br>
#### 配置环境变量
进入Settings->Secret and variables->Actions->New Repository secret
![](doc/配置环境变量.png)
将上一步的**四个值**<br>
ALIYUN_NAME_SPACE,ALIYUN_REGISTRY_USER，ALIYUN_REGISTRY_PASSWORD，ALIYUN_REGISTRY<br>
配置成环境变量

### 添加镜像
打开images.txt文件，添加你想要的镜像
可以加tag，也可以不用(默认latest)<br>
可添加 --platform=xxxxx 的参数指定镜像架构<br>
可使用 k8s.gcr.io/kube-state-metrics/kube-state-metrics 格式指定私库<br>
可使用 #开头作为注释<br>
![](doc/images.png)
文件提交后，自动进入Github Action构建

### 使用镜像
回到阿里云，镜像仓库，点击任意镜像，可查看镜像状态。(可以改成公开，拉取镜像免登录)
![](doc/开始使用.png)

在国内服务器pull镜像, 例如：<br>
```
docker pull registry.cn-hangzhou.aliyuncs.com/shrimp-images/alpine
```
registry.cn-hangzhou.aliyuncs.com 即 ALIYUN_REGISTRY(阿里云仓库地址)<br>
shrimp-images 即 ALIYUN_NAME_SPACE(阿里云命名空间)<br>
alpine 即 阿里云中显示的镜像名<br>

### 多架构
需要在images.txt中用 --platform=xxxxx手动指定镜像架构
指定后的架构会以前缀的形式放在镜像名字前面
![](doc/多架构.png)

### 镜像重名
程序自动判断是否存在名称相同, 但是属于不同命名空间的情况。
如果存在，会把命名空间作为前缀加在镜像名称前。
例如:
```
xhofe/alist
xiaoyaliu/alist
```
![](doc/镜像重名.png)

### 定时执行
修改/.github/workflows/docker.yaml文件
添加 schedule即可定时执行(此处cron使用UTC时区)
![](doc/定时执行.png)

## 方案二：Docker Image Clash 内网代理


## **Docker 部署 clash 代理翻墙**


1. 准备一个`docker-compose`文件

```
$ cat docker-compose.yml
version: '3'
services:
  clash:
    container_name: clash_proxy
    image: centralx/clash:1.18.0
    restart: always
    ports:
      - "7890:7890"
      - "8090:80"
    volumes:
      - ./config/clash/config.yaml:/home/runner/.config/clash/config.yaml

```

2. 获取clash config文件

- Mac 和 Windows 获取方式是一致的

Clash -- 配置 -- 文件（每个人的配置文件都是不一样的。）

- 将配置文件中的内容拷贝到一个文件中，并命名为config.yaml

```
  cat config.yaml
  port: 7890
  socks-port: 7891
  allow-lan: true
  mode: rule
  log-level: info
  ipv6: true
  external-controller: :9090
  secret: xxxxxxxxxxxxxxxxxx  //需要注意这一段的内容为密钥。容器起来需要输入
  profile:
    store-selected: true
  .......

```

1. docker-compose 启动容器

```
[cpu] root@node5:/home/lixie/Docker-proxy-clash# tree
.
├── config
│   └── clash
│       └── config.yaml
├── docker-compose.yml
└── run.sh

2 directories, 3 files


$ docker-compose up -d
```


1. 浏览器测试访问：{IP-address:8090},输入secret中密钥。

## **测试**

5.1 docker 配置代理拉镜像测试。
```
[cpu] root@node5:/home/lixie/Docker-proxy-clash# cat /etc/docker/daemon.json
{
    "proxies": {
        "http-proxy": "http://10.0.10.158:7890",
        "https-proxy": "http://10.0.10.158:7890",
        "no-proxy": "*.test.example.com,.example.org,127.0.0.0/8"
    },
    "default-ulimits": {
        "memlock": {
            "Hard": 4294967296,
            "Name": "memlock",
            "Soft": 4294967296
        },
        "nofile": {
            "Hard": 1048576,
            "Name": "nofile",
            "Soft": 1048576
        }
    },
    "exec-opts": [
        "native.cgroupdriver=systemd"
    ],
    "insecure-registries": [],
    "log-driver": "json-file",
    "log-opts": {
        "max-file": "3",
        "max-size": "50m"
    },
    "registry-mirrors": []
}
```

5.2 linux 配置代理
可以在 ~/.bash_profile 定义 proxy_on 和 proxy_off 函数，用于快速开启和关闭代理:
```
function proxy_off() {
unset no_proxy
unset http_proxy
unset https_proxy
unset all_proxy
echo -e "Proxy Disabled"
}

function proxy_on() {
    # CIDR 网段表示法只在部份受支持的程序里起效
    # 如果不起效，那么需要直接设置具体的 IP 地址
    export no_proxy=localhost,127.0.0.1,127.0.0.0/8,10.0.0.0/8,172.16.0.0/12,192.168.0.0/16
    export http_proxy=http://<host>:7890
    export https_proxy=http://<host>:7890
    export all_proxy=socks://<host>:7890
    echo -e "Proxy Enabled"
}

```

## 对比

### 方案一:

**优点**
- 无需对docker 做额外的配置，就可以在国内获取镜像。

**缺点**
- 需要针对不同的镜像，来单独的配置，并提交
- 针对一个混合架构环境中，拉取不同架构的镜像配置繁琐。

### 方案二：

**优点**
- 直接通过代理来拉镜像。方便，简单。
- 不区分架构直接拉取

**缺点**
- 需要对docker 进行代理配置，这个可以通过ansible自动化实现。
- 每个环境都需要同一内网找一台节点作为代理服务器

## 文献参考

- https://hub.docker.com/r/centralx/clash
