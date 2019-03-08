---
title: linux_4
tags: Linux
---
## 第四章 首次登入与在线求助

### 4.1 首次登入系统

  * #： 管理员
  * $： 一般用户

### 4.2 文本模式下指令的下达

  * 4.2.1 开始下达指令
    + 语法： command [-options] parameter1 parameter1 ...
  * 4.2.2 基础指令的操作
    + 日期：date
      - date 
      - date +%Y/%m/%d
      - date +%H:%M
    + 日历：cal
      - cal [month] [year]
      - cal 2019
      - cal 10 2019
    + 计算器：bc
      - scale：指定小数位数
  * 4.2.3 重要的热键
    + [Tab]：自动补全
    + [ctrl]-c：退出执行
    + [ctrl]-d：结束输入，可以替代`exit`
    + [shift]+{[PageUP]|[PageDown]}
  * 4.3 Linux系统的在线求助 man page 和info page
    + 4.3.1 指令的 --help 求助说明
    + 4.3.2 man page (manual)
      - 查看系统有哪些跟【man】这个指令有关的说明文件
        - man -f man
        - whatis => man -f
        - apropos => man -k
    + 4.3.3 info page
      - 支持info指令的文件默认放置在`/usr/share/info/`
    + 4.3.4 其他有用文档
      - 位置： `/usr/share/doc`
  *  4.4 nano
  *  4.5 正确的关机方法
     +  注意事项
        1. 观察系统的使用状态
           - who: 查看谁在线
           - netstat -a: 查看网络联机状态
           - ps -aux: 查看背景执行的程序
        2. 通知在线使用者关机的时刻
           - shutdown
        3. 正确的关机指令使用
           - shutdown
           - reboot
   + 数据同步写入磁盘： sync
     > 将内存中的数据写入到磁盘，关机或者重启前执行
   + 关机指令：shutdown
     > 实体终端机 （tty1~tty7）登录系统，所有身份都可以关机，室友远程管理工具（透过pietty使用ssh），只能使用root才能关机。
     - 语法：shutdown [-krhc] [时间] [警告信息]
       - -k: 不要真关机，只是发送警告信息
       - -r: 在将系统的符文停掉之后就重新启动
       - -h: 将系统服务停掉后，立即关机
       - -c: 取消进行中的shutdown指令
       - 时间: 指定系统关机时间，默认1分钟，格式如下：
         - shutdown -h now  
           - 立即关机
         - shutdown -h 20:25
           - 今天20:25分关机，当前时间超过该时间，会在第隔天这个时间关机
         - shutdow -h +30
           - 30分钟后关机
     + 重新启动，关机：reboot,halt,poweroff
     + 时间使用管理工具 systemctl 关机
       - 语法：systemctl [指令]
       - halt: 进入系统停止模式，屏幕可能会保留一些讯息，这与你的电源模式有关
       - poweroff： 进入系统关机模式，直接关机没哟提供电力
       - reboot: 直接重新启动
       - suspend：进入休眠模式