* 远程仓库MyBlog为源码存放位置
* MyBlog分支backup为源码备份
* MyBlog主干master为主要工作目录
* 本地MyBlogTemp目录clone远程MyBlog仓库代码，用作编译部署

工作方式
1 本地MyBlog主干master编写源码
2 推送到MyBlog远程仓库
3 本地MyBlogTemp目录拉取远程仓库MyBlog的代码进行编译
4 执行部署，将页面部署到github.io

### 必要时备份MyBlog分支backup