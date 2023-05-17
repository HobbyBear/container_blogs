# (1)500代码手写docker开篇-goland远程编译环境配置
> 本系列教程主要是为了弄清楚容器化的原理，纸上得来终觉浅，绝知此事要躬行，理论始终不及动手实践来的深刻，所以这个系列会用go语言实现一个类似docker的容器化功能，最终能够容器化的运行一个进程。


在开始写代码之前，先介绍下我的实验环境，本地开发环境是arm64 mac m1，为了能方便的在linux上进行调试，我买了一个amd64的云linux 服务器，其实也可以本地搭建一个linux虚拟机代。 代码编辑器选择了goland，并在goland配置了远程编译，这样便能在本地编写调试 适合amd64 linux环境的代码了。

下面是我配置的详细步骤。

## goland 配置

我创建了一个名为tidydocker的项目，然后用goland打开，进入到goland配置界面配置sftp

![image.png](https://s2.loli.net/2023/05/13/eP1zZHdxwA5TBDq.png)
配置远程的部署路径,注意我已经在linux服务器上提前创建好了projects和tinydocker 目录了。到时候goland在寻找部署目录时会根据上一个截图的root path 和下面截图的Deployment path 结合起来寻找部署目录。
![image.png](https://s2.loli.net/2023/05/13/2aTtH5bxZIgUqom.png)

接着配置go remote，这样到时候我们便能够远程调试代码。

![image.png](https://s2.loli.net/2023/05/13/JCTFhiunzk53fqt.png)
在接着配置goland之前，还需要在远程linux机器上部署调试工具。

首先肯定要有golang环境
```shell
root@ecs-295280:~# go version
go version go1.20.3 linux/amd64
root@ecs-295280:~# 
```

接着安装dlv调试工具
```shell
 go install github.com/go-delve/delve/cmd/dlv@latest
```

写一个简单hello world程序

![image.png](https://s2.loli.net/2023/05/13/1FlakU9DT3wQieV.png)

配置远程编译，编译的选项选择run on 在我们远程linux主机上。

![image.png](https://s2.loli.net/2023/05/13/aGzePSWdV7fvTQl.png)
注意编译时候设置-o参数这样能让我们编译后的文件名称为tinydocker，不然就是goland为我们自动生成的一串很长的文件名。

点击manager targets 配置编译后的文件输出目录
![image.png](https://s2.loli.net/2023/05/13/nlLtzV2uj7YbcQD.png)

## 运行效果

这下配置就算全部完成了，点击编译，goland便会将代码自动上传到远端，然后执行编译过程。
![image.png](https://s2.loli.net/2023/05/13/B56vKF7hfdQoEYI.png)
上一步完成后，登录到远端看看，可以发现已经生成了tinydocker的可执行文件
```shell
root@ecs-295280:~/projects/tinydocker# ls
go.mod  main.go  ReadMe.md  tinydocker
root@ecs-295280:~/projects/tinydocker# pwd
/root/projects/tinydocker
root@ecs-295280:~/projects/tinydocker# 

```

接着远端执行调试命令
```shell
root@ecs-295280:~/projects/tinydocker#  dlv exec  tinydocker  --headless --listen=:2345 --api-version=2 --accept-multiclient 
API server listening at: [::]:2345
2023-05-02T01:27:04+08:00 warning layer=rpc Listening for remote connections (connections are not authenticated nor encrypted)

```

然后本地goland 给hellow world 程序打上断点 执行remote
![image.png](https://s2.loli.net/2023/05/13/gFL4My7H9lfRwPb.png)
可以看到断点已经生效了，这样便配置完成了goland的远程编译调试环境。


