# 指月小筑
![](./static/img/jinx.png)

博客在本地修改及预览完成后 push 到该仓库，由 github action 推送到指定服务器以及 github page 上。
> 相关教程：[基于 Github Action 自动构建 Hugo 博客](https://www.lixueduan.com/post/blog/01-github-action-deploy-hugo/)

在线查看：
* http://blog.yudlk.com.com
* http://qpt.github.io





## 切换部署环境

迁移到新服务器具体流程见 [deploy 教程](./deploy/readme.md)



## 切换写作环境

环境配置步骤,写给自己，备忘。

**1）安装 hugo**

到 [github release page](https://github.com/gohugoio/hugo/releases) 下载压缩包，解压并配置环境变量即可。

> 当前使用的是 v0.100.2 extended 版本，更新版本可能出现不兼容情况。

```bash
$ hugo version
hugo v0.100.2-d25cb2943fd94ecf781412aeff9682d5dc62e284+extended windows/amd64 BuildDate=2022-06-08T10:25:57Z VendorInfo=gohugoio
```

**2）clone 此仓库**

```
git@github.com:barrypt/qpt.github.io.git
```

使用 submodules 方式，将 themes 合并到了当前仓库，不需要再额外拉取了。

> 具体配置见 .gitmodules 文件

不过 submodule 还是需要手动更新：

```bash
git submodule update --init --recursive
```
如果需要添加、更新或删除 submodule 的话，参考 [submodule 常用命令](./deploy/readme.md)

至此，环境就 ok 了。

## hugo 常用命令

```
# 查看hugo版本号
hugo version 
# 本地运行
hugo server
# 本地运行 指定为 production 环境，默认为 development 环境，该环境下部分特性不会开启
hugo serve -e production
# 生成public文件
hugo
```