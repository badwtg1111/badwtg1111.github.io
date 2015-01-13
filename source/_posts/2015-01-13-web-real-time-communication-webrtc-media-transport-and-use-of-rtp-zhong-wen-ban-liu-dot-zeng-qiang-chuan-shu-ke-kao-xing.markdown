---
layout: post
title: "Web Real-Time Communication (WebRTC): Media Transport and Use of RTP 中文版(六.增强传输可靠性)"
date: 2015-01-13 20:06:39 +0800
comments: true
categories: 
---

21 5月 13 Web Real-Time Communication (WebRTC): Media Transport and Use of RTP 中文版(六.增强传输可靠性）
原文：http://tools.ietf.org/html/draft-ietf-rtcweb-rtp-usage-06#section-6

6. WebRTC如何使用RTP：提供传输可靠性

有许多工具可以防止包丢失来提高RTP流的可靠性并减少对媒体质量的影响。但是，对比

无可靠性(no-robust)流，他们都要添加额外的bit到包中。这些额外的bit需要认真考虑下，

而且合计的bit率必须可控。这样，提高可靠性可能需要使用一个比较低的编码质量，但是在

这样的质量可以换来更少的错误。在下面的几节中描述的机制可以用来增强对保丢失的忍耐

性。

 

6.1. 负面确认和RTP重传(Negative Acknowledgements and RTP Retransmission)

   作为支持RTP/SAVPF profile的结果，WebRTC实现将会为RTP数据包[RFC4585]支持负面
确认NACK消息(negative acknowledgements)。这个反馈根据RTCP反馈通道的能力限制，
可以用来通知发送者特定的RTP包丢失。比如，一个发送者可以用这个信息调整媒体编码补偿从而
优化用户体验。
   发送者必须理解一般的在Section 6.2.1 of [RFC4585]定义的NACK消息，但是可以选择
忽略这个反馈 (根据 Section 4.2 of [RFC4585]).接收者可以为丢失的RTP包发送NACKs;
[RFC4585]提供了一些关于何时发送NACKs的指导。不能期望一个接受者会为每个丢失的RTP包
发送NACK消息，它需要考虑发送NACK反馈的成本，以及丢失包的重要性，从而决定是否值得告诉
发送者有包丢失了。
Perkins, et al.          Expires August 29, 2013               [Page 15]
 
Internet-Draft               RTP for WebRTC                February 2013
 RTP重发Payload格式 [RFC4588]提供了根据NACK反馈重复丢失包的能力。重发需要小心的在
实时交互应用程序中使用使用，来保证  重发的包按时到达且有用，但是能在相对低的RTT网络
环境下有效（一个RTP发送者可以使用RTCP SR和RR包中的信息估计到接受者的RTT)。重传的
使用也可以提高前转(forward)RTP带宽，如果丢包是因为网络拥堵造成的，也可能使问题恶化。
我们仍然要注明，重传一个重要的丢包来修复解码状态比重发一个完整的帧内帧(intraframe)
可以降低损耗。接收到NACK消息就轻率的重复RTP包并不合适。在重复之前需要丢失包的重要性
以及它们按时到达的可能性。
接受者必须实现RTP重传[RFC4588]。如果RTP重发payload格式已经在会话中协商过，而且
如果发送者根据NACK相信重发包是有用的，发送者可以发送RTP重传包。不能期望发送者会
重复每个回复了NACK的包。
 

6.2. 转发错误修正Forward Error Correction (FEC)

    转发错误修正(FEC)的使用可以对一定程度上的包丢失提供保护，同时会消耗一定的带宽。
有很多FEC方案供RTP使用。有些方案是给特定RTP payload格式使用的，而其它的对RTP包的
操作可以用在任意payload 格式上。需要注意的是使用冗余的编码或者FEC会导致延时增加，
在选择冗余或者FEC格式以及它们对应的参数时需要考虑这一点。
    
   如果在WebRTC会话中作为标准特性来使用的RTP payload格式 支持冗余传送或者FEC，
那么根据任何合适的信号，这个支持就可以用在WebRTC会话中。

   有许多基于块(block-based)FEC方案，是设计成与RTP payload格式无关的。在写这
个文档时，在WebRTC上下文中使用这些FEC方案中的哪一个还没有共识。因此，这个备忘录
无法推荐基于块（block-based)FEC供WebRTC使用。
Perkins, et al.          Expires August 29, 2013               [Page 16]
 
Internet-Draft               RTP for WebRTC                February 2013
原创文章，转载请注明： 转载自讨论关于webrtc html5 extjs nodejs等技术和产品

本文链接地址: Web Real-Time Communication (WebRTC): Media Transport and Use of RTP 中文版(六.增强传输可靠性）


