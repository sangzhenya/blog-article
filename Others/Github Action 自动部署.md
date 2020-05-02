# Github Action 自动部署

博客的前端项目同样需要在每次修改之后自动部署到服务器上，但是由于服务器内存太小导致，运行 `npm run build` 命令的时候内存不足导致其他程序异常退出，所以没有使用 Jenkins 进行自动化部署。然后发现 Github 提供了 Action 可以进行自动化 build 部署。所以对于前端项目使用 GitHub Action 进行部署。

*本文仅记录自己前端项目的自动构建配置流程，更详细的 Github Action 用法可以参考 [GitHub Actions 入门教程](http://www.ruanyifeng.com/blog/2019/09/getting-started-with-github-actions.html)*

打开项目首页，发现有一个 Actions tab 即可以打开 Actions type。

![Github Action](https://i.loli.net/2020/05/02/sLVnQCWywORK92M.png)

然后点击 `New workflow` 创建一个新的 Action。

![选择 Action 界面](https://i.loli.net/2020/05/02/6o3ChnbdVgqWmFu.png)

这里使用的 Node.js 的为模板，使用 yaml 格式的一个 workflow，如下所示：

![Node.js CI](https://i.loli.net/2020/05/02/p3Nf6hHSgyqG1Mj.png)

根据自己的实际情况完成编写即可，如果需要 ssh 部署可以在右侧 `Marketplace` 中搜索 `ssh deploy` 选择第一个 `easingthemes/ssh-deploy`，点击复制然后粘贴到 `workflow` 中即可。

![ssh deploy](https://i.loli.net/2020/05/02/ZKCLu4nbv1m9eOX.png)

这个 Action 需要配置 ssh key，可以在 项目 setting - secrets 中添加一个 secret，即自己登陆服务器 ssh 私钥，然后可以在 workflow 引用 key。

![Secrets](https://i.loli.net/2020/05/02/GUv2qjslf1QmOXM.png)

一个简单的例子如下：
```yaml
name: Node.js CI

# 在 master branch 上有 push 操作的时候执行
on:
  push:
    branches: [ master ]

# jobs
jobs:
  # 第一个 job 内容
  build:

    # 基于的平台
    runs-on: ubuntu-latest

    # 具体的步骤，包含使用的 action，步骤的名称。
    steps:
    # build 项目
    - uses: actions/checkout@v2
    - name: Use Node.js 10
      uses: actions/setup-node@v1
      with:
        node-version: 10
    - run: npm ci
    - run: npm run build --if-present

    - run: cp -rf dist blog
          
    # 部署到服务上
    - name: ssh deploy
      uses: easingthemes/ssh-deploy@v2.1.2
      env: 
        SSH_PRIVATE_KEY: ${{ secrets.KEY_3 }}
        REMOTE_HOST: "{ip_address}"
        REMOTE_USER: ubuntu
        SOURCE: blog
        TARGET: /var/www/html
```