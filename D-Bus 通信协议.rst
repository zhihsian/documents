D-Bus 通信协议
##############

注：

1. 此文由 `D-Bus Specification <https://dbus.freedesktop.org/doc/dbus-specification.html>`_ 整理而来
1. 此文假定你已了解 D-Bus 的数据类型
1. 数据结构以类似于 C++ struct 伪代码编写

连接方式
********

Unix Domain Sockets
===================

*nix 系统上的默认连接方式。

在一些分发版上，默认的地址为：

* System Bus: :code:`/run/dbus/system_bus_socket`
* Session Bus: :code:`/run/user/[UID]/bus`

TCP Sockets
===========
通过 TCP 协议来传输数据

SystemD
=======

仅 Linux 系统可使用 SystemD，实际上 :code:`dbus-daemon` 使用 SystemD 的 socket activation 来创建基于 UDS 或 TCP 的服务端监听，客户端依然使用 UDS 或 TCP 来连接 D-Bus 服务。

实现方式：向 SystemD 提供 `*.socket` 和 `*.service` 配置文件。其中， `.socket` 中配置监听地址， `.service` 中配置服务的启动方式、依赖等。

有了 socket activation，服务甚至无需立即启动，当有客户端连接监听的地址时，SystemD 负责开启服务，然后将监听的 socket 交给服务端。

launchd
=======
MacOS 上使用的服务管理系统，同样使用 socket activation。

消息格式
********

D-Bus 消息包含消息头和消息体。

消息头的长度必须是 8 的倍数，若内容长度不是 8 的倍数，则必须以不多于 7 个字节的 0 进行填充。

消息体无长度必须是 8 的倍数的限制。

整个消息的长度必须是 2 到 2^27 字节（128 MiB），实现不允许发送或接受超过此大小的消息。

消息头格式
==========

.. code-block:: c++

    struct message_header {
        byte_t   endianness_flag;   // 字节序，'l'（0x42）表示小端序，'B'（0x6C）表示大端序
        byte_t   message_type;      // 消息类型，未知类型必须被忽略，具体值在下面给出
        byte_t   flags;             // 标志，按位每位的解释下面表格给出
        byte_t   major;             // 协议主版本号，当前是 1
        uint32_t body_length;       // 消息体字节长度
        uint32_t serial;            // 消息的序列号，由发送方用作 cookie，以标识与此请求相对应的答复，不允许为 0
        array<field_item> fields;   // 零个或多个字段的数组，由消息类型决定哪些字段是必需的
    };

    struct field_item {
        byte_t  code;   // 字段
        variant value;  // 字段值
    };

array 封送格式见 `数组 <#数组>`_

variant 封送格式见 `variant <#variant>`_

类型
----

+-------------+--+-------------+
|名称         |值|说明         |
+=============+==+=============+
|INVALID      |0 |无效的类型   |
+-------------+--+-------------+
|METHOD_CALL  |1 |方法调用     |
+-------------+--+-------------+
|METHOD_RETURN|2 |方法返回     |
+-------------+--+-------------+
|ERROR        |3 |错误返回     |
+-------------+--+-------------+

标志
----

+-------------------------------+-------+------------------------------------------------------------------------+
|名称                           |值     |说明                                                                    |
+===============================+=======+========================================================================+
|NO_REPLY_EXPECTED              |1 << 0 |此消息不期望方法返回或错误返回                                          |
|                               |       |                                                                        |
|                               |       |METHOD_CALL 是唯一可以期望方法返回或错误返回的消息类型，所以此标志对其它|
|                               |       |消息类型是无意义的                                                      |
+-------------------------------+-------+------------------------------------------------------------------------+
|NO_AUTO_START                  |1 << 1 |若是目的服务没启动，D-Bus 不应该启动它                                   |
+-------------------------------+-------+------------------------------------------------------------------------+
|ALLOW_INTERACTIVE_AUTHORIZATION|1 << 2 |可以在方法调用消息上设置此标志，以通知接收方调用者已准备好等待交互式授权|
|                               |       |，这可能需要花费大量时间才能完成。例如：如果设置了此标志，则可以通过    |
|                               |       |Polkit 或类似框架向用户查询密码或确认。                                 |
|                               |       |                                                                        |
|                               |       |该标志仅对方法调用消息有效，否则将被忽略。客户端不应该默认设置此标志，仅|
|                               |       |当非特权代码调用特权更大的方法调用并且部署了允许进行交互式授权的授权框架|
|                               |       |时，此标志才有用。如果没有部署这样的框架，它将没有效果。如果已设置，则调|
|                               |       |用方还应在方法调用上设置适当的长时间超时，以确保用户交互可以完成。      |
|                               |       |                                                                        |
|                               |       |如果未在方法调用上设置此标志，并且服务确定在没有交互式授权的情况下不允许|
|                               |       |请求的操作，但是在成功进行交互式授权后可以允许该操作，则它可能返回      |
|                               |       |org.freedesktop.DBus.Error.InteractiveAuthorizationRequired 错误。      |
|                               |       |                                                                        |
|                               |       |缺少此标志并不能保证不会应用交互式授权，因为在此标志之前的现有服务可能已|
|                               |       |经使用了交互式授权。但是，将使用交互式授权的现有 D-Bus API              |
|                               |       |应该记录该调用可能比平时花费更长的时间，并且新的 D-Bus API              |
|                               |       |应该在没有此标志的情况下避免交互式授权。                                |
+-------------------------------+-------+------------------------------------------------------------------------+

字段
----

+---------------+-----------+-----------+-------------------+-----------+------------------------------------------------------+
|名称           |字段代码   |字段值类型 |必传               |控制       |说明                                                  |
+===============+===========+===========+===================+===========+======================================================+
|INVALID        |0          |N/A        |不允许             |           |                                                      |
+---------------+-----------+-----------+-------------------+-----------+------------------------------------------------------+
|PATH           |1          |objectPath | | METHOD_CALL     |发送者     |要发送呼叫的对象或发出信号的对象。特殊路径            |
|               |           |           | | SIGNAL          |           |/org/freedesktop/DBus/Local 是保留路径；实现不应使用此|
|               |           |           |                   |           |路径发送消息，并且总线守护程序的参考实现将断开任何尝试|
|               |           |           |                   |           |这样做的应用程序的连接。                              |
+---------------+-----------+-----------+-------------------+-----------+------------------------------------------------------+
|INTERFACE      |2          |string     |SIGNAL             |发送者     |调用方法调用或发出信号的接口。对于方法调用是可选的，对|
|               |           |           |                   |           |于信号是必需的。 特殊接口 org.freedesktop.DBus.Local  |
|               |           |           |                   |           |是保留接口；实现不应使用此接口发送消息，并且总线守护程|
|               |           |           |                   |           |序的参考实现将断开任何尝试这样做的应用程序的连接。    |
+---------------+-----------+-----------+-------------------+-----------+------------------------------------------------------+
|MEMBER         |3          |string     | | METHOD_CALL     |发送者     |成员名，比如方法名、信号名。                          |
|               |           |           | | SIGNAL          |           |                                                      |
+---------------+-----------+-----------+-------------------+-----------+------------------------------------------------------+
|ERROR_NAME     |4          |string     |ERROR              |           |发生的错误的名称，用于错误。                          |
+---------------+-----------+-----------+-------------------+-----------+------------------------------------------------------+
|REPLY_SERIAL   |5          |uint32     | | METHOD_RETURN   |发送者     |用于标明返回数据所答复的调用的序列号（也就是调用时，消|
|               |           |           | | ERROR           |           |息头中的 serial），用于告知调用者这是哪个调用的返回。 |
+---------------+-----------+-----------+-------------------+-----------+------------------------------------------------------+
|DESTINATION    |6          |string     |可选               |发送者     |此消息打算用于的连接的名称。该字段通常仅与消息总线结合|
|               |           |           |                   |           |使用才有意义，但其他服务器可能会为其定义自己的含义。  |
+---------------+-----------+-----------+-------------------+-----------+------------------------------------------------------+
|SENDER         |7          |string     |可选               |消息总线   |发送链接的唯一名称，该字段由消息总线控制，所以它的值与|
|               |           |           |                   |           |消息总线本身一样可靠且值得信赖。                      |
+---------------+-----------+-----------+-------------------+-----------+------------------------------------------------------+
|SIGNATURE      |8          |signature  |可选               |发送者     |发送的数据的签名，如果没有此字段，则假定它是空的，此时|
|               |           |           |                   |           |消息体也必须是空的。                                  |
+---------------+-----------+-----------+-------------------+-----------+------------------------------------------------------+
|UNIX_FDS       |9          |uint32     |可选               |发送者     |消息附带的 Unix                                       |
|               |           |           |                   |           |文件描述符的数量。 如果省略，则假定该消息中没有 Unix  |
|               |           |           |                   |           |文件描述符。实际文件描述符需要通过特定于平台的机制进行|
|               |           |           |                   |           |带外传输。它们必须作为消息本身的一部分同时发送。在消息|
|               |           |           |                   |           |本身的第一个字节被传输之前或消息本身的最后一个字节之后|
|               |           |           |                   |           |，可能不会发送它们。 该头字段由消息发送者控制。       |
+---------------+-----------+-----------+-------------------+-----------+------------------------------------------------------+

消息体格式
==========

可以看到，消息头已经描述了消息的协议版本、daemon 收到消息后的操作、来源、目的，剩下的消息体就只需要放数据了，比如方法调用的参数、方法的返回值、方法的错误返回、信号的值。

消息体的结构很简单，就是将数据连接封送到一起。

数据封送
========

对齐
----

字节块中的每个值都「自然」对齐，例如 4 字节值与 4 字节边界对齐、8 字节值与 8 字节边界对齐。对齐的边界是全局计算的，即相对于整个消息的第一个字节计算。要对齐一个值，可能需要在该值之前添加对齐填充。

字符串、容器，由于数据大小并不固定，要做到边界对齐，不仅需要在数据之前对齐填充，还可能需要在后面填充，他们的填充大小规则见下文。

作为自然对齐的一个例外，struct 和 dictEntry 的值始终与 8 字节边界对齐，而不考虑其值的类型。

对齐填充必须遵循以下标准：

* 必须是以正确对齐为前提的最小填充
* 必须由「0」组成

以下以 nature_align(T) 表示类型 T，或者值 T 的类型的自然对齐大小。

字符串类型（string、objectPath、signature）
-------------------------------------------

.. code-block:: c++

    struct string {
        uint32_t length;
        byte_t   data[length];
        byte_t   end_zero[1];
    };

    typedef string objectPath;

    struct signature {
        uint8_t length;
        byte_t  data[length];
        byte_t  end_zero[1];
    };

字符串类型的结构包含 4 部分：

1. 用来描述字符串长度的无符号整数，其中，string、objectPath 以 uint32 来描述，signature 以 uint8 来描述
2. 紧接着的是字符串内容
3. 后置 0（不被计入字符串长度）
4. 填充 0

字符串结构整体需要以 :code:`nature_align(length)` 边界来对齐，即 :code:`nature_align(string) == nature_align(string.length)` ，若是不足，则需要以小于 :code:`nature_align(length)` 个 0 来填充

容器
----

数组
^^^^

.. code-block:: c++

    struct array<T> {
        uint32_t length;            // 数组内容的字节大小
        byte_t   items_padding_zero[(offset(length) + sizeof(length) + (nature_align(T) - 1)) & ~(nature_align(T) - 1)];
        T        items[];
    };

数组整体同样需要以 :code:`nature_align(length)` 字符边界来对齐，即 :code:`nature_align(array) == nature_align(array.length)`

数组中的每一项，需要以 :code:`nature_align(T)` 字符边界来对齐，所以 length 与 items[] 之间，可能需要不小于 :code:`nature_align(T)` 个填充 0，而每一项之间，由于类型相同，所以紧密连接时不需要再有额外的填充来使之对齐

struct、dictEntries
^^^^^^^^^^^^^^^^^^^

与数组类似，不同的是，由于不同的项可能是不同的类型，所以他们之间会有填充，使每一项能够有与自己的类型相关的对齐。

由于在消息头中的 fields 里的 SIGNATURE 项，已经标明了数据类型，所以 struct 和 dictEntry 不需要再声明容器中值的类型了。

variant
^^^^^^^

.. code-block:: c++

    struct variant {
        signature sign;     // variant 包含数据的数据类型
        byte_t    value_padding_zero[(offset(sign) + sizeof(sign) + (nature_align(T) - 1)) & ~(nature_align(T) - 1)];
        T         value;
    };

variant 整体以 :code:`nature_align(sign)` 字符边界来对齐，即 :code:`nature_align(varian) == nature_align(variant.sign)`

而 variant 的值，以 :code:`nature_align(T)` 字符边界来对齐，所以 sign 和 value 之间，可能需要不小于 :code:`nature_align(T)` 个填充 0

例子
****

.. code-block:: c++

    struct message_header {
        byte_t   endianness_flag;                       // 6c               'l'，小端序
        byte_t   message_type;                          // 01               0，METHOD_CALL
        byte_t   flags;                                 // 00               0，无 flag
        byte_t   major;                                 // 01               1，版本号 1
        uint32_t body_length;                           // 32 00 00 00      消息体长 0x32 字节
        uint32_t serial;                                // 58 02 00 00      消息序列号 0x0258
        struct array<field_item> {
            uint32_t length;                            // 76 00 00 00      数组内容字节长度 0x76 字节
            byte_t   items_padding_zero[0];             //                  这里刚好是 8 的倍数，所以无需 padding
            struct field_item {
                byte_t  code;                           // 08               SIGNATURE
                struct variant {
                    struct signature {
                        uint8_t length;                 // 01               字符串长度 1
                        byte_t  data[length];           // 67               'g'
                        byte_t  end_zero[1];            // 00
                    } sign;
                    byte_t    value_padding_zero[0];    //                  T 是 signature，以 1 对齐，所以无需填充
                    struct signature {
                        uint8_t length;                 // 02               字符串长度 2
                        byte_t  data[length];           // 73 73            "ss"
                        byte_t  end_zero[1];            // 00
                    } value;
                } value;
            } item0;
            struct filed_item {
                byte_t  code;                           // 01               PATH
                struct variant {
                    struct signature {
                        uint8_t length;                 // 01               字符串长度 1
                        byte_t  data[length];           // 6f               'o'
                        byte_t  end_zero[1];            // 00
                    } sign;
                    byte_t    value_padding_zero[0];    //                  objectPath 以 4 字节边界对齐，这里刚好，无需填充
                    struct string {
                        uint32_t length;                // 1d 00 00 00      29
                        byte_t   data[length];          // 2f 63 6f 6d      "/com/deepin/daemon/SystemInfo"
                                                        // 2f 64 65 65
                                                        // 70 69 6e 2f
                                                        // 64 61 65 6d
                                                        // 6f 6e 2f 53
                                                        // 79 73 74 65
                                                        // 6d 49 6e 66
                                                        // 6f
                        byte_t   end_zero[1];           // 00
                        byte_t   end_padding[3]         // 00 00            字符串整体需要以 length 的类型的边界对齐，所以需要填充俩 0
                    };
                } value;
            } item1;
            struct field_item {
                byte_t  code;                           // 03               MEMBER
                struct variant {
                    struct signature {
                        uint8_t length;                 // 01               字符串长度 1
                        byte_t  data[length];           // 73               's'
                        byte_t  end_zero[1];            // 00
                    } sign;
                    byte_t    value_padding_zero[0];    //                  string 以 4 字节边界对齐，这里刚好，无需填充
                    struct string {
                        uint32_t length;                // 03 00 00 00      长度 0x03
                        byte_t   data[length];          // 47 65 74         "Get"
                        byte_t   end_zero[1];           // 00
                    };
                } value;
            } item2;
                                                        // 00 00 00 00      D-Bus 的 struct 需要以 8 字节边界对齐，所以这里填充 4 字节的 0
            struct field_item {
                byte_t  code;                           // 02               INTERFACE
                struct variant {
                    struct signature {
                        uint8_t length;                 // 01               字符串长度 1
                        byte_t  data[length];           // 73               's'
                        byte_t  end_zero[1];            // 00
                    } sign;
                    byte_t    value_padding_zero[0];    //                  string 以 4 字节边界对齐，这里刚好，无需填充
                    struct string {
                        uint32_t length;                // 1f 00 00 00      31
                        byte_t   data[length];          // 6f 72 67 2e      "org.freedesktop.DBus.Properties"
                                                        // 66 72 65 65
                                                        // 64 65 73 6b
                                                        // 74 6f 70 2e
                                                        // 44 42 75 73
                                                        // 2e 50 72 6f
                                                        // 70 65 72 74
                                                        // 69 65 73 00
                                                        // 06 01 73
                        byte_t   end_zero[1];           // 00
                    };
                } value;
            } item3;
            struct field_item {
                byte_t  code;                           // 06               DESTINATION
                struct variant {
                    struct signature {
                        uint8_t length;                 // 01               字符串长度 1
                        byte_t  data[length];           // 73               's'
                        byte_t  end_zero[1];            // 00
                    } sign;
                    byte_t    value_padding_zero[0];    //                  string 以 4 字节边界对齐，这里刚好，无需填充
                    struct string {
                        uint32_t length;                // 05 00 00 00      长度 0x05
                        byte_t   data[length];          // 3a 31 2e 32      ":1.27"
                                                        // 37
                        byte_t   end_zero[1];           // 00
                                                        //                  到这里 header 已经结束，所以虽然 string 末尾没有到边界，依然不需要再填充
                    };
                } value;
            } item4;
        } fields;
    };
    byte_t   padding[3];                                // 00 00            字符串整体需要以 length 的类型的边界对齐，所以需要填充俩 0
    struct string {
        uint32_t length;                                // 1c 00 00 00      长度 28
        byte_t   data[length];                          // 63 6f 6d 2e      "com.deepin.daemon.SystemInfo"
                                                        // 64 65 65 70
                                                        // 69 6e 2e 64
                                                        // 61 65 6d 6f
                                                        // 6e 2e 53 79
                                                        // 73 74 65 6d
                                                        // 49 6e 66 6f
        byte_t   end_zero[1];                           // 00
        byte_t   end_padding[3]                         // 00 00 00         字符串整体需要以 length 的类型的边界对齐，所以需要填充仨 0
    };
    struct string {
        uint32_t length;                                // 09 00 00 00      长度 9
        byte_t   data[length];                          // 63 6f 6d 2e      "Processor"
        byte_t   end_zero[1];                           // 00
    };
                                                        //                  已经结束，无需填充
