* 远程仓库MyBlog为源码存放位置
* MyBlog分支backup为源码备份
* MyBlog主干master为主要工作目录
* 本地MyBlogTemp目录clone远程MyBlog仓库代码，用作编译部署

工作方式
1. 本地MyBlog主干master编写源码
2. 推送到MyBlog远程仓库
3. 本地MyBlogTemp目录拉取远程仓库MyBlog的代码进行编译
4. 执行部署，将页面部署到github.io

具体步骤
1. 打开MyBlog仓库并cd到_post目录，copy一份post文稿，并创建对应名字的资源目录
2. 修改post内容，可以顺便配图
3. git提交并push
4. cd到MyBlogTemp仓库，git拉取更新
5. hexo clean清除，然后hexo d部署，完成！

编写post建议
1. cd MyBlogTemp
2. 执行hexo server
3. 打开http://localhost:4000/admin，便能实现markdown实时解析

### 必要时备份MyBlog分支backup
### 修改hueman主题时，务必同时更新根目录下的hueman主题