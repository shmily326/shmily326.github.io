---
layout: post
title: 利用 GitHub 和 GitPython 做增量图床
categories: [python]
description: 
keywords: Python, GitHub, ImageBed
---

# 0x00
参考 [Github图床](https://www.jianshu.com/p/c794bad425e5) 给出的方法，在 GitHub 上新建一个仓库：

>1. 新建一个 repositories
>2. 进入 repositories 下的 wiki 选项中
>3. 复制 wiki 对应的克隆地址
>4. 克隆该项目到本地
>5. 进入文件夹，将要上传的图片放在文件夹中

# 0x01
至此，图床算是创建完毕了，但是仍有两点不方便的地方：

* 每次存入新图片后，需要手工操作 git 进行 add/commit/push
* 若删除了本地图床文件夹中不想留存的图片（临时性图片/防止占用空间过大），则在下次 git add/commit/push 中很有可能导致线上图片也被删除

因此，这里利用 GitPython 库来操作 Git，免去一些重复的工作。  

```Python
import pyperclip # 操作剪切板
from git import Repo # 实际包名为 GitPython
import os

# 克隆到本地的图床文件夹绝对地址
PATH = r'E:\XXX\ImageBed'
# 下面是上传后图片地址的前缀，将其复制到剪切板
pyperclip.copy("https://raw.githubusercontent.com/wiki/YOUR_GitHub_NAME/ImageBed/")

def find_new(dirname):
    '''
        用于找到图床文件夹中存在的以规定格式结尾的文件名，
        即只会 `git add` 这些文件（图片），而忽略在本地已被删除的文件（图片）
    '''
    name_list = []
    postfix = ['jpg','png','gif','bmp','jpeg'] # 自行添加格式
    for maindir, subdir, file_name_list in os.walk(dirname):
        for name in file_name_list:
            if name.split('.')[-1] in postfix:
                name_list.append(name)
    return name_list

def push():
    # 新建版本库对象
    repo = Repo(PATH)
    git = repo.git
    for file_name in find_new(PATH):
        git.add('./' + file_name)
    git.commit('-m', 'update')
    # 获取远程仓库
    remote = repo.remote()
    # 推送本地修改到远程仓库
    remote.push()

push()
```
# 0x02
至此，一些仍然需要注意的事：
* 虽然以上代码已经可以工作，但是可能你每次仍然需要输入 GitHub 账号和密码，那么你可以在 Git 中通过配置 GitHub SSH key 或者通过简单的全局配置 credential.helper 将你的账号和密码保存在本地：
    * `git config --global credential.helper store`
    * `git pull / git push (这里需要输入一次用户名和密码，以后就不用了)`

    可以参考：[git不用输入用户名和密码](https://blog.csdn.net/LosingCarryJie/article/details/73801554), [git配置免密登录](https://blog.csdn.net/lqlqlq007/article/details/79065095)

* 一切配置好后，可以将上述代码后缀改为 `.pyw`，即以无窗口（no console）模式运行。在图床中放入新图片后只需双击该文件，然后就可以 Ctrl+V+图片文件名，使用该图片的链接了

* 当然该方法也不是很便捷，代码健壮性也并不强，这里仅仅给出一点点思路，如果您有更好更方便的方法，请分享给大家！
