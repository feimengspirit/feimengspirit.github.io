---
layout: post
title: iOS安全系列(一)
description: 系统安全概述
category: blog
tags: iOS security
---

## 概述
Apple在设计iOS时，安全是其核心。不惜耗费数年时间去建立一个新的架构-封闭的生态。每个iOS设备只有集成了软件、硬件和服务一起才能提供最大程度的安全性和透明的用户体验。iOS保护的不仅仅是设备和数据，而是整个生态系统，包括用户本地或联网状态所有服务。

所有iOS设备的主要安全特性(数据加密)都是强制执行的，用户无法进行配置，当然也无法关闭。其他诸如Touch ID等提升用户体验的特性则保留给用户自由设置。

本系列文章将对iOS的安全技术和特性进行详细探讨。主要包含以下几部分内容:

* 系统安全:iOS设备上的安全软件和硬件的一体化集成。
* 加密和数据保护:用于防御设备丢失或被未授权人访问或者篡改的架构和设计。
* App安全:保障多个app能够安全运行。
* 网络安全:提供安全授权和数据加密透传的工业标准网络协议。
* Internet服务:苹果内置的基于网络的消息、同步和备份服务。
* 设备管理:保证设备不被未授权的人使用，设备丢失时远程擦除等。

## 系统安全
系统安全贯彻整个iOS设备上的每个核心组件的软件和硬件。包括启动、软件更新和SE(secure enclave).该架构处于iOS安全的核心，但绝对不会影响设备的可用性。
<img src="/images/blog/iOS/iOS-arch.png" width="50%">
iO设备上软件和硬件的紧密集成保证了系统的每个组件都是可信的，从而保证系统的完整性。从最初的启动到软件更新到第三方app，每个步骤都被分析和诊断，以确保软件爱你和硬件的最优结合以及对资源的合理使用。

### 安全启动链
启动过程的每个组件都被Apple进行了数字签名以保证完整性。从bootloader到内核到内核扩展再到固件，只有该启动链上的每个环节的可信都得到验证，启动才能正常进行。

当iOS设备启动时，处理器首先从Boot ROM中读取可执行代码。这些代码是制造芯片时烧入的，所以是默认可信的。Boot ROM代码中包含了Apple的CA根证书公钥。该公钥由于验证底层的BootLoader(LLB)。这是信任链的头，它确保了以后的每个步骤都必须是Apple签名过的。

当LLB完成任务后，它将负责运行下一阶段的bootloader-iBoot。iBoot则会验证iOS内核的完整性。

安全启动链确保了iOS底层软件不被篡改，并且只能运行在Apple的设备上。

对于使用A7处理器的设备，Secure Enclave协处理器同样使用一个安全启动的过程去确保其软件是经过Apple前面的。

如果启动中的任何一个缓解没有完成，或者其下个环节未能通过验证，启动将被中指。设备会显示”连接iTunes“的字样。这被称作恢复模式，如果Boot ROM甚至未能完成LLB的验证和加载，将进入设备固件升级(DFU)模式，两种情况下，设备都只有通过连接iTunes并且恢复到出厂设置才能继续使用。如果想要手动进入恢复模式，可以参考这里[http://support.apple.com/kb/HT1808][1]

### 系统软件授权
Apple会定期更新软件来修补安全漏洞和提供新特性。这些更新大部分情况下会同时提供给所有支持的设备。用户慧收到iOS更新的通知，以鼓励用户完成更新，尽快弥补漏洞。

之前的安全措施保证了只有经过Apple官方签名的软件才能被安装到iOS设备上。所以为了防止用户将iOS系统降级到存在安全漏洞的低版本，Apple使用了一种称作系统软件授权的过程。对于使用A7处理器的设备，Secure Enclave协处理器也应用了相似的措施防止软件的降级使用。

iOS软件更新可以通过两种方式:

* 通过连接iTunes。这种方式将会完整下载iOS，并降旗拷贝安装到设备上。
* 通过设备本身的无线网络(OTA)。这种方式会采用增量更新的方式，只下载需要的组件即可，节约网络流量。
系统更新是，iTunes(或iOS设备本身)连接到Apple的授权服务器，附带一份经过加密的需要更新的组件列表(如LLB、iBoot、内核和OS映像)，一个用于防止重放攻击的随机值和设备ID(ECID)

授权服务器检查更新列表和现有更新，如果发现匹配，将ECID附加在组件列表当中并且实施签名。再将该结果返回给客户端设备。

启动时的信任链机制会验证服务端下发的组件的签名。这些步骤保证了授权只针对当前请求的设备，更新文件无法拷贝到其他设备进行复用。

### Secure Enclave
Secure Enclave是A7芯片上的一个协处理器，有自己独立的安全启动和软件更新控制。同时提供用于数据保护、密钥管理和数据完整性校验的安全功能。由于是独立的组件，即使iOS内核失陷，Secure Enclave依然可以完好无损。

Secure Enclave的内存是经过加密的，配有一个硬件的随机数生成器。Secure Enclave与处理器的通信基于中断驱动的消息盒子和共享内存。

每个Secure Enclave制造时被赋予了一个唯一的ID(UID),系统的其他组件甚至Apple自己都无法访问到。设备启动时，会生成一个临时密钥，该密钥与UID绑定，被用于加密Secure Enclave的内存空间。Secure Enclave保存的文件也会经另一个与UID绑定的密钥加密。

Secure Enclave负责处理Touch ID传感器的指纹数据，以确定是否与系统中注册的指纹匹配。A7处理器与Touch ID传感器之间通过外围的串行接口进行通信。然后A7处理器将指纹数据输入到
Secure Enclave中。数据经过Touch ID传感器和Secure Enclave的共享密钥协商而来的会话密钥加密。所以A7处理器无法读取到这些数据。

### Touch ID
Touch ID是从iPhone 5s开始引入的指纹识别系统。使得对设备的访问更加便捷。这种计数从不同的角度识别指纹，并且随着用户的使用，也还在不停的训练数据。Touch ID使得复杂的长密码变得可行了，因为用户不必每次输入了。解锁也更加方便。但Touch ID并没有替代passcode.其中还有些边界。

如果要使用Touch ID,用户必须开启passcode。一般情况下只要用户通过指纹就可以实现解锁，但在下面几种情况下，还是必须要输入passcode.

* 设备刚刚启动或者发生了重启；
* 设备超过48小时没有解锁过；
* 5次指纹识别失败以后；
* 当设置或添加了新的指纹之后；
* 设备接受到远程解锁命令；

设置了Touch ID以后，只需轻轻一按home键即可解锁，而未设置则必须每次都输入passcode，这就迫使很多用户将锁屏时间设长，带来安全隐患。Touch ID可以同时设置5个指纹，任何一个随机的手指能够通过的概率是1/50000。由于Touch ID控制了5次失败后必须输入passcode，随意暴力破解难度很大。

当环绕home键的钢圈识别到手指触碰时就会激活指纹传感器，进而出发高级图像扫描阵列扫描指纹，再将扫描的结果输入到Secure Enclave. 指纹数据在Secure Enclave中是经过加密的。分析完成后随即丢弃掉。

Secure Enclave持有数据加密的密钥，当设备锁住后，密钥即被丢弃，文件和keychain都将无法访问，除非用户重新解锁手机。

而当用户开启了Touch ID之后，设备锁住后密钥将不被丢弃，而是被分配给Touch ID子系统的密钥进行加密封装。当用户通过Touch ID解锁了设备后，Touch ID子系统的密钥解密数据加密密钥。Touch ID密钥也是存放在内存中，一旦重启将会丢失掉。

另外，如果设备超过48小时未解锁，或者5次Touch ID解锁均失败后，Secure Enclave也将丢弃密钥。

<div id="ckepop">
<span class="jiathis_txt">分享到：</span>
<a class="jiathis_button_tsina">新浪微博</a>
<a class="jiathis_button_weixin">微信</a>

<a href="http://www.jiathis.com/share" class="jiathis jiathis_txt jiathis_separator jtico jtico_jiathis" target="_blank">更多</a>
<a class="jiathis_counter_style"></a>
</div>
<script type="text/javascript" src="http://v2.jiathis.com/code/jia.js" charset="utf-8"></script>

[1]:	http://support.apple.com/kb/HT1808