###socks5原理
*介绍一下socks5协议： SOCKS协议位于传输层(TCP/UDP等)与应用层之间，其工作流程为:

1.client向proxy发出请求信息，用以协商传输方式
    
    * client连接proxy的第一个报文信息，进行认证机制协商
    +------------------------------------------+
    | version   |   nmethod  |    methods      |
    +-----------+------------+-----------------+
    | 1 bytes   |  1 bytes   | 1~255 bytes     +
    +------------------------------------------+
    *一般是 hex: 05 01 00 即：版本5，1种认证方式，NO AUTHENTICATION REQUIRED(无需认证 0x00)
    
2.proxy作出应答

    *proxy从methods字段中选中一个字节(一种认证机制)，并向Client发送响应报文
    +------------------------+
    |  version  |  methods   |
    +-----------+------------+
    |     1     |      1     |
    +------------------------+
    *一般是 hex: 05 00 即：版本5，无需认证
    
3.client接到应答后向proxy发送目的主机（destination server)的ip和port

    *认证机制相关的子协商完成后，client提交转发请求
    +----------------------------------------------------------+
    |  VER  |  CMD  |  RSV  |  ATYP  |  DST.ADDR  |  DST.PORT  |
    +-------+-------+-------+--------+------------+------------+
    |   1   |   1   | 0x00  |    1   |  variable  |      2     |
    +----------------------------------------------------------+
    *前3个一般是 hex: 05 01 00 地址类型可以是 * 0x01 IPv4地址 * 0x03 FQDN(全称域名) * 0x04 IPv6地址
    *对应不同的地址类型，地址信息格式也不同： * IPv4地址，这里是big-endian序的4字节数据 * FQDN，比如”www.nsfocus.net”，
    这里将是:0F 77 77 77 2E 6E 73 66 6F 63 75 73 2E 6E 65 74。注意，第一字节是长度域 * IPv6地址，这里是16字节数据。

4.proxy评估该目的主机地址，返回自身IP和port，此时C/P连接建立。

    *proxy评估来自client的转发请求并发送响应报文
    +----------------------------------------------------------------+
    |  VER  |     REP     |  RSV  |  ATYP  |  BND.ADDR  |  BND.PORT  |
    +-------+-------------+-------+--------+------------+------------+
    |   1   | 1(response) | 0x00  |    1   |  variable  |      2     |
    +----------------------------------------------------------------+
    *proxy可以靠DST.ADDR、DST.PORT、SOCKSCLIENT.ADDR、SOCKSCLIENT.PORT进行评估，以决定建立到转发目的地的
    *TCP连接还是拒绝转发。若允许则响应包的REP为0，非0则表示失败（拒绝转发或未能成功建立到转发目的地的TCP连接）

5.proxy与dst server连接

6.proxy将client发出的信息传到server，将server返回的信息转发到client。代理完成


# shadowsocks-php
A php port of shadowsocks based on Workerman

# Config
App/config.php

## Start

php start.php start -d

## Stop

php start.php stop

## Status

php start.php status

原ShadowSocks协议的TCP连接部分，使用非常简单的协议，加密前是这样子的：
地址类型(1byte)|地址(不定长)|端口(2byte)
其中，地址类型有三个可能取值：1，3，4，分别对应IPv4，Remote DNS，IPv6
地址的长度就按这个地址类型取值来决定，后面这些不详细介绍了，不是关键的东西。

而加密的时候，会根据不同的加密算法，在前面添加一些随机字节，再进行加密，以使得即使发送相同的数据，加密的结果也不会相同。但是，同一个加密算法下，前面添加的随机字节的长度是固定的，而最常用的加密算法只有rc4-md5和aes系列
（aes-128-cf8和aes-256-cfb等等），而非常的不巧，这些加密算法在使用的时候前面均添加16字节，于是前文所说的表示地址类型的那个字节的位置是完全固定的（在第17字节上）。

服务端为了判断数据是否有效，判断的依据就是表示地址类型的那个字节，看它是不是在那三个可能取值，如果不是，立即断开连接，如果是，就尝试解析后面的地址和端口进行连接。