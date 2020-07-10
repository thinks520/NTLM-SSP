# NTLM认证协议与SSP\(NTLM中高级进阶\)

内容参考原文链接：[http://davenport.sourceforge.net/ntlm.html](http://davenport.sourceforge.net/ntlm.html) 翻译人：rootclay（香山）[https://github.com/rootclay](https://github.com/rootclay)

## 说明

本文是一篇NTLM中高级进阶进阶文章，文中大部分参考来自于[Sourceforge](http://davenport.sourceforge.net/ntlm.html)，原文中已经对NTLM讲解非常详细，在学习的过程中思考为何不翻译之，做为学习和后续回顾的文档，并在此基础上添加自己的思考，因此出现了这篇文章，在翻译的过程中会有部分注解与新加入的元素，后续我也会在Github对此文进行持续性的更新NTLM以及常见的协议中高级进阶并计划开源部分协议调试工具，望各位issue勘误。

## 摘要

本文旨在以中级到高级的详细级别描述NTLM身份验证协议\(authentication protocol\)和相关的安全支持提供程序功能\(security support provider functionality\)，作为参考。希望该文档能发展成为对NTLM的全面描述。目前，无论是在作者的知识还是在文档方面，都存在遗漏，而且几乎可以肯定的说本文是不准确的。但是，该文档至少应能够为进一步研究提供坚实的基础。本文提供的信息用作在开放源代码jCIFS库中实现NTLM身份验证的基础，该库可从 [http://jcifs.samba.org获得。本文档基于作者的独立研究，并分析了\[Samba\]\(http://www.samba.org/\)软件套件。](http://jcifs.samba.org获得。本文档基于作者的独立研究，并分析了[Samba]%28http://www.samba.org/%29软件套件。)

## 什么是NTLM？

NTLM是一套身份验证和会话安全协议，用于各种Microsoft网络协议的实现中（注：NTLM为嵌套协议，被嵌套在各种协议，如HTTP、SMB、SMTP等协议中），并由NTLM安全支持提供程序（"NTLMSSP"）支持。NTLM最初用于DCE/RPC的身份验证和协商（Negotiate），在整个Microsoft系统中也用作集成的单点登录机制（SSO）。可以认为NTLM是HTTP身份验证的技术栈的一部分。同时，它也用于SMTP，POP3，IMAP（Exchange的所有部分），CIFS/SMB，Telnet，SIP以及其他可能的Microsoft实现中。

NTLM Security Support Provider在Windows Security Support Provider（SSPI）框架内提供身份验证（Authentication），完整性（Signing）和机密性（Sealing）服务。SSPI定义了由支持提供程序\(supporting providers\)实现的一组核心安全功能集；NTLMSSP就是这样的提供程序\(supporting providers\)。SSPI定义并由NTLMSSP实现以下核心操作：

1. 身份验证（Authentication）-NTLM提供了质询响应\(challenge-response\)身份验证机制，在这种机制中，客户端无需向服务器发送密码即可证明其身份。\(注：我们常说的PTH等等操作都发生在这里。\)
2. 签名（Signing）-NTLMSSP提供了一种对消息应用数字"签名"的方法。这样可以确保已签名的消息未被（偶然或有意地）修改，并且确保签名方知道共享机密。NTLM实现了对称签名方案（消息身份验证码或MAC）；也就是说，有效的签名只能由拥有公共共享Key的各方生成和验证。
3. Sealing（注：找不到合适的词来翻译，可以理解为加密封装）-NTLMSSP实现了对称Key加密机制，该机制可提供消息机密性。对于NTLM，Sealing还意味着签名（已签名的消息不一定是已Sealing的，但是所有已Sealing的消息都已签名）。

Kerberos已取代NTLM成为基于域的方案的首选身份验证协议。但是，Kerberos是需要有受信任的第三方方案，不能在不存在受信任的第三方的情况下使用。例如，成员服务器（不属于域的服务器），本地帐户以及对不受信任域中资源的身份验证。在这种情况下，NTLM仍然是主要的身份验证机制（可能会持续很长时间）。

## NTLM通用术语

在开始深入研究之前，我们需要定义各种协议中使用的一些术语。 由于翻译的原因我们大概约定一些术语： 协商 = Negotiate 质询 = Challenge 响应 = Response 身份验证 = Authentication 签名 = Signing

NTLM身份验证是一种质询-响应方案，由三个消息组成，通常称为Type 1（协商），Type 2（质询）和Type 3（身份验证）。它基本上是这样的：

1. 客户端向服务器发送Type 1消息。它主要包含客户端支持和服务器请求的功能列表。
2. 服务器以Type 2消息响应。这包含服务器支持和同意的功能列表。但是，最重要的是，它包含服务器产生的challenge。
3. 客户用Type 3消息答复质询。其中包含有关客户端的几条信息，包括客户端用户的域和用户名。它还包含对Type 3 challenge 的一种或多种响应。

   Type 3消息中的响应是最关键的部分，因为它们向服务器证明客户端用户已经知道帐户密码。

认证过程建立了两个参与方之间的共享上下文；这包括一个共享的Session Key，用于后续的签名和Sealing操作。

在本文档中，为避免混淆（无论如何，尽可能避免混淆），将遵循以下约定（除了“NTLM2会话响应”身份验证（NTLMv1身份验证的一种变体，与NTLM2会话安全性结合使用）的特殊情况外。）：

* 在讨论身份验证时，协议版本将使用"v编号"。例如" NTLM v1身份验证"。
* 在讨论会话安全性（签名和Sealing）时，"v"将被省略；例如" NTLM 1会话安全性"。 

" short "是一个低位（little-endian，即小端，这在实现协议库时会有小差别，不写代码可以忽略）字节的16位无符号值。例如，表示为short的十进制值"1234" 将以十六进制物理布局为" 0xd204 "。

" long "是32位无符号小尾数。以十六进制表示的长整数十进制值" 1234 "为" 0xd2040000 "。

Unicode字符串是一个字符串，其中每个字符都表示为一个16位的little-endian值（16位UCS-2转换格式，little-endian字节顺序，没有字节顺序标记，没有空终止符）。Unicode中的字符串" hello"将以十六进制表示为" 0x680065006c006c006f00 "。

OEM字符串是一个字符串，其中每个字符都表示为本地计算机的本机字符集（DOS代码页）中的8位值。没有空终止符。在NTLM消息中，OEM字符串通常以大写形式显示。OEM中的字符串" HELLO"将用十六进制表示为" 0x48454c4c4f "。

"安全缓冲区"（security buffer）是用于指向二进制数据缓冲区的结构。它包括：

1. 一个short的内容，包含缓冲区内容的长度（以字节为单位）（可以为零）。
2. 一个short的信息，包含为缓冲区分配的空间（以字节为单位）（大于或等于长度；通常与长度相同）。
3. 一个long，包含到缓冲区开头的偏移量（以字节为单位）（从NTLM消息的开头）。

因此，安全缓冲区" 0xd204d204e1100000 "将被读取为： Length: 0xd204 \(1234 bytes\) Allocated Space: 0xd204 \(1234 bytes\) Offset: 0xe1100000 \(4321 bytes\)

比如下图表示的数据就是一个安全缓冲区： ![-w598](https://p1.ssl.qhimg.com/t0189cb3609b67c976a.jpg)

如果您从消息中的第一个字节开始，并且向前跳过了4321个字节，那么您将位于数据缓冲区的开头。您将读取1234个字节（这是缓冲区的长度）。由于为缓冲区分配的空间也是1234字节，因此您将位于缓冲区的末尾。

## NTLM Message Header Layout（NTLM消息头）

现在，我们准备看一下NTLM身份验证消息头的布局。

所有消息均以NTLMSSP签名开头，该签名（适当地）是以null终止的ASCII字符串" NTLMSSP"（十六进制的" 0x4e544c4d53535000 "）。

![-w572](https://p0.ssl.qhimg.com/t014edb771996e94f6e.jpg)

下一个是包含消息Type （1、2或3）的long。例如，Type 1消息的十六进制Type 为" 0x01000000 "。

![-w622](https://p2.ssl.qhimg.com/t01de539edb2c0528f0.jpg)

这之后是特定于消息的信息，通常由安全缓冲区和消息Flags组成。

### NTLM Flags

消息Flags包含在头的位域中。这是一个 long，其中每个位代表一个特定的Flags。这些内容中的大多数出现在特定消息中，但是我们将全部在这里介绍它们，可以为其余的讨论建立参考框架。下表中标记为"unidentified"或"unknown"的Flags暂时不在作者的知识范围之内。

注：下表第一次使用中文描述，之后不再使用英文描述。

| flag | 名称 | 描述 |
| :--- | :--- | :--- |
| 0x00000001 | Negotiate Unicode | 指示在安全缓冲区数据中支持使用Unicode字符串。 |
| 0x00000002 | Negotiate OEM | 表示支持在安全缓冲区数据中使用OEM字符串。 |
| 0x00000004 | Request Target | 请求将服务器的身份验证领域包含在Type 2消息中。 |
| 0x00000008 | unknown | 该Flags的用法尚未确定。 |
| 0x00000010 | Negotiate Sign | 指定客户端和服务器之间的经过身份验证的通信应带有数字签名（消息完整性）。 |
| 0x00000020 | Negotiate Seal | 指定应该对客户机和服务器之间的已验证通信进行加密（消息机密性）。 |
| 0x00000040 | Negotiate Datagram Style | 指示正在使用数据报认证。 |
| 0x00000080 | Negotiate Lan Manager Key | 指示应使用Lan ManagerSession Key来签名和Sealing经过身份验证的通信。 |
| 0x00000100 | Negotiate Netware | 该Flags的用法尚未确定。 |
| 0x00000200 | Negotiate NTLM Key | 指示正在使用NTLM身份验证。 |
| 0x00000400 | Negotiate Only NT （unknown） | 该Flags的用法尚未确定。 |
| 0x00000800 | Negotiate Anonymous | 客户端在Type 3消息中发送以指示已建立匿名上下文。这也会影响响应字段（如" 匿名响应 "部分中所述）。 |
| 0x00001000 | Negotiate OEM Domain Supplied | 客户端在Type 1消息中发送的消息，指示该消息中包含客户端工作站具有成员资格的域的名称。服务器使用它来确定客户端是否符合本地身份验证的条件。 |
| 0x00002000 | Negotiate OEM Workstation Supplied | 客户端在"Type 1"消息中发送以指示该消息中包含客户端工作站的名称。服务器使用它来确定客户端是否符合本地身份验证的条件。 |
| 0x00004000 | Negotiate Local Call | 由服务器发送以指示服务器和客户端在同一台计算机上。表示客户端可以使用已建立的本地凭据进行身份验证，而不是计算对质询的响应。 |
| 0x00008000 | Negotiate Always Sign | 指示应使用"虚拟"签名对客户端和服务器之间的已验证通信进行签名。 |
| 0x00010000 | Target Type Domain | 服务器在Type 2消息中发送以指示目标身份验证领域是域。 |
| 0x00020000 | Target Type Server | 服务器在Type 2消息中发送的消息，指示目标身份验证领域是服务器。 |
| 0x00040000 | Target Type Share | 服务器在Type 2消息中发送以指示目标身份验证领域是共享。大概是用于共享级别的身份验证。用法尚不清楚。 |
| 0x00080000 | Negotiate NTLM2 Key（Negotiate Extended Security） | 说明应使用NTLM2签名和Sealing方案来保护经过身份验证的通信。请注意，这是指特定的会话安全方案，与NTLMv2身份验证的使用无关。但是，此Flags可能会影响响应计算（如" NTLM2会话响应 "部分中所述）。 |
| 0x00100000 | Request Init Response（Negotiate Identify） | 该Flags的用法尚未确定。 |
| 0x00200000 | Request Accept Response（Negotiate 0x00200000） | 该Flags的用法尚未确定。 |
| 0x00400000 | Request Non-NT Session Key | 该Flags的用法尚未确定。 |
| 0x00800000 | Negotiate Target Info | 服务器在Type 2消息中发送的消息，表明它在消息中包含目标信息块。目标信息块用于NTLMv2响应的计算。 |
| 0x01000000 | 未知 | 该Flags的用法尚未确定。 |
| 0x02000000 | 未知 | 该Flags的用法尚未确定。 |
| 0x04000000 | 未知 | 该Flags的用法尚未确定。 |
| 0x08000000 | 未知 | 该Flags的用法尚未确定。 |
| 0x10000000 | 未知 | 该Flags的用法尚未确定。 |
| 0x20000000 | Negotiate 128 | 表示支持128位加密。 |
| 0x40000000 | Negotiate Key Exchange | 指示客户端将在Type 3消息的"Session Key"字段中提供加密的Master Key。 |
| 0x80000000 | Negotiate 56 | 表示支持56位加密。 |

下面使用NTLM认证流程的数据包内容做示例参考学习研究： NTLM Type1 Flag ![-w1202](https://p2.ssl.qhimg.com/t01c7d02d2dbaf15d91.jpg)

NTLM Type2 Flag ![-w1174](https://p0.ssl.qhimg.com/t01df76f3e9e760e056.jpg)

NTLM Type3 Flag ![-w1112](https://p1.ssl.qhimg.com/t01baf73be430bce4e8.jpg)

例如，考虑一条消息，该消息指定Flag：

Negotiate Unicode \(0x00000001\) Request Target \(0x00000004\) Negotiate NTLM \(0x00000200\) Negotiate Always Sign \(0x00008000\)

结合以上flag位" 0x00008205 "。但是这在物理传输时上面的数据将被设置为" 0x05820000 "（因为它以小尾数字节顺序表示，这个非常重要，因为很多时候看到位置不对会产生疑问）。

## Type 1消息

我们来看看Type 1消息：

| 偏移量 | 描述 | 内容 |
| :--- | :--- | :--- |
| 0 | NTLMSSP签名 | Null-terminated ASCII "NTLMSSP" \(0x4e544c4d53535000\) |
| 8 | NTLM消息Type | long \(0x0000001\) |
| 12 | Flags | long |
| （16） | 提供的域（可选） | security buffer |
| （24） | 提供的工作站（可选） | security buffer |
| （32） | 操作系统版本结构（可选） | 8 Bytes |
| （32） （40） | 数据块的开始（如果需要） |  |

Type 1消息从客户端发送到服务器以启动NTLM身份验证。其主要目的是通过Flags指示受支持的选项，从而建立用于认证的"基本规则"。作为可选项，它还可以为服务器提供客户端的工作站名称和客户端工作站具有成员资格的域；服务器使用此信息来确定客户端是否符合本地身份验证的条件。

通常，Type 1消息包含来自以下集合的Flags：

| Flags | 说明 |
| :--- | :--- |
| Negotiate Unicode \(0x00000001\) | The client sets this flag to indicate that it supports Unicode strings. |
| Negotiate OEM \(0x00000002\) | This is set to indicate that the client supports OEM strings. |
| Request Target \(0x00000004\) | This requests that the server send the authentication target with the Type 2 reply. |
| Negotiate NTLM \(0x00000200\) | Indicates that NTLM authentication is supported. |
| Negotiate Domain Supplied \(0x00001000\) | When set, the client will send with the message the name of the domain in which the workstation has membership. |
| Negotiate Workstation Supplied \(0x00002000\) | Indicates that the client is sending its workstation name with the message. |
| Negotiate Always Sign \(0x00008000\) | Indicates that communication between the client and server after authentication should carry a "dummy" signature. |
| Negotiate NTLM2 Key \(0x00080000\) | Indicates that this client supports the NTLM2 signing and sealing scheme; if negotiated, this can also affect the response calculations. |
| Negotiate 128 \(0x20000000\) | Indicates that this client supports strong \(128-bit\) encryption. |
| Negotiate 56 \(0x80000000\) | Indicates that this client supports medium \(56-bit\) encryption. |

提供的域是一个安全缓冲区，其中包含客户机工作站具有其成员资格的域。即使客户端支持Unicode，也始终采用OEM格式。

提供的工作站是包含客户端工作站名称的安全缓冲区。这也是OEM而不是Unicode。

![-w590](https://p3.ssl.qhimg.com/t0117ea937e19c0dee9.jpg)

在Windows的最新更新中引入了OS版本结构。它标识主机的操作系统构建级别，其格式如下：

| Description | Content |
| :--- | :--- |
| 0 | Major Version Number |
| 1 | Minor Version Number |
| 2 | Build Number |
| 4 | Unknown |

![-w583](https://p0.ssl.qhimg.com/t01f3c900e5a6d6c8f2.jpg)

你可以通过CMD运行"winver.exe"来找到操作系统版本。它应该提供类似于以下内容的字符串：

请注意，操作系统版本结构和提供的域/工作站是可选字段。在所有消息类型中发现了三种Type 1型消息：

版本1-完全省略了提供的域和工作站安全缓冲区以及操作系统版本结构。在这种情况下，消息在Flags字段之后结束，并且是固定长度的16字节结构。这种形式通常出现在较旧的基于Win9x的系统中，并且在Open Group的ActiveX参考文档（[第11.2.2节](http://www.opengroup.org/comsource/techref2/NCH1222X.HTM#ntlm.2.2)）中有大致记录。

版本2-存在提供的域和工作站缓冲区，但没有操作系统版本结构。数据块在安全缓冲区标头之后的偏移量32处立即开始。在大多数Windows现成的出厂版本中都可以看到这种形式。

版本3-既提供了域/工作站缓冲区，也提供了OS版本结构。数据块从OS版本结构开始，偏移量为40。此格式是在相对较新的Service Pack中引入的，并且可以在Windows 2000，Windows XP和Windows 2003的当前修补版本中看到。

注：目前一般来说都是版本3了。

因此，"最最少的"格式正确的Type 1消息为：

```text
4e544c4d535350000100000002020000
```

这是"版本1"Type 1消息，仅包含NTLMSSP签名，NTLM消息Type 和最少的Flags集（NegotiateNTLM和NegotiateOEM）。

### Type 1消息示例

请考虑以下十六进制Type 1消息：

```text
4e544c4d53535000010000000732000006000600330000000b000b0028000000 
050093080000000f574f524b53544154494f4e444f4d41494e
```

我们将其分解如下：

| 0 | 0x4e544c4d53535000 | NTLMSSP Signature |
| :--- | :--- | :--- |
| 8 | 0x01000000 | Type 1 Indicator |
| 12 | 0x07320000 | Flags: Negotiate Unicode \(0x00000001\) Negotiate OEM \(0x00000002\) Request Target \(0x00000004\) Negotiate NTLM \(0x00000200\) Negotiate Domain Supplied \(0x00001000\) Negotiate Workstation Supplied \(0x00002000\) |
| 16 | 0x0600060033000000 | Supplied Domain Security Buffer: Length: 6 bytes \(0x0600\) Allocated Space: 6 bytes \(0x0600\) Offset: 51 bytes \(0x33000000\) |
| 24 | 0x0b000b0028000000 | Supplied Workstation Security Buffer: Length: 11 bytes \(0x0b00\) Allocated Space: 11 bytes \(0x0b00\) Offset: 40 bytes \(0x28000000\) |
| 32 | 0x050093080000000f | OS Version Structure: Major Version: 5 \(0x05\) Minor Version: 0 \(0x00\) Build Number: 2195 \(0x9308\) Unknown/Reserved \(0x0000000f\) |
| 40 | 0x574f524b53544154494f4e | Supplied Workstation Data \("WORKSTATION"\) |
| 51 | 0x444f4d41494e | Supplied Domain Data \("DOMAIN"\) |

分析这些信息，我们可以看到：

* 这是一条NTLM Type 1消息（来自NTLMSSP签名和Type 1指示符）。
* 此客户端可以支持Unicode或OEM字符串（同时设置了"Negotiate Unicode"和"Negotiate OEM"Flags）。
* 该客户端支持NTLM身份验证（Negotiate NTLM）。
* 客户端正在请求服务器发送有关身份验证目标的信息（设置了Request Target）。
* 客户端正在运行Windows 2000（5.0），内部版本2195（Windows 2000系统的生产内部版本号）。
* 该客户端正在发送其域" DOMAIN "（设置了"Negotiate Domain Supplied flag"Flags，并且该域名称存在于"提供的域安全性缓冲区"中）。
* 客户端正在发送其工作站名称，即" WORKSTATION "（已设置"Negotiate Workstation Supplied flag"Flags，并且工作站名称出现在"提供的工作站安全缓冲区"中）。

请注意，提供的工作站和域为OEM格式。此外，安全缓冲区数据块的布局顺序并不重要；在该示例中，工作站数据位于域数据之前。

创建Type 1消息后，客户端将其发送到服务器。服务器会像我们刚刚所做的那样分析消息，并创建答复。这将我们带入下一个主题，即Type 2消息。

## Type 2消息

| 偏移量 | Description | Content |
| :--- | :--- | :--- |
| 0 | NTLMSSP Signature | Null-terminated ASCII "NTLMSSP" \(0x4e544c4d53535000\) |
| 8 | NTLM Message Type | long \(0x02000000\) |
| 12 | Target Name | security buffer |
| 20 | Flags | long |
| 24 | Challenge | 8 bytes |
| \(32\) | Context \(optional\) | 8 bytes \(two consecutive longs\) |
| \(40\) | Target Information \(optional\) | security buffer |
| \(48\) | OS Version Structure \(Optional\) | 8 bytes |
| 32 \(48\) \(56\) | start of data block |  |

服务器将Type 2消息发送给客户端，以响应客户端的Type 1消息。它用于完成与客户端的选项Negotiate，也给客户端带来了challenge。它可以选择包含有关身份验证目标的信息。

典型的2类消息flags包括（前面已经翻译过，这里就不翻译了，正好也可以看看原文）：

| Flags | 说明 |
| :--- | :--- |
| Negotiate Unicode \(0x00000001\) | The server sets this flag to indicate that it will be using Unicode strings. This should only be set if the client indicates \(in the Type 1 message\) that it supports Unicode. Either this flag or Negotiate OEM should be set, but not both. |
| Negotiate OEM \(0x00000002\) | This flag is set to indicate that the server will be using OEM strings. This should only be set if the client indicates \(in the Type 1 message\) that it will support OEM strings. Either this flag or Negotiate Unicode should be set, but not both. |
| Request Target \(0x00000004\) | This flag is often set in the Type 2 message; while it has a well-defined meaning within the Type 1 message, its semantics here are unclear. |
| Negotiate NTLM \(0x00000200\) | Indicates that NTLM authentication is supported. |
| Negotiate Local Call \(0x00004000\) | The server sets this flag to inform the client that the server and client are on the same machine. The server provides a local security context handle with the message. |
| Negotiate Always Sign \(0x00008000\) | Indicates that communication between the client and server after authentication should carry a "dummy" signature. |
| Target Type Domain \(0x00010000\) | The server sets this flag to indicate that the authentication target is being sent with the message and represents a domain. |
| Target Type Server \(0x00020000\) | The server sets this flag to indicate that the authentication target is being sent with the message and represents a server. |
| Target Type Share \(0x00040000\) | The server apparently sets this flag to indicate that the authentication target is being sent with the message and represents a network share. This has not been confirmed. |
| Negotiate NTLM2 Key \(0x00080000\) | Indicates that this server supports the NTLM2 signing and sealing scheme; if negotiated, this can also affect the client's response calculations. |
| Negotiate Target Info \(0x00800000\) | The server sets this flag to indicate that a Target Information block is being sent with the message. |
| Negotiate 128 \(0x20000000\) | Indicates that this server supports strong \(128-bit\) encryption. |
| Negotiate 56 \(0x80000000\) | Indicates that this server supports medium \(56-bit\) encryption. |

Target Name是包含身份验证目标信息的安全缓冲区。 这通常是响应客户端请求目标而发送的（通过设置Type 1消息中的Request Target Flags）。 它可以包含域，服务器或（显然）网络共享。 通过Target Type Domain, Target Type Server, and Target Type Share flags指示目标类型。 Target Name可以是Unicode或OEM，如Type 2消息中存在适当的Flags所指示。

challenge是一个8字节的随机数据块。客户端将使用它来制定响应。

设置"Negotiate Local Call"时，通常会填充上下文字段。它包含一个SSPI上下文句柄，该句柄允许客户端"short-circuit"身份验证并有效规避对challenge的响应。从物理上讲，上下文是两个长值。稍后将在"Local Authentication "部分中对此进行详细介绍。

Target information是一个包含目标信息块的安全缓冲区，该缓冲区用于计算 NTLMv2响应（稍后讨论）。它由一系列子块组成，每个子块包括：

| Field | Content | Description |
| :--- | :--- | :--- |
| Type | short | Indicates the type of data in this subblock: 1 \(0x0100\):    Server name 2 \(0x0200\):    Domain name 3 \(0x0300\):    Fully-qualified DNS host name \(i.e., server.domain.com\) 4 \(0x0400\):    DNS domain name \(i.e., domain.com\) |
| Length | short | Length in bytes of this subblock's content field |
| Content | Unicode string | Content as indicated by the type field. Always sent in Unicode, even when OEM is indicated by the message flags. |

前面已经描述了OS版本的结构。

与Type 1消息一样，已经观察到一些Type 2的版本：

版本1-上下文，目标信息和操作系统版本结构均被省略。数据块（仅包含Target Name安全缓冲区的内容）从偏移量32开始。这种形式在较旧的基于Win9x的系统中可见，并且在Open Group的ActiveX参考文档（第11.2.3节）中得到了大致记录。

版本2-存在Context 和 Target Information fields字段，但没有OS版本结构。数据块在目标信息标题之后的偏移量48处开始。在大多数现成的Windows发行版中都可以看到这种形式。

版本3-上下文，目标信息和操作系统版本结构均存在。数据块在OS版本结构之后开始，偏移量为56。同样，缓冲区可能为空（产生零长度的数据块）。这种形式是在相对较新的Service Pack中引入的，并且可以在Windows 2000，Windows XP和Windows 2003的当前修补版本中看到。

最小的Type 2消息如下所示：

```text
4e544c4d53535000020000000000000000000000020200000123456789abcdef
```

该消息包含NTLMSSP签名，NTLM消息Type ，空Target Name，最少Flags（NegotiateNTLM和NegotiateOEM）以及质询。

### Type 2消息示例

让我们看下面的十六进制Type 2消息：

```text
4e544c4d53535000020000000c000c003000000001028100 
0123456789abcdef0000000000000000620062003c000000 
44004f004d00410049004e0002000c0044004f004d004100 
49004e0001000c0053004500520056004500520004001400 
64006f006d00610069006e002e0063006f006d0003002200 
7300650072007600650072002e0064006f006d0061006900 
6e002e0063006f006d0000000000
```

将其分为几个组成部分可以得出：

| 偏移量 | 值 | 说明 |
| :--- | :--- | :--- |
| 0 | 0x4e544c4d53535000 | NTLMSSP Signature |
| 8 | 0x02000000 | Type 2 Indicator |
| 12 | 0x0c000c0030000000 | Target Name Security Buffer: Length: 12 bytes \(0x0c00\) Allocated Space: 12 bytes \(0x0c00\) Offset: 48 bytes \(0x30000000\) |
| 20 | 0x01028100 | Flags: Negotiate Unicode \(0x00000001\) Negotiate NTLM \(0x00000200\) Target Type Domain \(0x00010000\) Negotiate Target Info \(0x00800000\) |
| 24 | 0x0123456789abcdef | Challenge |
| 32 | 0x0000000000000000 | Context |
| 40 | 0x620062003c000000 | Target Information Security Buffer: Length: 98 bytes \(0x6200\) Allocated Space: 98 bytes \(0x6200\) Offset: 60 bytes \(0x3c000000\) |
| 48 | 0x44004f004d004100    49004e00 | Target Name Data \("DOMAIN"\) |
| 60 | 0x02000c0044004f00    4d00410049004e00    01000c0053004500    5200560045005200    0400140064006f00    6d00610069006e00    2e0063006f006d00    0300220073006500    7200760065007200    2e0064006f006d00    610069006e002e00    63006f006d000000    0000 | Target Information Data:      接下一表格 |

Target Information Data：

| 偏移量 | 值 |
| :--- | :--- |
| 0x02000c0044004f00   4d00410049004e00 | Domain name subblock: Type: 2 \(Domain name,   0x0200 \) Length: 12 bytes \( 0x0c00 \) Data: " DOMAIN" |
| 0x01000c0053004500   5200560045005200 | Server name subblock: Type: 1 \(Server name,   0x0100 \) Length: 12 bytes \( 0x0c00 \) Data: " SERVER" |
| 0x0400140064006f00   6d00610069006e00   2e0063006f006d00 | DNS domain name subblock: Type: 4 \(DNS domain name,   0x0400 \) Length: 20 bytes \( 0x1400 \) Data: " domain.com" |
| 0x0300220073006500   7200760065007200   2e0064006f006d00   610069006e002e00   63006f006d00 | DNS server name subblock: Type: 3 \(DNS server name,   0x0300 \) Length: 34 bytes \( 0x2200 \) Data: " server.domain.com" |
| 0x00000000 | Terminator subblock: Type: 0 \(terminator,   0x0000 \) Length: 0 bytes \( 0x0000\) |

对此消息的分析显示：

* 这是一条NTLM 2类消息（来自NTLMSSP Signature and Type 2指示符）。
* 服务器已指示将使用Unicode编码字符串（已设置Negotiate UnicodeFlags）。
* 服务器支持NTLM身份验证（Negotiate NTLM）。
* 服务器提供的Target Name将被填充并代表一个域（"Target Type Domain"Flags已设置并且该域名存在于"Target Name Security Buffer"中）。
* 服务器正在提供目标信息结构（已设置Negotiate目标信息）。此结构存在于目标信息（Target Info）安全缓冲区（域名" DOMAIN "，服务器名称" SERVER "，DNS域名" domain.com "和DNS服务器名称" server.domain.com "）中。
* 服务器生成的challenge是" 0x0123456789abcdef "。
* 空上下文。

请注意，Target Name采用Unicode格式（由"NegotiateUnicode"Flags指定）。

服务器创建Type 2消息后，它将发送给客户端。客户端的Type 3消息中提供了对服务器质询的响应。

## Type 3消息

|  | Description | Content |
| :--- | :--- | :--- |
| 0 | NTLMSSP Signature | Null-terminated ASCII "NTLMSSP" \(0x4e544c4d53535000\) |
| 8 | NTLM Message Type | long \(0x03000000\) |
| 12 | LM/LMv2 Response | security buffer |
| 20 | NTLM/NTLMv2 Response | security buffer |
| 28 | Target Name | security buffer |
| 36 | User Name | security buffer |
| 44 | Workstation Name | security buffer |
| \(52\) | Session Key \(optional\) | security buffer |
| \(60\) | Flags \(optional\) | long |
| \(64\) | OS Version Structure \(Optional\) | 8 bytes |
| 52 \(64\) \(72\) | start of data block |  |

Type 3消息是身份验证的最后一步。此消息包含客户对Type 2质询的响应，这证明客户端无需直接发送密码进行认证而使用NTLM HASH认证。Type 3消息还指示身份验证目标（域或服务器名称）和身份验证帐户的用户名，以及客户端工作站名称。

请注意，Type 3消息中的Flags是可选的；较旧的客户端在消息中既不包含Session Key也不包含Flags。通过实验确定，Type 3 Flags（如果包含）在面向连接的身份验证中不带有任何其他语义。它们似乎对身份验证或会话安全性都没有任何明显的影响。发送Flags的客户端通常会非常紧密地镜像已建立的Type 2设置。可能会将Flags作为已建立选项的"提醒"发送，以允许服务器避免缓存Negotiate的设置。但是，Type 3 Flags在数据报样式身份验证期间是有意义的的 。

LM/LMv2和NTLM/NTLMv2响应是安全缓冲区，其中包含根据用户的密码响应Type 2质询而创建的答复。下一节概述了生成这些响应的过程。

target name是一个安全缓冲区，其中包含身份验证领域，其中身份验证帐户具有成员身份（域帐户的域名，或本地计算机帐户的服务器名）。根据Negotiate的编码，它可以是Unicode或OEM。

user name是包含身份验证帐户名的安全缓冲区。根据Negotiate的编码，它可以是Unicode或OEM。

workstation name是包含客户端工作站名称的安全缓冲区。根据Negotiate的编码，它可以是Unicode或OEM。

session key值在Key交换期间由会话安全性机制使用；"会话安全性"部分将对此进行详细讨论 。

当在Type 2消息中建立了"Negotiate Local Call"时，Type 3消息中的安全缓冲区通常都为空（零长度）。客户端"采用"在Type 2消息中发送的SSPI上下文，从而有效地避免了计算适当响应的需求。

OS版本结构与前面描述的格式相同。

同样，Type 3消息有一些变体：

版本1-Session Key，Flags和OS版本结构被省略。在这种情况下，数据块在"工作站名称"安全缓冲区标头之后的偏移量52处开始。此格式在基于Win9x的较旧系统中可见。

版本2-包含Session Key和Flags，但不包含OS版本结构。在这种情况下，数据块在Flags字段之后的偏移量64处开始。这种形式在大多数现成的Windows版本中都可以看到，并且在Open Group的ActiveX参考文档（第11.2.4节）中有大致记录。。

版本3-Session Key，Flags和OS版本结构均存在。数据块从OS版本结构开始，偏移量为72。此格式是在相对较新的Service Pack中引入的，并且可以在Windows 2000，Windows XP和Windows 2003的当前修补版本中看到。

### 名称可变性（Name Variations）

除了消息的布局变化之外，user和Target Name还可以在Type 3消息中以几种不同的格式显示。通常情况下，"User Name"字段将使用Windows帐户名称填充，而"Target Name"将使用NT域名填充。但是，用户名和/或域也可以采用Kerberos样式的" user@domain.com"格式进行多种组合。已经支持多种变体，并具有一些可能的含义如下表：

| Format | Type 3 Field Content | Notes |
| :--- | :--- | :--- |
| DOMAIN\user | User Name = "user" Target Name = "DOMAIN" | Target Name=“ DOMAIN”这是“常规”格式； 用户名字段包含Windows用户名，目标名包含NT样式的NetBIOS域或服务器名。 |
| domain.com\user | User Name = "user" Target Name = "domain.com" | Target Name=“ domain.com”在此，Type 3消息中的“Target Name”字段填充有DNS域名/领域名称（对于本地计算机帐户，则为标准DNS主机名）。 |
| user@DOMAIN | User Name = "user@DOMAIN" Target Name is empty | Target Name为空在这种情况下，“Target Name”字段为空（零长度），而“用户名”字段使用Kerberos样式的“ user @ realm”格式。 但是，使用NetBIOS域名代替DNS域。 已经观察到，本地计算机帐户不支持此格式。 此外，NTLMv2 / LMv2身份验证似乎不支持此格式。 |
| user@domain.com | User Name = "user@domain.com" Target Name is empty | 类型3消息中的Target Name字段为空； “用户名”字段包含Kerberos样式的“ user @ realm”格式以及DNS域。本地计算机帐户似乎不支持此方式。 |

### 应答challenge（Responding to the Challenge）

客户端创建一个或多个对Type 2质询的响应，并将响应以Type 3消息发送给服务器。有六种Type 的响应：

* LM（LAN管理器）响应-由大多数较旧的客户端发送，这是"原始"响应Type 。
* NTLM响应-这是由基于NT的客户端（包括Windows 2000和XP）发送的。
* NTLMv2响应-一种较新的返回类型，Windows NT Service Pack 4中引入更新的响应Type。它替换了启用了NTLMv2的系统上的NTLM响应。
* LMv2响应-替换NTLMv2系统上的LM响应。
* NTLM2会话响应-在未经NTLMv2身份验证的情况下Negotiate NTLM2会话安全性时使用，此方案会更改LM和NTLM响应的语义。
* 匿名响应\(Anonymous Response\)-建立匿名上下文时使用；不会显示实际凭据，也不会进行真正的身份验证。"Stub"字段显示在Type 3消息中。

有关这些方案的更多信息，强烈建议您阅读Christopher Hertel的[《实现CIFS》](http://ubiqx.org/cifs)，尤其是有关[身份验证的部分](http://ubiqx.org/cifs/SMB.html#SMB.8)。

响应\(responses\)用作客户端拥有密码口令的间接证明。客户端使用密码导出LM和/或NTLM哈希（在下一节中讨论）；这些值依次用于计算对challenge的适当响应。域控制器（或本地计算机帐户的服务器）存储LM和NTLM哈希作为密码；当从客户端收到响应时，这些存储的值将用于计算适当的响应值，并将其与客户端发送的响应值进行比较。匹配会成功验证用户。

请注意，与Unix密码哈希不同，LM和NTLM哈希在响应计算的上下文中是与密码等效的。它们必须受到保护，因为即使不知道实际密码本身，也可以使用它们来通过网络对用户进行身份验证。（注：也就是说知道密码和指导哈希是同样的都可以用于认证）

#### LM响应（The LM Response）

LM响应是由大多数客户端发送的。此方案比NTLM响应要旧，并且安全性较低。虽然较新的客户端支持NTLM响应，但它们通常会同时发送这两个响应以与旧服务器兼容。因此，在支持NTLM响应的许多客户端中，仍然存在LM响应中存在的安全漏洞。

LM响应的计算方式如下（有关 Java中的示例实现，请参阅附录D）：

1. 首先将用户密码（作为OEM字符串）转换为大写。
2. 将其填充为14个字节，不足则用0填充。
3. 将此固定长度的密码分为两个7字节。
4. 这些值做为两个DESKey（每个7字节为一个）。
5. 这些Key中的每一个都用于对特定的ASCII字符串"KGS!@\#$%" 进行DES加密（产生两个8字节密文值）。
6. 将这两个密文值连接起来以形成一个16字节的值-LM哈希。
7. 16字节的LM哈希被空填充为21个字节。
8. 该值分为三个7字节。
9. 这些值用于创建三个DESKey。
10. 这些Key中的每一个都用于对来自Type 2消息的质询进行DES加密（产生三个8字节密文值）。
11. 将这三个密文值连接起来形成一个24字节的值。这就是LM的回应。

    如果用户的密码长度超过15个字符，则主机或域控制器将不会为该用户存储LM哈希。在这种情况下，LM响应不能用于认证用户。仍会生成一个响应并将其放置在LM响应字段中，并使用16字节的空值（0x00000000000000000000000000000000000000）作为计算中的LM哈希。该值将被目标忽略。（注：平时在利用工具时LM字段可以放置空值即可）

最好用一个详细的例子说明响应计算过程。考虑一个密码为" SecREt01 " 的用户，它响应Type 2质询" 0x0123456789abcdef "。

1. 密码（作为OEM字符串）将转换为大写，并以十六进制的形式给出" SECRET01 "（或" 0x5345435245543031 "）。
2. 将其填充为14个字节，为" 0x534543524554303031000000000000 "。
3. 此值分为两个7字节，即" 0x53454352455430 "和" 0x31000000000000 "。
4. 这两个值用于创建两个DESKey。DESKey的长度为8个字节；每个字节包含7位Key材料和1个奇偶校验位（根据基础DES的实现，可以检查或不检查奇偶校验位）。我们的第一个7字节值" 0x53454352455430 "将以二进制形式表示为： 01010011 01000101 01000011 01010010 01000101 01010100 00110000

   此值的未经奇偶校验调整的DESKey为：

   0101001 0 1010001 0 0101000 0 0110101 0 0010010 0 0010101 0 0101000 0 0110000 0

   （奇偶校验位在上方以最后一位显示）。十六进制为" 0x52a2506a242a5060 "。应用奇数奇偶校验以确保每个八位位组中的总置位位数为奇数可得出：

   0101001 0 1010001 0 0101000 1 0110101 1 0010010 1 0010101 0 0101000 1 0110000 1

   这是第一个DESKey（十六进制为" 0x52a2516b252a5161 "）。然后，我们对第二个7字节值" 0x31000000000000 " 应用相同的过程，以二进制表示：

   00110001 00000000 00000000 00000000 00000000 00000000 00000000

   创建一个非奇偶校验的DESKey可以得到：

   0011000 0 1000000 0 0000000 0 0000000 0 0000000 0 0000000 0 0000000 0 0000000 0

   （十六进制为" 0x3080000000000000 "）。调整奇偶校验位可得出：

   0011000 1 1000000 0 0000000 1 0000000 1 0000000 1 0000000 1 0000000 1 0000000 1

   这是我们的第二个DESKey，十六进制为" 0x3180010101010101 "。请注意，如果我们的特定DES实现不强制执行奇偶校验（很多都不强制），则可以跳过奇偶校验调整步骤；然后，将非奇偶校验调整后的值用作DESKey。在任何情况下，奇偶校验位都不会影响加密过程。

5. 我们的每个Key都用于DES加密常量ASCII字符串"KGS!@\#$% "（十六进制为" 0x4b47532140232425 "）。这使我们获得" 0xff3750bcc2b22412 "（使用第一个Key）和" 0xc2265b23734e0dac "（使用第二个Key）。
6. 这些密文值被连接起来以形成我们的16字节LM哈希-" 0xff3750bcc2b22412c2265b23734e0dac "。
7. 将其空填充到21个字节，得到" 0xff3750bcc2b22412c2265b23734e0dac0000000000 "。
8. 该值分为三个7字节的三分之三：" 0xff3750bcc2b224 "，" 0x12c2265b23734e "和" 0x0dac0000000000 "。
9. 这三个值用于创建三个DESKey。使用前面概述的过程，我们的第一个价值是： 11111111 00110111 01010000 10111100 11000010 10110010 00100100

   给我们提供经过奇偶校验调整的DESKey：

   1111111 0 1001101 1 1101010 1 0001011 0 1100110 1 0001010 1 1100100 0 0100100 1

   （十六进制为" 0xfe9bd516cd15c849 "）。第二个值：

   00010010 11000010 00100110 01011011 00100011 01110011 01001110

   结果的关键：

   0001001 1 0110000 1 1000100 1 1100101 1 1011001 1 0001101 0 1100110 1 1001110 1

   （" 0x136189cbb31acd9d "）。最后，第三个值：

   00001101 10101100 00000000 00000000 00000000 00000000 00000000

   给我们：

   0000110 1 1101011 0 0000000 1 0000000 1 0000000 1 0000000 1 0000000 1 0000000 1

   这是第三个DESKey（" 0x0dd6010101010101 "）。

10. 三个Key中的每个Key都用于对来自Type 2消息的质询进行DES加密（在我们的示例中为" 0x0123456789abcdef "）。这给出结果" 0xc337cd5cbd44fc97 "（使用第一个键），" 0x82a667af6d427c6d "（使用第二个键）和" 0xe67c20c2d3e77c56 "（使用第三个键）。
11. 将这三个密文值连接起来以形成24字节LM响应：

    0xc337cd5cbd44fc9782a667af6d427c6de67c20c2d3e77c56

该算法有几个弱点，使其容易受到攻击。尽管在Hertel文本中详细介绍了这些内容，但最突出的问题是：在计算响应之前，密码将转换为大写。

#### NTLM响应\(The NTLM Response\)

NTLM响应由较新的客户端发送。该方案解决了LM响应中的一些缺陷。但是，它仍然被认为是相当薄弱的。此外，NTLM响应几乎总是与LM响应一起发送。可以利用该算法的弱点来获得不区分大小写的密码，并通过反复试验来找到NTLM响应所采用的区分大小写的密码。

NTLM响应的计算方式如下（有关示例Java实现，请参阅附录D）：

1. MD4消息摘要算法（在RFC 1320中描述 ）应用于Unicode大小写混合密码。结果为16字节的值-NTLM哈希。
2. 16字节的NTLM哈希值被空填充为21个字节。
3. 该值分为三个7字节的三分之二。
4. 这些值用于创建三个DESKey（每个7字节的三分之一）。
5. 这些Key中的每一个都用于对来自Type 2消息的质询进行DES加密（产生三个8字节密文值）。
6. 将这三个密文值连接起来形成一个24字节的值。这是NTLM响应。

注意，只有哈希值的计算与LM方案有所不同；响应计算是相同的。为了说明此过程，我们将其应用到之前的示例（密码为"SecREt01" 的用户，响应Type 2质询" 0x0123456789abcdef "）。

1. Unicode混合大小写密码为" 0x53006500630052004500740030003100 "（十六进制）；计算出该值的MD4哈希，结果为" 0xcd06ca7c7e10c99b1d33b7485a2ed808 "。这是NTLM哈希。
2. 将其空填充到21个字节，得到" 0xcd06ca7c7e10c99b1d33b7485a2ed8080000000000 "。
3. 该值分为三个7字节的三分之三：" 0xcd06ca7c7e10c9 "，" 0x9b1d33b7485a2e "和" 0xd8080000000000 "。
4. 这三个值用于创建三个DESKey。我们的第一个值： 11001101 00000110 11001010 01111100 01111110 00010000 11001001

   得出奇偶校验调整后的Key：

   1100110 1 1000001 1 1011001 1 0100111 1 1100011 1 1111000 1 0100001 1 1001001 0

   （十六进制为" 0xcd83b34fc7f14392 "）。第二个值：

   10011011 00011101 00110011 10110111 01001000 01011010 00101110

   给出Key：

   1001101 1 1000111 1 0100110 0 0111011 0 0111010 1 0100001 1 0110100 0 0101110 1

   （" 0x9b8f4c767543685d "）。我们的第三个价值：

   11011000 00001000 00000000 00000000 00000000 00000000 00000000

   产生我们的第三个关键：

   1101100 1 0000010 0 0000000 1 0000000 1 0000000 1 0000000 1 0000000 1 0000000 1

   （十六进制为" 0xd904010101010101 "）。

5. 这三个Key中的每一个都用于对来自Type 2消息（" 0x0123456789abcdef "）的质询进行DES加密。这将产生结果" 0x25a98c1c31e81847 "（使用第一个Key），" 0x466b29b2df4680f3 "（使用第二个Key）和" 0x9958fb8c213a9cc6 "（使用第三个Key）。
6. 这三个密文值连接起来形成24字节NTLM响应：

   0x25a98c1c31e81847466b29b2df4680f39958fb8c213a9cc6

#### NTLMv2响应

NTLM版本2（"NTLMv2"）专门用于解决NTLM中存在的安全问题。启用NTLMv2时，NTLM响应将替换为NTLMv2响应，而LM响应将替换为LMv2响应（我们将在下面讨论）。

NTLMv2响应的计算方式如下（有关 Java中的示例实现，请参阅附录D）：

1. 获取NTLM密码哈希（如前所述，这是Unicode混合大小写密码的MD4摘要）。
2. Unicode大写用户名与Unicode身份验证目标（在Type 3消息的"Target Name"字段中指定的域或服务器名称）串联在一起。请注意，即使已协商使用OEM编码，此计算也始终使用Unicode表示形式。还请注意，用户名将转换为大写，而身份验证目标（authentication target）区分大小写，并且必须与"Target Name"字段中显示的大小写匹配。使用16字节NTLM哈希作为Key，将HMAC-MD5消息认证代码算法（[RFC 2104](http://www.ietf.org/rfc/rfc2104.txt)）应用于该值。结果为16字节的值-NTLMv2哈希。
3. 构造一个称为"Blob"的数据块。Hertel text更详细地讨论了这种结构的格式。简要说明如下：

   | offset | Description | Content |
   | :--- | :--- | :--- |
   | 0 | Blob Signature | 0x01010000 |
   | 4 | Reserved | long \(0x00000000\) |
   | 8 | Timestamp | Little-endian, 64-bit signed value representing the number of tenths of a microsecond since January 1, 1601. |
   | 16 | Client Nonce | 8 bytes |
   | 24 | Unknown | 4 bytes |
   | 28 | Target Information | Target Information block \(from the Type 2 message\). |
   | \(variable\) | Unknown | 4 bytes |

4. 来自Type 2消息的质询与Blob连接在一起。使用16字节NTLMv2哈希（在步骤2中计算）作为Key，将HMAC-MD5消息认证代码算法应用于此值。结果是一个16字节的输出值。
5. 该值与Blob连接起来形成NTLMv2响应。

让我们来看一个例子。由于我们需要更多信息来计算NTLMv2响应，因此我们将使用前面提供的示例中的以下值：

| 名称 | 值 |
| :--- | :--- |
| Target: | DOMAIN |
| Username: | user |
| Password: | SecREt01 |
| Challenge: | 0x0123456789abcdef |
| Target Information: | 0x02000c0044004f004d00410049004e0001000c005300450052005600450052000400140064006f006d00610069006e002e0063006f006d00030022007300650072007600650072002e0064006f006d00610069006e002e0063006f006d0000000000 |

1. Unicode混合大小写密码为" 0x53006500630052004500740030003100 "（十六进制）；计算出该值的MD4哈希，结果为" 0xcd06ca7c7e10c99b1d33b7485a2ed808 "。这是NTLM哈希。
2. Unicode大写的用户名与Unicode身份验证目标连接在一起，并提供" USERDOMAIN "（或十六进制的" 0x55005300450052004200444f004d00410049004e00 "）。使用上一步中的16字节NTLM哈希作为Key，将HMAC-MD5应用于此值，这将产生" 0x04b8e0ba74289cc540826bab1dee63ae "。这是NTLMv2哈希。
3. 接下来，构建Blob。时间戳是其中最繁琐的部分，添加11644473600将使我们在1601年1月1日之后获得秒数（12700317600）。乘以10的7次方（10000000）将得到十分之一微秒（127003176000000000）。作为小端64位值，它是" 0x0090d336b734c301 "（十六进制）。 我们还需要生成一个8字节的随机"客户随机数"；我们将使用不太随机的" 0xffffff0011223344 "。构造其余的Blob很容易；我们只是串联：

   0x01010000 （blob签名）

   0x00000000 （保留值）

   0x0090d336b734c301 （我们的时间戳）

   0xffffff0011223344 （随机的客户随机数）

   0x00000000 （未知，但零将起作用）

   0x02000c0044004f00 （我们的目标信息块） 4d00410049004e00 01000c0053004500 5200560045005200 0400140064006f00 6d00610069006e00 2e0063006f006d00 0300220073006500 7200760065007200 2e0064006f006d00 610069006e002e00 63006f006d000000 0000

   0x00000000 （未知，但零会起作用）

4. 我们将Type 2challenge与Blob连接起来：

   0x0123456789abcdef0101000000000000 0090d336b734c301ffffff0011223344 0000000002000c0044004f004d004100 49004e0001000c005300450052005600 450052000400140064006f006d006100 69006e002e0063006f006d0003002200 7300650072007600650072002e006400 6f006d00610069006e002e0063006f00 6d000000000000000000 使用第2步中的NTLMv2哈希作为Key，将HMAC-MD5应用于该值，即可得到16个字节的值" 0xcbabbca713eb795d04c97abc01ee4983 "。

5. 此值与Blob串联以获得NTLMv2响应：

   0xcbabbca713eb795d04c97abc01ee4983 01010000000000000090d336b734c301 ffffff00112233440000000002000c00 44004f004d00410049004e0001000c00 53004500520056004500520004001400 64006f006d00610069006e002e006300 6f006d00030022007300650072007600 650072002e0064006f006d0061006900 6e002e0063006f006d00000000000000 0000

#### LMv2响应\(The LMv2 Response\)

LMv2响应用于提供与旧服务器的直通身份验证兼容性。与客户端通信的服务器很可能不会实际执行身份验证；而是将响应传递到域控制器进行验证。较旧的服务器仅传递LM响应，并且期望它恰好是24个字节。LMv2响应旨在使此类服务器正常运行。它实际上是一个"微型" NTLMv2响应，如下所示（有关示例Java实现，请参阅附录D）：

1. 将计算NTLM密码哈希（Unicode大小写混合的密码的MD4摘要）。
2. Unicode大写用户名与Type 3消息的"Target Name"字段中显示的Unicode身份验证目标（域或服务器名称）串联。使用16字节NTLM哈希作为Key，将HMAC-MD5消息认证代码算法应用于该值。结果为16字节的值-NTLMv2哈希。
3. 创建一个随机的8字节client nonce（这与NTLMv2 blob中使用的client nonce相同）。
4. 来自Type 2消息的质询与client nonce串联在一起。使用16字节NTLMv2哈希（在步骤2中计算）作为Key，将HMAC-MD5消息认证代码算法应用于此值。结果是一个16字节的输出值。
5. 该值与8字节client nonce串联在一起，以形成24字节LMv2响应。

我们将使用一个久经考验的样本值通过一个简短的示例来说明此过程： 目标： 域 用户名： 用户 密码： SecREt01 challenge： 0x0123456789abcdef

1. Unicode混合大小写密码为" 0x53006500630052004500740030003100 "（十六进制）；计算出该值的MD4哈希，结果为" 0xcd06ca7c7e10c99b1d33b7485a2ed808 "。这是NTLM哈希。
2. Unicode大写的用户名与Unicode身份验证目标连接在一起，并提供" USERDOMAIN "（或十六进制的" 0x55005300450052004200444f004d00410049004e00 "）。使用上一步中的16字节NTLM哈希作为Key，将HMAC-MD5应用于此值，这将产生" 0x04b8e0ba74289cc540826bab1dee63ae "。这是NTLMv2哈希。
3. 创建一个随机的8字节client nonce。在我们的NTLMv2示例中，我们将使用" 0xffffff0011223344 "。
4. 然后，我们将Type 2challenge与客户现时串联起来： 0x0123456789abcdefffffff0011223344

   使用第2步中的NTLMv2哈希作为Key，将HMAC-MD5应用于该值，即可得到16字节的值" 0xd6e6152ea25d03b7c6ba6629c2d6aaf0 "。

5. 此值与client nonce连接在一起，以获得24字节LMv2响应： 0xd6e6152ea25d03b7c6ba6629c2d6aaf0ffffff0011223344

#### NTLM2会话响应\(The NTLM2 Session Response\)

NTLM2会话响应可以与NTLM2会话安全性（session security）结合使用（可通过"Negotiate NTLM2 Key"Flags使用）。这用于在不支持完整NTLMv2身份验证的环境中提供增强的保护，以抵御预计算的字典攻击（尤其是基于Rainbow Table的攻击）。

NTLM2会话响应将替换LM和NTLM响应字段，如下所示（有关 Java中的示例实现，请参阅附录D）：

1. 创建一个随机的8字节client nonce。
2. client nonce被空填充为24个字节。此值放在Type 3消息的LM响应字段中。
3. 来自Type 2消息的质询（challenge）与8字节的client nonce串联在一起以形成session nonce。
4. 将MD5消息摘要算法（[RFC 1321](http://www.ietf.org/rfc/rfc1321.txt) ）应用于session nonce，产生16字节的值。
5. 该值将被截断为8个字节，以形成NTLM2会话哈希。
6. 获得NTLM密码哈希（如所讨论的，这是Unicode混合大小写密码的MD4摘要）。
7. 16字节的NTLM哈希值被空填充为21个字节。
8. 该值分为三个7字节。
9. 这些值用于创建三个DESKey（每个7字节）。
10. 这些Key中的每一个都用于对NTLM2会话散列进行DES加密（产生三个8字节密文值）。
11. 将这三个密文值连接起来形成一个24字节的值。这是NTLM2会话响应，放置在Type 3消息的NTLM响应字段中。

为了用我们先前的示例值（用户密码为" SecREt01 "，响应Type 2质询" 0x0123456789abcdef "）进行演示：

1. 创建一个随机的8字节client nonce；与前面的示例一样，我们将使用" 0xffffff0011223344 "。
2. challenge是将空值填充为24个字节：

   0xffffff001122334400000000000000000000000000000000000000

   此值放在Type 3消息的LM响应字段中。

3. 来自Type 2消息的质询与client nonce串联在一起，形成session nonce（" 0x0123456789abcdefffffff0011223344 "）。
4. 将MD5摘要应用于该随机数将产生16字节的值" 0xbeac9a1bc5a9867c15192b3105d5beb1 "。
5. 它被截断为8个字节，以获得NTLM2会话哈希（" 0xbeac9a1bc5a9867c "）。
6. Unicode大小写混合密码为" 0x53006500630052004500740030003100 "；将MD4摘要应用于此值将为我们提供NTLM哈希（" 0xcd06ca7c7e10c99b1d33b7485a2ed808 "）。
7. 将其空填充到21个字节，得到" 0xcd06ca7c7e10c99b1d33b7485a2ed8080000000000 "。
8. 该值分为三个7字节的三分之三：" 0xcd06ca7c7e10c9 "，" 0x9b1d33b7485a2e "和" 0xd8080000000000 "。
9. 这些值用于创建三个DESKey（如在我们之前的NTLM响应示例中计算的" 0xcd83b34fc7f14392 "，" 0x9b8f4c767543685d "和" 0xd904010101010101 "）。
10. 这三个Key中的每一个都用于对NTLM2会话哈希（" 0xbeac9a1bc5a9867c "）进行DES加密。这将产生结果" 0x10d550832d12b2cc "（使用第一个Key），" 0xb79d5ad1f4eed3df "（使用第二个Key）和" 0x82aca4c3681dd455 "（使用第三个Key）。
11. 这三个密文值被连接起来以形成24字节的NTLM2会话响应：

    0x10d550832d12b2ccb79d5ad1f4eed3df82aca4c3681dd455

    放置在Type 3消息的NTLM响应字段中。

#### 匿名响应\(The Anonymous Response\)

当客户端建立匿名上下文而非真正的基于用户的上下文时，将看到“匿名响应”。 当不需要“已验证”用户的操作需要“占位符”时，通常会看到这种情况。 匿名连接与Windows“来宾”用户不同（后者是实际用户帐户，而匿名连接则根本没有帐户关联）。

在匿名的Type 3消息中，客户端指示“ Negotiate Anonymous”标志。 NTLM响应字段为空（零长度）； LM响应字段包含单个空字节（“ 0x00”）。

### Type 3 消息示例

现在我们已经熟悉了类型3的响应，现在可以检查类型3的消息了：

```text
4e544c4d5353500003000000180018006a00000018001800
820000000c000c0040000000080008004c00000016001600
54000000000000009a0000000102000044004f004d004100
49004e00750073006500720057004f0052004b0053005400
4100540049004f004e00c337cd5cbd44fc9782a667af6d42
7c6de67c20c2d3e77c5625a98c1c31e81847466b29b2df46
80f39958fb8c213a9cc6
```

此消息被分解为：

| 偏移量 | 值 | 说明 |
| :--- | :--- | :--- |
| 0 | 0x4e544c4d53535000 | NTLMSSP Signature |
| 8 | 0x03000000 | Type 3 Indicator |
| 12 | 0x180018006a000000 | LM Response Security Buffer: Length: 24 bytes \(0x1800\) Allocated Space: 24 bytes \(0x1800\) Offset: 106 bytes \(0x6a000000\) |
| 20 | 0x1800180082000000 | NTLM Response Security Buffer: Length: 24 bytes \(0x1800\) Allocated Space: 24 bytes \(0x1800\) Offset: 130 bytes \(0x82000000\) |
| 28 | 0x0c000c0040000000 | Target Name Security Buffer: Length: 12 bytes \(0x0c00\) Allocated Space: 12 bytes \(0x0c00\) Offset: 64 bytes \(0x40000000\) |
| 36 | 0x080008004c000000 | User Name Security Buffer: Length: 8 bytes \(0x0800\) Allocated Space: 8 bytes \(0x0800\) Offset: 76 bytes \(0x4c000000\) |
| 44 | 0x1600160054000000 | Workstation Name Security Buffer: Length: 22 bytes \(0x1600\) Allocated Space: 22 bytes \(0x1600\) Offset: 84 bytes \(0x54000000\) |
| 52 | 0x000000009a000000 | Session Key Security Buffer: Length: 0 bytes \(0x0000\) Allocated Space: 0 bytes \(0x0000\) Offset: 154 bytes \(0x9a000000\) |
| 60 | 0x01020000 | Flags: Negotiate Unicode \(0x00000001\) Negotiate NTLM \(0x00000200\) |
| 64 | 0x44004f004d004100   49004e00 | Target Name Data \("DOMAIN"\) |
| 76 | 0x7500730065007200 | User Name Data \("user"\) |
| 84 | 0x57004f0052004b00   5300540041005400   49004f004e00 | Workstation Name Data \("WORKSTATION"\) |
| 106 | 0xc337cd5cbd44fc97   82a667af6d427c6d   e67c20c2d3e77c56 | LM Response Data |
| 130 | 0x25a98c1c31e81847   466b29b2df4680f3   9958fb8c213a9cc6 | NTLM Response Data |

分析表明：

1. 这是一条NTLM Type 3消息（来自NTLMSSP签名和3类指示符）。
2. 客户端已指示使用Unicode编码字符串（已设置"Negotiate Unicode"Flags）。
3. 客户端支持NTLM身份验证（NegotiateNTLM）。
4. 客户的域是" DOMAIN "。
5. 客户端的用户名是" user "。
6. 客户的工作站是" WORKSTATION "。
7. 客户端的LM响应为" 0xc337cd5cbd44fc9782a667af6d427c6de67c20c2d3e77c56 "。
8. 客户端的NTLM响应为" 0x25a98c1c31e81847466b29b2df4680f39958fb8c213a9cc6 "。
9. 空的Session Key已发送。
10. 收到Type 3消息后，服务器将计算LM和NTLM响应，并将它们与客户端提供的值进行比较。如果它们匹配，则用户已成功通过身份验证。

## NTLM版本2\(NTLM Version 2\)

NTLM版本2包含三种新的响应算法（NTLMv2，LMv2和NTLM2会话响应，如前所述）和新的签名和Sealing方案（NTLM2会话安全性）。NTLM2会话安全性是通过"Negotiate NTLM2 Key"FlagsNegotiate的；但是，可以通过修改注册表来启用NTLMv2身份验证。此外，客户端和域控制器上的注册表设置必须兼容才能成功进行身份验证（尽管NTLMv2身份验证有可能通过较旧的服务器传递到NTLMv2域控制器）。部署NTLMv2所需的配置和计划的结果是，许多主机仅使用默认设置（NTLMv1），而不怎么使用NTLMv2进行身份验证。

Microsoft知识库文章239869 中详细介绍了启用NTLM版本2的说明 。简要地，对注册表值进行了修改： `HKEY_LOCAL_MACHINE\System\CurrentControlSet\Control\LSA\LMCompatibilityLevel` （基于Win9x的系统上的LMCompatibility）。这是一个 REG\_DWORD条目，可以设置为以下值之一：

| Level | Sent by Client | Accepted by Server |
| :--- | :--- | :--- |
| 0 | LM NTLM | LM NTLM LMv2 NTLMv2 |
| 1 | LM NTLM | LM NTLM LMv2 NTLMv2 |
| 2 | NTLM | LM NTLM LMv2 NTLMv2 |
| 3 | LMv2 NTLMv2 | LM NTLM LMv2 NTLMv2 |
| 4 | LMv2 NTLMv2 | NTLM LMv2 NTLMv2 |
| 5 | LMv2 NTLMv2 | LMv2 NTLMv2 |

在所有级别中，都支持NTLM2会话安全性并在可用时进行Negotiate（大多数可用文档表明NTLM2会话安全性仅在级别1和更高级别上启用，但实际上在级别0上也可以看到）。默认情况下，在Windows 95和Windows 98平台上仅支持LM响应。安装Directory Services之后的客户端使NTLMv2也可以在这些主机上使用（并启用LMCompatibility 设置，尽管仅级别0和3可用）。

在级别2中，客户端两次发送NTLM响应（在LM和NTLM响应字段中）。在级别3和更高级别，LMv2和NTLMv2响应分别替换LM和NTLM响应。

协商了NTLM2会话安全性后（由"Negotiate NTLM2 Key"Flags指示），可以在级别0、1和2中使用NTLM2会话响应来代替较弱的LM和NTLM响应。与NTLMv1相比，这可以提供针对基于服务器的预先计算的字典攻击的增强保护。通过向计算中添加随机的客户随机数，可以使客户对给定challenge的响应变得可变。

NTLM2 session response很有趣，因为它可以在支持较新方案的客户端和服务器之间进行Negotiate，即使存在不支持较旧域控制器的情况也是如此。在通常情况下，身份验证事务中的服务器实际上并不拥有用户的密码哈希；而是保留在域控制器中。将计算机加入使用NT风格认证的域时，它会建立到域控制器（俗称" NetLogon pipe"）的经过加密，相互认证的通道。当客户端使用"原始" NTLMv1握手向服务器进行身份验证时，在后台进行以下事务：

1. 客户端发送Type 1消息，其中包含Flags和其他信息，如前所述。
2. 服务器为客户端生成一个质询，并发送包含Negotiate Flags集的Type 2消息。
3. 客户响应challenge，提供LM / NTLM响应。
4. 服务器通过NetLogon管道将质询和客户端响应发送到域控制器。
5. 域控制器使用存储的哈希值和服务器给出的质询来重现身份验证计算。如果它们与响应匹配，则认证成功。
6. 域控制器计算Session Key并将其发送到服务器，该Session Key可用于服务器和客户端之间的后续签名和Sealing操作。

在NTLM2会话响应的情况下，可能已升级了客户端和服务器以允许较新的协议，而域控制器却没有。为了考虑到这种情况，对上述握手进行了如下修改：

1. 客户端发送Type 1消息，在这种情况下，该消息指示"Negotiate NTLM2 Key"Flags。
2. 服务器为客户端生成质询，并发送包含NegotiateFlags集（还包括"Negotiate NTLM2 Key"Flags）的Type 2消息。
3. 客户端响应challenge，在LM字段中提供client nonce，并在NTLM字段中提供NTLM2会话响应（NTLM2 Session Response）。请注意，后者的计算与NTLM响应完全相同，只是客户端没有对服务器质询进行加密，而是对与client nonce连接的服务器质询的MD5哈希进行了加密。
4. 服务器不是将服务器质询直接通过NetLogon管道直接发送到域控制器，而是将服务器质询的MD5哈希与client nonce连接在一起（从LM响应字段中提取）。此外，它还发送客户端响应（照常）。
5. 域控制器使用存储的哈希作为Key对服务器发送的质询字段进行加密，并验证它与NTLM响应字段匹配；因此，客户端已成功通过身份验证。
6. 域控制器计算正常的NTLM用户Session Key并将其发送到服务器；服务器在次要计算中使用它来获取NTLM2会话响应用户Session Key（在后续部分中讨论 ）

   本质上，这允许已升级的客户端和服务器在尚未将域控制器升级到NTLMv2（或者网络管理员尚未将LMCompatibilityLevel注册表设置配置为使用NTLMv2）的网络中使用NTLM2会话响应。

与LMCompatibilityLevel设置相关的是 NtlmMinClientSec和NtlmMinServerSec设置。这些规定了由NTLMSSP建立的NTLM上下文的最低要求。两者都是 REG\_WORD条目，并且是指定以下NTLMFlags组合的位域：

1. Negotiate Sign（0x00000010）-指示必须在支持消息完整性（签名）的情况下建立上下文。
2. Negotiate Seal（0x00000020）-指示必须在支持消息机密性（Sealing）的情况下建立上下文。
3. Negotiate NTLM2 Key（0x00080000）-指示必须使用NTLM2会话安全性来建立上下文。
4. Negotiate 128（0x20000000）-指示上下文必须至少支持128位签名/SealingKey。
5. Negotiate 56（0x80000000）-指示上下文必须至少支持56位签名/SealingKey。

尽管其中大多数都更适用于NTLM2签名和Sealing，但"Negotiate NTLM2 Key"对于身份验证很重要，因为它可以防止与无法NegotiateNTLM2会话安全性的主机建立会话。这用于确保不发送LM和NTLM响应（要求认证在所有情况下至少将使用NTLM2会话响应）。

## NTLMSSP和SSPI

在这一点上，我们将开始研究NTLM如何适应"大局"（big picture）。关于SSPI内容也可以查看本链接中的简单说明[SSPI](https://daiker.gitbook.io/windows-protocol/ntlm-pian/4#0x04-ssp-and-sspi)

Windows提供了一个称为SSPI的安全框架-安全支持提供程序接口。这与GSS-API（通用安全服务应用程序接口，RFC 2743 ）在Microsoft中等效。 ），并允许应用认证，完整性和机密性原语的非常高级的机制无关的方法。SSPI支持多个基础提供程序（Kerberos、Cred SSP、Digest SSP、Negotiate SSP、Schannel SSP、Negotiate Extensions SSP、PKU2U SSP）。其中之一就是NTLMSSP（NTLM安全支持提供程序），它提供了到目前为止我们一直在讨论的NTLM身份验证机制。SSPI提供了一个灵活的API，用于处理不透明的，特定于提供程序的身份验证令牌。NTLM Type 1，Type 2和Type 3消息就是此类令牌，专用于NTLMSSP并由其处理。SSPI提供的API几乎抽象了NTLM的所有细节。应用程序开发人员甚至不必知道正在使用NTLM，并且可以交换另一种身份验证机制（例如Kerberos），而在应用程序级别进行的更改很少或没有更改。

在系统层面，SSP就是一个dll，来实现身份验证等安全功能，实现的身份验证机制是不一样的。比如 NTLM SSP 实现的就是一种 Challenge/Response 验证机制。而 Kerberos 实现的就是基于 ticket 的身份验证机制。我们可以编写自己的 SSP，然后注册到操作系统中，让操作系统支持更多的自定义的身份验证方法。

我们不会对SSPI框架进行深入研究，但这是研究应用于NTLM的SSPI身份验证握手的好方法：

1. 客户端通过SSPI AcquireCredentialsHandle函数为用户获取证书集的表示。
2. 客户端调用SSPI InitializeSecurityContext函数以获得身份验证请求令牌（在我们的示例中为Type 1消息）。客户端将此令牌发送到服务器。该函数的返回值表明身份验证将需要多个步骤。
3. 服务器从客户端接收令牌，并将其用作AcceptSecurityContext SSPI函数的输入 。这将在服务器上创建一个表示客户端的本地安全上下文，并生成一个身份验证响应令牌（Type 2消息），该令牌将发送到客户端。该函数的返回值指示需要客户端提供更多信息。
4. 客户端从服务器接收响应令牌，然后再次调用 InitializeSecurityContext，并将服务器的令牌作为输入传递。这为我们提供了另一个身份验证请求令牌（Type 3消息）。返回值指示安全上下文已成功初始化；令牌已发送到服务器。
5. 服务器从客户端接收令牌，并使用Type 3消息作为输入再次调用 AcceptSecurityContext。返回值指示上下文已成功接受；没有令牌产生，并且认证完成。

### 本地认证（Local Authentication）

我们在讨论的各个阶段都提到了本地身份验证序列。对SSPI有基本的了解后，我们可以更详细地研究这种情况。

基于NTLM消息中的信息，客户端和服务器通过一系列决策来协商本地身份验证。其工作方式如下：

1. 客户端调用AcquireCredentialsHandle函数，通过将null传递给"pAuthData"参数来指定默认凭据。这将获得用于单点登录的登录用户凭据的句柄。
2. 客户端调用SSPI InitializeSecurityContext函数来创建Type 1消息。提供默认凭据句柄时，Type 1消息包含客户端的工作站和域名。这由"Negotiate Domain Supplied"和"Negotiate Workstation Supplied"Flags的存在以及消息中包含已填充的"已提供的域（Supplied Domain）"和"工作站的安全性（Supplied Workstation security）"标记来表明。
3. 服务器从客户端接收Type 1消息，并调用 AcceptSecurityContext。这将在服务器上创建一个代表客户端的本地安全上下文。服务器检查客户端发送的域和工作站信息，以确定客户端和服务器是否在同一台计算机上。如果是这样，则服务器通过在结果2类消息中设置"Negotiate Local Call"Flags来启动本地身份验证。Type 2消息的Context字段中的第一个long填充了新获得的SSPI上下文句柄的"upper"部分（特别是SSPI CtxtHandle结构的" dwUpper"字段）。第二个long在所有情况下，"上下文"字段中的"空白"都为空。（尽管从逻辑上讲，它会假定它应包含上下文句柄的"下部"部分）。
4. 客户端从服务器接收Type 2消息，并将其传递给 InitializeSecurityContext。注意了"Negotiate Local Call"Flags的存在之后，客户端检查服务器上下文句柄以确定它是否代表有效的本地安全上下文。如果无法验证上下文，则身份验证将照常进行-计算适当的响应，并将其包含在Type 3消息中的域，工作站和用户名中。如果来自Type 2消息的安全上下文句柄可以验证，但是，没有准备任何答复。而是，默认凭据在内部与服务器上下文相关联。生成的Type 3消息完全为空，其中包含响应长度为零的安全缓冲区以及用户名，域和工作站。
5. 服务器收到Type 3消息，并将其用作AcceptSecurityContext函数的输入 。服务器验证安全上下文已与用户关联；如果是这样，则认证已成功完成。如果上下文尚未绑定到用户，则身份验证失败。

### 数据报认证（Datagram Authentication）（面向无连接）

数据报样式验证用于通过无连接传输Negotiate NTLM。尽管消息周围的许多语义保持不变，但仍存在一些重大差异：

1. 在第一次调用InitializeSecurityContext的过程中，SSPI不会创建Type 1消息 。
2. 身份验证选项由服务器提供，而不是由客户端请求。
3. Type 3消息中的Flags将会有用（如在面向连接的身份验证中）。

在"normal"（面向连接）身份验证期间，在交换Type 1和Type 2消息期间，所有选项都在客户端和服务器之间的第一个事务中Negotiate。Negotiate的设置由服务器"remembered"，并应用于客户端的Type 3消息。尽管大多数客户端发送带有Type 3消息的Negotiate一致的Flags，但它们未用于连接身份验证。（注：也就是Type3消息的Flag是没有用的）

但是，在数据报身份验证中，规则发生了一些变化。为了减轻服务器跟踪Negotiate选项的需要（如果没有持久连接，这将变得困难），将Type 1消息完全删除。服务器生成包含所有受支持Flags的Type 2消息（当然还有质询）。然后，客户端决定它将支持哪些选项，并以Type 3消息进行答复，其中包含对质询的响应和一组选定Flags。数据报认证的SSPI握手序列如下：

1. 客户端调用AcquireCredentialsHandle以获得用户证书集的表示。
2. 客户端调用InitializeSecurityContext，并通过fContextReq参数将 ISC\_REQ\_DATAGRAMFlags作为上下文要求传递。这将启动客户端的安全上下文的建设，但并没有产生令牌的请求（Type 1的消息）。
3. 服务器调用AcceptSecurityContext函数，指定 ASC\_REQ\_DATAGRAM上下文要求Flags并传入空输入令牌。这将创建本地安全上下文，并生成身份验证响应令牌（Type 2消息）。此Type 2消息将包含"Negotiate数据报样式"Flags，以及服务器支持的所有Flags。照常发送给客户端。
4. 客户端收到Type 2消息，并将其传递给 InitializeSecurityContext。客户端从服务器提供的选项中选择适当的选项（包括必须设置的"Negotiate数据报样式"），创建对质询的响应，并填充Type 3消息。然后，该消息将中继到服务器。
5. 服务器将Type 3消息传递到AcceptSecurityContext 函数中。根据客户端选择的Flags来处理消息，并且上下文被成功接受。

与SSPI一起使用时，显然无法产生数据报样式的Type 1消息。但是，有趣的是，我们可以通过巧妙地操纵NTLMSSP令牌来产生我们自己的数据报Type 1令牌，从而在较低级别上"诱导"数据报语义。

这可以通过在将令牌传递到服务器之前，在面向连接的SSPI握手中在第一个InitializeSecurityContext调用产生的Type 1消息上设置"NegotiateNegotiate Datagram Style"Flags来实现。当将修改后的Type 1消息传递到 AcceptSecurityContext函数中时，服务器将采用数据报语义（即使未指定ASC\_REQ\_DATAGRAM）。这将产生设置了"Negotiate Datagram Style"Flags的2类消息，但与通常会生成的面向连接的消息相同；也就是说，在构造Type 2消息时会考虑客户端发送的Type 1Flags，而不是简单地提供所有受支持的选项。

然后，客户端可以使用此Type 2令牌调用InitializeSecurityContext。请注意，客户端仍处于面向连接的模式。生成的Type 3消息将忽略应用于Type 2消息的"Negotiate Datagram Style"Flags。但是，服务器正在执行数据报语义，并且现在将要求正确设置Type 3Flags。在将"Negotiate Datagram Style"Flags添加到Type 3消息之前，将其手动发送到服务器之前，可以使服务器使用修改后的令牌成功调用 AcceptSecurityContext。

这样可以成功进行身份验证；"篡改"Type 1消息有效地将服务器切换到数据报式身份验证，其中将观察并强制使用Type 3Flags。目前没有已知的实际用途，但是它确实演示了可以通过策略性地处理NTLM消息来观察到的一些有趣和意外的行为。

## 会话安全性-签名和盖章概念（Session Security - Signing & Sealing Concepts）

除了SSPI身份验证服务，还提供了消息完整性和机密性功能。这也由NTLM安全支持提供程序实现。"签名"由SSPI MakeSignature函数执行，该函数将消息验证码（MAC）应用于消息（message）。收件人可以对此进行验证，并且可以强有力地确保消息在传输过程中没有被修改。签名是使用发送方和接收方已知的Key生成的；MAC只能由拥有Key的一方来验证（这反过来可以确保签名是由发送方创建的）。"Sealing"由SSPI EncryptMessage执行功能。这会对消息应用加密，以防止传输中的第三方查看它（类似HTPPS）；NTLMSSP使用多种对称加密机制（使用相同的Key进行解密和加密）。

NTLM身份验证过程的同时会建立用于签名和Sealing的Key。除了验证客户端的身份外，身份验证握手还在客户端和服务器之间建立了一个上下文，其中包括在各方之间签名和Sealing消息所需的Key。我们将讨论这些Key的产生以及NTLMSSP用于签名和Sealing的机制。

在签名和盖章过程中采用了许多关键方案。我们将首先概述不同Type的Key和核心会话安全性概念。

### The User Session Key

这是会话安全中使用的基本Key Type。有很多变体：

* LM User Session Key
* NTLM User Session Key
* LMv2 User Session Key
* NTLMv2 User Session Key
* NTLM2 Session Response User Session Key

所使用的推导方法取决于Type 3消息中发送的响应。这些变体及其计算概述如下。

#### LM User Session Key

仅在提供LM响应时（即，对于Win9x客户端）使用。LM用户Session Key的得出如下：

1. 16字节LM哈希（先前计算）被截断为8字节。
2. 将其空填充为16个字节。该值是LM用户Session Key。

与LM哈希本身一样，此Key仅响应于用户更改密码而更改。还要注意，只有前7个密码字符输入了Key（请参阅LM响应的计算过程 ； LM用户Session Key是LM哈希的前半部分）。此外，Key空间实际上要小得多，因为LM哈希本身基于大写密码。所有这些因素加在一起使得LM用户Session Key非常难以抵抗攻击。

#### NTLM User Session Key

客户端发送NTLM响应时，将使用此变体。Key的计算非常简单：

1. 获得NTLM哈希（Unicode大小写混合的密码的MD4摘要，先前已计算）。
2. MD4消息摘要算法应用于NTLM哈希，结果为16字节。这是NTLM用户Session Key。

NTLM用户Session Key比LM用户Session Key有了很大的改进。密码空间更大（区分大小写，而不是将密码转换为大写）；此外，所有密码字符都已输入到Key生成中。但是，它仍然仅在用户更改其密码时才更改。这使得离线攻击变得更加容易。

#### LMv2 User Session Key

发送LMv2响应（但不发送NTLMv2响应）时使用。派生此Key有点复杂，但并不十分复杂：

1. 获得NTLMv2哈希（如先前计算的那样）。
2. 获得LMv2client nonce（用于LMv2响应）。
3. 来自Type 2消息的质询与client nonce串联在一起。使用NTLMv2哈希作为Key，将HMAC-MD5消息认证代码算法应用于此值，从而得到16字节的输出值。
4. 再次使用NTLMv2哈希作为Key，将HMAC-MD5算法应用于该值。结果为16个字节的值是LMv2 User Session Key。

LMv2 User Session Key相对于基于NTLMv1的Key提供了一些改进。它是从NTLMv2哈希派生而来的（它本身是从NTLM哈希派生的），它特定于用户名和域/服务器。此外，服务器质询和client nonce都为Key计算提供输入。Key计算也可以简单地表示为LMv2响应的前16个字节的HMAC-MD5摘要（使用NTLMv2哈希作为Key）。

#### NTLMv2 User Session Key

发送NTLMv2响应时使用。该Key的计算与LMv2用户Session Key非常相似：

1. 获得NTLMv2哈希（如先前计算的那样）。
2. 获得NTLMv2"blob"（与NTLMv2响应中使用的一样）。
3. 来自Type 2消息的challenge与Blob连接在一起作为待加密值。使用NTLMv2哈希作为Keykey，将HMAC-MD5消息认证代码算法应用于此值，从而得到16字节的输出值。
4. 再次使用NTLMv2哈希作为Key，将HMAC-MD5算法应用于第三步的值。结果为16个字节的值是NTLMv2用户Session Key。

NTLMv2 User Session Key在密码上与LMv2 User Session Key非常相似。可以说是NTLMv2响应的前16个字节的HMAC-MD5摘要（使用NTLMv2哈希作为关键字）。

#### NTLM2 Session Response User Session Key

当NTLMv1身份验证与NTLM2会话安全性一起使用时使用。该Key是从NTLM2会话响应信息中派生的，如下所示：

1. 如前所述，将获得NTLM User Session Key。
2. 获得session nonce（先前已讨论过，这是Type 2质询和NTLM2会话响应中的随机数的串联）。
3. 使用NTLM User Session Key作为Key，将HMAC-MD5算法应用于session nonce。结果为16个字节的值是NTLM2会话响应用户Session Key。

NTLM2会话响应用户Session Key的显着之处在于它是在客户端和服务器之间而不是在域控制器上计算的。域控制器像以前一样导出NTLM用户Session Key，并将其提供给服务器。如果已经与客户端Negotiate了NTLM2会话安全性，则服务器将使用NTLM用户Session Key作为MACKey来获取session nonce的HMAC-MD5摘要。

#### 空用户Session Key（The Null User Session Key）

当执行匿名身份验证时，将使用Null用户Session Key。这很简单；它只有16个空字节（" 0x000000000000000000000000000000000000 "）。

### Lan Manager Session Key

Lan Manager Session Key是User Session Key的替代方法，用于在设置"Negotiate Lan Manager Key" NTLM Flags时派生NTLM1签名和Sealing中的Key。Lan ManagerSession Key的计算如下：

1. 16字节LM哈希（先前计算）被截断为8字节。
2. 这将填充为14个字节，其值为"0xbdbdbdbdbdbdbd "。
3. 该值分为两个7字节的一半。
4. 这些值用于创建两个DESKey（每个7字节的一半为一个）。
5. 这些Key中的每一个都用于对LM响应的前8个字节进行DES加密（导致两个8字节密文值）。
6. 这两个密文值连接在一起形成一个16字节的值-Lan ManagerSession Key。

请注意，Lan ManagerSession Key基于LM响应（而不是简单的LM哈希），这意味着它将响应于不同的服务器challenge而更改。与仅基于密码哈希的LM和NTLM用户Session Key相比，这是一个优势。Lan ManagerSession Key会针对每个身份验证操作进行更改，而LM / NTLM用户Session Key将保持不变，直到用户更改其密码为止。因此，Lan ManagerSession Key比LM用户Session Key（两者具有相似的Key强度，但Lan ManagerSession Key可以防止重放攻击）要强得多。NTLM用户Session Key具有完整的128位Key空间，但与LM用户Session Key一样，在每次身份验证时也不相同。

### Key Exchange（密钥交换）

当设置"Negotiate Key Exchange"Flags时，客户端和服务器将会就"secondary"Key达成共识，该Key用于代替Session Key进行签名和Sealing。这样做如下：

1. 客户端选择一个随机的16字节Key（辅助Key，也就是py-ntlm中的exported\_session\_key）。
2. Session Key（User Session Key或Lan Manager Session Key，取决于"Negotiate Lan  ManagerKey"Flags的状态）用于RC4加密辅助Key。结果是一个16字节的密文值（注：也就是py-ntlm中的encrypted\_random\_session\_key）。
3. 此值在Type 3消息的"Session Key"字段中发送到服务器。
4. 服务器接收Type 3消息并解密客户端发送的值（使用带有用户Session Key或Lan ManagerSession Key的RC4）。
5. 结果值是恢复的辅助Key，并代替Session Key进行签名和Sealing。

此外，密钥交换过程巧妙地更改了NTLM2会话安全性中的签名协议（在后续部分中讨论）。

### 弱化Key（Key Weakening）

根据加密输出限制，用于签名和Sealing的Key已被弱化（"weakened"）（注：可能是由于加密性能原因）。Key强度由"Negotiate128"和"Negotiate56"Flags确定。使用的最终Key的强度是客户端和服务器都支持的最大强度。如果两个Flags都未设置，则使用默认的Key长度40位。NTLM1签名和Sealing支持40位和56位Key；NTLM2会话安全性支持40位，56位和不变的128位Key。

## NTLM1会话安全

NTLM1是"原始" NTLMSSP签名和Sealing方案，在未协商"Negotiate NTLM2 Key"Flags时使用。此方案中的Key派生由以下NTLM的Flags驱动：

| Flag | 说明 |
| :--- | :--- |
| Negotiate Lan Manager Key | 设置后，Lan Manager会话密钥将用作签名和密封密钥（而不是用户会话密钥）的基础。 如果未建立，则用户会话密钥将用于密钥派生。（When set, the Lan Manager Session Key is used as the basis for the signing and sealing keys \(rather than the User Session Key\). If not established, the User Session Key will be used for key derivation. ） |
| Negotiate 56 | 表示支持56位密钥。 如果未协商，将使用40位密钥。 这仅适用于与“协商Lan Manager密钥”结合使用； 在NTLM1下，用户会话密钥不会减弱（因为它们已经很弱）。（Indicates support for 56-bit keys. If not negotiated, 40-bit keys will be used. This is only applicable in combination with "Negotiate Lan Manager Key"; User Session Keys are not weakened under NTLM1 \(as they are already weak\).） |
| Negotiate Key Exchange | 表示将执行密钥交换以协商用于签名和密封的辅助密钥。（Indicates that key exchange will be performed to negotiate a secondary key for signing and sealing.） |

### NTLM1Key派生

要产生或是派生NTLM1Key本质上是一个三步过程：

1. Master key negotiation
2. Key exchange
3. Key weakening

#### Master key negotiation

第一步是Negotiate128位"Master Key"，从中将得出最终的签名和Sealing的Key。这是由NTLMFlags"Negotiate Lan Manager Key"驱动的；如果设置，则Lan ManagerSession Key将用作Master Key。否则，将使用适当的用户Session Key。

例如，考虑我们的示例用户，其密码为" SecREt01"。如果未设置"Negotiate Lan Manager"Key，并且在Type 3消息中提供了NTLM响应，则将选择NTLM用户Session Key作为Master Key。这是通过获取NTLM哈希的MD4摘要（本身就是Unicode密码的MD4哈希）来计算的：

0x3f373ea8e4af954f14faa506f8eebdc4

#### 密钥交换（Key exchange）

如果设置了"Negotiate Key exchange"Flags，则客户端将使用新的Master Key（使用先前选择的Master Key进行RC4加密）填充Type 3消息中的"Session Key"字段。服务器将解密该值以接收新的Master Key。

例如，假定客户端选择随机Master Key" 0xf0f0aabb00112233445566778899aabb "。客户端将使用先前Negotiate的Master Key（" 0x3f373ea8e4af954f14faa506f8eebdc4 "）做为Key使用RC4加密此随机Master Key，以获取该值：

0x1d3355eb71c82850a9a2d65c2952e6f3

它在Type 3消息的"Session Key"字段中发送到服务器。服务器RC4-使用旧的Master Key对该值解密，以恢复客户端选择的新的Master Key（" 0xf0f0aabb00112233445566778899aabb "）。

#### 弱化Key（Key Weakening）

最后，关键是要弱化以遵守出口限制。NTLM1支持40位和56位Key。如果设置了" Negotiate 56" NTLMFlags，则128位Master Key将减弱为56位；如果不设置，它将被削弱到40位。请注意，仅在使用Lan Manager Session Key（设置了"NegotiateLan ManagerKey"）时，才在NTLM1下采用Key弱化功能。LM和NTLM 的 User Session Key基于密码散列，而不是响应。给定的密码将始终导致NTLM1下具有相同的用户Session Key。显然不需要弱化，因为给定用户的密码哈希可以轻松恢复User Session Key。

NTLM1下的Key弱化过程如下：

* 要生成56位Key，Master Key将被截断为7个字节（56位），并附加字节值" 0xa0 "。
* 要生成40位Key，Master Key将被截断为5个字节（40位），并附加三个字节的值" 0xe538b0 "。

以Master Key" 0x0102030405060708090a0b0c0d0e0f00 "为例，用于签名和Sealing的40位Key为" 0x0102030405e538b0 "。如果Negotiate了56位Key，则最终Key将为" 0x01020304050607a0 "。

### 签名

一旦协商了Key，就可以使用它来生成数字签名，从而提供消息完整性。通过存在"Negotiate Flags" NTLMFlags来指示对签名的支持。

NTLM1签名（由SSPI MakeSignature函数完成）如下：

1. 使用先前Negotiate的Key初始化RC4密码。只需执行一次（在第一次签名操作之前），并且Key流永远不会重置。
2. 计算消息的CRC32校验和；它表示为长整数（32位Little-Endian值）。
3. 获得序列号；它从零开始，并在每条消息签名后递增。该数字表示为长号。
4. 将四个零字节与CRC32值和序列号连接起来，以获得一个12字节的值（" 0x00000000 " + CRC32（message）+ sequenceNumber）。
5. 使用先前初始化的RC4密码对该值进行加密。
6. 密文结果的前四个字节被伪随机计数器值覆盖（使用的实际值无关紧要）。
7. 将版本号（" 0x01000000 "）与上一步的结果并置以形成签名。

例如，假设我们使用上一个示例中的40位Key对消息" jCIFS "（十六进制" 0x6a43494653 "）进行签名：

1. 计算CRC32校验和（使用小端十六进制" 0xa0310宝宝7 "）。
2. 获得序列号。由于这是我们签名的第一条消息，因此序列号为零（" 0x00000000 "）。
3. 将四个零字节与CRC32值和序列号连接起来，以获得一个12字节的值（" 0x00000000a0310宝宝700000000 "）。
4. 使用我们的Key（" 0x0102030405e538b0 "）对这个值进行RC4加密；这将产生密文" 0xecbf1ced397420fe0e5a0f89 "。
5. 前四个字节被计数器值覆盖；使用" 0x78010900 "给出" 0x78010900397420fe0e5a0f89 "。
6. 将版本图章与结果连接起来以形成最终签名：

    0x0100000078010900397420fe0e5a0f89

下一条签名的消息将接收序列号1；同样，再次注意，用第一个签名初始化的RC4Key流不会为后续签名重置。

### Sealing

除了消息完整性之外，还通过Sealing来提供消息机密性。"Negotiate Sealing" NTLMFlags表示支持Sealing。在具有NTLM提供程序的SSPI下，Sealing总是与签名结合进行（Sealing消息会同时生成签名）。相同的RC4Key流用于签名和Sealing。

NTLM1Sealing（由SSPI EncryptMessage函数完成）如下：

1. 使用先前Negotiate的Key初始化RC4密码。只需执行一次（在第一次Sealing操作之前），并且Key流永远不会重置。
2. 使用RC4密码对消息进行加密；这将产生Sealing的密文。
3. 如前所述，将生成消息的签名，并将其放置在安全尾部缓冲区中。

例如，考虑使用40位Key" 0x0102030405e538b0 " 对消息" jCIFS "（" 0x6a43494653 "）进行Sealing：

1. 使用我们的Key（" 0x0102030405e538b0 "）初始化RC4密码。
2. 我们的消息通过RC4密码传递，并产生密文" 0x86fc55abca "。这是Sealing消息。
3. 我们计算出消息的CRC32校验和（使用小尾数十六进制" 0xa0310宝宝7 "）。
4. 获得序列号。由于这是第一个签名，因此序列号为零（" 0x00000000 "）。
5. 将四个零字节与CRC32值和序列号连接起来，以获得一个12字节的值（" 0x00000000a0310宝宝700000000 "）。
6. 该值是使用来自密码的Key流进行RC4加密的；这将产生密文" 0x452b490efa3e828bcc8affc3 "。
7. 前四个字节被计数器值覆盖；使用" 0x78010900 "给出" 0x78010900fa3e828bcc8affc3 "。
8. 版本标记与结果串联在一起，以形成最终签名，并将其放置在安全尾部缓冲区中： 0x0100000078010900fa3e828bcc8affc3

   整个Sealing结构的十六进制转储为：

   0x86fc55abca0100000078010900fa3e828bcc8affc3

## NTLM2会话安全

NTLM2是更新的签名和Sealing方案，在建立"NegotiateNTLM2Key"Flags时使用。此方案中的Key派生由以下NTLMFlags驱动：

| Flags | 说明 |
| :--- | :--- |
| Negotiate NTLM2 Key | 表示支持NTLM2会话安全性。 |
| Negotiate56 | 表示支持56位Key。如果既未指定此Flags也未指定"Negotiate128"，则将使用40位Key。 |
| Negotiate128 | 表示支持128位Key。如果既未指定此Flags也未指定"Negotiate56"，则将使用40位Key。 |
| NegotiateKey交换 | 表示将执行Key交换以Negotiate用于签名和Sealing的辅助基本Key。 |

### NTLM2Key派生

NTLM2中的Key派生分为四个步骤：

1. Master Key Negotiate
2. Key exchange
3. Key weakening
4. Subkey generation

#### Master Key Negotiate

用户Session Key在NTLM2签名和Sealing中始终用作基本Master Key。使用NTLMv2身份验证时，LMv2或NTLMv2用户Session Key将用作Master Key。当NTLMv1身份验证与NTLM2会话安全一起使用时，NTLM2会话响应用户Session Key将用作Master Key。请注意，NTLM2中使用的用户Session Key比NTLM1对应的用户Session Key或Lan Manager Session Key要强得多，因为它们同时包含服务器质询和client nonce。

#### Key交换

如先前针对NTLM1所讨论的那样执行Key交换。客户端选择一个辅助Master Key，RC4用基本Master Key对其进行加密，然后在Type 3"Session Key"字段中将密文值发送到服务器。这由"Negotiate Key exchange"Flags的存在指示。

#### 弱化Key

NTLM2中的Key弱化仅通过将Master Key（或辅助Master Key，如果执行了Key交换）截短到适当的长度即可完成；例如，Master Key" 0xf0f0aabb00112233445566778899aabb "将减弱为40位，如" 0xf0f0aabb00 "和56位为" 0xf0f0aabb001122 "。请注意，NTLM2支持128位Key。在这种情况下，Master Key直接用于生成子Key（不执行弱化操作）。

仅当生成Sealing子Key时，Master Key才会在NTLM2下减弱。完整的128位Master Key始终用于生成签名Key。

#### 子项生成

在NTLM2下，最多可以建立四个子项。Master Key实际上从未用于Signing或Sealing消息。子项生成如下：

1. 128位（无弱点）Master Key与以空值终止的ASCII常量字符串连接： 客户端到服务器签名的Session Key魔术常数

   以十六进制表示，此常数是：

   0x73657373696f6e206b657920746f2063 6c69656e742d746f2d73657276657220 7369676e696e67206b6579206d616769 6320636f6e7374616e7400 上面的换行符仅用于显示目的。将MD5消息摘要算法应用于此算法，从而得到一个16字节的值。这是客户端Signing Key，客户端使用它来为消息创建签名。

2. 原生的128位Master Key与以空值终止的ASCII常量字符串连接： 服务器到客户端签名的Session Key魔术常数

   以十六进制表示，此常数是：

   0x73657373696f6e206b657920746f2073 65727665722d746f2d636c69656e7420 7369676e696e67206b6579206d616769 6320636f6e7374616e7400 将使用此内容的MD5摘要，从而获得16字节的服务器Signing Key。服务器使用它来创建消息的签名。

3. 弱化的Master Key（取决于Negotiate的是40位，56位还是128位加密）与以空值结尾的ASCII常量字符串连接： 客户端到服务器的Session KeySealingKey魔术常数

   以十六进制表示，此常数是：

   0x73657373696f6e206b657920746f2063 6c69656e742d746f2d73657276657220 7365616c696e67206b6579206d616769 6320636f6e7374616e7400 使用MD5摘要来获取16字节的客户端Sealing Key。客户端使用它来加密消息。

4. 弱化的主键与以空值终止的ASCII常量字符串连接： 服务器到客户端的Session KeySealingKey魔术常数

   以十六进制表示，此常数是：

   0x73657373696f6e206b657920746f2073 65727665722d746f2d636c69656e7420 7365616c696e67206b6579206d616769 6320636f6e7374616e7400 应用MD5摘要算法，产生16字节的服务器Sealing Key。服务器使用此Key来加密消息。

#### 签名（Signing）

签名支持再次由"Negotiate Signing" NTLMFlags指示。客户端签名是使用客户端签名Key完成的；服务器使用服务器签名Key对消息进行签名。签名Key是从无损的Master Key生成的（如前所述）。

NTLM2签名（由SSPI MakeSignature函数完成）如下：

1. 获得序列号；它从零开始，并在每条消息签名后递增。该数字表示为长整数（32位Little-endian值）。
2. 序列号与消息串联在一起。HMAC-MD5消息认证代码算法使用适当的签名Key应用于此值。这将产生一个16字节的值。
3. 如果已NegotiateKey交换，则使用适当的SealingKey初始化RC4密码。这一次完成（在第一次操作期间），并且Key流永远不会重置。HMAC结果的前八个字节使用此RC4密码加密。如果Key交换还没有经过谈判，省略这个Sealing操作。
4. 将版本号（" 0x01000000 "）与上一步的结果和序列号连接起来以形成签名。

例如，假设我们使用Master Key" 0x0102030405060708090a0b0c0d0e0f00 " 在客户端上对消息" jCIFS "（十六进制" 0x6a43494653 "）进行签名。这与客户端到服务器的签名常量连接在一起，并应用MD5生成客户端签名Key（" 0xf7f97a82ec390f9c903dac4f6aceb132 "）和客户端SealingKey（" 0x2785f595293f3e2813439d73a223810d "）；这些用于签名消息如下：

1. 获得序列号。由于这是我们签名的第一条消息，因此序列号为零（" 0x00000000 "）。
2. 序列号与消息串联在一起： 0x000000006a43494653

   使用客户端签名Key（" 0xf7f97a82ec390f9c903dac4f6aceb132 "）应用HMAC-MD5 。结果是16字节的值" 0x0a003602317a759a720dc9c7a2a95257 "。

3. 使用我们的SealingKey（" 0x2785f595293f3e2813439d73a223810d "）初始化RC4密码。先前结果的前八个字节通过密码传递，产生密文" 0xe37f97f2544f4d7e "。
4. 将版本标记与上一步的结果和序列号连接起来，以形成最终签名：

    0x01000000e37f97f2544f4d7e00000000

#### Sealing

"Negotiate Sealing" NTLMFlags再次表明支持NTLM2中的消息机密性。NTLM2Sealing（由SSPI EncryptMessage函数完成）如下：

1. RC4密码使用适当的SealingKey初始化（取决于客户端还是服务器正在执行Sealing）。只需执行一次（在第一次Sealing操作之前），并且Key流永远不会重置。
2. 使用RC4密码对消息进行加密；这将产生Sealing的密文。
3. 如前所述，将生成消息的签名，并将其放置在安全尾部缓冲区中。请注意，签名操作中使用的RC4密码已经初始化（在前面的步骤中）；它不会为签名操作重置。

例如，假设我们使用Master Key" 0x0102030405060060090090a0b0c0d0e0f00 " 在客户端上Sealing消息" jCIFS "（十六进制" 0x6a43494653 "）。与前面的示例一样，我们使用未减弱的Master Key生成客户端签名Key（" 0xf7f97a82ec390f9c903dac4f6aceb132 "）。我们还需要生成客户SealingKey；我们将假定已经Negotiate了40位弱化。我们将弱化的Master Key（" 0x0102030405 "）与客户端到服务器的Sealing常数连接起来，并应用MD5产生客户端SealingKey（" 0x6f0d9953503333cbe499cd1914fe9ee "）。以下过程用于Sealing消息：

1. RC4密码使用我们的客户SealingKey（" 0x6f0d99535033951cbe499cd1914fe9ee "）初始化。
2. 我们的消息通过RC4密码传递，并产生密文" 0xcf0eb0a939 "。这是Sealing消息。
3. 获得序列号。由于这是第一个签名，因此序列号为零（" 0x00000000 "）。
4. 序列号与消息串联在一起： 0x000000006a43494653

   使用客户端签名Key（" 0xf7f97a82ec390f9c903dac4f6aceb132 "）应用HMAC-MD5 。结果是16字节的值" 0x0a003602317a759a720dc9c7a2a95257 "。

5. 该值的前八个字节通过Sealing密码，得到的密文为" 0x884b14809e53bfe7 "。
6. 将版本标记与结果和序列号连接起来以形成最终签名，该最终签名被放置在安全性尾部缓冲区中： 0x01000000884b14809e53bfe700000000

   整个Sealing结构的十六进制转储为：

   0xcf0eb0a93901000000884b14809e53bfe700000000

## 会话安全主题（Miscellaneous Session Security Topics）

还有其他几个会话安全性主题，这些主题实际上并不适合其他任何地方：

* 数据报的签名和Sealing
* "虚拟"的签名

### 数据报签名和Sealing

在建立数据报上下文时使用此方法（由数据报身份验证握手和"Negotiate Datagram Style"Flags的存在指示）。关于数据报会话安全性的语义有些不同；首次调用SSPI InitializeSecurityContext函数之后（即，在与服务器进行任何通信之前），签名可以立即在客户端上开始。这意味着需要预先安排的签名和Sealing方案（因为可以在与服务器Negotiate任何选项之前创建签名）。数据报会话安全性基于具有密钥交换的40位Lan Manager Session Key NTLM1（尽管可能有一些方法可以通过注册表预先确定更强大的方案）。

在数据报模式下，序列号不递增；它固定为零，每个签名都反映了这一点。同样，每次签名或Sealing操作都会重置RC4Key流。这很重要，因为消息可能容易受到已知的明文攻击。

### "虚拟"签名

如果初始化SSPI上下文而未指定对消息完整性的支持，则使用此方法。如果建立了"始终NegotiateNegotiate" NTLMFlags，则对MakeSignature的调用将成功，并返回常量" signature"：

0x01000000000000000000000000000000

对EncryptMessage的调用通常会成功（包括安全性尾部缓冲区中的"真实"签名）。如果未Negotiate"Negotiate始终签名"，则签名和Sealing均将失败。

## 附录A：链接和参考

请注意，由于Web具有高度动态性和瞬态性，因此这些功能可能可用或可能不可用。

jCIFS项目主页 [http://jcifs.samba.org/](http://jcifs.samba.org/) jCIFS是CIFS / SMB的开源Java实现。本文中提供的信息用作jCIFS NTLM身份验证实现的基础。jCIFS为NTLM HTTP身份验证方案的客户端和服务器端以及非协议特定的NTLM实用程序类提供支持。 Samba主页 [http://www.samba.org/](http://www.samba.org/) Samba是一个开源CIFS / SMB服务器和客户端。实现NTLM身份验证和会话安全性，以及本文档大部分内容的参考。 实施CIFS：通用Internet文件系统 [http://ubiqx.org/cifs/](http://ubiqx.org/cifs/) Christopher R. Hertel撰写的，内容丰富的在线图书。与该讨论特别相关的是有关 身份验证的部分。 Open Group ActiveX核心技术参考（第11章，" NTLM"） [http://www.opengroup.org/comsource/techref2/NCH1222X.HTM](http://www.opengroup.org/comsource/techref2/NCH1222X.HTM) 与NTLM上"官方"参考最接近的东西。不幸的是，它还很旧并且不够准确。 安全支持提供者界面 [http://www.microsoft.com/windows2000/techinfo/howitworks/security/sspi2000.asp](http://www.microsoft.com/windows2000/techinfo/howitworks/security/sspi2000.asp) 白皮书，讨论使用SSPI进行应用程序开发。 HTTP的NTLM身份验证方案 [http://www.innovation.ch/java/ntlm.html](http://www.innovation.ch/java/ntlm.html) 有关NTLM HTTP身份验证机制的内容丰富的讨论。 Squid NTLM认证项目 [http://squid.sourceforge.net/ntlm/](http://squid.sourceforge.net/ntlm/) 为Squid代理服务器提供NTLM HTTP身份验证的项目。 Jakarta Commons HttpClient [http://jakarta.apache.org/commons/httpclient/](http://jakarta.apache.org/commons/httpclient/) 一个开放源Java HTTP客户端，它提供对NTLM HTTP身份验证方案的支持。 GNU加密项目 [http://www.gnu.org/software/gnu-crypto/](http://www.gnu.org/software/gnu-crypto/) 一个开放源代码的Java密码学扩展提供程序，提供了MD4消息摘要算法的实现。 RFC 1320-MD4消息摘要算法 [http://www.ietf.org/rfc/rfc1320.txt](http://www.ietf.org/rfc/rfc1320.txt) MD4摘要的规范和参考实现（用于计算NTLM密码哈希）。 RFC 1321-MD5消息摘要算法 [http://www.ietf.org/rfc/rfc1321.txt](http://www.ietf.org/rfc/rfc1321.txt) MD5摘要的规范和参考实现（用于计算NTLM2会话响应）。 RFC 2104-HMAC：消息身份验证的键哈希 [http://www.ietf.org/rfc/rfc2104.txt](http://www.ietf.org/rfc/rfc2104.txt) HMAC-MD5算法的规范和参考实现（用于NTLMv2 / LMv2响应的计算）。 如何启用NTLM 2身份验证 [http://support.microsoft.com/default.aspx?scid=KB;zh-cn;239869](http://support.microsoft.com/default.aspx?scid=KB;zh-cn;239869) 描述了如何启用NTLMv2身份验证的Negotiate并强制执行NTLM安全Flags。 Microsoft SSPI功能文档 [http://windowssdk.msdn.microsoft.com/en-us/library/ms717571.aspx\#sspi\_functions](http://windowssdk.msdn.microsoft.com/en-us/library/ms717571.aspx#sspi_functions) 概述了安全支持提供程序接口（SSPI）和相关功能。

## 附录B：NTLM的应用协议用法

本节研究了Microsoft的某些网络协议实现中NTLM身份验证的使用。

### NTLM HTTP身份验证

Microsoft已经为HTTP建立了专有的" NTLM"身份验证方案，以向IIS Web服务器提供集成身份验证。此身份验证机制允许客户端使用其Windows凭据访问资源，通常用于公司环境中，以向Intranet站点提供单点登录功能。从历史上看，Internet Explorer仅支持NTLM身份验证。但是，最近，已经向其他各种用户代理添加了支持。

NTLM HTTP身份验证机制的工作方式如下：

1. 客户端从服务器请求受保护的资源：

    GET /index.html HTTP / 1.1

2. 服务器以401状态响应，指示客户端必须进行身份验证。通过" WWW-Authenticate "标头将" NTLM"表示为受支持的身份验证机制。通常，服务器此时会关闭连接： HTTP / 1.1 401未经授权的 WWW身份验证：NTLM 连接：关闭 请注意，如果Internet Explorer是第一个提供的机制，它将仅选择NTLM。这与RFC 2616不一致，RFC 2616指出客户端必须选择支持最强的身份验证方案。
3. 客户端使用包含Type 1消息参数的" Authorization "标头重新提交请求。Type 1消息经过Base-64编码以进行传输。从这一点开始，连接保持打开状态。关闭连接需要重新验证后续请求。这意味着服务器和客户端必须通过HTTP 1.0样式的" Keep-Alive"标头或HTTP 1.1（默认情况下采用持久连接）来支持持久连接。相关的请求标头显示如下（下面的" Authorization "标头中的换行符仅用于显示目的，在实际消息中不存在）： GET /index.html HTTP / 1.1 授权：NTLM TlRMTVNTUAABAAAABzIAAAYABgArAAAACwALACAAAABXT1 JLU1RBVElPTkRPTUFJTg ==
4. 服务器以401状态答复，该状态在" WWW-Authenticate "标头中包含Type 2消息（再次，以Base-64编码）。如下所示（" WWW-Authenticate "标头中的换行符仅出于编辑目的，在实际标头中不存在）。

    HTTP / 1.1 401未授权

    WWW验证：NTLM TlRMTVNTUAACAAAADAAMADAAAAABAoEAASNFZ4mrze8 

    AAAAAAAAAAGIAYgA8AAAARABPAE0AQQBJAE4AAgAMAEQATwBNAEEASQBOAAEADABTA 

    EUAUgBWAEUAUgAEABQAZABvAG0AYQBpAG4ALgBjAG8AbQADACIAcwBlAHIAdgBlAHI 

    ALgBkAG8AbQBhAGkAbgAuAGMAbwBtAAAAAAA =

5. 客户端通过使用包含包含Base-64编码的Type 3消息的" Authorization "标头重新提交请求来响应Type 2消息（同样，下面的" Authorization "标头中的换行符仅用于显示目的）：

    GET /index.html HTTP / 1.1 

    授权：NTLM TlRMTVNTUAADAAAAGAAYAGoAAAAYABgAggAAAAwADABAAA 

    AACAAIAEwAAAAWABYAVAAAAAAAAACaAAAAAQIAAEQATwBNAEEASQBOAHUAcwBlAHIA 

    VwBPAFIASwBTAFQAQQBUAEkATwBOAMM3zVy9RPyXgqZnr21CfG3mfCDC0 + d8ViWpjB 

    wx6BhHRmspst9GgPOZWPuMITqcxg ==

6. 最后，服务器验证客户端的Type 3消息中的响应，并允许访问资源。

    HTTP / 1.1 200 OK

此方案与大多数"常规" HTTP身份验证机制不同，因为通过身份验证的连接发出的后续请求本身不会被身份验证；NTLM是面向连接的，而不是面向请求的。因此，对" /index.html " 的第二个请求将不携带任何身份验证信息，并且服务器将不请求任何身份验证信息。如果服务器检测到与客户端的连接已断开，则对" /index.html " 的请求将导致服务器重新启动NTLM握手。

上面的一个显着例外是客户端在提交POST请求时的行为（通常在客户端向服务器发送表单数据时使用）。如果客户端确定服务器不是本地主机，则客户端将通过活动连接启动POST请求的重新认证。客户端将首先提交一个空的POST请求，并在" Authorization "标头中带有Type 1消息。服务器以Type 2消息作为响应（如上所示，在" WWW-Authenticate "标头中）。然后，客户端使用Type 3消息重新提交POST，并随请求发送表单数据。

NTLM HTTP机制也可以用于HTTP代理身份验证。该过程类似，除了：

* 服务器使用407响应代码（指示需要进行代理身份验证）而不是401。
* 客户端的1类和3类消息是在" 代理授权 "请求标头中发送的，而不是在" 授权 "标头中发送的。
* 服务器的Type 2质询在" Proxy-Authenticate "响应头中发送（而不是" WWW-Authenticate "）。

在Windows 2000中，Microsoft引入了"Negotiate" HTTP身份验证机制。虽然其主要目的是提供一种通过Kerberos通过Active Directory验证用户身份的方法，但它与NTLM方案向后兼容。当在"传统"模式下使用Negotiate机制时，在客户端和服务器之间传递的标头是相同的，只是将"Negotiate"（而不是" NTLM"）指定为机制名称。

### NTLM POP3身份验证

Microsoft的Exchange服务器为POP3协议提供了NTLM身份验证机制。这是RFC 1734中记录的与POP3 AUTH命令 一起使用的专有扩展 。在客户端，Outlook和Outlook Express支持此机制，称为"安全密码身份验证"。

POP3 NTLM身份验证握手在POP3"授权"状态期间发生，其工作方式如下：

1. 客户端可以通过发送不带参数的AUTH命令来请求支持的身份验证机制的列表：

    AUTH

2. 服务器以成功消息响应，然后是受支持机制的列表；此列表应包含" NTLM "，并以包含单个句点（" 。 "）的行结尾。

    +OK The operation completed successfully.

    NTLM

    .

3. 客户端通过发送一个将NTLM指定为身份验证机制的AUTH命令来启动NTLM身份验证：

    AUTH NTLM

4. 服务器将显示一条成功消息，如下所示。注意，" + "和" OK " 之间有一个空格；RFC 1734指出服务器应以质询进行答复，但是NTLM要求来自客户端的Type 1消息。因此，服务器发送"非challenge"消息，基本上是消息" OK "。

    +OK

5. 然后，客户端发送Type 1消息，以Base-64编码进行传输：

    TlRMTVNTUAABAAAABzIAAAYABgArAAAACwALACAAAABXT1JLU1RBVElPTkRPTUFJTg ==

6. 服务器回复Type 2质询消息（再次，以Base-64编码）。它以RFC 1734指定的质询格式发送（" + "，后跟一个空格，后跟质询消息）。如下所示；换行符是出于编辑目的，不出现在服务器的答复中：
   * TlRMTVNTUAACAAAADAAMADAAAAABAoEAASNFZ4mrze8AAAAAAAAAAGIAYgA8AAAA 

     RABPAE0AQQBJAE4AAgAMAEQATwBNAEEASQBOAAEADABTAEUAUgBWAEUAUgAEABQAZA 

     BvAG0AYQBpAG4ALgBjAG8AbQADACIAcwBlAHIAdgBlAHIALgBkAG8AbQBhAGkAbgAu 

     AGMAbwBtAAAAAAA =
7. 客户端计算并发送Base-64编码的Type 3响应（下面的换行符仅用于显示目的）：

    TlRMTVNTUAADAAAAGAAYAGoAAAAYABgAggAAAAwADABAAAAACAAIAEwAAAAWABYAVA 

    AAAAAAAACaAAAAAQIAAEQATwBNAEEASQBOAHUAcwBlAHIAVwBPAFIASwBTAFQAQQBU 

    AEkATwBOAMM3zVy9RPyXgqZnr21CfG3mfCDC0 + d8ViWpjBwx6BhHRmspst9GgPOZWP 

    uMITqcxg ==

8. 服务器验证响应并指示认证结果：

    +OK User successfully logged on

成功进行身份验证后，POP3会话进入"事务"状态，从而允许客户端检索消息。

### NTLM IMAP身份验证

Exchange提供了一种IMAP身份验证机制，其形式类似于前面讨论的POP3机制。RFC 1730中记录了IMAP身份验证 ；NTLM机制是Exchange提供的专有扩展，并由Outlook客户端家族支持。

握手序列类似于POP3机制：

1. 服务器可以在能力响应中指示对NTLM身份验证机制的支持。连接到IMAP服务器后，客户端将请求服务器功能列表：

    0000 CAPABILITY

2. 服务器以支持的功能列表作为响应；服务器回复中字符串" AUTH = NTLM " 的存在指示了NTLM身份验证扩展名：
   * CAPABILITY IMAP4 IMAP4rev1 IDLE LITERAL+ AUTH=NTLM

     0000 OK CAPABILITY completed.
3. 客户端通过发送 将NTLM指定为身份验证机制的AUTHENTICATE命令来启动NTLM身份验证：

    0001授权NTLM

4. 服务器以一个空的质询作为响应，该质询仅由" + "组成：

    +

5. 然后，客户端发送Type 1消息，以Base-64编码进行传输：

    TlRMTVNTUAABAAAABzIAAAYABgArAAAACwALACAAAABXT1JLU1RBVElPTkRPTUFJTg ==

6. 服务器回复Type 2质询消息（再次，以Base-64编码）。它以RFC 1730指定的质询格式发送（" + "，后跟一个空格，后跟质询消息）。如下所示；换行符是出于编辑目的，不出现在服务器的答复中：
   * TlRMTVNTUAACAAAADAAMADAAAAABAoEAASNFZ4mrze8AAAAAAAAAAGIAYgA8AAAA 

     RABPAE0AQQBJAE4AAgAMAEQATwBNAEEASQBOAAEADABTAEUAUgBWAEUAUgAEABQAZA 

     BvAG0AYQBpAG4ALgBjAG8AbQADACIAcwBlAHIAdgBlAHIALgBkAG8AbQBhAGkAbgAu 

     AGMAbwBtAAAAAAA =
7. 客户端计算并发送Base-64编码的Type 3响应（下面的换行符仅用于显示目的）：

    TlRMTVNTUAADAAAAGAAYAGoAAAAYABgAggAAAAwADABAAAAACAAIAEwAAAAWABYAVA 

    AAAAAAAACaAAAAAQIAAEQATwBNAEEASQBOAHUAcwBlAHIAVwBPAFIASwBTAFQAQQBU 

    AEkATwBOAMM3zVy9RPyXgqZnr21CfG3mfCDC0 + d8ViWpjBwx6BhHRmspst9GgPOZWP 

    uMITqcxg ==

8. 服务器验证响应并指示认证结果：

    0001 OK AUTHENTICATE NTLM completed.

身份验证完成后，IMAP会话将进入身份验证状态。

### NTLM SMTP身份验证

除了为POP3和IMAP提供的NTLM身份验证机制外，Exchange还为SMTP协议提供了类似的功能。这样可以对发送外发邮件的用户进行NTLM身份验证。这是与SMTP AUTH命令一起使用的专有扩展（在 RFC 2554中记录）。

SMTP NTLM身份验证握手的操作如下：

1. 服务器可以在EHLO答复中指示支持NTLM作为身份验证机制。连接到SMTP服务器后，客户端将发送初始EHLO消息：

    EHLO client.example.com

2. 服务器以支持的扩展列表进行响应。NTLM身份验证扩展由其在AUTH机制列表中的存在指示，如下所示。请注意，AUTH 列表发送了两次（一次带有" = "，一次没有）。显然在RFC草案中指定了" AUTH = "形式。发送两种表格都可以确保支持针对该草案实施的客户。

    250-server.example.com Hello \[10.10.2.20\]

    250-HELP

    250-AUTH LOGIN NTLM

    250-AUTH=LOGIN NTLM

    250 SIZE 10240000

3. 客户端通过发送一个AUTH 命令来启动NTLM身份验证，该命令将NTLM指定为身份验证机制，并提供Base-64编码的Type 1消息作为参数：

   AUTH NTLM TlRMTVNTUAABAAAABzIAAAYABgArAAAACwALACAAAABXT1JLU1RBVElPTkRPTUFJTg ==

根据RFC 2554，客户端可以选择不发送初始响应参数（而是仅发送" AUTH NTLM "并等待空服务器质询，然后以Type 1消息答复）。但是，在针对Exchange测试时，这似乎无法正常工作。

1. 服务器回复334响应，其中包含Type 2质询消息（同样是Base-64编码）。如下所示；换行符是出于编辑目的，不出现在服务器的答复中：

    334 TlRMTVNTUAACAAAADAAMADAAAAABAoEAASNFZ4mrze8AAAAAAAAAAGIAYgA8AAAA 

    RABPAE0AQQBJAE4AAgAMAEQATwBNAEEASQBOAAEADABTAEUAUgBWAEUAUgAEABQAZA 

    BvAG0AYQBpAG4ALgBjAG8AbQADACIAcwBlAHIAdgBlAHIALgBkAG8AbQBhAGkAbgAu 

    AGMAbwBtAAAAAAA =

2. 客户端计算并发送Base-64编码的Type 3响应（下面的换行符仅用于显示目的）：

    TlRMTVNTUAADAAAAGAAYAGoAAAAYABgAggAAAAwADABAAAAACAAIAEwAAAAWABYAVA 

    AAAAAAAACaAAAAAQIAAEQATwBNAEEASQBOAHUAcwBlAHIAVwBPAFIASwBTAFQAQQBU 

    AEkATwBOAMM3zVy9RPyXgqZnr21CfG3mfCDC0 + d8ViWpjBwx6BhHRmspst9GgPOZWP 

    uMITqcxg ==

3. 服务器验证响应并指示认证结果：

   235 NTLM authentication successful.

验证后，客户端可以正常发送消息。

## 附录C：NTLMSSP操作分解示例

此部分内容较多，后续持续更新于Github。
