# SPDY接入服务

## 概述

> SPDY是Google开发的下一代网络协议，它并不是一种用于替代HTTP的协议，而是对HTTP协议的增强。谷歌表示，引入SPDY协议后，在实验室测试中页面加载速度比原先快64%，并且应用到Gmail中，目前业界支持SPDY的服务器有Netty和Nginx。

* SPDY增加了一个帧层用于多路复用，多个并发流通过一个TCP连接（或者其他可靠传输流）。这个帧层为类似HTTP的请求响应流进行了优化，现在运行在HTTP之上的应用也能运行在SPDY之上，对web应用开发者来说几乎不需要做什么改变。

* SPDY会话在HTTP的基础之上提供了四项改进：

  * 多路复用请求：在单个SPDY连接能并发的发起请求，并不限制请求数；
  * 请求优先级：客户端能请求某个资源被优先传输。这避免了高优先级请求被非关键资源堵塞网络通道的问题；
  * 头部压缩：客户端现在发送了大量冗余的HTTP头部信息。因为一个页面可能有50到100个子请求，这些数据是巨大的；
  * 服务端推送流：服务端能向客户端推送数据不需要客户端发起一个请求。

* SPDY视图保持已有的HTTP语义。所有的特性比如cookies，Etags，Vary headers，Content-Encoding协商，SPDY仅仅替换了网络数据传输方法。

* Google之所以要改动HTTP协议而不是TCP/IP，是因为改变HTTP只需更新Browser和web server就行了，而改动TCP/IP就困难多了，牵扯面广，需要更新巨量的路由器，服务器和客户端的操作系统。

## SPDY通过移动网络连接流程图

![Alt text](https://raw.githubusercontent.com/zqjflash/spdy-browser/master/SPDY.png)

## SPDY文档结构

  * SPDY分为两层：Framing Layer和HTTP Layer。Framing Layer位于TCP协议层之上，传输的数据单元是frame，同一个TCP连接可以传输多个独立的frame。HTTP Layer则位于Framing Layer之上，负责把HTTP协议与SPDY协议的转换

* Session层作用：在一个单独到TCP连接上传输多个有序的frame，一个session与一个TCP连接一一对应。Session概念等同于framing layer。为了最好的性能，SPDY期望客户端不要关闭连接直到用户离开这个连接引用的所有页面，或者直到服务端关闭连接。服务端鼓励尽可能长的打开连接，但是，如果需要，能终止连接，当任何一段关闭连接，必须首先发送GOAWAY(2.6.6节)帧，这样端点就能确定在关闭连接前完成请求。

  * Session层整个网络连接链路的位置如图：

    ![Alt text](https://raw.githubusercontent.com/zqjflash/spdy-browser/master/spdy_session.png)

  * frame：一旦建立连接，客户端和服务端交换帧消息。分为两类：Control frame和Data frame。Frame由frame header和frame data两部分组成。frame header是固定的8字节，包含8bits的Flags字段和24bits的Length字段，Flags用来标识frame的类别，Length则是用来表示该frame的数据长度。

  * stream：由多个相关联的frame组成。一个stream只能用于传输一次HTTP请求及该请求对应的HTTP响应。

## 控制帧(Control frame)的格式：

  ```js
	+----------------------------------+
	|C| Version(15bits) | Type(16bits) |
	+----------------------------------+
	| Flags (8)  |  Length (24 bits)   |
	+----------------------------------+
	|               Data               |
	+----------------------------------+
  ```

  * 控制帧各个位置参数详解：

      * 控制位(Control bit)： 'C'位用一个比特定义控制消息，控制帧的值恒定1;
      * 版本(Version)：SPDY协议的版本号;
      * 类型(Type)：控制帧的类型;
      * 标识(Flags)：和这个帧有关的标识。控制帧和数据帧的标识不同;
      * 长度(Length)：一个无符号24bit数字表示之后域的长度;
      * 数据(Data)：控制帧相关的数据。格式和长度取决于控制帧的类型。
    
  * 控制帧处理需求：
    * 实现者必须能接收至少8192字节的控制帧。
    
  * 控制帧类型：
    * SYN_STREAM：允许发送者在两端之间异步的创建流；
    * SYN_REPLY：接收一个由SYN_STREAM帧接受者创建的流；
    * RST_STREAM：不正常终止一个流；
    * SETTINGS：包含一组id/value对的配置数据来告诉两端可以怎样通信；
    * PING：是一种测试发送者最小回路时间的机制；
    * GOAWAY：是一种通知连接的远端在这个会话上停止创建流的机制；
    * HEADERS：给一个流扩展额外的头部；
    * WINDOW_UPDATE：在SPDY中用于实现每个流的流量控制；
    * CREDENTIAL：用于客户端发送附加的客户端证书到服务端；

## 数据帧(Data frame)的格式：

  ```js
	+----------------------------------+
	|C|       Stream-ID (31bits)       |
	+----------------------------------+
	| Flags (8)  |  Length (24 bits)   |
	+----------------------------------+
	|               Data               |
	+----------------------------------+
  ```

  * 数据帧各个位置参数详解：

    * 控制位(Control bit)：数据帧的值恒定0；
    * 流ID(Stream-ID)：一个31bit的值标识这个流；
    * 标识(Flags)：和这个帧有关的标识；有效标识是：0x01 = FLAG_FIN：表示这个帧是流里面最后的帧；
    * 长度(Length)：一个无符号24位数字表示之后域的字节长度；数据帧总长度是8byte+数据长度。数据长度为0是有效；
    * 数据(Data)：存放变长数据；数据长度在长度域定义。

## SPDY特性

  * 流复用：允许在一个TCP连接里面，允许无限并发流（在双方资源可承受的情况下）。因为请求是在一个单一的通道交错传输，TCP可以达到很高的效率，从而减少网络连接需要，可以以很高的数据密度做传输。虽然无限的并行数据流解决了序列化问题，但是他们引入了另一个问题，如果由于信道带宽的限制，客户端可能会阻止怕堵塞通道的要求。SPDY实现请求的优先次序：客户端可以请求尽可能多的项目，每个请求分配一个优先级。这样即使高优先级的请求仍处在pending状态，通道也不会传输非关键的，低优先级的请求，这样就有效地阻止了传输拥塞。

## SPDY协议的优缺点

  | 优点        | 缺点           |

  | ------------- |:-------------:|

  | 下载速度快：连接复用、头部压缩 | 需要客户端和服务端同时支持SPDY |

  | 节省流量：头部压缩 | 协议解析更复杂 |

  | 页面加载快：请求分优先级、关键资源push |  |

  | 安全：采用SSL |  |

## SPDY协议与http协议相比

  | SPDY协议 | HTTP1.1协议 |

  | -------- | ---------- |

  | 一个SPDY连接允许建立多条stram（虚拟流），并发送多个HTTP请求 | 一个连接同时只能处理一个请求 |

  | HTTP请求可以具有优先级，客户端可以要求服务器优先发送重要资源 | 一个处理时间很长的非关键请求阻塞服务器对后面请求的处理 |

  | SPDY协议允许压缩头部，减少HTTP头部大小，减小带宽占用 | http请求和响应头未压缩，头部冗余，User-Agent,Host重复发送 |

  | 服务器可以主动给客户端发送数据，不需要客户端主动请求 | 只有客户端才能发送请求 |

## SPDY协议与http测速对比（数据采样）

  | 测试网络 | 测试网址 | 测试指标 | spdy代理 | http代理 | spdy与http代理对比 |

  | WIFI | m.sohu.com | 页面首字(ms) | 976 | 1168 | 16% |

  | WIFI | m.sohu.com | 页面首屏(ms) | 1046 | 1237 | 15% |

  | WIFI | m.sohu.com | 页面全部完成(ms) | 1046 | 1293 | 19% |

  | WIFI | m.sohu.com | 网络完成时间(ms) | 2047 | 2367 | 14% |

  | WIFI | m.sohu.com | 流量(KB) | 123 | 135 | 9% |

  | WIFI | www.lashou.com | 页面首字(ms) | 1837 | 2026 | 9% |

  | WIFI | www.lashou.com | 页面首屏(ms) | 3100 | 3776 | 18% |

  | WIFI | www.lashou.com | 页面全部完成(ms) | 3100 | 3776 | 18% |

  | WIFI | www.lashou.com | 网络完成时间(ms) | 2906 | 3563 | 18% |

  | WIFI | www.lashou.com | 流量(KB) | 265 | 298 | 11% |

## SPDY和HTTP协议转换

  * http请求在Framing Layer会被拆分为一个SYN_STREAM frame和零个或多个data frame。其中，http请求头转换为一个SYN_STREAM frame。如果http请求有消息体，则该消息体被转换为一个或多个data frame。
  * http响应在Framing Layer会被拆分为一个SYN_REPLY frame和零个或多个data frame。其中，http响应头转换为一个SYN_STREAM frame。如果http响应有消息体，则该消息体被转换为一个或多个data frame。
  * HTTP消息头转换为frame时，对应frame的frame data格式如下：

  +------------------------------------+

  |X|          Stream-ID (31bits)         |

  +------------------------------------+

  |  Unused (16 bits)|                     |

  |-------------------                     |

  | Name/value header block               |

  +------------------------------------+

  | Number of Name/Value pairs (int16) |

  +------------------------------------+

  |     Length of name (int16)            |

  +------------------------------------+

  |           Name (string)                 |

  +------------------------------------+

  |     Length of value  (int16)          |

  +------------------------------------+

  |          Value   (string)               |

  +------------------------------------+

  |           (repeats)                      |

  +------------------------------------+

## 省时与省流

  * 子资源push
    * 浏览器通常是边解析边发起子资源请求，包括js、css和图片等。当浏览器解析到HTML以link方式出现的js和css时，页面解析会暂停。直到这些资源回来之后，浏览器才能继续解析页面。其直接后果就是，页面加载完成时间变长。同时图片的加载快慢也会影响用户的首屏体验。
  * 子资源push时序图

      ![Alt text](https://raw.githubusercontent.com/zqjflash/spdy-browser/master/spdy_sub_push.png)
    
    * 测试数据显示采用子资源push，首屏完成时间减少10%以上。


  * 预连接和连接池
    * 不论是2G/3G还是WIFI，网络的延时(RTT）都很大，具体的延时如下：

    | 网络 | 2G | 3G | WIFI |

    | RTT(ms) | 200~400 | 50~100 | 0 ~ 100 |

  * 为了减少连接建立带来的额外消耗，采用预连接策略，处理流程，同时，在网络不稳定的情况下，TCP的拥塞算法会直接减缓资源的下载速度。如果只采用一个TCP连接，那么多个资源的下载速度总体会很慢。因此，采用多个TCP连接进行资源并行下载的连接池方式可以提升资源的总体下载速度。预连接在2G/3G网络下，平均可以节省大小200ms，在WIFI下平均可以节省50ms左右。