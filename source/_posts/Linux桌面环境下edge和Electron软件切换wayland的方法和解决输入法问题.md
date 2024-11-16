---
title: Linux桌面环境下edge和Electron软件切换wayland的方法和解决输入法问题
date: 2024-11-16 12:44:52
tags: 
    - Linux
    - Wayland
    - 折腾
abbrlink: wayland-input-method
---
本篇博客是对过去一段时间折腾Linux桌面环境时解决切换wayland和输入法问题的一些汇总和吐槽，之后如果换了其他的发行版，还能想起来怎么解决。本文的解决方案仅保证在2024年11月16日这个时间点对作者而的环境有效，不保证以后或其他环境下仍然有效。

本博客全文在KDE plasma 6 + wayland环境下运行的VSCode中写成，要是使用GNOME，那可能只能在X11环境下写了。

## 笔者的环境
![fastfetch](env.jpg)

## Edge浏览器

### 开启Wayland

Chrome浏览器可以直接通过`chrome://flags`里设置`Preferred Ozone Platform`为`Wayland`，但是Edge里没有这个flags，所以需要添加命令行参数来开启：
```shell
microsoft-edge-stable --ozone-platform=wayland
```
在KDE plasma中，你可以通过右击开始菜单里的Edge -> 编辑应用程序 -> 应用程序 -> 参数 来给图形界面启动的Edge增加上参数。

### 输入法支持

本文假定你已经安装并配置好了fcitx5输入法，如果你还没有安装，可以参考[官网的说明](https://fcitx-im.org/wiki/Fcitx_5/zh-cn)，或者如果是ArchLinux+KDE plasma，也可以参考[Arch简明指南](https://arch.icekylin.online/guide/rookie/desktop-env-and-app.html#_10-%E5%AE%89%E8%A3%85%E8%BE%93%E5%85%A5%E6%B3%95)。

有三种方法可以开启edge在wayland下的输入法：
1. 使用text-input-v3 (适用于Edge 129版本以上，推荐)
2. 使用gtk4 (KDE plasma下可能会有错位问题，GNOME可以通过插件解决这个问题)
3. 使用text-input-v1 (适用于KDE plasma/Kwin 5.27以上，GNOME不支持)

#### 使用text-input-v3
如果你的Edge版本是129以上，直接添加两个参数来开启text-input-v3，它相比过去的方式效果要好：
```shell
microsoft-edge-stable --enable-wayland-ime --wayland-text-input-version=3
```

这个方法也同样适用于129版本以后的Chrome浏览器。最终你的启动参数应该类似这样：
```shell
microsoft-edge-stable --ozone-platform=wayland --enable-wayland-ime --wayland-text-input-version=3
```

![添加了所有启动参数的Edge](edge.jpg)

#### 使用gtk4
如果你的Edge版本在129以下，可以通过修改gtk版本的方式来通过gtk支持输入法：
```shell
microsoft-edge-stable --gtk-version=4
```

最终你的启动参数应该类似这样：
```shell
microsoft-edge-stable --ozone-platform=wayland --gtk-version=4
```
这个方式在一些条件下可能会出现候选词错位的问题，如果是GNOME可以通过kimpanel插件来解决。

#### 使用text-input-v1
通过只添加`--enable-wayland-ime`的方法来开启对text-input-v1的支持，这个方法适用于KDE/kwin5.27以上，而GNOME的mutter至今不支持text-input-v1，所以无法使用这个方法。

最终你的启动参数应该类似这样：
```shell
microsoft-edge-stable --ozone-platform=wayland --enable-wayland-ime
```

## Electron

### 开启wayland

和浏览器一样，electron的软件也可以通过添加`--ozone-platform=wayland`的方式来切换到wayland。如果你觉得一个一个添加太麻烦的话，也可以通过环境变量的方式来让所有的electron版本28以上的软件都切换到wayland：
```
ELECTRON_OZONE_PLATFORM_HINT=wayland
```

### 输入法支持

对于Edge支持的开启输入法的三种方法中，Electron仅支持方法3，即通过text-input-v1的方式支持输入法。添加启动参数```--enable-wayland-ime```即可。但是由于GNOME的mutter合成器截至目前**即使已经有开发者提了Merge Request的情况下，仍然不支持text-input-v1**[^1]，所以GNOME还是老老实实退回x11来运行吧。

<p style="color: rgba(255,255,255,0.1); font-size: 14px"><del>然而GNOME上的xwayland对非整数倍缩放和多屏不同缩放都有问题，退回x11也只是换了一种难受方式</del></p>

如果你一定要在GNOME下的Electron软件里使用输入法的话，可以尝试安装用户自己增加了text-input-v1的mutter，例如Arch下的[mutter-text-input-v1](https://aur.archlinux.org/packages/mutter-text-input-v1)，Fedora下的[mutter-im-fix](https://copr.fedorainfracloud.org/coprs/kanru/mutter-im-fixes/)。

注：使用第三方包有风险，请确保你有相关经验再使用

## 参考
[^1]: [wayland/text-input-v1: Implement basic text-input-v1 support](https://gitlab.gnome.org/GNOME/mutter/-/merge_requests/3751)
[^2]: [常见问题 - fcitx](https://fcitx-im.org/wiki/FAQ/zh-hans)
[^3]: [Chrome/Chromium 今日 Wayland 输入法支持现状 | CS Slayer](https://www.csslayer.info/wordpress/fcitx-dev/chrome-state-of-input-method-on-wayland/)
[^4]: [开发输入法是一种妥协的艺术 | CS Slayer](https://www.csslayer.info/wordpress/diary/%E5%BC%80%E5%8F%91%E8%BE%93%E5%85%A5%E6%B3%95%E6%98%AF%E4%B8%80%E7%A7%8D%E5%A6%A5%E5%8D%8F%E7%9A%84%E8%89%BA%E6%9C%AF/)
[^5]: [Wayland 中文输入法 fcitx5 无法输入中文 - 阿冰的小屋](https://blog.coldbin.top/posts/ac56d16/)
[^6]: [Microsoft Edge浏览器129现可使用text-input-v3 - 哔哩哔哩](https://www.bilibili.com/opus/979150396325888006)
[^7]: [How to globally set all electron apps to have --enable-features=UseOzonePlatform --ozone-platform=wayland options?](https://unix.stackexchange.com/questions/736187/how-to-globally-set-all-electron-apps-to-have-enable-features-useozoneplatform)