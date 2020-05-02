# Github Action 自动部署

博客的前端项目同样需要在每次修改之后自动部署到服务器上，但是由于服务器内存太小导致，运行 `npm run build` 命令的时候内存不足导致其他程序异常退出，所以没有使用 Jenkins 进行自动化部署。然后发现 Github 提供了 Action 可以进行自动化 build 部署。所以对于前端项目使用 GitHub Action 进行部署。

