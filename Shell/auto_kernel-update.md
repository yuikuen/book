# Auto Kernel-Update

> CentOS7 内核升级脚本

```bash
#!/bin/bash

source /etc/init.d/functions

# 检查脚本运行用户是否为Root
if [ $(id -u) != 0 ];then
        echo -e "\033[31mError! You must be root to run this script! \033[0m"
        exit 1
fi

function update_kernel() {
    # 安装EL源
    rpm --import https://www.elrepo.org/RPM-GPG-KEY-elrepo.org
    rpm -Uvh https://www.elrepo.org/elrepo-release-7.el7.elrepo.noarch.rpm
    if [[ $? -ne 0 ]];then
        echo -e "\033[31mError! EL 源安装失败，请检查是否存在问题！\033[0m"
        exit 1
    fi

    # 查看可提供升级的版本
    yum --disablerepo="*" --enablerepo="elrepo-kernel" list available
    VAR_KERNEL_NAME="kernel-lt"
    read -p "请输入上面列出的版本中你想安装的版本（默认 lt 版本） [lt/ml]: " VAR_VERSION_CHOICE
    if [[ ${VAR_VERSION_CHOICE} == "ml" ]];then
        VAR_KERNEL_NAME="kernel-ml"
    fi

    echo "本次选择升级的版本为：${VAR_KERNEL_NAME}"

    # 升级内核
    yum -y --enablerepo=elrepo-kernel install ${VAR_KERNEL_NAME}
    if [[ $? -ne 0 ]];then
        echo "内核升级失败，请根据报错检查是否存在问题！"
        exit 1
    fi

    # 查看目前版本
    echo "系统当前所安装的内核版本如下："
    awk -F\' '$1=="menuentry " {print i++ " : " $2}' /etc/grub2.cfg

    # 选择默认内核版本
    grub2-set-default 0 && grub2-mkconfig -o /etc/grub2.cfg
}

function uninstall_kernel() {
    # 显示内核版本
    echo "系统当前所安装的内核版本如下："
    rpm -qa | grep kernel

    # 提示卸载
    echo "你可以手动卸载旧版本：yum -y remove 包名字，然后重启使用：uname -r 查看升级结果"
}

read -p "是否继续安装升级（默认 y） [y/n]: " VAR_CHOICE
case ${VAR_CHOICE} in
    [yY][eE][sS]|[yY])
    update_kernel
    uninstall_kernel
    ;;
    [nN][oO]|[nN])
    echo "安装升级即将终止..."
    exit
    ;;
  *)
    update_kernel
    uninstall_kernel
esac
```