---
title: 在 Termux 中安装 Arch Linux 并配置 Neovim (LazyVim)
description: 平板秒变轻薄本~妈妈再也不用担心我扛着5公斤笔记本一去不复返了
pubDate: 2025-5-2 # 发表日期，注意格式，可以参考其他文章或config里的date_format
image: /img/projects-bg.jpg # 可选，文章封面图路径，图片放public/image/下
categories:
  - tech
tags:
  - linux
  - neovim
# badge: Pin # 如果想置顶，可以加上
draft: false # false 表示发布，true 表示草稿
---

# 在 Termux 中安装 Arch Linux 并配置 Neovim (LazyVim)

## 0x00 闲言碎语

我用我的三星Tab S7 FE已经一年多了，回想起刚拿起它时候的雄心壮志，那种渴望改变一切，命运在握，Turn My Situation Around的感觉，恍如昨日。我也确实用它度过了我极度充实的2024年春天，有种下水道呆久了回光返照的感觉，这确实是我大学生涯的一个转折，我由此建立了上课认真听课，不玩手机的底线。最近，因为我想用Markdown记笔记来提交到我的GitHub Pages上，所以，我给我的三星TabS7FE买了蓝牙键鼠，极大的利用了三星的DeX功能。如果你现在正需要平板来记笔记，看文献，甚至是coding(GitHub不是也有命令行工具CLI了么)，我的这一套你都可以复刻。

平板：三星Tab S7 FE

比iPad便宜太多了，现在闲鱼基本上二手的一千以内了，甚至有的还带原装键盘，大学四年靠Z-library的电子书代替教材都能回本。而且Spen真的好用，和在纸上写没任何区别。哦，说到这里我的Spen是改装过的不会磨损的中性笔笔头，平板随便贴了个类纸膜。

键盘：罗技MX Keys Mini

这太有的说了，我闲鱼只花了￥180就买到了99新的，因为没有Bolt接收器，日语键位配置，所以卖的很便宜，我第一盲打很熟练了，第二也不需要接收器，就爽快收下了。

鼠标：英菲克pm6

便宜，性价比之王，很好用，除了侧键太软容易误触外也没有明显缺点。

## 0x01：Termux 基础环境准备

首先，先在Pad上装个Termux，推荐用F-Droid装，方便后面装插件改变字体图标和Termux主题。

然后我们需要更新 Termux 的包列表并安装一些必要的工具。

1.  **更新包列表和升级已安装包:**
    打开 Termux，运行以下命令：

    ```bash
    pkg update -y && pkg upgrade -y
    ```

2.  **安装 `proot-distro`:**
    `proot-distro` 是一个脚本，可以帮助我们在 Termux 中轻松管理 Linux 发行版。

    ```bash
    pkg install proot-distro -y
    ```

3.  **安装 `vim`:**

    ```bash
    pkg install vim -y
    ```

4.  **安装 `pulseaudio`:**
    如果你计划在 Arch Linux 环境中使用音频，可能需要安装它。对于纯 Neovim 配置，这不是必需的。我安装这个主要是当时对GUI抱有幻想。我给我的Arch甚至安装了Xfce，配置了VNC，计划用VScode来着，后面经常宕机，因为我的Pad的RAM不到6G，还是Neovim更适合我啊。

    ```bash
    pkg install pulseaudio -y
    ```

## 0x02：安装 Arch Linux

现在，我们使用 `proot-distro` 来安装 Arch Linux。

1.  **查看可用的发行版:**

    ```bash
    proot-distro list
    ```

2.  **安装 Arch Linux:**
    下载 Arch Linux 的 rootfs。

    ```bash
    proot-distro install archlinux
    ```

## 0x03：登录并初始化 Arch Linux 环境

安装完成后，我们需要登录到 Arch Linux 环境并进行一些基础配置。

1.  **登录 Arch Linux:**

    ```bash
    proot-distro login archlinux
    ```

2.  **更新 Arch Linux 包数据库和系统:**

    ```bash
    pacman -Syu
    ```

3.  **安装基础工具 (`sudo`, `vim`):**

    ```bash
    pacman -S sudo vim --noconfirm
    ```

4.  **创建普通用户:**

    ```bash
    useradd -m -g users -G wheel -s /bin/bash <your_username>
    ```

5.  **设置用户密码:**
    为新创建的用户设置密码

    ```bash
    passwd <your_username>
    ```

6.  **配置 `sudo`:**

    ```bash
    visudo
    ```

    ```
    # %wheel ALL=(ALL:ALL) ALL
    ```

    取消注释：

    ```
    %wheel ALL=(ALL:ALL) ALL
    ```

7.  **切换到新用户:**

    ```bash
    su - <your_username>
    ```

## 0x04：安装 Neovim 和相关依赖

1.  **安装 Neovim:**

    ```bash
    sudo pacman -S neovim --noconfirm
    ```

2.  **安装 LazyVim 依赖:**

    ```bash
    sudo pacman -S git base-devel ripgrep fd xsel --noconfirm
    ```

## 0x05：安装 LazyVim

1.  **备份现有 Neovim 配置 (如果存在):**

    ```bash
    # 确保你在你的用户家目录下 (~)
    mv ~/.config/nvim{,.bak}       # 备份旧的 nvim 配置 (如果存在)
    mv ~/.local/share/nvim{,.bak} # 备份旧的 nvim 数据 (如果存在)
    mv ~/.local/state/nvim{,.bak} # 备份旧的 nvim 状态 (如果存在)
    mv ~/.cache/nvim{,.bak}       # 备份旧的 nvim 缓存 (如果存在)
    ```

2.  **克隆 LazyVim Starter 模板:**

    ```bash
    git clone https://github.com/LazyVim/starter ~/.config/nvim
    ```

3.  **移除 .git 目录:**

    ```bash
    rm -rf ~/.config/nvim/.git
    ```

## 0x06：安装 Termux:Styling插件

直接在F-Droid里安装即可，完成后长按Termux任意位置->More->Style->CHOOSE FONT修改字体，推荐DejaVu Sans Mono，因为我喜欢这个字体，UI图标也能正常显示。

## 0x07：启动 Neovim

1. **启动 Neovim:**
   在 Arch Linux 环境中 (以你的普通用户身份)，运行：

   ```bash
   nvim
   ```

## 0x08 结语

OK了，后面需要什么配什么就OK，我第一次尝试用Pad coding，还挺爽的，主要轻便续航久，感觉当个轻薄本用也问题不大。
![7f7a115eef626080fa00733046bef3e2](/img/posts/7f7a115eef626080fa00733046bef3e2.jpg)
