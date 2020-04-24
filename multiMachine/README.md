# EcshopX本地开发环境说明

EcshopX 采用前后端分离的方式开发，代码结构由后端api和多个前端组成。

后端api由lumen开发，部署时包含了web、scheduler、worker和websocket。

前端包含管理端、小程序、PC端和H5端。

本环境目前只包含了web和管理的部署，后续会根据需求将其他环境加入。

## 准备
开始前，需要确保本地已经安装 docker。然后先将 docker 开发环境的脚本 git clone 到本地：
```shell
git clone git@github.com:shopex/ecshopx-dev-docker.git
cd ecshopx-dev-docker
```

## 第一步：初始化代码

将管理端的代码复制到 espier-retail-manage 目录中

将api端的代码复制到到 espier-bloated 目录中

完成后目录结构如下：
```shell
ecshopx-dev-docker
    ├── config
    ├── data
    ├── espier-retail-manage
    │   ├── app
    │   └── docker
    ├── espier-bloated
    │   ├── app  
    │   ├── bootstrap
    │   ├── config
    │   ├── ...
    │   └── README.md
    ├── docker-compose.yml
    └── README.md
```
### 配置管理端 api 地址
在 `espier-retail-manage/app/config/dev.env.js` 中配置后端 api 地址：

```js
BASE_API: '"api"',
```
## 第二步：启动开发环境
在ecshopx-dev-docker下执行
```
docker-compose up
```

> 第一次启动时，需要通过 docker pull 拉去镜像到本地，所以需要等待一段时间

## 第三步：配置数据库

### 创建 MySQL 数据库
为保证安装环境的统一性，本环境自带了 `phpmyadmin`，账号密码为: root root 。

现在我们要创建程序需要的数据库，访问<http://localhost:8004> 。进入 phpmyadmin ，创建项目数据库 `ecshopx` ，字符集选用 `utf8mb4_general_ci`

> 在开发过程中，也可使用其他数据库工具创建数据库，连接地址为：localhost:8806

### 修改 Neo4j 密码
由于 Neo4j 不修改密码，无法通过 API 访问。所以我们需要通过浏览器客户端修改 Neo4j 的密码。

访问<http://localhost:7474> 输入账号密码： `neo4j`  `neo4j`。 登录后会进入重置密码的页面，输入两遍新密码： `123456`，点击确认即可。


## 第四步：配置安装

### 1、配置`.env`
代码中不包`.env`，可以将 `.env.production` 复制改名为 `.env`。

如果你完全按照第二步来配置数据库，以下配置可以直接复制替换 `.env` 原有配置。
* 修改数据库配置
```
DB_CONNECTION=default
DB_HOST=mysql
DB_PORT=3306
DB_DATABASE=ecshopx
DB_USERNAME=root
DB_PASSWORD=root
```
* 修改redis配置
```
REDIS_CLIENT=predis
REDIS_HOST=redis
REDIS_PORT=6379
REDIS_PASSWORD=123456
```
* 修改neo4j配置
```
NEO4J_DEFAULT_PROTOCOL=http
NEO4J_DEFAULT_HOST=neo4j
NEO4J_DEFAULT_PORT=7474
NEO4J_DEFAULT_USERNAME=neo4j
NEO4J_DEFAULT_PASSWORD=123456
```

### 2、composer&&migrate

由于 API 的环境在 docker 的容器内，所以开发时，需要进入到容器中来执行相关命令。

#### 进入容器
```shell
docker exec -it ecshopx-dev-docker_espier-bloated-web_1 sh 
```
进入容器之后，会自动进入代码目录 `/app`，所以可以直接运行命令：

#### 安装依赖
```
composer install
```

#### 初始化数据库表
```
php ./artisan doctrine:migrations:migrate
```

## 访问环境

在访问环境前，请先确认管理端代码是否编译完成

```shell
#查看管理端容器名称
docker ps | grep ecshop-admin-build
#查看日志
docker logs ecshopx-dev-docker_ecshop-admin-build_1 
```
如过有看到以下信息，说明管理端已经编译启动完成：
```shell
ecshop-admin-build_1  | > espier@1.0.0 dev /usr/share/nginx/html/app
ecshop-admin-build_1  | > node build/dev-server.js
ecshop-admin-build_1  |
ecshop-admin-build_1  | > Starting dev server...
ecshop-admin-build_1  |  DONE  Compiled successfully in 62395ms03:19:56
ecshop-admin-build_1  |
ecshop-admin-build_1  | > Listening at http://localhost:4000
```
如果看不到 ecshopx-dev-docker_ecshop-admin-build_1 说明编译出错，可通过以下命令重新启动：
```shell
docker start ecshopx-dev-docker_ecshop-admin-build_1
```

##  `本地可以选择先编译好后，直接运行代码(本地开发重点看这个)`
#### 代码编译过程

进入前端代码所在路径

```
cd /data/httpd/shopex/ecshopX/espier-shop/app
npm install
```

> 请选择合适你的node 版本，经测试，有的是系统适合用node12 有的用node13，推荐使用node12 可以本地安装，也可以自己安装docker，请自行选择

修改 app/config/prod.env.js 将BASE_API设置为你自己的url/api
```
let args = process.argv
let argv = {}
for(let i=2; i<args.length; ++i){
  let cur = args[i]
  let arg = cur.split('=')
  argv[arg[0]] = `'${arg[1]}'`
}

module.exports = {
  NODE_ENV: '"production"',
  IS_SAAS: '"true"',
  PRODUCT_MODEL: '"standard"',
  BASE_API: '"api"',
  WXAUTHCALL_Url: '"api"',
  PREFIXES: '""',
  WXIMG_URL: '""',
}
```
然后请初始化
```
npm run build
```
然后更改当前目录下/config/nginx/default.conf
```shell
location / {
    #「需要修改」 前端编译好的代码路径
    root  /usr/share/nginx/html/espier-shop/app/dist/prod;
    index  index.html index.htm;
    try_files $uri $uri/ /index.html =404;
}
```

替换这段代码，如果想要做watch模式，请使用对应的配置 主要有yml 跟ng的配置，后缀名为bak

你需要做的事情是修改.env文件，改成符合你自己身份气质的状态即可

多项目并存，请修改ng配置，启用域名访问即可（记住要修改本地host文件，做dns映射）

## `总结一下`

`A:修改.env文件 改成你自己的目录  `

`B:修改config/nginx/default.conf或者自行添加自定义域名conf（参照lacoste.conf），按照{需要修改}A B C 三步修改完`


在完成以上几步之后，就可以访问开发环境了，具体访问地址如下：

| 项目 | 地址 | 
| - | - |
| 管理端 | <http://127.0.0.1:8080/> | 
| 后端api | <http://127.0.0.1:8080/api> | 
| 微信授权 | <http://127.0.0.1:8080/wechatAuth> | 
| phpmyadmin | <http://127.0.0.1:8004> | 
| neo4j | <http://127.0.0.1:7474> |

## 补充信息
我们可以通过以下命令查看web容器名称：
```shell
docker ps | grep espier-bloated-web
```
> espier-bloated-web 是在 docker-compose.yml 中定义的服务的名称

如果没问题，可以看下以下信息：
```shell
55763c866ca7  hub.ishopex.cn/espier-docker-library/php:7.2-composer-alpine3.10   "php-fpm"                2 hours ago         Up 2 hours          9000/tcp, 0.0.0.0:8085->8005/tcp                           ecshopx-dev-docker_espier-bloated-web_1
```
`ecshopx-dev-docker_espier-bloated-web_1` 就是实际运行的容器的名称。
