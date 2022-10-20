# 中间件篇

- [中间件篇](#中间件篇)
  - [Kafka](#kafka)
  - [nginx](#nginx)
  - [docker](#docker)
    - [1.CMD和ENTRYPOINT的区别?](#1cmd和entrypoint的区别)
      - [覆盖](#覆盖)
      - [shell和exec表示法](#shell和exec表示法)
    - [ENTRYPOINT和CMD组合使用](#entrypoint和cmd组合使用)
    - [2.COPY和ADD的区别?](#2copy和add的区别)
    - [3.常用的Dockerfile命令有哪些?](#3常用的dockerfile命令有哪些)

## Kafka

## nginx

## docker

### 1.CMD和ENTRYPOINT的区别?

**在功能上,都是在imagin启动时执行一条命令**,像一些基础的镜像,如alpine,Ubuntu等是使用的CMD指定执行的 /bin/bash或sh,这样在docker run时,即使不指定任何参数,其默认的就是启动一个终端,这样能使容器正常运行,因此,Dockerfile中必定有CMD或ENTRYPOINT,否则会报错

- Entrypoint指令用于**设定容器启动时第一个运行的命令及其参数**。
在执行`docker run <镜像名> <arg1,arg2,arg3>`时,镜像名之后的参数都会被附加到ENTRYPOINT指定的命令之后,如果有指定CMD,这些参数会覆盖掉CMD指定的内容

- CMD指令的**主要用意是设置容器的默认执行的命令**。CMD设定的内容会放在entrypoint之后

可以从几个方面来理解这两个命令的区别:

#### 覆盖

ENTRYPOINT和CMD会自动覆盖之前的指定的命令(即最后一个CMD或ENTRYPOINT才会生效)

假设创建一个Dockerfile如下:

```docker
FROM ubuntu:trusty
CMD ping localhost 
```

直接使用`docker run demo`,将会执行CMD指定的 `ping localhost`

如果使用`docker run demo hostname`,则会执行hostname命令,即显示指定的命令会覆盖CMD指定的命令

ENTRYPOINT指定的命令也会被覆盖,使用`---entrypoint`指定,如:
`docker run --entrypoint hostname demo`

因为CMD更容易被覆盖(不需要使用flag),如果希望docker镜像更灵活,建议使用CMD;

相反, ENTRYPOINT的作用不同, **如果你希望你的docker镜像只执行一个具体程序, 不希望用户在执行docker run的时候随意覆盖默认程序. 建议用ENTRYPOINT.**

#### shell和exec表示法

ENTRYPOINT和CMD均支持shell和exec表示法,如shell表示法:
`CMD <命令> parama1 parama2`  

当使用shell表示法时, **命令行程序作为sh程序的子程序运行, docker用/bin/sh -c的语法调用**. 如果我们用docker ps命令查看运行的docker, 就可以看出实际运行的是/bin/sh -c命令,而不是ping命令,这种方式会出现的问题就是信号无法正确传递给运行的命令,比如传递SIGINT终止信号会发送至sh,而不是ping;而且如果构建的是一个小到连sh都没有的镜像,则容器都无法运行

因此更好的选择是用exec表示法:
`CMD ["executable","param1","param2"]`

CMD指令后面用了类似于JSON的语法表示要执行的命令. 这种用法告诉docker不需要调用/bin/sh执行命令.

Dockerfile修改为:

```docker
FROM ubuntu:trusty
CMD ["/bin/ping","localhost"] 
```

此时通过docker ps 查看就能看到执行的命令是 `/bin/ping`而不是sh -c

一定要使用exec表示法

### ENTRYPOINT和CMD组合使用

组合使用ENTRYPOINT和CMD, **ENTRYPOINT指定默认的运行命令, CMD指定默认的运行参数**

如:

```docker
FROM ubuntu:trusty
ENTRYPOINT ["/bin/ping","-c","3"]
CMD ["localhost"] 
```

在直接执行docker run时将会执行`/bin/ping -c 3 localhost`,如果指定了docker run的参数,则localhost将被覆盖

ENTRYPOINT和CMD同时存在时, **docker把CMD的命令拼接到ENTRYPOINT命令之后**, 拼接后的命令才是最终执行的命令.

### 2.COPY和ADD的区别?

都是从上下文目录拷贝内容到镜像中(只复制目录中的内容而不包含目录自身),但ADD会对内容做一些处理,如将压缩文件解压,URL自动下载等,而COPY就是纯粹的拷贝,实际使用一般都是COPY,语义更明确

### 3.常用的Dockerfile命令有哪些?

[参考](https://vuepress.mirror.docker-practice.com/image/build/#from-%E6%8C%87%E5%AE%9A%E5%9F%BA%E7%A1%80%E9%95%9C%E5%83%8F)

1. FROM:指定基础镜像;以scratch为基础镜像的话，意味着你不以任何镜像为基础，接下来所写的指令将作为镜像第一层开始存在。
2. RUN:执行命令,最常用,使用RUN时应该尽量将多种命令连接执行,避免使用多个RUN命令,因为每一次RUN都会构建一层镜像,在撰写 Dockerfile 的时候，要经常提醒自己，这并不是在写 Shell 脚本，而是在定义每一层该如何构建,合理利用`\`和`&&`,并且RUN最后不要忘了清理本层构建产生的临时文件
3. COPY:`COPY <源文件> <镜像内路径>`,源文件可以多个,COPY时源文件的各种元数据都会保留,如读写执行权限等
4. ADD:更高级的COPY,拓展了一些功能,如自动解压,URL自动下载等;ADD会是镜像构建缓存失效
5. CMD:容器是进程而不是虚拟机,既然是进程,那么启动时就可以指定参数,CMD就是用于指定默认的启动命令的,可被docker run <镜像名> <命令 参数>直接覆盖;一定要注意,容器中运行的程序一定要是在前台运行的,不要想主机一样用Systemd去启动服务,容器内没有后台服务的概念,正确的做法是直接执行可执行文件,如:`CMD ["nginx", "-g", "daemon off;"]`
6. ENTRYPOINT:目的和 CMD 一样，都是在指定容器启动程序及参数。ENTRYPOINT 在运行时也可以替代，不过比 CMD 要略显繁琐，需要通过 docker run 的参数 --entrypoint 来指定;当指定了 ENTRYPOINT 后，CMD 的含义就发生了改变，不再是直接的运行其命令，而是将 CMD 的内容作为参数传给 ENTRYPOINT 指令，换句话说实际执行时，将变为：`<ENTRYPOINT> "<CMD>"`;几个使用场景:1.使用镜像作为命令工具时,如果使用CMD,那么如果想要在docker run时修改某个参数,将不得不输入完整的CMD后的命令,应该使用ENTRYPOINT来将此命令写入,而CMD就会成为参数,每次再docker run之后的内容将会替换CMD的值传给ENTRYPOINT;2.使用ENTRYPOINT来执行一些准备工作,如用root配置数据库然后切换回普通用户等;此时ENTRYPOINT一般会是一个shell脚本,根据CMD被覆盖的内容来决定怎么执行程序
7. ENV:定义环境变量,定义之后,就可以在后续的指令中使用
8. ARG:和ENV一样,都是设置环境变量,所不同的是，ARG所设置的构建环境的环境变量，**在将来容器运行时是不会存在这些环境变量的**,**ARG指令有生效范围**，如果在 FROM 指令之前指定，那么只能用于 FROM 指令中,多阶段构建要尤其注意这个问题
9. VALUME:定义匿名卷,容器运行时应该**尽量保持容器存储层不发生写操作，对于数据库类需要保存动态数据的应用，其数据库文件应该保存于卷(volume)中**,为了防止运行时用户忘记将动态文件所保存目录挂载为卷，在 Dockerfile 中，我们可以事先指定某些目录挂载为匿名卷，这样在运行时如果用户不指定挂载，其应用也可以正常运行，避免向容器存储层写入大量数据。
10. EXPOSE:声明容器运行时提供服务的端口,**仅仅只是声明**,运行时不会直接开启这个端口,在 Dockerfile 中写入这样的声明有两个好处，**一个是帮助镜像使用者理解这个镜像服务的守护端口，以方便配置映射；另一个用处则是在运行时使用随机端口映射时，也就是 docker run -P 时，会自动随机映射 EXPOSE 的端口。**,和run时-p的功能要区分开,-p，是映射宿主端口和容器端口，换句话说，就是将容器的对应端口服务公开给外界访问，而 EXPOSE 仅仅是声明容器打算使用什么端口而已，并不会自动在宿主进行端口映射,主机使用host模式时,可以直接暴露EXPOSE声明的端口(对应主机的端口,不存在映射关系)
11. WORKDIR:指定工作目录,以后各层的当前目录就被改为指定的目录，如该目录不存在，WORKDIR 会帮你建立目录,每一个RUN都是启动一个容器,执行命令,然后提交存储层文件变更,多个RUN之间执行环境根本不相同,因此需要WORKDIR来显示的指定各层的工作目录的位置
12. USER:和WORKDIR类似,都是改变各层的运行状态;USER 则是改变之后层的执行 RUN, CMD 以及 ENTRYPOINT 这类**命令的身份**,注意,USER只是切换到目标用户,而**用户必须是事先创建好的**(在RUN中使用useradd groupadd等)
13. HEALTHCHECK:推荐使用exec格式,指定健康检查要执行的操作
14. LABEL:添加一些镜像的元数据,作者,文档地址等等
15. SHELL:可以指定,在运行RUN,CMD,ENTRYPOINT时所用的SHELL,如`SHELL: ["/bin/sh","-c"]`
16. ONBUILD: ONBUILD后面的RUN,COPY等指令,只有在自己作为其他镜像的基础镜像时才会执行,一般用于构建公共的基础镜像(多个dockerfile基于一个基础镜像,便于基础镜像更新),减少代码重复
