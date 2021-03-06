# 第四部分Capture15：实体和编码
#计算机网络/HTTP

HTTP 会确保它的报文被正确 传送、识别、提取以及适当处理，了实现这些目标，HTTP 使用了完善的标签来描述承载内容的实体。

* 可以被正确地识别(通过 Content-Type 首部说明媒体格式，Content- Language 首部说明语言)，以便浏览器和其他客户端能正确处理内容。 
* 可以被正确地解包(通过 Content-Length 首部和 Content-Encoding 首部)。 
* 是最新的(通过实体验证码和缓存过期控制) 
* 符合用户的需要(基于 Accept 系列的内容协商首部) 
* 在网络上可以快速有效地传输(通过范围请求、差异编码以及其他数据压缩方法) 
* 完整到达、未被篡改(通过传输编码首部和 Content-MD5 校验和首部) 

## 15.1 报文是箱子，实体是货物
![](%E7%AC%AC%E5%9B%9B%E9%83%A8%E5%88%86Capture15%EF%BC%9A%E5%AE%9E%E4%BD%93%E5%92%8C%E7%BC%96%E7%A0%81/EAB8E8BA-02ED-4C82-8A84-9409ADD3E6FB.png)
一个空白行(CRLF)把首部字段 同主体的开始部分分隔开来 

*HTTP/1.1 种的10 个基本实体首部* 

* Content-Type 
实体中所承载对象的类型。 
* Content-Length
所传送实体主体的长度或大小。 
* Content-Language 
与所传送对象最相配的人类语言。 
* Content-Encoding
对象数据所做的任意变换(比如，压缩)。 
* Content-Location 
一个备用位置，请求时可通过它获得对象。 
* Content-Range 
如果这是部分实体，这个首部说明它是整体的哪个部分。 
* Content-MD5
实体主体内容的校验和。 
* Last-Modified 
所传输内容在服务器上创建或最后修改的日期时间。 
* Expires 
实体数据将要失效的日期时间。 
* Allow
该资源所允许的各种请求方法，例如，GET 和 HEAD。 

> ETag  
> 这份文档特定实例(参见 15.7 节)的唯一验证码。ETag 首部没有正式定义为实 体首部，但它对许多涉及实体的操作来说，都是一个重要的首部。   
> Cache-Control  
> 指出应该如何缓存该文档。和 ETag 首部类似，Cache-Control 首部也没有正 式定义为实体首部。   

*实体主体*
 实体主体中就是所要传输的原始货物 
![](%E7%AC%AC%E5%9B%9B%E9%83%A8%E5%88%86Capture15%EF%BC%9A%E5%AE%9E%E4%BD%93%E5%92%8C%E7%BC%96%E7%A0%81/BBCF939B-6F65-4B0B-A959-A16990038A48.png)

## 15.2 实体的大小（Content-Length）
Content-Length 首部指示出报文中实体主体的字节大小。除非使用了分块编码，否则 Content-Length 首部就是带有实体主体的报文必须使用的。

> 这个大小是包含了所有内容编码的。比如，对文本文件进行了 gzip 压缩的话，Content-Length 首部就是压缩后的大小，而不是原始数据的大小。   

### 15.2.1 使用 Content-Length 首部的目的
1. 能够检测出服务器崩溃而导致的报文截尾（长度不一致）
2. 对共享持久连接的多个报文正确分段，标识位置（对比实际和当前的大小）

### 15.2.2 如何确定实体主体的大小
1. 如果特定的 HTTP 报文类型中不允许带有主体，就忽略 Content-Length 首 部（HEAD响应）
2. 如果报文中含有描述传输编码的 Transfer-Encoding 首部(不采用默认的 HTTP“恒等”编码)，那就必须忽略 Content-Length （因为传输编码会改变实体主体的表示和传输方式，因此可能就会改变传输的字节数）
3. 如果报文中含有 Content-Length 首部(并且报文类型允许有实体主体)，而 且没有非恒等的 Transfer-Encoding 首部字段，那么 Content-Length 的值 就是主体的长度。 
4. 如果报文使用了 multipart/byteranges(多部分 / 字节范围)媒体类型，并且没 有用 Content-Length 首部指出实体主体的长度，那么多部分报文中的每个部 分都要说明它自己的大小。这种多部分类型是唯一的一种自定界的实体主体类型，因此除非发送方知道接收方可以解析它，否则就不能发送这种媒体类型。

## 15.3 实体的修改检测（content-MD5）
为检测实体主体的数据是否被不经意(或不希望有)地修改，发送方可以在生成初始的主体时，生成一个数据的校验和（content-MD5首部字段），这样接收方就可以通过检查这个校验和来捕获所有意外的实体修改。 

1. 需要保证只有产生响应的原始服务器计算并发送 Content-MD5 首部。
2. Content-MD5 首部是在对内容做了所有需要的内容编码之后，还没有做任何传输编码之前，计算出来的。 
3. 为了验证报文的完整性，客户端必须先进行传输编码的解码，然后计算所得到的未进行传输编码的实体主体的 MD5 
4. 当然，这种方法对同时替换报文主体和摘要首部的恶意攻击无效。这只是为了检测不经意的修改。对付恶意篡改，需要使用别的机制，比如摘要认证。 

##  15.4 实体资源类型（Content-Type） 
Content-Type 首部字段说明了实体主体的 MIME 类型。
MIME 类型是标准化的名字，用以说明作为货物运载实体的基本媒体类型(比如:HTML 文件、Microsoft Word 文档或是 MPEG 视频等)。
 MIME 类型由一个主媒体类 型(比如:text、image 或 audio 等)后面跟一条斜线以及一个子类型组成。

![](%E7%AC%AC%E5%9B%9B%E9%83%A8%E5%88%86Capture15%EF%BC%9A%E5%AE%9E%E4%BD%93%E5%92%8C%E7%BC%96%E7%A0%81/53556C24-27B8-4895-B917-7E902C80C2FB.png)

### 15.4.1 Content-Type 额外参数
Content-Type 首部还支持可选的参数来进一步说明内容的类型。 
```
Content-Type: text/html; charset=iso-8859-4 
```
它说明把实体中的比特转换为文本文件中的字符的方法。

### 15.4.2 多部分媒体类型（multipart ） 
当 HTTP 实体主体由多个主体合在一起作为单一的复杂报文发送时，用此类型描述复杂报文的资源类型且每一部分都是独立的，有各自的描述其内容的集，一般用在下面2种情况

*多部分表格提交* 
同时提交多种类型的资源数据，HTTP 使用 Content-Type:multipart/form-data 或 Content-Type:multipart/ mixed 这样的首部以及多部分主体来发送这种请求。
![](%E7%AC%AC%E5%9B%9B%E9%83%A8%E5%88%86Capture15%EF%BC%9A%E5%AE%9E%E4%BD%93%E5%92%8C%E7%BC%96%E7%A0%81/F14AFE67-712F-489C-A802-80AA5093CBBF.png)
/boundary 参数说明了分割主体中不同部分所用的字符串/ 

假定一个用户在文本输入字段中键入 Sally，并选择了文本文件 essayfile.txt 
![](%E7%AC%AC%E5%9B%9B%E9%83%A8%E5%88%86Capture15%EF%BC%9A%E5%AE%9E%E4%BD%93%E5%92%8C%E7%BC%96%E7%A0%81/DA3F5672-4B34-4F3B-B6E0-4605C42AD65C.png)

*多部分范围响应* 
bytes 0-1744/1441
bytes 552-761/1441
bytes 1344-1441/1441
![](%E7%AC%AC%E5%9B%9B%E9%83%A8%E5%88%86Capture15%EF%BC%9A%E5%AE%9E%E4%BD%93%E5%92%8C%E7%BC%96%E7%A0%81/4A8AD541-89B1-4484-AA6C-4D25D0AE57EF.png)

## 15.5 内容编码 
一般在发送前，会对内容进行编码，再放在实体主体中，这样有助于减少传输实体的时间和防止未经授权的第三方看到文档的内容。

一个压缩后的图像响应报文
![](%E7%AC%AC%E5%9B%9B%E9%83%A8%E5%88%86Capture15%EF%BC%9A%E5%AE%9E%E4%BD%93%E5%92%8C%E7%BC%96%E7%A0%81/E9B65895-0DA1-4195-BCF0-F49126A232EB.png)
*Content- Length 首部现在代表的是编码之后的主体长度* 

15.5.1 内容编码过程 

1. 网站服务器生成原始响应报文，其中有原始的 Content-Type 和 Content- Length 首部。 
2. 内容编码服务器(也可能就是原始的服务器或下行的代理)创建编码后的报文。 编码后的报文有同样的 Content-Type 但 Content-Length 可能不同(比如 主体被压缩了)。内容编码服务器在编码后的报文中增加 Content-Encoding 首部，这样接收的应用程序就可以进行解码了。 
3. 接收程序得到编码后的报文，进行解码，获得原始报文。 
 
15.5.2 内容编码类型 
![](%E7%AC%AC%E5%9B%9B%E9%83%A8%E5%88%86Capture15%EF%BC%9A%E5%AE%9E%E4%BD%93%E5%92%8C%E7%BC%96%E7%A0%81/A5836351-7C48-4D03-A193-84BF70C1D242.png)

### 15.5.3 Accept-Encoding 首部 
客户端就把自己支持的内容编码方式列表放在请求的 Accept-Encoding 首部里发出去。如果 HTTP 请求中没有包含 Accept-Encoding 首部，服务器就可以假设客户端能够接受任何编码方式(等价于发送 Accept-Encoding: *) 

![](%E7%AC%AC%E5%9B%9B%E9%83%A8%E5%88%86Capture15%EF%BC%9A%E5%AE%9E%E4%BD%93%E5%92%8C%E7%BC%96%E7%A0%81/845F5B05-8705-47F1-8078-2265B8CC5A0C.png)
> 客户端可以给每种编码附带 Q(质量)值参数来说明编码的优先级。Q 值的范围从 0.0 到 1.0，0.0 说明客户端不想接受所说明的编码，1.0 则表明最希望使用的编码。 “*”表示“任何其他方法”。   

## 15.6 传输编码 
内部编码是对实体主体部分进行编码，传输编码是作用于整个报文上，改变报文自身结构，HTTP 规范只定义了一种传输编码，就是分块编码。
![](%E7%AC%AC%E5%9B%9B%E9%83%A8%E5%88%86Capture15%EF%BC%9A%E5%AE%9E%E4%BD%93%E5%92%8C%E7%BC%96%E7%A0%81/4F15874C-4DE1-44B9-B01D-4FE04611DC7D.png)

### 15.6.1 传输编码的作用
* 希望在知道内容大小之前就开始传输数据（ HTTP 协议要求 Content-Length 首部必须在数据之前）
* 用传输编码来把报文内容扰乱，可以保证安全（一般用SSL）

### 15.6.2 传输编码的首部字段 
* Transfer-Encoding 
用在响应首部中，告知接收方已经对其进行了何种编码。 
* TE 
用在请求首部中，告知服务器可以使用哪些传输编码扩展。 
![](%E7%AC%AC%E5%9B%9B%E9%83%A8%E5%88%86Capture15%EF%BC%9A%E5%AE%9E%E4%BD%93%E5%92%8C%E7%BC%96%E7%A0%81/DF344F92-6276-43F2-AB56-2773AA59D19A.png)

### 15.6.3 分块编码 
分块编码把报文分割为若干个大小已知的块。块之间是紧挨着发送的，这样就不需 要在发送之前知道整个报文的大小了。 分块编码是一种传输编码，因此是报文的属性，而不是主体的属性。 

![](%E7%AC%AC%E5%9B%9B%E9%83%A8%E5%88%86Capture15%EF%BC%9A%E5%AE%9E%E4%BD%93%E5%92%8C%E7%BC%96%E7%A0%81/26EE88D0-6529-4E26-8F8E-1D3BE49212E3.png)
* 由起始的 HTTP 响应首部块开始，随后就是一系列分块。
* 每个分块包含一个长度值和该分块的数据。长度值是十六进制形式并将 CRLF 与数据分隔开。分块中数据的大小以字节计算，不包括长度值与数据之间的 CRLF 序列以及分块结尾的 CRLF 序列。
* 最后一个块有点特别，它的长度值为 0，表示“主体结束”。 
* 如果客户端的 TE 首部中说明它可以接受拖挂的话，就可以在分块的报文最后加上拖挂。 
* *拖挂的内容是可选的元数据，拖挂中可以包含附带的首部字段，它们的值在报文开始的时候可能是无法确定的 (例如：必须要先生成主体的内容)*。Content-MD5 首部就是一个可以在拖挂中发送的首部，因为在文档生成之前，很难算出它的 MD5。
* 除了 Transfer-Encoding、Trailer （拖挂关键字）以及 Content-Length 首部之外，其他 HTTP 首部都可以作为拖挂发送。 
* 图 15-6 中展示了拖挂的使用方式。报文首部中包含一个 Trailer 首部，列出了跟在分块报文之后的首部列表。 在 Trailer 首部中列出的首部就紧接在最后一个分块之后。 

## 15.7 随时间变化的请求实例
同样的 URL 会随着时间变化而指向对象的不同版本。以 CNN 的主页为例，同一天里多次访问 http://www.cnn.com，可能每次得到的返回页面都会略有不同。 
![](%E7%AC%AC%E5%9B%9B%E9%83%A8%E5%88%86Capture15%EF%BC%9A%E5%AE%9E%E4%BD%93%E5%92%8C%E7%BC%96%E7%A0%81/DD9514AF-5538-4246-9C97-74C41265C8C5.png)

HTTP 协议规定了称为实例操控(instance manipulations)的一系列请求和响应操作，用以操控对象的实例。两个主要的实例操控方法是范围请求和差异编码。这两种方法都要求客户端能够标识它所拥有(如果有的话)的资源的特定副本，并在一定的条件下请求新的实例。 
 
## 15.8 验证码和新鲜度 

有条件的请求(conditional request)，要求客户端使用验证码 (validator)来告知服务器它当前拥有的版本号，并仅当它的当前副本不再有效时才要求发送新的副本。 

### 15.8.1 新鲜度 
服务器应当告知客户端能够将内容缓存多长时间，在这个时间之内就是新鲜的。 服务器可以用这两个首部之一来提供这种信息: 
Expires(过期)和 Cache- Control(缓存控制)。 

Expires 首部
规定文档“过期”的具体时间——此后就不应当认为它还是最新的。 
```
Expires: Sun Mar 18 23:59:59 GMT 2001 
```

Cache-Control 首部
功能很强大，服务器和客户端都可以用它来说明新鲜度。
![](%E7%AC%AC%E5%9B%9B%E9%83%A8%E5%88%86Capture15%EF%BC%9A%E5%AE%9E%E4%BD%93%E5%92%8C%E7%BC%96%E7%A0%81/1D5CF471-E951-403B-9D8F-1837CB286089.png)

### 15.8.2 有条件的请求与验证码 
如果服务器上的文档和已过期的缓存副本相同，而缓存服务器还是要从原始服务器 上取文档的话，那缓存服务器就是在浪费网络带宽。

所以仅当资源改变时才请求副本， 这种特殊请求称为有条件的请求。有条件的请求是标准的 HTTP 请求报文，但仅当某个特定条件为真时才执行。例如，某个缓存服务器可能发送下面的有条件 GET 报 文给服务器，仅当文件 /announce.html 从 2002 年 6 月 29 日(这是缓存的文档最后 被作者修改的时间)之后发生改变的情况下才发送它。
```
GET /announce.html HTTP/1.0
If-Modified-Since: Sat, 29 Jun 2002, 14:30:00 GMT 
```
![](%E7%AC%AC%E5%9B%9B%E9%83%A8%E5%88%86Capture15%EF%BC%9A%E5%AE%9E%E4%BD%93%E5%92%8C%E7%BC%96%E7%A0%81/4AAAA03C-C660-490F-9A01-FC857FF763EA.png)

## 15.9 范围请求 
范围请求属于一类实例操控，因为它们是在客户端和服务器之间针对特定的对象实例来交换信息的。也就是说，客户端的范围请求仅当客户端和服务器拥有 文档的同一个版本时才有意义。 

HTTP 客户端可以通过请求曾获取失败的实体的一个范围(或者说 一部分)，来恢复下载该实体。当然这有一个前提，那就是从客户端上一次请求该实 体到这次发出范围请求的时段内，该对象没有改变过。 
![](%E7%AC%AC%E5%9B%9B%E9%83%A8%E5%88%86Capture15%EF%BC%9A%E5%AE%9E%E4%BD%93%E5%92%8C%E7%BC%96%E7%A0%81/6F79D7B8-E8D7-4DB1-AEE3-3B4FBA527B9E.png)
请求从文档开头4000以后的字节数据

为了加速下载文档而从不同的服务器下载同一个文档的不同部分。对于客户端在一个请求内请求多个不同范围的情况，返回的响应也是单个实体，它有一个多部分主体及Content-Type: multipart/byteranges首部 。

通过Accept-Ranges 首部的形式向客户端说明当前服务器可以接受的范围请求，这个首部的 值是计算范围的单位，通常是以字节计算（btyes表示整个范围） 
![](%E7%AC%AC%E5%9B%9B%E9%83%A8%E5%88%86Capture15%EF%BC%9A%E5%AE%9E%E4%BD%93%E5%92%8C%E7%BC%96%E7%A0%81/FD84BB8B-01AB-4289-A016-7A16E19F82F9.png)
![](%E7%AC%AC%E5%9B%9B%E9%83%A8%E5%88%86Capture15%EF%BC%9A%E5%AE%9E%E4%BD%93%E5%92%8C%E7%BC%96%E7%A0%81/C3AFD6E9-AA86-40DD-8273-FA3E9D67C04D.png)

## 15.10 差异编码 
差异编码是一类实例操控，因为它依赖客户端和服务器之间针对特定的对象实例来 交换信息。 差异编码是 HTTP 协 议的一个扩展，它通过交换对象改变的部分而不是完整的对象来优化传输性能。比如：一个页面若改变的地方比较少，与其发送完整的新页面给客户端，客户端更愿意服务器只发 送页面发生改变的部分。

















