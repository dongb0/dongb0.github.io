---
layout: post
title: "Linux command (2) -- service and systemctl"
subtitle: 
author: "Dongbo"
header-style: text
tags:
  - linux
  - note
  - cmd
---

刚接触linux的时候，要用什么命令都是现查，碰到好几个博客重启服务时分别用 `service ufw restart` 和 `systemctl restart ufw` 这样的命令。当时只知道这两种使用方式效果一样，没有深究，现在是时候填一下坑了（Ubuntu 20.04)。

### service 

  `man service` 我们可以看到手册对该命令的描述： run a System V init script

  能看出来 `service` 命令就是调用某个init脚本来启动/停止应用程序或系统服务的，这些脚本都放在`/etc/init.d`目录下。我简单理解为系统启动时将有一个初始化程序扫描该目录下的脚本，将目录下所有的服务初始化并启动为守护进程。

  ![img](/img/in-post/post-linux-service/content-of-init-d.png) 

  可以看下ufw里的内容

    #!/bin/sh

    ### BEGIN INIT INFO
    # Provides:          ufw
    # Required-Start:    $local_fs
    # Required-Stop:     $local_fs
    # Default-Start:     S
    # Default-Stop:      1
    # Short-Description: start firewall
    # Description: Start ufw firewall
    ### END INIT INFO

    set -e

    PATH="/sbin:/bin"

    [ -d /lib/ufw ] || exit 0

    . /lib/lsb/init-functions

    for s in "/lib/ufw/ufw-init-functions" "/etc/ufw/ufw.conf" "/etc/default/ufw" ; do
        if [ -s "$s" ]; then
            . "$s"
        else
            log_failure_msg "Could not find $s (aborting)"
            exit 1
        fi
    done

    error=0
    case "$1" in
    start)
        if [ "$ENABLED" = "yes" ] || [ "$ENABLED" = "YES" ]; then
            log_action_begin_msg "Starting firewall:" "ufw"
            output=`ufw_start` || error="$?"
            if [ "$error" = "0" ]; then
                log_action_cont_msg "Setting kernel variables ($IPT_SYSCTL)"
            fi
            if [ ! -z "$output" ]; then
                echo "$output" | while read line ; do
                    log_action_cont_msg "$line"
                done
            fi
        else
            log_action_begin_msg "Skip starting firewall:" "ufw (not enabled)"
        fi
        log_action_end_msg $error
        exit $error
        ;;
    stop)
        if [ "$ENABLED" = "yes" ] || [ "$ENABLED" = "YES" ]; then
            log_action_begin_msg "Stopping firewall:" "ufw"
            output=`ufw_stop` || error="$?"
            if [ ! -z "$output" ]; then
                log_action_cont_msg "$output"
            fi
        else
            log_action_begin_msg "Skip stopping firewall:" "ufw (not enabled)"
        fi
        log_action_end_msg $error
        exit $error
        ;;
    restart|force-reload)
        log_action_begin_msg "Reloading firewall:" "ufw"
        output=`ufw_reload` || error="$?"
        if [ ! -z "$output" ]; then
            log_action_cont_msg "$output"
        fi
        log_action_end_msg $error
        exit $error
        ;;
    status)
        output=`ufw_status` || error="$?"
        if [ ! -z "$output" ]; then
            log_action_cont_msg "$output"
        fi
        log_action_end_msg $error
        exit $error
        ;;
    *)
        echo "Usage: /etc/init.d/ufw {start|stop|restart|force-reload|status}"
        exit 1
        ;;
    esac

    exit 0

  可以看到`start`命令是通过调用`/lib/ufw/ufw-init-functions`中的 `ufw_start()` 函数来完成启动的，其他命令也在该目录下有对应的函数。所以redis的开机自启动也是通过自行编写脚本并放到该目录下完成的。（话说redis现在还没有加个自动启动的脚本吗，大家都得自己上网复制一段代码然后手动塞到这个目录，也太麻烦了）

  这里提到的 `System V`，其实是来自于 Unix 的某个版本 System V，使用 `/etc/init.d` 的脚本进行系统初始化就是这个系统最先采用的，linux 延续了下来。其他没有太多要说的，基本上用法就是`service program-name {start|stop|restart|status}`，具体程序的不同用法自行跑一下就知道怎么用了。
  

### systemctl

  按惯例我们看一下man手册怎么说： Control the sytemd system and service manager。这里的 `systemd` 指的是 linux 的service manager，是系统启动时第一个运行的进程（是不是可以当作内核的核心？）；systemctl 命令是用来控制 systemd的，同时也保持了对 SysV 的兼容性。可以简单理解为 systemd 是 system V 的升级版，这部分了解不多，贴一个随手搜到的[帖子](https://danielmiessler.com/study/the-difference-between-system-v-and-systemd/)吧。

  总的来说使用 `systemctl` 的话，命令的用法只是变为 `systemctl {start|stop|restart|status} service-name` 而已，如果只想了解用法的区别的话，到这就差不多了。不过偶尔我们会看到 `systemctl status ufw.service` 这个 `.service` 又是怎么回事呢？这就要再介绍一下 systemctl 了。

  systemd 里引入了 Unit 和 Unit file的概念[^1]  [^2]。 Unit 可以看作 sytemd 可以操控的所有资源，比如硬件、socket、进程，甚至包括挂载点（mount point)、交换区、计时器等，而 Unit file 就是他们的启动或者管理文件。unit file按优先级从高到低存放在 `/etc/systemd/system`、`/run/systemd/system`、`/lib/systemd/system` 路径下，默认的 Unit 安装路径是在 /lib 下。Unit的类型可以根据 Unit File 的后缀判断，比如 ufw 的 Unit File 是 `ufw.service`，表明这是一个 service；其他的还有 `.socket` `.mount` `.device` 等，更详细的可以看下这个[博客](https://www.digitalocean.com/community/tutorials/understanding-systemd-units-and-Unit-files)。所以有些时候我们看到的带后缀的 systemctl 命令，就是在直接使用用 Unit File 的全名；如果没带后缀，则会在上述目录下逐个搜索是否存在对应的 Unit File。


        [Unit]
        Description=Uncomplicated firewall
        Documentation=man:ufw(8)
        DefaultDependencies=no
        Before=network.target
        
        [Service]
        Type=oneshot
        RemainAfterExit=yes
        ExecStart=/lib/ufw/ufw-init start quiet
        ExecStop=/lib/ufw/ufw-init stop
        
        [Install]
        WantedBy=multi-user.target


  Unit 部分下可以有 Requires 和 Wants 字段来指定 Unit 的依赖关系，在启动本 Unit 之前会先启动依赖中指定的 Units。不过 Requires 字段需要依赖服务全部成功启动，才会继续启动本服务；而 Wants 不管启动是否成功都会继续启动。Install 部分中的 WantedBy 字段也是类似的功能，该字段指定 Unit 在启动时，还会在 `/etc/systemd/system` 下创建一个 `unit-file.prefix.wants`。

  基本上我想了解的关于 service 和 systemctl 命令的内容就这么多了。

  ------------------------------------

  [^1]: [systemd unit](https://www.freedesktop.org/software/systemd/man/systemd.Unit.html)
  [^2]: [systemd syntax](https://www.freedesktop.org/software/systemd/man/systemd.syntax.html#) 


