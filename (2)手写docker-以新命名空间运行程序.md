# (2)手写docker-以新命名空间运行程序

> 本系列教程主要是为了弄清楚容器化的原理，纸上得来终觉浅，绝知此事要躬行，理论始终不及动手实践来的深刻，所以这个系列会用go语言实现一个类似docker的容器化功能，最终能够容器化的运行一个进程。

本章要完成的任务则是golang启动一个sh的进程，并且sh的进程将在新的命名空间中运行。

届时运行效果如下:

![tty.gif](https://s2.loli.net/2023/05/13/O3fFxyNWnhoakHJ.gif)

在正式开始编写代码前，先来看看linux namespace涉及的一些原理。

## linux namespace 原理

命名空间是linux为了隔离各种资源而形成的一个概念，不同类型的命名空间能够对不同类型的资源进行隔离。

目前linux支持的命名空间有:

| 命名空间              |  |
|:----------------------|---|
| syscall.CLONE_NEWUTS  | 对主机名进行隔离 |
| syscall.CLONE_NEWPID | 对pid空间进行隔离 |
| syscall.CLONE_NEWNS   | 对mount命名空间进行隔离 |
| syscall.CLONE_NEWNET  | 对网络进行隔离 |
| syscall.CLONE_NEWIPC  | 对进程通信组件进行隔离，我认为主要是针对消息队列 |

对主机名的隔离 和ipc进程消息通信的隔离 比较好理解，不同uts 命名空间和ipc命名空间，其主机名和各自在ipc命名空间内部创建的 ipc组件对彼此都不可见。

着重来看下剩下的3种类型的命名空间。

### syscall.CLONE_NEWPID

内核在为新进程分配pid时对不同pid的namespace是进行了隔离的，同一个pid 的namespace下的pid是不会重复的，但不同pid的namespace下的pid可以重复。

当执行mount 命名挂载procfs时，也是先从当前进程 获取到 与进程相同的pid namespace，然后进行挂载的。所以后续你会看到当一个处于新pid namespace里的程序，执行mount 重新挂载proc文件系统后，proc文件系统中进程号为1的进程就变了。


###  syscall.CLONE_NEWNS

再来看看mnt namespace， 当执行完一个mount 命名后，再访问挂载的目标目录，你会发现目标目录的内容已经变成了你指定的挂载目录，mnt namespace就是为了让你的挂载目录对于其他的mnt namespace变成不可见的，其他mnt namespace该目录下依然是原先的目录结构。

> 📢📢❗️不过注意下，当用systemd作为init进程启动时，mount 默认的挂载方式是共享模式，这意味着你在一个mnt namespace下执行mount命令后的挂载对其他mnt 的namespace是可见的。
>
> 比如我在新mnt namespace下挂载procfs，这将会导致主机上的procfs失效，然后你访问主机的/proc 目录将会发现主机/proc目录下的内容和新mnt namespace /proc目录下的内容是一样的。所以当你回到主机的mnt namespace去执行top命令时，将会提示你需要将procfs重新进程挂载
>
> 解决这个问题的办法则是将新mnt namespace设置为私有模式，后续我会在代码里体现这一部分。


### syscall.CLONE_NEWNET

最后我们来看看network namespace起的作用，网络传输过程涉及到网络设备，域名解析，路由表，防火墙等等网络配置，network namespace的出现就是为了将这些各种各样的网络配置进行隔离。所以你会发现，当你最开始进入一个新network namespace时，你用ping 命名是ping不通任何地址的，因为你并没有为新的network namespace配置任何网络配置信息，比如路由表。


在大致了解了各种命名空间之后，那么究竟该如何在创建一个进程时指定新命名空间呢，让我们来看看用go如何实现。

## golang 如何以新命名空间启动程序

```go
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
```
golang 可以通过exec包下的Command 方法构建一个Command结构体，调用其Run 方法将会启动一个新的进程，其本质也是先clone系统调用 然后再 进行exec调用，可以看到在构建Command 时指定了clone所需要的flag参数 。

> 💡💡❗️clone系统调用其实和fork系统调用类似，不过clone系统调用可以指定在创建子进程时对哪些资源进行复制，比如上述例子中我们指定了各种命名空间的flag，这代表新启动的子进程将会在新的命名空间下运行。

Command 启动结构体其实有两种方式，一种是Start 方式，一种就是像代码里的Run 方法启动。

两种方法区别在于调用Start 不会等待子进程结束，而Run 方法将会等待子进程结束。而这里为什么要调用Run 方法呢，因为这里需要用到标准输入输出流，可以看到，我将控制台输入输出流传递给了Command的Stddin,Stdout参数，如果父进程在调用Start后关闭了进程，进程关闭将导致自身的文件描述符也关闭，所以标准输入输出也会关闭，那么子进程将不不能从标准输入中获取到信息了。

其父子进行通信的原理是通过建立一个管道，通过管道将标准输入的消息传递给了子进程，子进程也通道管道将自身的输出 输出到 标准输出。详细的细节可以参数 这篇 todo ❗️

总之，到这里算是明白了如何用golang启动一个新进程，并且新进程将拥有自己的命名空间。

现在让我们看下完整的这段代码
```go
func main() {
	switch os.Args[1] {
	case "run":
		initCmd, err := os.Readlink("/proc/self/exe")
		if err != nil {
			fmt.Println("get init process error ", err)
			return
		}
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
		fmt.Println("init proc end", initCmd)
		return
	case "init":
		cmd := os.Args[2]
		err := syscall.Exec(cmd, os.Args[2:], os.Environ())
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
来简单分析下这段代码，如果传递给程序的参数是run 那么将会在一个新的命名空间内 启动一个子进程，子进程运行的代码也是当前可执行程序的代码。

> 💡💡👨‍🦰 /proc/self/exe 是一个软链接，程序内部读取到的链接是自身可执行文件的路径。比如执行
> ```shell 
> root@ecs-295280:~/projects/tinydocker# ls -l  /proc/self/exe
> lrwxrwxrwx 1 root root 0 May 11 17:32 /proc/self/exe -> /usr/bin/ls
> ```
> ls 执行后返回/usr/bin/ls

接着传递给子进程init 参数 ，子进程接手到init参数后将会用 后续的 可执行文件 程序 调用exec覆盖当前进程。所以可以看到 用init 参数启动的进程，是新的命名空间内的第一个进程，后续用exec系统调用，将覆盖这个进程的堆栈，内存空间等信息，从而让init 后面的可执行文件变成命名空间内的第一个进程。

我们运行这段程序时便可以这样运行。
```shell
root@ecs-295280:~/projects/tinydocker# ./tinydocker run /bin/ls
go.mod	main.go  ReadMe.md  tinydocker
init proc end /root/projects/tinydocker/tinydocker
root@ecs-295280:~/projects/tinydocker# ./tinydocker run /bin/sh
#
#
#
#
#
````
可以看到run 后面接着 执行/bin/ls 成功输出了当前目录下的文件，执行 /bin/sh后启动了一个sh进程以便于我们同控制台进行交互，这得益于我们 在cmd.Run 启动新进程前 将标准输入输出赋值给了cmd的标准输入输出参数。


不过可以看到 输出的目录还是主机上的目录，并没有达到隔离的效果，这是因为即使声明了创建新进程时在新的命名空间内部，但是因为没有重新挂载相关目录，新的mnt namespace依然是继承自主机的mnt namespace，所以在没有重新挂载的前提下，新的mnt namespace下看到的目录和主机的是一样的。

所以现在让我们来重新挂载下目录。

## 为程序重新挂载根文件系统

首先要明白寻找文件系统地址的原理，当程序在寻找地址时，会首先判断地址时相对地址还是绝对地址，如果是相对地址则会从进程当前的地址开始寻找，如果是以‘/’ 开头，则说明是绝对地址，那么将会从根路径开始寻找，根路径涉及到两个点，一个是mnt namespace的根路径，一个是进程自身的根路径，比如进程将自身根路径设置为/home  那么进程自身在寻找/lanpangzi 时，实际是从 /home/lanpangzi 开始寻找。

现在来看看替换进程能够看到的文件范围时涉及的两种方式，这两种方式也是和刚才提到的根路径涉及的两个点有关。

### chroot 替换方式

首先是chroot的方式，使用chroot可以替换进程自身的根目录，这样进程自身能够寻找到的范围就变到了设置的根目录下。

不过在看具体的代码前，我们还得有这么一个目录，到时候让进程把这个目录设置为其根目录，这个目录下的下文件通常会用linux的根目录文件系统(rootfs)填充，可以从这个[网址上下载](https://mirrors.tuna.tsinghua.edu.cn/ubuntu-cdimage/ubuntu-base/releases/16.04.6/release) 。

之后我们便可以用chroot 替换程序的根目录了。

```golang
syscall.Chroot("./ubuntu-base-16.04.6-base-amd64")
syscall.Chdir("/")
```

我的程序目录如下
```shell
(base) ➜  tinydocker git:(main) ✗ tree -L 1
.
├── ReadMe.md
├── go.mod
├── main.go
├── ubuntu-base-16.04.6-base-amd64
└── ubuntu-base-16.04.6-base-amd64.tar.gz
```

注意为什么在更换了进程根目录后为啥还有个syscall.Chdir 切换进程当前工作目录的系统调用，因为即使切换了进程的根目录，进程还是在tinydocker 下，由于 相对地址的寻址不会寻找根目录，所以如果此时进程用相对地址'./go.mod'去访问go mod文件还是能访问到，当使用syscall.Chdir后，将会更新进程的当前目录到ubuntu-base-16.04.6-base-amd64下，这样后续进程对文件的访问范围将限制在ubuntu-base-16.04.6-base-amd64目录下。

不过chroot切换 文件系统根目录的方式只能改变该进程能看到的文件范围，并不能改变mnt namespace的根目录，所以替换的并不彻底。

可以用nsenter 命令进入mnt namespace 去看下mnt namespace的目录有哪些验证这一点。

首先是我在新进程下，查看根目录下的目录
```shell
root@ecs-295280:~/projects/tinydocker# ./tinydocker run /bin/sh
# ls /
bin  boot  dev	etc  home  lib	lib64  media  mnt  opt	proc  root  run  sbin  srv  sys  tmp  usr  var
```

运行的sh进程pid是 184295 ，然后我再在主机上用nsenter 命令 进入该进程的命名空间 查看该mnt namespace 根目录下有哪些文件

```shell
root@ecs-295280:~# nsenter -t 184295 -m ls /
bin   CloudResetPwdUpdateAgent	conf  etc   lib    lib64   log	       media  opt   root  sbin	src  swapfile  tmp  var
boot  CloudrResetPwdAgent	dev   home  lib32  libx32  lost+found  mnt    proc  run   snap	srv  sys       usr
root@ecs-295280:~#
```

可以看到两个目录下的文件是不一样的，而nsenter进入mnt namespace 下查看的根目录 文件 则是我主机的mnt namespace上的根目录文件。

关于nsenter 使用的一些参数设置如下:

> ```shell
> nsenter [options] [program [arguments]]
>
>options:
>-t, --target pid：指定被进入命名空间的目标进程的pid
>-m, --mount[=file]：进入mount命令空间。如果指定了file，则进入file的命令空间
>-u, --uts[=file]：进入uts命令空间。如果指定了file，则进入file的命令空间
>-i, --ipc[=file]：进入ipc命令空间。如果指定了file，则进入file的命令空间
>-n, --net[=file]：进入net命令空间。如果指定了file，则进入file的命令空间
>-p, --pid[=file]：进入pid命令空间。如果指定了file，则进入file的命令空间
>-U, --user[=file]：进入user命令空间。如果指定了file，则进入file的命令空间
>-G, --setgid gid：设置运行程序的gid
>-S, --setuid uid：设置运行程序的uid
>-r, --root[=directory]：设置根目录
>-w, --wd[=directory]：设置工作目录
>
>如果没有给出program，则默认执行$SHELL。
> ```


### pivot_root 替换方式
接着来看看pivot root 的方式，使用pivot root 的方式替换挂载目录，可以把mnt 命名空间的根目录也替换掉。

```golang
func PivotRoot(newroot string, putold string) (err error) 
````

pivot root 系统调用有些限制，它要求传入两个目录，首先newroot 和putold目录是要求在同一个挂载命名空间中，putold会存放之前旧的命名空间的文件，并且newroot 和putold处于的挂载命名空间和旧的挂载命名空间不能是同一个。


由于子进程默认会继承父进程的命名空间，所以需要对新命名空间的目录进行重新挂载一下。

```golang
syscall.Mount("", "/", "", syscall.MS_PRIVATE|syscall.MS_REC, "")
syscall.Mount(newroot, newroot, "bind", syscall.MS_BIND|syscall.MS_REC, "")
```
注意在使用mount bind 重新挂载 newroot之前，我先通过一条mount语句声明了挂载命名空间从根目录开始以及其子目录都是私有模式挂载，因为我的主机是ubuntu，默认的init进程已经由systemd进程代替，systemd进程的默认挂载方式是共享挂载，后续会导致pivot_root的调用失败，声明为私有模式挂载后，再调用mount bind对newroot目录重新挂载，那么newroot目录就会被单独挂载到新命名空间了。

此时我再在主机上把程序启动起来，然后运行nsenter 命令进入进程的mnt namepspace去查看根目录就发现它和主机上面的根目录不一样了。以下是相关命令。

将程序启动起来
```shell
root@ecs-295280:~# cd projects/tinydocker/
root@ecs-295280:~/projects/tinydocker# ./tinydocker run /bin/sh
#
```

查看程序的进程号
```shell
root@ecs-295280:~# ps -ef | grep /bin/sh
root      186426  186410  0 10:40 pts/0    00:00:00 ./tinydocker run /bin/sh
root      186430  186426  0 10:40 pts/0    00:00:00 /bin/sh
root      186534  186506  0 10:45 pts/1    00:00:00 grep --color=auto /bin/sh
```

其中186430是 我启动的进程，进入该进程的mnt namespace 去执行 ls命令查看根目录

```shell
root@ecs-295280:~# nsenter -t 186430 -m /bin/ls /
bin  boot  dev	etc  home  lib	lib64  media  mnt  opt	proc  root  run  sbin  srv  sys  tmp  usr  var
root@ecs-295280:~#
```

发现mnt namepsace的根目录已经被新的newroot下的rootfs替换掉了，已经和主机的根目录不同了。

## 为程序挂载proc文件系统

看到这里，还没有结束，我们刚刚仅仅把系统的根文件系统替换掉了,不过这个时候，如果你执行top命令会发现它提示你错误。

```shell
# top
Error, do this: mount -t proc proc /proc
```

这是由于top命令默认会从/proc 路径去读取内核的进程信息，而替换了根文件系统后，/proc下还没有挂载procfs，所以需要重新挂载下。

> 🧚🏻🧚🏻🧚‍♀️ procfs是一个内存文件系统，当用mount 挂载proc类型的文件系统时，内核会从当前进程的pid namespace下找到该pid namespace下的所有进程，然后将进程的各种信息通过访问 /proc目录的方式暴露出来。这样再访问/proc目录时，就能访问到该pid namespace下的所有进程信息了。

添加上挂载proc文件系统的代码。
```shell
defaultMountFlags := syscall.MS_NOEXEC | syscall.MS_NOSUID | syscall.MS_NODEV
syscall.Mount("proc", "/proc", "proc", uintptr(defaultMountFlags), "")
```

再执行top命令.

```shell
top - 03:04:51 up 14 days, 10:23,  0 users,  load average: 0.00, 0.00, 0.00
Tasks:   2 total,   1 running,   1 sleeping,   0 stopped,   0 zombie
%Cpu(s):  0.0 us,  0.0 sy,  0.0 ni, 99.7 id,  0.3 wa,  0.0 hi,  0.0 si,  0.0 st
KiB Mem :  2030524 total,   193608 free,   159664 used,  1677252 buff/cache
KiB Swap:        0 total,        0 free,        0 used.  1623628 avail Mem

    PID USER      PR  NI    VIRT    RES    SHR S %CPU %MEM     TIME+ COMMAND
      1 root      20   0    4500    752    684 S  0.0  0.0   0:00.00 sh
      4 root      20   0   36528   3044   2640 R  0.0  0.1   0:00.00 top
```

可以看到当前我的pid namespace下有两个进程，一个是sh进程，一个是top进程，而sh由于是该命名空间下运行的第一个进程，所以进程号为1。

这样便完成了在新命名空间内部运行一个进程，并且将进程的文件系统和主机的文件系统进行了隔离。不过隔离仅仅做到这一步还不算完，回忆下，当我们用docker启动一个进程时，是不是可以用同一份镜像启动多个容器，类比下现在的实现，你会发现，如果用一份rootfs来启动多个进程，那么多个进程最后改变的将会是同一个rootfs下的文件，这样将达不到将文件系统隔离的目的。

所以在下面一讲，我将演示下如何用内核联合文件系统的特质，达到一分镜像多次运行的效果。




