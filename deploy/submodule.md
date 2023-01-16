## submodule 常用命令
> git submodule -h

### 添加
语法
```bash
git submodule add {url} [path]
```
demo
```bash
git submodule add https://github.com/lixd/LoveIt themes/LoveIt
```

### 更新
```bash
git submodule update --init --recursive
```

### 删除
* 1）更新 .gitmodules，移除该 submodule 对应配置
* 2）从主仓库中移除该 submodule 对应文件
* 3）从 .git/config 中移除该 submodule 对应配置
* 4）删除 .git/modules 目录下该 submodule 对应文件

以移除该仓库中的 LoveIt 为例
* 1）更新 .gitmodules，移除该 submodule 对应配置
  需要移除的内容如下：
```text
[submodule "themes/LoveIt"]
path = themes/LoveIt
url = https://github.com/lixd/LoveIt
```
* 2）从主仓库中移除该 submodule 对应文件
  该 submodule 对应 themes/LoveIt 目录，因此直接移除 themes 下的 LoveIt 目录
* 3）从 .git/config 中移除该 submodule 对应配置
  需要移除的内容如下：
```text
[submodule "themes/LoveIt"]
	url = https://github.com/lixd/LoveIt
	active = true
```
* 4）删除 .git/modules 目录下该 submodule 对应文件
  该 submodule 对应 themes/LoveIt 目录,因此 init 之后会在 .git/modules 目录下创建 themes/LoveIt 两级目录，
  同样的，这里直接移除整个 themes/LoveIt 目录。

至此，submodule 就删除干净了。