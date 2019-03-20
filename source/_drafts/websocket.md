---
title: websocket
tags: node websocket
---

## node 原生实现服务端 websocket

本文主要介绍 webSocket(下文简写为 ws)，并使用 node 原生实现基本功能，难点主要是解析和组装数据。需要的知识点：

- [WebSocket](https://developer.mozilla.org/zh-CN/docs/Web/API/WebSocket)
- [Buffer](http://nodejs.cn/api/buffer.html)
- [按位操作符](https://developer.mozilla.org/zh-CN/docs/Web/JavaScript/Reference/Operators/Bitwise_Operators)
- 了解二进制
- 了解十六进制

首先我们看看 ws 数据帧格式：

```
  0                  1                   2                   3
  0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
 +-+-+-+-+-------+-+-------------+-------------------------------+
 |F|R|R|R| opcode|M| Payload len |    Extended payload length    |
 |I|S|S|S|  (4)  |A|     (7)     |             (16/64)           |
 |N|V|V|V|       |S|             |   (if payload len==126/127)   |
 | |1|2|3|       |K|             |                               |
 +-+-+-+-+-------+-+-------------+ - - - - - - - - - - - - - - - +
 |     Extended payload length continued, if payload len == 127  |
 + - - - - - - - - - - - - - - - +-------------------------------+
 |                               |Masking-key, if MASK set to 1  |
 +-------------------------------+-------------------------------+
 | Masking-key (continued)       |          Payload Data         |
 +-------------------------------- - - - - - - - - - - - - - - - +
 :                     Payload Data continued ...                :
 + - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - +
 |                     Payload Data continued ...                |
 +---------------------------------------------------------------+

```

要理解 ws 就离不开上面这个图，但是对数据帧不熟悉的，会完全搞不懂这个图是表达的啥意思。所以我们先解释下这个图是干嘛的，我们应该看。

### 数据帧

- 位（bit）
  - 计算机最小数据存储单位是，简称 b，也称比特(bit)。每个 0 或 1 就是一个位。
- 字节（Byte）
  - 八个位表示一个字节

有上面这两个概念再看上面的图：

- 第一行（占 32 位）
  - 表格左上角有个 FIN，这个就表示一个`位`，在这个位上可能值就只能是 0 或者 1
  - 接下来是 RSV1、RSV2、RSV3，它们也分别占用 1 位，
  - 再后面是`opcode(4)`这里表示数据操作码，占据 4 位，取值返回是：0000-1111，注意是二进制
  - 然后是`MASK`掩码标识，占 1 位，
  - `payload len(7)`，接受到的数据长度，占 7 位。
  - `Extended payload length(16/54)...`第一行的最后一格，占 8 位这里的数据含义会有变化，稍后详说。
- 第二行（占 32 位）

  - `Extended payload length continued, if payload len == 127`扩展数据长度，这里为什么要分行呢？

    - 其实分行只是为了显示方便而已，我们完全可以把第二行拼接到第一行后面，其实我们在处理数据时也是这么做的，没有分行一说。

    ```
      0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
       +-+-+-+-+-------+-+-------------+-------------------------------+------------------------------------+
       |F|R|R|R| opcode|M| Payload len |    Extended payload length    |                                    |
       |I|S|S|S|  (4)  |A|     (7)     |             (16/64)           | Extended payload length continued, |
       |N|V|V|V|       | |             |                               |  if payload len == 127             |
       | | | | |       |S|             |   (if payload len==126/127)   |                                    |
       | |1|2|3|       |K|             |                               |                                    |
       +-+-+-+-+-------+-+-------------+ - - - - - - - - - - - - - - - +------------------------------------+
    ```

所以后面几行都是可以以此拼接到后面。

如果客户端（浏览器）要发送一个`hello`给服务器，我们服务端收到的数据其实是一个二进制数据一系列的 0 或者 1，就像这样`10001000111...`，我们要知道到底发给我们的是啥，就需要对这一些列的 0/1 做解析，上面的图就解析这系列 0/1 的规则，我们按照上面的规则一步步解析就能得到我们想要的数据。

举个例子：

> 假如收到客户端发来的数据`10000001`（这里只是截取数据开始的一部分(第一个字节)，后面还有很多），对应的值如下：

| FIN | RSV1 | RSV2 | RSV3 | opcode |
| --- | ---- | ---- | ---- | ------ |
| 1   | 0    | 0    | 0    | 0001   |

### 数据帧格式详解

* FIN: 1bit
  > 表示这是一个消息的最后的一帧。第一个帧也可能是最后一个。  
  > %x0 : 还有后续帧  
  > %x1 : 最后一帧
* RSV1, RSV2, RSV3: 各占1bit
  > 除非一个扩展经过协商赋予了非零值以某种含义，否则必须为0 如果没有定义非零值，并且收到了非零的RSV，则websocket链接会失败
* Opcode: 4bit
  > 解释说明 “Payload data” 的用途/功能
  > 如果收到了未知的opcode，最后会断开链接
  > 定义了以下几个opcode值:
  >     %x0 : 代表连续的帧
  >     %x1 : text帧
  >     %x2 ： binary帧
  >     %x3-7 ： 为非控制帧而预留的
  >     %x8 ： 关闭握手帧
  >     %x9 ： ping帧
  >     %xA :  pong帧
  >     %xB-F ： 为非控制帧而预留的
* Mask: 1bit
  > 定义“payload data”(实际提交的数据)是否被添加掩码如果置1， “Masking-key”就会被赋值所有从客户端发往服务器的帧都会被置1
* Payload length： 7 bit | 7+16 bit | 7+64 bit
  >  如果是0~125，它就是“payload length”（收到数据的长度，比如收到的是`hello`，那么就是5），
  > 如果是126，紧随其后的被表示为16 bits无符号整型就是“payload length”，
  > 如果是127，紧随其后的被表示为64 bits无符号整型就是“payload length”
  - 为什么会有这三种情况呢？
    由于`payload length`只有7位，二级制最大是`1111111`转换为十进制就是`127`，如果“payload length”大于`127`了，就没法正确的表示。我们需要更多的位来表示“payload length”，所以我们在`Payload length`后面用另外的位来表示。那直接定义一个64位来表示不就行了么？虽然这样能行，但是也得考虑到性能问题，如上面说的`hello`长度只有“5”，转换为二进制是`101`，三位就可以了，如果用64位就有点太浪费了。所以分别定义了这三种情况。
* Masking-key： 0 or 32bit
  > 所有从客户端发送到服务器的帧都包含一个32 bits的掩码（如果“mask bit”被设置成1），否则为0 bit。一旦掩码被设置，所有接收到的payload data都必须与该值以一种算法做异或运算来获取真实值。
* Payload data: (x+y) bytes
  > 它是"Extension data"和"Application data"的总和，一般扩展数据为空。
* Extension data: x bytes
  > 除非扩展被定义，否则就是0，任何扩展必须指定其Extension data的长度
* Application data: y bytes
  > 占据"Extension data"之后的剩余帧的空间

### 实战

知道了帧结构和含义，接下来就可以按照规则解析数据

* 解析数据

```javascript
  function parseFrams() {
    // buffer接受到的数据
    const buffer = this.buffer;
    // 数据默认从第三个字节开始，默认数据长度小于125
    let payloadIndex = 2;

    // 获取第字节，包含FIN和操作码（opcode）
    const byte1 = buffer.readUInt8(0);

    // 0：还有后续帧
    // 1：最后一帧
    const FIN = (byte1 >>> 7) & 0x1;

    // 获取操作码，后面会根据操作码处理数据
    const opcode = byte1 & 0x0f;

    if (!FIN) {
      // 不是最后一帧需要暂存当前的操作码，协议要求:
      // 必须要暂存第一帧的操作码
      // 分片编号  0  1 ...  N-2  N-1
      //   FIN    0  0 ...  0    1
      // opcode  !0  0 ...  0    0
      this.frameOpcode = opcode;
    }

    // 获取掩码（MASK）和数据长度（payload length）
    let byte2 = buffer.readUInt8(1);

    // 定义“payload data”是否被添加掩码
    // 如果置1， “Masking-key”就会被赋值
    // 所有从客户端发往服务器的帧都会被置1
    let MASK = (byte2 >>> 7) & 0x1;

    // 获取数据长度
    let payloadLength = byte2 & 0x7f;

    let mask_key;

    if (payloadLength === 126) {
      // 大于126小于65536，那么后面字节表示的是数据的长度，那么真实的数据就会后移两字节
      payloadLength = buffer.readUInt16BE(payloadIndex);

      // 真实数据后移2位
      payloadIndex += 2;
    } else if (payloadLength === 127) {
      // 大于等于65536，那么后面字节表示的是数据的长度，数据最长为64位，但是数据太大就不好处理了，这里限制最大为32位
      // 所以第2-6字节的数据始终应该为0，真实数据的长度在6-10字节
      // 4：2-6字节的位置
      payloadLength = buffer.readUInt32BE(payloadIndex + 4);
      // 8：数据长度占据了8字节，真实数据就需要后移8字节
      payloadIndex += 8;
    }

    // 如果MASK位被置为1那么Mask_key将占据4位 MASK_KEY_LENGTH===4
    const maskKeyLen = MASK ? MASK_KEY_LENGTH : 0;

    // 如果当前接受到的数据长度小于发送的数据总长度加上协议头部的数据长度，表示数据没有接受完，暂不处理，需要等到所有数据都接受到后再处理
    if (buffer.length < payloadIndex + maskKeyLen + payloadLength) {
      return;
    }

    // 如果有掩码，那么在真实数据之前会有四字节的掩码key（Masking-key）
    let payload = Buffer.alloc(0);
    if (MASK) {
      // 获取掩码
      mask_key = buffer.slice(payloadIndex, payloadIndex + MASK_KEY_LENGTH);

      // 真实数据再次后移4位
      payloadIndex += MASK_KEY_LENGTH;

      // 有掩码需要解码，解码算法是规定死的，可见文后源码
      payload = unmask(mask_key, buffer.slice(payloadIndex));
    } else {
      // 没有掩码就直接截取数据
      payload = buffer.slice(payloadIndex);
    }

    // 可能是分片传输，需要缓存数据帧，等待所有帧接受完毕后再处理完整数据
    this.payloadFrames = Buffer.concat([this.payloadFrames, payload]);
    this.buffer = Buffer.alloc(0);

    // 数据接受完毕
    if (FIN) {
      const _opcode = opcode || this.frameOpcode;
      const payloadFrames = this.payloadFrames.slice(0);
      this.payloadFrames = Buffer.alloc(0);
      this.frameOpcode = 0;

      // 根据不同opcode处理成不同的数据
      this.processPayload(_opcode, payloadFrames);
    }
  }
  
```

* 构建返回数据，返回数据就是解析数据的逆操作

```javascript
  /**
  *
  * @param {number} opcode
  * @param {string|buffer} payload
  * @param {boolean} isFinal
  */
  function encodeMessage(opcode, payload, isFinal = true) {
    const len = payload.length;
    let buffer;
    let byte1 = (isFinal ? 0x80 : 0x00) | opcode;

    if (len < 126) {
      // 数据长度0~125

      // 构建返回数据容器
      buffer = Buffer.alloc(2 + len); // 2：[FIN+RSV1/2/3+OPCODE](占1bytes) + [MASK+payload length](占1bytes)

      // 写入FIN+RSV1/2/3+OPCODE
      buffer.writeUInt8(byte1);

      // 从第二字节写入MASK+payload length
      buffer.writeUInt8(len, 1);

      // 从第三字节写入真实数据
      payload.copy(buffer, 2);
    } else if (len < 1 << 16) {
      // 数据长度126~65535
      buffer.Buffer.alloc(2 + 2 + len);
      buffer.writeUInt8(byte1);
      buffer.writeUInt8(126, 1);
      buffer.writeUInt16(len, 2);
      payload.copy(buffer, 4);
    } else {
      // 数据长度65536~..
      buffer.Buffer.alloc(2 + 8 + len);
      buffer.writeUInt8(byte1);
      buffer.writeUInt8(127, 1);
      buffer.writeUInt32(0, 2);
      buffer.writeUInt32(len, 6);
      payload.copy(buffer, 10);
    }
    return buffer;
  }
```

上面两段代码都有很详细的注释，应该能看懂，就不再具体的解析，实现源码见[github](https://github.com/xblj/demos/tree/master/websocket)