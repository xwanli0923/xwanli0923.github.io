---
layout: post
title: "在红帽企业版Linux9中如何重置ROOT密码"
date: 2022-12-25
tag: linux
---
![rhel9](/assets/images/2022-12-25/rhel9-backlit-lines.jpg)

*[图片来源：RHEL9 内置壁纸]*

*作者：邢万里*

## 文档说明：

本文叙述了如何在没有对应版本的系统LiveCD进入救援模式下重置 **root** 账户密码的使用场景。

> 系统版本: Red Hat Enterprise Linux 9.0
> 
> Kernel: 5.14.0-70.13.1.el9_0.x86_64


## 重置 ROOT 密码

1. 在计算机的本地控制台，发送 **Ctrl+Alt+Del** 组合键来重启系统。

2. 当我们看到系统引导菜单界面，按下任意按键（不包括**Enter**)中止倒计时。

3. 使用方向按键移动光标到当前内核版本对应的 rescue 条目:
![select-kernel-item](/assets/images/2022-12-25/select-kernel-item.png)

   根据屏幕下方的提示，按下 **e** 编辑选择的条目；如果在编辑页面发生改变后，想要恢复默认内容，按下**Esc**回退到当前页面菜单，重新进入即可。

4. 进入到编辑页面后，使用方向键移动光标，找到 **linux** 开头的一行，按下 **Ctrl+e** 或者 **End** 键，来到行尾，输入 **rd.break** ；这可以使系统从 **initramfs** 向实际系统移交控制权前中断前进。
![kernel-command-line](/assets/images/2022-12-25/kernel-command-line.png)
  按下 **Ctrl+x** 使用当前更改进行启动

5. 启动之后，系统会进入 **emergency mode**, 并提示按下 **Enter** 进入维护,按下 **Enter** 按键：
![chroot-jail](/assets/images/2022-12-25/chroot-jail.png)

6. 在当前的 **shell (switch_root)** 环境中，磁盘上的系统根目录会以只读的方式挂载在 **/sysroot** 。我们在修改密码时需要更新 **/etc/shadow** 文件，因此我们需要将 **sysroot** 目录重新以读写权限进行挂载：
   ```shell
   $ mount -o remount,rw /sysroot
   ```

7. 使用 **chroot** 命令切换系统根目录到 **/sysroot** ，**/sysroot** 是我们系统真实的根目录：
   ```shell
   $ chroot /sysroot
   ```

8. 切换成功后 ，设置新的密码:
   ```shell
   $ passwd root
   ```

9. 检查系统是否开启了 **SELinux**，如果开启，上述更改会导致系统在下次启动失败，需要创建 **/.autorelabel** 文件，通知系统在下次启动时，确保所有未标记文件能够重新获得 **SELinux** 安全上下文标签:
   ```shell
   $ grep ^SELINUX= /etc/selinux/config
   SELINUX=enforcing
   $ touch /.autorelabel
   ```
10. 输入 **exit** 命令退出当前 **chroot** 环境 ,并再次输入 **exit** 命令退出 **initramfs** 调试 **shell** 。

11. 系统将自动重启两次，第二次可以看到 **login** 登录提示，输入 **root** 账户和对应新密码登录系统。



