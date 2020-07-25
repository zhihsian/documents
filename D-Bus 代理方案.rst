D-Bus 代理方案
##############

调研过程
********

直接转发流量来代理
==================

#. 使用 socat 创建一个 UDS 透明代理：
    .. code-block:: shell

        sudo socat -t100 -x -v UNIX-LISTEN:/run/dbus/system_bus_socket1,reuseaddr,fork UNIX-CONNECT:/run/dbus/system_bus_socket

#. 使用 d-feet 来连接 /run/dbus/system_bus_socket1
    失败，在 AUTH 过程直接被拒绝了，socat 输出如下：

    .. code-block::

        > 2020/07/17 16:44:40.018158  length=7 from=0 to=6
         00 41 55 54 48 0d 0a                             .AUTH..
        --
        < 2020/07/17 16:44:40.018364  length=19 from=0 to=18
         52 45 4a 45 43 54 45 44 20 45 58 54 45 52 4e 41  REJECTED EXTERNA
         4c 0d 0a                                         L..
        --
        > 2020/07/17 16:44:40.018493  length=24 from=7 to=30
         41 55 54 48 20 45 58 54 45 52 4e 41 4c 20 33 31  AUTH EXTERNAL 31
         33 30 33 30 33 30 0d 0a                          303030..
        --
        < 2020/07/17 16:44:40.018618  length=19 from=19 to=37
         52 45 4a 45 43 54 45 44 20 45 58 54 45 52 4e 41  REJECTED EXTERNA
         4c 0d 0a                                         L..


分析源码
========

因 sd-bus 的代码比 libdbus 更易懂，所以选择了 sd-bus。

在 bus-socket.c 中，找到了 :code:`bus_socket_start_auth` 函数，发现他一开始就调用了 :code:`bus_get_peercred`，并最终通过 :code:`SO_PEERCRED` 调用了 :code:`getsocketopt`，获取到了如下信息：

.. code-block:: c

    struct ucred
    {
      pid_t pid;			/* PID of sending process.  */
      uid_t uid;			/* UID of sending process.  */
      gid_t gid;			/* GID of sending process.  */
    };

继续往下找，看到了到了 is_server 的判断，验证是在 server 端做的，自然得走 :code:`is_server == true` 的分支。

进入 bus_socket_read_auth，再进入调用的第一个函数 :code:`bus_socket_read_auth`，再继续进入 :code:`bus_socket_read_auth`。

由上面 socat 的输出可知，验证是停在了「AUTH EXTERNAL」这一步，而 :code:`line_begins(line, l, "AUTH EXTERNAL")` 分支调用了 :code:`verify_external_token`。

进入 :code:`verify_external_token` 可发现，它将 :code:`AUTH EXTERNAL` 后面的字符串作十六进制解码处理，如 :code:`31 30 30 30` 解码为字符串 :code:`1000`，就是我运行 d-feet 用户的 UID。

结论
****

D-Bus 客户端在发送验证的时候，会将当前进程的 UID 转为 16 进制字符串发送过去，服务端会通过 SO_PEERCRED 来获取链接所属的 UID，若此 UID 与客户端发送过来的 UID 不匹配，则会拒绝链接。

D-Bus 代理实现思路
******************

代理程序以 root 权限运行，当有程序连接到时，先使用 SO_PEERCRED 调用 getsocketopt，来获取连接过来的进程所属用户信息，然后使用 setruid 设置进程 UID，再连接 dbus-daemon。
