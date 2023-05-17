# (3)500行代码代码手写docker-将rootfs设置为只读镜像

> 本系列教程主要是为了弄清楚容器化的原理，纸上得来终觉浅，绝知此事要躬行，理论始终不及动手实践来的深刻，所以这个系列会用go语言实现一个类似docker的容器化功能，最终能够容器化的运行一个进程。

本章的源码已经上传到githuhub，地址如下:
```shell
https://github.com/HobbyBear/tinydocker/tree/chapter3
```


前文提到，如果仅仅将ubuntu-base-16.04.6-base-amd64 目录作为容器的根目录， 那么当运行多个容器，就会同时修改到ubuntu-base-16.04.6-base-amd64目录，这样将达不到不同容器使用不同的根文件系统的目的。

所以这节我将会演示如何运行内核提供到联合文件系统的功能，来达到一份镜像，多次运行的目的。

这节代码运行效果:

![image.png](https://s2.loli.net/2023/05/14/TfOhnxlvF6eAYMK.png)

可以看到我其实启动了两个容器 hello1 ,hello2 然后在hello1 下创建test目录，但是test目录在hello2容器里是不可见的。


## 联合文件系统原理
首先，来先简单的看看联合文件系统的概念。
>🦧🦧🦧 联合文件系统可以把其他文件系统的文件和目录挂载到同一个挂载点下，形成统一的文件系统，在挂载点下形成统一的文件视图

在linux内核里，自带了一种叫做overlay类型的文件系统类型，它是一种联合文件系统，类似的还有aufs，不过本文还是用overlay 类型进行举例。

如下是一个挂载overlay 文件系统的mount命令
```shell
sudo mount -t overlay overlay -o lowerdir=image-layer1:image-layer2,upperdir=container-layer,workdir=work mnt/
```
其中contailber-layer 后续会作为容器的读写层，image-layer会作为镜像层，mnt作为overlay联合文件系统的挂载目录，而work后续会作为overlay联合文件系统的工作目录，这个目录是overlay自己用的，对用户不可见。挂载目录为mnt。

也就是说后续进程可以统一访问mnt目录就能看到image-layer 和contailber-layer 这两个目录的内容，但是对mnt目录进行修改的话，则只会将修改体现在contailber-layer这个目录下，image-layer这个目录永远不会变。

关于联合文件系统更详细的解释和命令演示可以参考之前我的一篇博文[容器镜像原理- 联合文件系统实践](https://mp.weixin.qq.com/s/2Zsg7PFbWWPbxgyPR6PHUg)

##  如何用go代码实现
接着，我们来看看如何对前文的代码进行改造。

已经知道了，当挂载一个overlay文件系统时，镜像层的文件是永远不会变的，所以ubuntu-base-16.04.6-base-amd64这个roofs目录毫无疑问将会作为镜像层进行参数传递，而我们还需要为容器创建其自身的可写层和工作层目录。因为可以运行多个容器，如何区分这些容器各自的可写层呢？最简单的方法就是拥有一个容器名，通过容器名创建属于他们自己的目录。

所以，现在运行命令的方式变了，之前我们是这样运行一个容器:
```shell
./tinydocker run /bin/sh
```
现在将变成这样
```shell
./tinydocker run 容器名 /bin/sh
```

先统一浏览下目前main方法中的代码
```go
func main() {

switch os.Args[1] {
case "run":
initCmd, err := os.Readlink("/proc/self/exe")
if err != nil {
fmt.Println("get init process error ", err)
return
}
// 获取容器名
containerName := os.Args[2]
os.Args[1] = "init"
cmd := exec.Command(initCmd, os.Args[1:]...)
cmd.SysProcAttr = &syscall.SysProcAttr{
Cloneflags: syscall.CLONE_NEWUTS | syscall.CLONE_NEWPID | syscall.CLONE_NEWNS |
syscall.CLONE_NEWNET | syscall.CLONE_NEWIPC,
}
cmd.Env = os.Environ()
cmd.Stdin = os.Stdin
cmd.Stdout = os.Stdout
cmd.Stderr = os.Stderr
err = cmd.Run()
if err != nil {
fmt.Println(err)
}
// 容器结束后要清理掉它的挂载点和目录
workspace.DelMntNamespace(containerName)
return
case "init":
var (
containerName = os.Args[2]
cmd           = os.Args[3]
)
// 创建挂载点和更换rootfs
if err := workspace.SetMntNamespace(containerName); err != nil {
fmt.Println(err)
return
}
syscall.Chdir("/")
defaultMountFlags := syscall.MS_NOEXEC | syscall.MS_NOSUID | syscall.MS_NODEV
syscall.Mount("proc", "/proc", "proc", uintptr(defaultMountFlags), "")
err := syscall.Exec(cmd, os.Args[3:], os.Environ())
if err != nil {
fmt.Println("exec proc fail ", err)
return
}
fmt.Println("forever exec it ")
return
default:
fmt.Println("not valid cmd")
}
}
```
可以看到，在以新命名空间启动一个子进程后，在workspace.SetMntNamespace 里将会进行相关目录的挂载，然后在执行cmd.Run 的父进程中，等待子进程结束后，调用了workspace.DelMntNamespace清理了子进程的挂载点和相关目录。

而workspace.SetMntNamespace 的源码如下:
```golang
func SetMntNamespace(containerName string) error {
if err := os.MkdirAll(mntLayer(containerName), 0700); err != nil {
return fmt.Errorf("mkdir mntlayer fail err=%s", err)
}
if err := os.MkdirAll(workerLayer(containerName), 0700); err != nil {
return fmt.Errorf("mkdir work layer fail err=%s", err)
}
if err := os.MkdirAll(writeLayer(containerName), 0700); err != nil {
return fmt.Errorf("mkdir write layer fail err=%s", err)
}

if err := syscall.Mount("overlay", mntLayer(containerName), "overlay", 0,
fmt.Sprintf("upperdir=%s,lowerdir=%s,workdir=%s",
writeLayer(containerName), imagePath, workerLayer(containerName))); err != nil {
return fmt.Errorf("mount overlay fail err=%s", err)
}

if err := syscall.Mount("", "/", "", syscall.MS_PRIVATE|syscall.MS_REC, ""); err != nil {
return fmt.Errorf("reclare rootfs private fail err=%s", err)
}

if err := syscall.Mount(mntLayer(containerName), mntLayer(containerName), "bind", syscall.MS_BIND|syscall.MS_REC, ""); err != nil {
return fmt.Errorf("mount rootfs in new mnt space fail err=%s", err)
}
if err := os.MkdirAll(mntOldLayer(containerName), 0700); err != nil {
return fmt.Errorf("mkdir mnt old layer fail err=%s", err)
}
if err := syscall.PivotRoot(mntLayer(containerName), mntOldLayer(containerName)); err != nil {
return fmt.Errorf("pivot root  fail err=%s", err)
}
return nil
}
```

workspace.SetMntNamespace 先是根据容器名创建了执行overlay挂载所需要的目录，然后通过mount命令讲一个overlay类型的文件系统挂载到mntLayer(containerName)的路径下，然后mntLayer(containerName)路径下文件将作为容器的根文件系统，后续会对其进行pivot root调用，然后mntLayer(containerName)的目录将会成为新的mnt namespace的根目录了。

这样，不同容器名的容器将会有自己独立的根目录。即避免了镜像层文件的改变，又达到了各容器文件系统隔离的目的。

