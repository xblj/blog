---
title: linux_5
tags: Linux
---

## 第五章、Linux的文件权限与目录配置

### 5.1 使用者与群组

  * owner/group/others
  * Linux用户身份与群组记录的文件
    - /etc/passwd: 记录所有账号
    - /etc/shadow: 记录个人密码
    - /etc/group: 记录所有群组
### 5.2 Linux 文件权限概念

  * 5.2.1 Linux 文件属性
    + ls -al
      - -rw-r--r-- 1 root root 1024 May 4 18:01 文件名称
      - 说明： [文件类型][rwx][rwx][rwx] 链接数 owner group 修改时间 文件名称
        - 文件类型： 
          - d： 目录
          - -： 文件
          - l: 连结档
          - b: 装置文件里面可供存储的接口设备（可随机存取装置）,例如硬盘
          - c: 装置文件里面串行端口，例如鼠标、键盘
        - 接下来的三个[rwx]，分别表拥有者、群组和其他人对这个文件的读取、写入和执行权限，没有该权限就是“-”
  * 5.2.2 如何改变文件属性与属性
    + chgrp：改变文件所属群组
      - chgrp [-R] group 文件名
    + chown：改变文件拥有者
      - 语法：chown [-R] [用户名][:组名] 文件或者目录
    + chmod：改变文件权限
      - 数字类型改变文件权限
        - r:4
        - w:2
        - x:1
        - 语法： chmod [-R] xyz 文件或者目录
          - xyz代表上面数字相加，x:拥有者，y:群组，z:其他人
      - 符号类型改变文件权限
        - 语法 chmod [u][+-=][rwx],[g][+-=][rwx],[o][+-=][rwx]
        - chomd u=rwx,go=rx example
          - 拥有者有读写执行，群组和其他有读和执行
### 5.3 Linux 目录配置

  * 5.3.1 Linux目录配置的依据 -- FHS
    |    |  可分享的  |  不可分享的  |
    |-----|--------|---------------|
    | 不变的 |/usr(软件放置处)，/opt（第三方协力软件）|/etc(配置文件), /boot(开机与核心档)|
    |可变的  | /var/mail(邮件信箱)，/var/spool/news(新闻组)| /var/run(程序相关)，/var/lock（程序相关）
