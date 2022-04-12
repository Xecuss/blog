---
title: 小米AX1800使用MIXBOX搭建射线版本2踩坑记
tags:
  - 网络
  - Linux
abbrlink: 12300
date: 2021-05-15 23:05:25
---
在《[0成本实现远程开机](/2021/02/09/0成本实现远程开机/)》这篇文章里提到：路由器本身就是一台小型的电脑，其本身搭载了一个特殊版本的Linux系统。既然是一台小型的电脑，那么就可以在上面搭载各种各样的工具，其中就包括著名的网络工具——射线2。

笔者最近就在AX1800上部署了射线2，这个过程中碰到一些坑。由于笔者既不是计算机专业出身，也没学过嵌入式开发，甚至仅有的一点C语言快要还给老师了，所以在解决这些坑的过程中走了不少弯路。因此将过程记录于此，希望能帮助到有需要的朋友。

> ps: 对解决过程不感兴趣的朋友请直接跳转到最后的[解决方法](#总结-解决方法)

## 安装过程

安装过程这里就不再多提了，按照《[0成本实现远程开机](/2021/02/09/0成本实现远程开机/)》里提到的，你的AX1800需要开启SSH并安装MIXBOX，然后可以直接通过MIXBOX安装阴影袜子，这个MIXBOX里安装的阴影袜子里面自带了射线2，只需要添加配置的时候选择射线2的配置即可。

## 问题

### etc分区太小
一上来首先碰到的问题是etc分区空间不够，AX1800的etc分区只有11+ M，安装时会出现```No space left on device```的错误。于是使用```df -h```命令查看了一下剩余空间。重新安装MIXBOX，将其转移到/tmp分区，随后就能顺利安装了。

> ps: 经笔者学嵌入式的朋友提醒，这个/tmp分区该是个临时分区，重启后内容就会消失

### 启用插件后其状态仍然为未启动
在启用插件之后可以看到其状态仍然为未启动，但是实际上笔者验证射线2的进程已经在运行，iptables的规则也已经生效，所以一开始并没有关注这里，认为这只是个无关紧要的小问题。

但是实际上这个问题和以下所产生的问题都只是两个问题的不同表现而已。

### 仅global模式生效
顺利安装并配置好射线2之后，发现这个射线2在禁止名单模式和通行名单模式下均无法正常生效，只有在global模式下才能生效。而且即使是生效了，也无法进行正常的访问操作，会出现各种各样的证书错误。

经过一番搜索，在[这个帖子](https://www.v2ex.com/t/471828)里了解到，这其实是因为DNS问题造成的。又经过一番搜索，最后在射线2的配置文件的任意门部分增加了sniffing选项，终于global模式下能够进行正常的访问操作了。

然而没高兴多久，随即就发现了另一个问题。

### 配置文件重置
笔者在运行射线2的过程中发现，在修改射线2配置过一段时间之后，配置文件总是会重置回之前的状态并且射线2会被重启。因为解决DNS问题需要在配置文件中添加sniffing选项，这样一来就导致每过一段时间就会失效。

百思不得其解下，笔者只好打开其配置脚本试图通过源码来解决问题。

>ps: 通过MIXBOX安装的插件，其配置脚本位于： ${安装位置}/mixbox/apps/${插件名}/scripts/${插件名}.sh

由于表现为每隔一段时间重置，那么应该和一些定时任务有关，所以笔者首先注意到了start() 中的
```shell
#添加定时更新规则
write_cron_job
```
查看其定义可以看到它使用cru命令添加了三个定时任务：
```shell
# 代码可能有删改，下同
write_cron_job() {
  cru a "${appname}"_rule "20 5 * * * ${mbroot}/apps/${appname}/scripts/**_rule_update.sh"
  cru a "${appname}"_online "0 */6 * * * ${mbroot}/apps/${appname}/scripts/**_online_update.sh"
  cru a "${appname}" "0 6 * * * ${mbroot}/apps/${appname}/scripts/${appname}.sh restart"
}
```
查阅[cru命令的使用方法](https://koolshare.cn/thread-170536-1-1.html)后发现，这三个任务一个在每天5点20分运行，一个每6小时运行一次，还有一个在每天6运行，显然不可能每几分钟进行重置。

于是笔者又列出了所有cron的定时任务，发现其中有一个叫monitor的任务，对其内容进行base64解码后得到：```*/10 * * * * /tmp/mixbox/scripts/monitor.sh```，这个任务每10分钟运行一次，应该就是它了。随后查阅了一下这个monitor.sh，发现这个是MIXBOX的管理任务，其中和我们的插件比较相关的是detect_apps部分
```shell
detect_apps() {

        applist installed -n| while read line
        do
                ${mbroot}/apps/${line}/scripts/${line}.sh status
                result1="$(mbdb get ${line}.main.enable)"
                result2="$(mbdb get ${line}.main.status | cut -d'|' -f2)"
                if [ "$result1" = '1' ] && [ "$result2" = '0' ]; then
                        ${mbroot}/apps/${line}/scripts/${line}.sh restart
                fi
        done

}
```
这段脚本主要功能是从数据库(mbdb)中获取插件的enable和status，如果enable=1且status=0（启用了却未处于运行状态），则restart插件。

这样一来就十分明了了，我们碰到的第一个问题就是，启用阴影袜子插件之后在MIXBOX查看其状态仍然为未启动。所以MIXBOX会反复尝试拉起插件进程，而每次拉起进程都会重新生成配置文件，也就表现成了配置文件被不断重置。

### 重新研究显示为未启动的问题
既然如此，就得排查一下为什么状态仍然显示为未启动了。我们查看上面detect_apps的代码，不难发现插件的状态是通过配置脚本的status参数来刷新的，于是笔者看了一下配置脚本中的status部分。
```shell
status() {

  result1=$(pssh | grep -v status | grep -c "${appname}")

  result2=$(iptables -t nat -S | grep ****)
  process_count=3
  [ "$ssgena" == '1' ] && ssgflag=", Game Node: $ssgid($ssg_mode)"
  if [ "$kcp_enable" == '1' ]; then
    ssgflag="$ssgflag, kcptun($ss_kcp_node):"
    let "process_count++"
        [ "$(pssh | grep -c kcptun)" -eq 1 ] && ssgflag="$ssgflag 运行中" || ssgflag="$ssgflag 未运行"
  fi

  if [ "$proxy_type" == "射线2" ]; then
    let "process_count--"
  fi

  if [ "$result1" -ge $process_count ]; then
    if [ -n "$result2" ]; then
      status="Running Node: $id($ss_mode)$ssgflag|1"
    else
      status="链路异常，可以尝试重启服务！|0"
    fi
  else
    status="未运行|0"
  fi
  mbdb set $appname.main.status="$status"

}
```
阅读脚本可以知道，如果要判定当前插件正在运行，需要两个条件：
1. 相关iptables规则已经生效
2. 插件运行的进程必须大于等于process_count这个变量

然后我们还可以知道，当我们实际运行的是射线2的时候，这个process_count会减去1，对于笔者的配置来说最终需要检测到两个进程正在运行。其中一个显然是射线2自己，那么另一个进程会是什么呢？

在配置脚本中进行搜索以及联系到前面的dns问题，笔者得到了结论：另一个进程是dns2socks

### dns2socks无法启动
这个[dns2socks](https://sourceforge.net/projects/dns2socks/)就是帮助我们解决前面那个dns问题的工具。笔者测试了一下运行插件自带的dns2socks工具，发现没有任何回显，也无法让进程常驻后台运行。推测可能是兼容问题，那么只好自己动手来编译一个可用的dns2socks工具了。

笔者首先尝试了一下[小米路由器插件开发文档](https://dev.mi.com/docs/routerplugin/user_guide/)里的交叉编译toolchain，然而编译出来并不能使用，使用readelf可以发现，这个结果里需要的lib在路由器里并没有，应该是toolchain的问题。
![并不包含该动态链接库](toolchain1.jpg)

小米官方似乎并没有提供AX1800的相关toolchain。不过通过```cat /etc/*release```可以看到，AX1800的路由器固件是基于OpenWRT18.06的，笔者便去OpenWRT官网下载OpenWRT 18.06的toolchain。官网上并没有ipq60xx的toolchain，最后选择了指令集相同的ipq40xx的toolchain。

交叉编译的过程不再赘述，编译好之后readelf，看到了确实存在于AX1800的动态链接库。上传，尝试运行
![运行dns2socks](dns2socks.png)

终于看到了回显，使用这个dns2socks替换掉插件自带的版本，重新启用插件，终于，MIXBOX插件status里显示为正在运行了。

### 禁止名单模式不可使用
然而，即使dns2socks已经启动，禁止名单模式仍然不能使用。查看dns2socks日志，发现并没有任何dns请求通过。

重新查阅插件配置脚本，发现dns2socks并没有配置在53端口上，而是通过dnsmasq在一些特定情况下将请求转发过来。然而配置脚本默认将dnsmasq规则写到了/tmp/etc/dnsmasq.d/下面，而我们的dnsmasq实际使用的配置文件是在/etc/dnsmasq.d/下，所以创建相关文件的软链即可解决这个问题。
```shell
ln -s /tmp/mbtmp/wblist.conf /etc/dnsmasq.d/wblist.conf
ln -s /tmp/mbtmp/功夫王列表.conf /etc/dnsmasq.d/功夫王列表_ipset.conf
```
>ps：如果自己添加软链，请关闭后记得删掉软链并重启dnsmasq，否则会无法上网，最好是直接修改原脚本，把其中的/tmp/etc/dnsmasq.d全替换成/etc/dnsmasq.d


至此，禁止名单模式终于能够正常使用了。

## 总结 解决方法
1. 使用MIXBOX停用正在运行的阴影袜子插件
2. 下载[需要的文件](https://1drv.ms/u/s!AoGA8iOeRdabk-80RCGoMOcx3JhL3A?e=CTIKV3)并解压，密码: XROSS THE XOUL
3. 上传并替换 ```${安装位置}/mixbox/apps/${插件名}/bin/dns2socks```为这个下载文件里的dns2socks（仅适用于AX1800，其他路由器请勿尝试），替换后使用```chmod 777 dns2socks```修改其权限
4. 上传并替换 ```${安装位置}/mixbox/apps/${插件名}/scripts/${插件名}.sh```为这个下载文件里的版本（仅适用于AX1800，其他路由器请勿尝试）
5. 使用MIXBOX重新启用阴影袜子插件

注意：射线2插件占用空间较大，所以只能安装在/tmp下，这就导致了每次重启后之前安装的东西会消失，需要重新配置。