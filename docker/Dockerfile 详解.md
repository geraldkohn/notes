## 镜像构建上下文

一个构建镜像的 docker 命令：docker build -t nginx:3 .

* 这里我们指定了最终镜像的名称 -t nginx:3

* docker build 命令最后有一个 . 表示当前目录

这个 . 表示的是上下文，不是 Dockerfile 文件的目录。有些 docker 命令需要从宿主机拷贝一些东西，比如通过 copy、 add 命令。上下文就是用来指定：copy，add 的相对路径，比如一个 copy 命令：

```dockerfile
COPY ./package.json /app/
```

指示将上下文目录中的 package.json 复制到 /app/ 这个镜像中的目录下。不是复制执行 docker build 命令所在的目录下，也不是 Dockerfile 所在的目录，而是上下文目录！

 
