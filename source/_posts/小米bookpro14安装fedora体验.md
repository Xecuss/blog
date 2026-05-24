---
title: 小米 book pro 14安装Fedora体验
abbrlink: mibookpro14-fedora
date: 2026-05-23 18:04:30
tags:
    - Linux
    - 折腾
---
最近看了下评测，intel家最新的panther lake架构的Core Ultra x7 358H有点厉害的啊，不仅待机功耗可以压到非常低，而且功耗上去了打游戏也非常厉害。

搭载了358H的小米book pro 14也非常强劲，不仅机身重量只有1.08kg非常的轻薄，而且它那块3kOLED屏显示效果也非常的棒。本来笔者兴趣没有那么大的，直到去小米之家摸了一下，瞬间被这个重量惊讶到了。而且后来发现一众超轻薄本的价格确实比它高了很多，所以等到有货就激情下单了一台。
 
有了这样强大的硬件，大家都说，现在拖累小米笔记本的已经是Windows系统本身了。不过还好它还有一个空的2280的硬盘位，笔者手上还正好有一块闲置的1TB 2280硬盘，刚好拿来装Linux，解决它这最后的弱点。

## 选择发行版
笔者选择的发行版是Fedora Workstation，为什么这次没有选择KDE，主要是之前的两个痛点得以解决：
1. 由于全面转向wayland，现在大部分Electron软件都已支持text-input-v3，这也就意味着不会再有输入法问题
2. 由于米book的屏幕更大，可以直接使用200%缩放无须担心缩放问题

另外GNOME的手势操作也非常顺手，非常适合米book这块超大触控板，所以笔者就决定回GNOME看看。

## 解决键盘和花屏问题

安装好Fedora之后会发现，最首要的问题是：
1. 键盘没响应
2. 屏幕花屏

这些都可以通过添加内核参数解决，笔者在这里先提供一条直接设置所有内核参数的命令，然后再分步解释：
```bash
sudo grubby --update-kernel=ALL --args="i8042.nopop=1 i8042.dumbkbd=1 xe.force_probe=b081 i915.force_probe=\!b081 xe.enable_psr=1"
```

### 键盘问题

对于键盘问题，我们可以先外接键盘（注：如果没有外接键盘，可以重启电脑，在grub菜单出现的时候按e，然后把内核参数添加上去，这样可以在这次启动使用键盘），然后通过grubby来设置启动参数：

```bash
sudo grubby --update-kernel=ALL --args="i8042.nopop=1 i8042.dumbkbd=1"
```

附上gemini 3给出的解释：

> 键盘与触控板相关参数 (i8042 子系统)
> i8042 是负责处理传统的 PS/2 键盘和鼠标（触控板）的内核模块。即使是现代笔记本，其内置键盘通常也是通过内部模拟这个老式接口来连接的。
>
> * i8042.nopnp=1
> 效果： 告诉内核不要通过 ACPI/PNP（即插即用）机制来检测键盘控制器，而是强制直接扫描传统的 I/O 硬件端口。
> 作用： 很多新笔记本的 ACPI 表（主板固件的一部分）在 Linux 下存在兼容性问题，导致内核检测不到键盘。加了这个参数可以绕过有缺陷的 ACPI，强制让键盘工作。
> * i8042.dumbkbd=1
> 效果： 开启“傻瓜键盘”模式。内核将不会尝试从键盘控制器读取硬件状态，而是直接假定它存在并跳过某些初始化检查。
> 作用： 解决某些笔记本在启动时键盘控制器卡死，导致开机极其缓慢或者键盘/触控板完全无法响应的问题。

### 显示花屏

对于显示花屏，主要是面板自刷新（PSR）的锅，笔者在之前用7940HS初期也碰到过这个问题，我们把默认的PSR2降低为PSR1就可以解决这个问题：

```bash
xe.force_probe=b081 i915.force_probe=\!b081 xe.enable_psr=1
```

同样附上gemini 3给出的解释：

> Intel 以前一直使用 i915 作为官方内核显卡驱动。但对于较新的显卡（如 Xe 架构、Arc 独立显卡或酷睿 Ultra 的核显），Intel 开发了全新的 xe 驱动。这两个参数通常成对出现，用于强制用新驱动替换老驱动。
> 其中的 b081 指的是特定 Intel 显卡的 PCI ID。
> 
> * i915.force_probe=!b081
> 效果： 注意前面的感叹号 !。它的意思是如果检测到 PCI ID 为 b081 的显卡，禁止旧版 i915 驱动去加载和接管它。
> * xe.force_probe=b081
> 效果： 强制让全新的 xe 显卡驱动去接管 PCI ID 为 b081 的显卡，即使该驱动对此显卡的支持还在实验性阶段。
> 作用： 大概率是因为 i915 驱动在这个的新硬件上表现极其糟糕（例如无法启动图形界面、严重卡顿等），所以用户通过这两个参数“屏蔽旧驱动，强上新驱动”。
> * xe.enable_psr=0
> 效果： 在 xe 显卡驱动中禁用 PSR（Panel Self Refresh，面板自刷新）功能。
> 作用： PSR 是一种屏幕省电技术：当屏幕画面静止不动时，显卡停止发送数据，让显示器用自己的缓存维持画面。然而，这项技术在 Linux 下经常存在严重的各种 Bug。禁用它（设为 0）是解决屏幕闪烁、花屏、局部画面卡死、或者鼠标移动有拖影的最常见且最有效的办法。 代价是会稍微增加一点点耗电量。

注：这里的解释是针对参考文档中设置的完全关闭psr的参数，后来了解到psr除了关闭，也可以设为1。笔者经过测试发现降级为PSR1也可以解决花屏，没必要完全关闭。

## 解决声音问题

对于没有声音的问题解决起来比上面两个简单多了，7.0内核已经修复了这个问题，只需要运行```sudo dnf update```更新到最新内核即可，这个问题只在默认的6.x内核上出现。

## 启用指纹

参考[这个博客](https://meixg.cn/2026/03/25/xiaomi-book-pro-14-omarchy/)内容，自己编译指纹模块就可以了，之后这个模块更新的话应该默认就支持指纹了。Fedora和博客里的内容稍微有点区别，附上测试可用的步骤：

### 第 0 步：确认硬件与现状
就像文章里说的，先确认一下你的设备 ID 和当前状态：
```bash
# 查看是否有 Goodix 设备，确认 ID 是 27c6:6890
lsusb | grep -i goodix

# 检查当前系统是否确实无法识别
fprintd-list $USER
```
如果输出显示 `No devices available`，就可以开始下面的步骤了。

---

### 第 1 步：安装构建依赖
在 Fedora 中，安装编译所需依赖非常简单。我们可以利用 `dnf builddep` 命令直接拉取官方打包 `libfprint` 时用到的所有依赖环境：

```bash
# 确保系统包列表是最新的
sudo dnf update

# 安装 git 和基础构建工具 (meson, ninja, gcc)
sudo dnf install git meson ninja-build gcc valgrind

# 自动安装编译 libfprint 所需的所有系统依赖（这一步是 Fedora 的神器）
sudo dnf builddep libfprint
```

---

### 第 2 步：拉取上游 `libfprint` 源码
从 Freedesktop 的官方 GitLab 仓库克隆最新的开发版源码：

```bash
# 回到用户主目录或你喜欢放源码的地方
cd ~

# 克隆源码（国内如果拉取较慢，请自备网络代理环境）
git clone https://gitlab.freedesktop.org/libfprint/libfprint.git

# 进入源码目录
cd libfprint
```

---

### 第 3 步：编译并安装到系统
这里是**最关键的一步**。Fedora 是典型的红帽系发行版，它的 64 位库目录在 `/usr/lib64`，而不是 `/usr/lib`。我们需要在配置编译环境时明确指定路径，这样才能正确替换系统的旧库。

```bash
# 1. 使用 meson 配置构建目录 (指定安装路径覆盖系统原版)
meson setup builddir --prefix=/usr --libdir=/usr/lib64 --sysconfdir=/etc

# 2. 开始编译
meson compile -C builddir

# 3. 将编译好的文件安装到系统中 (需要 root 权限)
sudo meson install -C builddir
```

> **补充操作**：安装完成后，新的硬件识别规则（udev rules）也一起被安装了，我们需要重载一下规则，让系统立刻识别 USB 硬件：
> ```bash
> sudo udevadm control --reload-rules
> sudo udevadm trigger
> ```

---

### 第 4 步：重启 `fprintd` 服务
底层驱动已经更新，重启上层指纹服务让其生效：

```bash
# 重启服务
sudo systemctl restart fprintd

# 查看服务状态，确保它处于 active (running) 没有报错
systemctl status fprintd
```

再次输入 `fprintd-list $USER`，这时候应该就能看到你的指纹识别器名字了。

---

### 第 5 步：录入指纹
你可以直接在命令行录入指纹：
```bash
fprintd-enroll
```
按照终端里的提示，多次将你的右手食指（默认）按在指纹识别器上。如果出现 `Enroll result: enroll-completed` 就说明成功了。

*提示：在桌面环境中，你现在也可以直接打开 **GNOME 设置 -> 系统 -> 用户 -> 指纹登录**，在图形界面里录入，体验会更好。*

## 解决终端行高问题

做完以上的步骤，已经能获得一个基本可用的Fedora环境了，不过底下还有两个优化点，可以让使用的体验更好。

众所周知，GNOME在CJK环境下存在行高问题，不仅会导致你的终端显示被拉长；更糟糕的是，如果你使用像powerlevel10k这样的终端主题，左边三角形会和矩形对不齐，非常的难看。而且这个问题的原因其实早已查明[是由于NotoSans字体导致的](https://gitlab.gnome.org/GNOME/vte/-/work_items/347)，然而两边都不愿意修改，于是就这么僵在这了，没有人在意中日韩用户的使用体验。

不过好在，笔者又在知乎搜到了[大佬的回答](https://www.zhihu.com/question/1938337986841408824/answer/1940895122414892599)，现在通过AI的加持，便可以生成安装修改版的字体，具体来说，AI提供了这样一个python脚本：

```python
#!/usr/bin/env python3
import sys
from fontTools.ttLib import TTCollection

def patch_ttc(input_path, output_path):
    print(f"Loading TrueType Collection (TTC): {input_path}...")
    try:
        collection = TTCollection(input_path)
    except Exception as e:
        print(f"Failed to load TTC: {e}")
        sys.exit(1)
        
    patched_count = 0
    for i, font in enumerate(collection.fonts):
        if 'OS/2' in font:
            os2 = font['OS/2']
            
            # 【关键修复】: 升级 OS/2 表的版本至 4
            if os2.version < 4:
                print(f"Font {i}: Upgrading OS/2 table version from {os2.version} to 4.")
                os2.version = 4
                
            fs_selection = os2.fsSelection
            
            if fs_selection & (1 << 7):
                print(f"Font {i}: USE_TYPO_METRICS is already set.")
            else:
                os2.fsSelection |= (1 << 7)
                print(f"Font {i}: Patched! USE_TYPO_METRICS bit set to 1.")
                patched_count += 1
        else:
            print(f"Font {i}: No OS/2 table found, skipped.")
            
    if patched_count > 0:
        print(f"\nSaving patched collection to {output_path}...")
        collection.save(output_path)
        print("All done!")
    else:
        print("\nNo fonts needed patching.")

if __name__ == "__main__":
    if len(sys.argv) != 3:
        print("Usage: python3 proc-font.py <input.ttc> <output.ttc>")
        sys.exit(1)
    
    patch_ttc(sys.argv[1], sys.argv[2])
```
这个脚本可以处理ttc文件，然后重新打包，我们复制系统默认的NotoSansCJK字体，然后运行此脚本生成修改后的字体文件，并作为用户字体安装，再进入终端，终于正常了：

系统的NotoSansCJK字体位于/usr/share/fonts/google-noto-sans-cjk-vf-fonts/NotoSansCJK-VF.ttc ，我们复制到和脚本同一目录，运行：
```bash
# 需要先安装对应的依赖
sudo dnf install fonttools
# 生成patch字体，如果你先运行了chmod +x proc-font.py，也可以直接执行
python3 proc-font.py NotoSansCJK-VF.ttc Patched-NotoSansCJK-VF.ttc
# 复制到用户字体目录
mv Patched-NotoSansCJK-VF.ttc ~/.local/share/fonts/NotoSansCJK-VF.ttc
# 强制刷新用户级与系统级字体缓存
fc-cache -f -v
```

![可以看到行高正常了，底部powerlevel10k也正常对齐了](terminal.png)

## 开启充电限制

众所周知，锂电池的一大敌人就是过充和过放，但是现在mibook的驱动里自然是没有相关驱动从而无法实现充电限制的。但是在Windows下，可以通过小米电脑管家开启充电限制。那么小米电脑管家是如何实现充电限制的呢？

笔者并不会逆向，不过还好现在有AI，在AI的帮助下，最终通过提取和反编译ACPI表，找到了进行充电限制的wmi调用，并在Fedora上实验通过

![成功设置80%充电限制](charginglimit.png)

> ⚠️ **<font color="red" size="5">免责声明</font>**
> 
> <font color="red">**本博客所述的 WMI 方案存在潜在风险。作者仅在 Xiaomi Book Pro 14 设备上进行过验证，其对其他笔记本电脑（包括相同型号）的适用性与安全性无法保证。使用者应自行评估相关风险。因参照本博客或使用相关脚本导致的任何硬件损坏、数据丢失或系统故障，作者不承担任何责任。请在充分了解风险后再决定是否继续操作。**</font>

## 参考

- [在 Xiaomi Book Pro 14 (2026) 上运行 Omarchy (Arch + Hyprland)](https://meixg.cn/2026/03/25/xiaomi-book-pro-14-omarchy/)
- [Locale-dependent increased line height after upgrade 0.64](https://gitlab.gnome.org/GNOME/vte/-/work_items/347)
- [如何解决 GNOME 中的终端行高不正常的问题？ - 知乎](https://www.zhihu.com/question/1938337986841408824/answer/1940895122414892599)
- [libfprint / libfprint](https://gitlab.freedesktop.org/libfprint/libfprint.git)
- [小米 Xiaomi Book Pro 14 （Ultra X7） Linux 兼容性实测](https://zhul.in/2026/04/30/xiaomi-book-pro-14-2026-on-linux/)

