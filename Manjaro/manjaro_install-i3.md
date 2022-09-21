# Manjaro Install-I3

> 操作系统安装的是 manjaro-i3-21.1.2-minimal，语言为简体中文

## 基本操作

```bash
$ sudo pacman -S  package # 安装
$ sudo pacman -R  package # 删除单个软件包，保留其全部已经安装的依赖关系
$ sudo pacman -Rs package # 除指定软件包，及其所有没有被其他已安装软件包使用的依赖关系
$ sudo pacman -Ss package # 查找软件
$ sudo pacman -Sc         # 清空并且下载新数据
$ sudo pacman -Syu        # 升级所有软件包
$ sudo pacman -Qs         # 搜索已安装的包
```

## 基础配置

### 替换国源

1）配置中国的 mirrors，在 终端 执行下面的命令从官方的源列表中对中国源进行测速和设置

```bash
$ sudo pacman-mirrors -i -c China -m rank
```

2）为 Manjaro 增加中文社区的源来加速安装软件，在 `/etc/pacman.conf` 中添加 `archlinuxcn` 源，末尾加上：

```bash
[archlinuxcn]
SigLevel = Optional TrustedOnly
Server = https://mirrors.tuna.tsinghua.edu.cn/archlinuxcn/$arch
```

3）安装 `archlinuxcn-keyring` 包以导入 `GPG key`，否则的话 key 验证失败会无法安装并安装 `base-devel` 工具包，最后同步更新系统：

```bash
$ sudo pacman -Syy
$ sudo pacman -S archlinuxcn-keyring base-devel
$ sudo pacman -Syyu
```

### 安装 yay

> Yay 是用 Go 编写的 Arch Linux AUR 包管理工具。具体可以查看 Arch Wiki

```bash
$ sudo pacman -S yay
$ yay --aururl "https://aur.tuna.tsinghua.edu.cn" --save # 配置镜像源
$ yay -P -g                                              # 配置文件位于 ~/.config/yay/config.json 

# 基本操作
$ yay -S package   # 从 AUR 安装软件包
$ yay -Rns package # 删除包
$ yay -Syu         # 升级所有已安装的包
$ yay -Ps          # 打印系统统计信息
$ yay -Qi package  # 检查安装的版本
```

### 安装 ZSH

> [Zsh](https://www.zsh.org/) 是一款功能强大的 [Shell](https://wiki.archlinux.org/title/Shell) 软件，既可以作为交互式终端来使用，也可以作为脚本语言解释器来使用

```bash
$ cat /etc/shells      # 查看系统所有shell
$ chsh -s /usr/bin/zsh # 更改默认shell为zsh

# 安装ohmyzsh
$ wget https://github.com/robbyrussell/oh-my-zsh/raw/master/tools/install.sh -O - | sh

# 安装插件
# zsh-syntax-highlighting：语法高亮
$ git clone https://github.com/zsh-users/zsh-syntax-highlighting.git ${ZSH_CUSTOM}/plugins/zsh-syntax-highlighting
# autosuggestions：记住用过的命令
$ git clone git://github.com/zsh-users/zsh-autosuggestions ~/.oh-my-zsh/custom/plugins/zsh-autosuggestions
# autojump：文件目录跳转
$ sudo pacman -S autojump
$ source /usr/share/autojump/autojump.zsh

# 修改主题，这里使用的主题是powerlevel10k，详细信息可从Github找到
$ git clone --depth=1 https://github.com/romkatv/powerlevel10k.git ${ZSH_CUSTOM:-~/.oh-my-zsh/custom}/themes/powerlevel10k

$ sudo vim ~/.zshrc                       # 修改配置文件
ZSH_THEME="powerlevel10k/powerlevel10k"   # 更改ZSH_THEME
plugins=(git                              # 更改plugins
       zsh-autosuggestions
       zsh-syntax-highlighting
       sudo
       extract
       autojump
     )
# 字体显示模糊 
export LC_ALL=en_US.UTF-8
export LANG=EN_US.UTF-8
# 文末添加
eval $(thefuck --alias)
$ source ~/.zshrc                         # 刷新配置,打开终端按提示进行配置即可
```

注：**使用vim编辑时提错E437: terminal capability "cm" required**

```bash
# 临时方法
export TERM=xterm
# 永久修改
$ sudo vim /etc/profile
$ sudo vim ~/.zshrc
export TERM=xterm
```

### 安装字体

```bash
# 安装常用的中文字体
$ sudo pacman -S ttf-roboto noto-fonts ttf-dejavu powerline powerline-fonts powerline-vim
# 文泉驿
$ sudo pacman -S wqy-bitmapfont wqy-microhei wqy-microhei-lite wqy-zenhei
# 思源字体
$ sudo pacman -S noto-fonts-cjk adobe-source-han-sans-cn-fonts adobe-source-han-serif-cn-fonts
# 其它字体
$ sudo pacman -S ttf-font-awesome ttf-font-icons ttf-font-logos ttf-roboto-mono awesome-terminal-fonts nerd-fonts-complete ttf-fireflysung firefox-i18n-zh-cn thunderbird-i18n-zh-cn gimp-help-zh_cn man-pages-zh_cn && fc-cache -fv && locale-gen

# 桌面乱码问题：修改conky_main乱码，将Bitstream Vera字体替换成anti
$ sudo vim /usr/share/conky/conky_maia
:%s/Bitstream Vera/anti/
```

**字体模糊发虚**
1. 先系统设置生成 `~/.config/fontconfig/fonts.conf`
2. mod+z – Settings – Qt5 settings – 标签栏” 字体” – 创建”fonts.conf” (可以前置选择好自己的字体再创建，下为参考参数)

```xml
<?xml version="1.0"?>
<!DOCTYPE fontconfig SYSTEM "fonts.dtd">

<fontconfig>

    <its:rules xmlns:its="http://www.w3.org/2005/11/its" version="1.0">
        <its:translateRule translate="no" selector="/fontconfig/*[not(self::description)]"/>
    </its:rules>

    <description>Manjaro Font Config</description>

    <!-- Font directory list -->
    <dir>/usr/share/fonts</dir>
    <dir>/usr/local/share/fonts</dir>
    <dir prefix="xdg">fonts</dir>
    <dir>~/.fonts</dir> <!-- this line will be removed in the future -->

    <!-- 自动微调 微调 抗锯齿 内嵌点阵字体 -->
    <match target="font">
        <edit name="autohint"> <bool>false</bool> </edit>
        <edit name="hinting"> <bool>true</bool> </edit>
        <edit name="antialias"> <bool>true</bool> </edit>
        <edit name="embeddedbitmap" mode="assign"> <bool>false</bool> </edit>
    </match>

    <!-- 英文默认字体使用 Roboto 和 Noto Serif ,终端使用 DejaVu Sans Mono. -->
    <match>
        <test qual="any" name="family">
            <string>serif</string>
        </test>
        <edit name="family" mode="prepend" binding="strong">
            <string>Noto Serif</string>
        </edit>
    </match>
    <match target="pattern">
        <test qual="any" name="family">
            <string>sans-serif</string>
        </test>
        <edit name="family" mode="prepend" binding="strong">
            <string>Roboto</string>
        </edit>
    </match>
    <match target="pattern">
        <test qual="any" name="family">
            <string>monospace</string>
        </test>
        <edit name="family" mode="prepend" binding="strong">
            <string>DejaVu Sans Mono</string>
        </edit>
    </match>

    <!-- 中文默认字体使用思源宋体,不使用 Noto Sans CJK SC 是因为这个字体会在特定情况下显示片假字. -->
    <match>
        <test name="lang" compare="contains">
            <string>zh</string>
        </test>
        <test name="family">
            <string>serif</string>
        </test>
        <edit name="family" mode="prepend">
            <string>Source Han Serif CN</string>
        </edit>
    </match>
    <match>
        <test name="lang" compare="contains">
            <string>zh</string>
        </test>
        <test name="family">
            <string>sans-serif</string>
        </test>
        <edit name="family" mode="prepend">
            <string>Source Han Sans CN</string>
        </edit>
    </match>
    <match>
        <test name="lang" compare="contains">
            <string>zh</string>
        </test>
        <test name="family">
            <string>monospace</string>
        </test>
        <edit name="family" mode="prepend">
            <string>Noto Sans Mono CJK SC</string>
        </edit>
    </match>

    <!-- 把Linux没有的中文字体映射到已有字体，这样当这些字体未安装时会有替代字体 -->
    <match target="pattern">
        <test qual="any" name="family">
            <string>SimHei</string>
        </test>
        <edit name="family" mode="assign" binding="same">
            <string>Source Han Sans CN</string>
        </edit>
    </match>
    <match target="pattern">
        <test qual="any" name="family">
            <string>SimSun</string>
        </test>
        <edit name="family" mode="assign" binding="same">
            <string>Source Han Serif CN</string>
        </edit>
    </match>
    <match target="pattern">
        <test qual="any" name="family">
            <string>SimSun-18030</string>
        </test>
        <edit name="family" mode="assign" binding="same">
            <string>Source Han Serif CN</string>
        </edit>
    </match>
    <!--
    <match target="pattern">
        <test qual="any" name="family">
            <string>Microsoft YaHei</string>
        </test>
        <edit name="family" mode="assign" binding="same">
            <string>Source Han Sans CN</string>
        </edit>
    </match>
    -->
    
    <!-- Load local system customization file -->
    <include ignore_missing="yes">conf.d</include>
    <!-- Font cache directory list -->
    <cachedir>/var/cache/fontconfig</cachedir>
    <cachedir prefix="xdg">fontconfig</cachedir>
    <!-- will be removed in the future -->
    <cachedir>~/.fontconfig</cachedir>

    <config>
        <!-- Rescan in every 30s when FcFontSetList is called -->
        <rescan> <int>30</int> </rescan>
    </config>

</fontconfig>
```

### 目录修改

> 先将中文目录名称修改成英文，再修改配置文件信息

```bash
$ vim ~/.config/usr-dirs.dirs
XDG_DESKTOP_DIR="$HOME/Desktop"
XDG_DOWNLOAD_DIR="$HOME/Downloads"
XDG_TEMPLATES_DIR="$HOME/Templates"
XDG_PUBLICSHARE_DIR="$HOME/Public"
XDG_DOCUMENTS_DIR="$HOME/Documents"
XDG_MUSIC_DIR="$HOME/Music"
XDG_PICTURES_DIR="$HOME/Pictures"
XDG_VIDEOS_DIR="$HOME/Videos"
```

### 输入法

> fcitx 是 Free Chinese Input Toy for X 的缩写，国内也常称作小企鹅输入法，是一款 Linux 下的中文输入法

```bash
$ sudo pacman -S fcitx5-im
$ sudo pacman -S fcitx5-chinese-addons
$ sudo pacman -S fcitx5-rime
$ sudo pacman -S fcitx5-configtool
$ sudo nano ~/.pam_environment
# 添加以下内容
INPUT_METHOD  DEFAULT=fcitx5
GTK_IM_MODULE DEFAULT=fcitx5
QT_IM_MODULE  DEFAULT=fcitx5
XMODIFIERS    DEFAULT="@im=fcitx5"
# 配置环境变量
$ sudo vim ~/.xprofile
export GTK_IM_MODULE=fcitx5
export QT_IM_MODULE=fcitx5
export XMODIFIERS=@im=fcitx
fcitx5 &

$ sudo vim ~/.i3/config
# Autostart applictions
exec --no-startup-id fcitx
```

### 挂载外盘

> 挂载 samba (按需配置)，因有多台 PC/Server 设备，为了 Win/Linux 之间互联，可略过

```bash
$ yay -S samba
$ mkdir -p /home/username/share
# 挂载并设置自启
$ smbclient //server-ip/share -U servername
$ sudo mount -t cifs -l //server-ip/share /home/username/share -o username=servername,password=serverpassword
$ sudo vim /etc/fstab
$ //server-ip/share /home/username/share cifs defaults,username=servername,password=serverpassword 0 0
```

## 日常工具

> 建议安装各自所需的，没必要全部安装

```bash
$ yay -S google-chrome                            # 浏览器
$ vim ~/.config/mimeapps.list                     # 默认值
:%s/userapp-Pale Moon/google-chrome-stable/
$ vim ~/.i3/config                                # 快捷建
bindsym $mod+F2 exec google-chrome-stable
$ vim ~/.profile                                  # 环境变量
export BROWSER=/usr/bin/google-chrome-stable
-------------------------------------------------------------
$ yay -S switchhosts                              # github访问加速,选择.appimage版本
- 标题：任意
- 类型：Remote
- URL: https://cdn.jsdelivr.net/gh/521xueweihan/GitHub520@main/hosts
- 自动刷新时间：第一次添加可以先选择1 minute，有了规则以后，就可以选择1 hour
-------------------------------------------------------------
$ sudo pacman -S unrar unzip p7zip file-roller    # 压缩软件
$ sudo pacman -S timeshift                        # 备份工具
$ yay -S wps-office wps-office-mui-zh-cn wps-office-fonts ttf-wps-fonts  # wps-office
$ sudo pacman-S netease-cloud-music               # 网易音乐
$ yay -S picgo                                    # 图床工具
$ sudo pacman -S screenkey                        # 屏幕键盘
$ sudo pacman -S typora                           # 编辑工具
$ sudo pacman -S net-tools                        # 网络工具
$ sudo pacman -S aria2 filezilla                  # 下载工具
$ yay -S xmind                                    # 思维导图
$ sudo pacman -S foxitreader bookworm             # pdf&电子书
$ sudo pacman -S wireshark-qt                     # 抓包工具
$ yay -S remmina freerdp                          # 远程桌面
$ sudo pacman -S mpv vlc                          # 播放器
```

## 开发工具

```bash
$ sudo pacman -S vim                              # vim
$ sudo pacman -S git                              # git
$ git config --global user.name "username"
$ git config --global user.email "usermail"
$ ssh-keygen -t rsa -c "usermail"
-------------------------------------------------------------
$ sudo pacman -S nodejs npm yarn                  # nodejs & npm & yarn
$ npm config set registry https://registry.npm.taobao.org
$ sudo npm install -g hexo-cli                    # hexo-blog
-------------------------------------------------------------
$ sudo pacman -S jdk8-openjdk jdk11-openjdk       # java
$ sudo archlinux-java set java-8-openjdk          # 版本转换
-------------------------------------------------------------
$ sudo pacman -S maven                            # maven
配置文件路径 `/opt/maven/conf/settings.xml`
-------------------------------------------------------------
$ sudo pacman -S python-pip                       # pip
$ pip install --user thefuck                      # thefuck
# 下载慢时可更换源
$ mkdir ~/.pip
$ nano ~/.pip/pip.conf
[global]
index-url = https://pypi.tuna.tsinghua.edu.cn/simple
[install]
trusted-host=mirrors.aliyun.com
-------------------------------------------------------------
$ yay -S pycharm                                  # pycharm
$ yay -S visual-studio-code-bin                   # VScode
$ yay -S jetbrains-toolbox                        # Jetbrains 全家桶管理工具（包含Android Studio）
$ sudo pacman -S make cmake                       # 编译器
$ sudo pacman -S clang                            # C
$ sudo pacman -S golang                           # Go
$ sudo pacman -S atom                             # Atom
$ sudo pacman -S gitkraken                        # Gitkraken
```

## 系统优化

> 系统优化包含于桌面美化之类，请各自选择是否适合自身

### 磁盘优化

```bash
$ sudo systemctl enable fstrim.timer 
```

### Zsh 主题

1）zsh 修改主题(系统内置主题)

```bash
$ vim ~/.zshrc
# 把默认的agnoster改为新的主题名称即可
ZSH_THEME="agnoster"
$ source ~/.zshrc

# 插件
$ cd ~/.oh-my-zsh/custom/plugins/
$ git clone https://github.com/zsh-users/zsh-syntax-highlighting.git
$ git clone https://github.com/zsh-users/zsh-autosuggestions
# 修改配置文件
plugins=(
	git
	zsh-autosuggestions
	zsh-syntax-highlighting
)
```

2）zsh 修改主题(系统外置主题)

> 更换没有的主题：[powerlevel9k](https://github.com/Powerlevel9k/powerlevel9k)  [Powerlevel10k](https://github.com/romkatv/powerlevel10k)

```bash
# 旧版本与新版本，现以9k为例
$ git clone https://github.com/bhilburn/powerlevel9k.git ~/.oh-my-zsh/custom/themes/powerlevel9k
$ git clone https://github.com/romkatv/powerlevel10k ~/.oh-my-zsh/custom/themes/powerlevel10k
$ sudo vim ~/.zshrc
ZSH_THEME="powerlevel9k/powerlevel9k"
#自定义提示符内容，具体请查看github
POWERLEVEL9K_LEFT_PROMPT_ELEMENTS=(os_icon context vcs)
POWERLEVEL9K_RIGHT_PROMPT_ELEMENTS=(dir time root_indicator)
POWERLEVEL9K_MODE='nerdfont-complete'
POWERLEVEL9K_TIME_FORMAT='%D{%H:%M}'

# ==== Theme Settings ====
# PowerLevel9k
# 左侧栏目显示的要素（指定的关键字参考官网）
POWERLEVEL9K_LEFT_PROMPT_ELEMENTS=(os_icon context dir vcs)
# 右侧栏目显示的要素
POWERLEVEL9K_RIGHT_PROMPT_ELEMENTS=(status root_indicator background_jobs time virtualenv)
#新起一行显示命令 (推荐！极其方便）
POWERLEVEL9K_PROMPT_ON_NEWLINE=true
#右侧状态栏与命令在同一行
POWERLEVEL9K_RPROMPT_ON_NEWLINE=true
#缩短目录层级
POWERLEVEL9K_SHORTEN_DIR_LENGTH=1
#缩短目录策略：隐藏上层目录中间的字
#POWERLEVEL9K_SHORTEN_STRATEGY="truncate_middle"
#添加链接上下链接箭头更方便查看
POWERLEVEL9K_MULTILINE_FIRST_PROMPT_PREFIX="↱"
POWERLEVEL9K_MULTILINE_LAST_PROMPT_PREFIX="↳ "
# 新的命令与上面的命令隔开一行
#POWERLEVEL9K_PROMPT_ADD_NEWLINE=true
# Git仓库状态的色彩指定
POWERLEVEL9K_VCS_CLEAN_FOREGROUND='blue'
POWERLEVEL9K_VCS_CLEAN_BACKGROUND='black'
POWERLEVEL9K_VCS_UNTRACKED_FOREGROUND='yellow'
POWERLEVEL9K_VCS_UNTRACKED_BACKGROUND='black'
POWERLEVEL9K_VCS_MODIFIED_FOREGROUND='red'
POWERLEVEL9K_VCS_MODIFIED_BACKGROUND='black'
```

### 安装 alacritty

> alacritty 是一款仿终端工具，下载安装后，根据自需求进行配置即可，配置可参考 i3 配置文件

```bash
$ sudo pacman -S alacritty
$ sudo vim ~/.i3/config
# start a terminal 修改默认终端
bindsym $mod+Return exec alacritty
```

**配置自定义**

[alacritty 配置文件](https://github.com/alacritty/alacritty/releases) 与 i3 配置文件类似，若 .config 目录下没有 alacritty 目录则自行创建，并将上述配置文件根据自身版本复制过来

```bash
$ sudo mkdir ~/.config/alacritty
$ sudo cp /home/usrname/Downloads/alacritty.yml ~/.config/alacritty/

# 例子：桌面背景透明化
$ sudo vim ~/.config/alacritty/alacritty.yml
# Background opacity 背景不透明度
background_opacity: 0.5
```

### 安装 Feh

> 背景壁纸自动刷新

```bash
$ sudo pacman -S feh
$ sudo vim ~/.i3/config
# 首先创建背景目录，并设置每次启动 feh 自动随机选取壁纸作桌面
exec --no-startup-id feh --randomize --big-fill /usr/share/backgrounds/*.jpg 
```

```bash
# 如上述不成功可尝试方法二
$ vim ~/shell/wallpic.sh
#!/bin/sh

while true; do
    find /usr/share/backgrounds -type f \( -name '*.jpg' -o -name '*.png' \) -print0 |
        shuf -n1 -z | xargs -0 feh --bg-scale
    sleep 15m
done
$ chmod +x wallpic.sh
# 设置开机自启脚本
exec_always --no-startup-id ~/shell/wallpic.sh
```

注：两种方法最后都使用 `$mod+shift+r` 重启 i3 就可以了

*关于 feh 配置背景不生效问题*，虽然设置了 feh 桌面背景，但每次开机启动还是默认桌面背景，原因在于在 `~/.i3/config` 配置文件中可知，manjaro-i3 默认使用 `nitrogen` 来管理桌面背景的，配置文件在 `~/.config/nitrogen` ，可根据需要进行修改

```bash
[xin_-1]
file=/usr/share/backgrounds/i3_default_background.jpg
mode=5
bgcolor=#002A36
```

### 安装 flameshot

> 截图软件安装并设置快捷键

```bash
$ sudo pacman -S flameshot
$ sudo vim ~/.i3/config
# 添加截图快捷方式
bindsym Print --release exec /usr/bin/flameshot gui
for_window [class="flameshot"] floating enable 
exec --no-startup-id flameshot
```

### 终端透明

```bash
$ sudo pacman -S picom
$ sudo vim ~/.i3/config
exec_always --no-startup-id picom
$ sudo vim ~/.config/picom.conf
# Opacity
inactive-opacity = 0.7;
active-opacity = 0.9;
```

### 关闭时间

> 关闭桌面上左下角提示及右上角时间：桌面上的信息都由 conky 服务提供，关闭 conky 服务并重置配置文件即可

```bash
$ sudo kill -9 $(ps aux | grep conky | awk '{print $2}')
$ sudo cd /usr/share/conky
$ sudo mv conky_maia conky_maia.bak
$ sudo mv conky_1.10_shortcuts_maia conky_1.10_shortcuts_maia.bak
# 将下行进行注释
$ sudo vim ~/.i3/config
exec --no-startup-id start_conky_maia
```

### 配置显示

> 首先要使用 `xrandr` 命令查看显示器参数，根据实际情况修改

```bash
$ sudo vim ~/.i3/config
xrandr --output HDMI-1 --same-as DVI-D-1 --auto           # 双屏幕显示相同的内容--克隆
xrandr --output HDMI-1 --same-as DVI-D-1 --mode 1280x1024 # 指定外接显示器的分辨率
xrandr --output HDMI-1 --right-of DVI-D-1 --auto          # 外接显示器，设置为右侧扩展
xrandr --output DVI-D-1 --off                             # 关闭显示器
xrandr --output HDMI-1 --auto --output DVI-D-1 --off      # 打开HDMI-1接口显示器，关闭DVI-D-1接口显示器
xrandr --output HDMI-0 --primary                          # 设置主屏幕
```