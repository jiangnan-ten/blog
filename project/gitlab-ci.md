# gitlab ci 实践之路




# why?

## 传统的部署方式
<img src="_media/p2/1.png" width="800"/>

缺点:

1. 需要登录N个服务器, 过程繁琐
1. 手动上传, 效率低下
1. 测试不通过, 需要重复以上操作
1. ...

## 理想化的部署模式
<img src="_media/p2/2.png" width="800"/>

优势:

- 无需任何远程登录
- gitlab 接管代码拉取, 编译, 部署(测试, 预发布, 线上)
- gitlab UI 查看进度, 支持分支部署, 回滚部署, 条件部署

<img src="_media/p2/3.png" />


# How
## 安装runner

[官方安装详细教程](https://docs.gitlab.com/runner/install/)

安装方式分为2种, 这里我们采用第二种方式, 以**docker container** 模式运行

- 传统的安装包模式
- docker runner

### 安装docker

```bash
curl -sSL https://get.docker.com/ | sh
```

### 运行gitlab-runner container

```bash
docker run -d --name gitlab-runner --restart always \
  -v /srv/gitlab-runner/config:/etc/gitlab-runner \
  -v /var/run/docker.sock:/var/run/docker.sock \
  gitlab/gitlab-runner:latest
```

> docker run 遇到本地没有的image, 会自动远程拉取对应的image


### 注册gitlab runner

[官方详细注册文档](https://docs.gitlab.com/runner/register/#docker)

运行如下命令, 会进入交互模式, 按照提示操作

```bash
docker run --rm -t -i -v /srv/gitlab-runner/config:/etc/gitlab-runner gitlab/gitlab-runner register
```

上述操作会形成下述配置文件, 默认保存在 /srv/gitlab-runner/config 中, 以后可更改配置

```yaml
concurrent = 1
check_interval = 0

[session_server]
  session_timeout = 1800

[[runners]]
  name = "web-ci/cd"
  url = "https://git.cas-pe.com/"
  token = "BNVcJD5AZMigxZB7zXMG"
  executor = "docker"
  [runners.custom_build_dir]
  [runners.docker]
    tls_verify = false
    image = "node:10.15.3"
    privileged = false
    disable_entrypoint_overwrite = false
    oom_kill_disable = false
    disable_cache = false
    volumes = ["/cache"]
    shm_size = 0
  [runners.cache]
    [runners.cache.s3]
    [runners.cache.gcs]
```

## gitlab ci 配置
配置路径: 项目xxx/settings/ci cd/runners

### 启用Specific Runners, 关闭Shared Runners
<img src="_media/p2/4.png" />


已注册的runner 会如下显示

<img src="_media/p2/5.png" />

点击编辑按钮, 可针对runner做配置修改, 诸如超时时间, tags等设置

<img src="_media/p2/6.png" />

### 环境变量配置

- SSH_PRIVATE_KEY 部署gitlab ci runner 的服务器私钥
- 其他.....

### ci 自动部署的必要配置

runner 服务器需要自动ssh登录到被部署的服务器实施部署, 因此需要将runner的公钥记住, 示例:

<img src="_media/p2/7.png" />


## .gitlab-ci.yml 配置文件

[官方配置文件详解](https://docs.gitlab.com/ee/ci/yaml/)

改文件为gitlab-ci的核心配置文件, 指明了gitlab如何运行, 编译, 部署

配置文件示例:

> 配置文件中涉及到的一些私密信息 诸如服务器ip 端口号 登录用户名 等最好还是用gitlab 的变量引入, 我在这里为了方便直观就省略了该步骤


```yaml
image: tencx/node-rsync-ssh

stages:
  - build
  - deploy-local
  - deploy-test
  - deploy-prod

build:
  stage: build
  artifacts:
    paths:
      - dist
    expire_in: 6 hour
  script:
    - npm config set unsafe-perm true
    - npm config set registry https://registry.npm.taobao.org
    - npm config set sass_binary_site "https://npm.taobao.org/mirrors/node-sass/"
#    - npm install -g @vue/cli@latest
    - npm install
    - npm run build
  only:
    variables:
      - $CI_COMMIT_MESSAGE =~ /^trigger-ci.*/

deploy-local:
  stage: deploy-local
  script:
    - mkdir -p ~/.ssh
    - echo "$SSH_PRIVATE_KEY" >> ~/.ssh/id_rsa
    - chmod 600 ~/.ssh/id_rsa
    - echo -e "Host *\n\tStrictHostKeyChecking no\n\n" > ~/.ssh/config
    - rsync -hrvz --delete dist/ platform_user@10.5.10.95:/home/platform_user/www/performance-for-web
  only:
    variables:
      - $CI_COMMIT_MESSAGE =~ /^trigger-ci.*/

deploy-test:
  stage: deploy-test
  script:
    - mkdir -p ~/.ssh
    - echo "$SSH_PRIVATE_KEY" >> ~/.ssh/id_rsa
    - chmod 600 ~/.ssh/id_rsa
    - echo -e "Host *\n\tStrictHostKeyChecking no\n\n" > ~/.ssh/config
    - rsync -hrvz --delete dist/ platform_user@10.5.10.45:/home/platform_user/web/performance-for-web
  only:
    variables:
      - $CI_COMMIT_MESSAGE =~ /^trigger-ci.*/
  when: manual


deploy-prod:
  stage: deploy-prod
  script:
    - mkdir -p ~/.ssh
    - echo "$SSH_PRIVATE_KEY" >> ~/.ssh/id_rsa
    - chmod 600 ~/.ssh/id_rsa
    - echo -e "Host *\n\tStrictHostKeyChecking no\n\n" > ~/.ssh/config
    - rsync -hrvz -e "ssh -p 22122" --delete dist/ platform_user@xx.xx.xx.xx:/home/platform_user/web/performance-for-web
  only:
    variables:
      - $CI_COMMIT_MESSAGE =~ /^trigger-ci.*/
  when: manual

```

### 核心概念

1. image

指明了container环境, 因为ci 的runner为docker, 所以这里指明所需要的镜像, 如不指明, 使用默认配置文件 /srv/gitlab-runner/config 中的image

2. stages

上述示例, 指明需要build, deploy-local, deploy-test, deploy-prod 4个阶段
build: 编译阶段
deploy-local: 部署local服务器
deploy-test: 部署test服务器
deploy-prod: 部署线上服务器

3. expire_in 

打包后的资源保存时间

4. script

stage 阶段 运行的脚本命令

5. only

触发条件 ([基础](https://docs.gitlab.com/ee/ci/yaml/#onlyexcept-basic)) ([高级](https://docs.gitlab.com/ee/ci/yaml/#onlyexcept-advanced))
上述配置的含义是, build, deploy-local 触发ci的条件是$CI_COMMIT_MESSAGE([更多内置变量]()[)]() 是以trigger-ci 开头, deploy-test, deploy-prod 则是需要再web界面手动点击按钮部署

## 使用示例

### 场景一

小明本地代码未提交, 准备在这次代码提交到远端后触发自动构建部署

触发构建部署
<img src="_media/p2/8.png" />

最后执行push, 可以再git pipeline 界面看到触发了ci:

<img src="_media/p2/9.png" />


点击stage, 可以查看详情
<img src="_media/p2/10.png" />


一段时间过后, 可以看到执行结果, 如果失败, 进入查看详情, 如果想取消本次构建, 点击cancel按钮取消

<img src="_media/p2/11.png" />


针对之前的配置 stage为deploy-prod, 需要手动触发部署, 因为是正式生产环境, 最好还是采取人工手动部署, 右侧按钮, 点击 manual job > deploy-prod 即可, 针对构建包, 也可以下载, 最右侧按钮如图所示

<img src="_media/p2/12.png" />


### 场景二
本地代码已提交和远程保持一致, 没有可提交的代码来通过commit message触发ci, 改如何操作?

点击右侧run pipeline, 进入
<img src="_media/p2/13.png" />

选择分支, 填入如下信息, 仍可触发pipeline

<img src="_media/p2/14.png" />


### 场景三
在push的时候, 遇到了这种冲突情况
<img src="_media/p2/15.png" />


那么, 先pull, 有冲突解决冲突, 待一切准备就绪
运行 git commit --amend 修改commit message, 最后再push
<img src="_media/p2/16.png" />


# 思考

现阶段的ci /cd 能够满足小规模团队的需求, 当团队规模扩大后, 会存在以下不足:

- 多人同时push 触发构建, 会造成runner资源浪费, 多个runner同时运行, 可能会出现位置问题, 如果runner等待, 则开发人员一直获取不到构建部署结果, 结果不可控
- 缺少必要的权限, 任何人员都可构建部署
- 集群部署架构设计不足, ui功能较弱, 缺少可选择性的部署集群服务器
- 和docker融合不深, 比如部署之前, 仍需提前配置服务器nginx, 不够智能, 是否可以本地有dockerfile直接将部署配置写入, 部署的时候充分利用docker实现编译打包, 部署, 服务器配置等一站式需求
