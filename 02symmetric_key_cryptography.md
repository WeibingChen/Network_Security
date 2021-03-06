# 对称密钥加密 Symmetric-Key Cryptography

## 基本设计思路

在对称秘钥加密中发送方和接收方使用同一套秘钥。

**五要素**

- 原文
- 密文
- 密钥
- 加密算法
- 解密算法

**两分类**

- 流加密
- 块加密

## 经典的对称秘钥加密算法

- 流加密
    - RC4

- 块加密
    - Feistel network
    - DES（数据加密标准）
    - AES（高级加密标准）

- 加密模式 modes of operation
    - ECB/CBC/CFB/OFB/CTR…

## 流加密

把源数据看成源源不断进入的数据流。

- 优点：速度快，一串完成

- 缺点：需要很长很随机的秘钥
    - 秘钥需要保证一定的随机性（凯撒密码）
    - 秘钥不能重复使用（Why？见实验一）
    - 这样需要一定代价
        - 复杂度高
        - 时间长
        - 安全性低

- 流加密也可以当成一种特殊的块加密（块大小等于原文长度）

### RC4算法

[维基](https://zh.wikipedia.org/wiki/RC4)上的解释还是比较好理解的。

RC4的核心是可变长度（但一般是256）的S盒，给定一个key，根据这个key打乱S盒的顺序：

```c
 for i from 0 to 255
     S[i] = i
 endfor
 j = 0
 for( i=0 ; i<256 ; i++)
     j = (j + S[i] + key[i % keylength]) % 256
     swap values of S[i] and S[j]

```

1. 第一个for循环将0到255的互不重复的元素装入S盒。
2. 第二个for循环根据密钥打乱S盒。

```c
 i = 0
 j = 0
 while GeneratingOutput:
     i = (i + 1) mod 256
     j = (j + S[i]) mod 256
     swap values of S[i] and S[j]
     k = inputByte ^ S[(S[i] + S[j]) % 256]
     output K

```
加/解密阶段，把输入作为源源不断的流来看，因此可以生成任意长度的子密钥。采用while循环，每输入一个字符就进行一次加/解密操作。因为RC4采用的加密方式是**密文=密码<img src="http://latex.codecogs.com/gif.latex?\oplus" /> 原文**
，所以按照异或的特性，解密时只要用密文异或密码就可以得到原文了，步骤是完全一样的。所以只要**发送方和接收方先协商好一个key**，就可以采用RC4源源不断地产生流密码（子密钥）了。

画成流程图如下：

![1](https://raw.githubusercontent.com/familyld/Network_Security/master/graph/RC4_algorithm2.png)

图中的数组K就是预先约定好的key，根据它生成一个与S等长的数组T，可以理解为在初始化阶段加入了：<img src="http://latex.codecogs.com/gif.latex?T[i]=K[i\%keylen]" />  <br>
然后在打乱S的阶段就可以直接调用T这个数组了。

#### 攻击RC4的手段

1. 针对机密性，就是实验一的方法，如果流密码多次使用则会被攻破。

2. 针对完整性，攻击者不需要知道密文对应的原文是什么，他在发送方和接收方中间，把密文异或一个别的字符串，这样接收方解密后就无法获取正确的原文了。
    - 发送方加密原文： <img src="http://latex.codecogs.com/gif.latex? m \oplus k = c" />

    - 攻击者截取并异或字串p：<img src="http://latex.codecogs.com/gif.latex? c \oplus p = m \oplus k \oplus p" />

    - 接收方解密得到错误的原文：<img src="http://latex.codecogs.com/gif.latex? m \oplus k \oplus p \oplus k = m \oplus p" />


## 块加密

块加密将信息分块，**每一块采用同样的算法进行多轮加密**。块的大小固定，最后输出的密文长度也为块的大小。

**块加密的要素**

- 块大小
- 秘钥长度
- 轮数
- 子秘钥的生成算法
- 轮函数(Round Function)

**块加密的优点**

可能具有较高安全性，对秘钥的要求不如流加密那么高。

**块加密的缺点**

需要较大内存存储（需要把整块内容存下来），前后数据关联，加密效率一般不如流加密。

### Feistel Network

Feistel Network是一个用于构建块加密密文的对称结构，结构图如下：

![2](https://raw.githubusercontent.com/familyld/Network_Security/master/graph/feistel_network.png)

局部（单轮的结构）：

![3](https://raw.githubusercontent.com/familyld/Network_Security/master/graph/feistel_network(local\).png)

####解读

其实结构图就像是搭积木一样，将每一轮联系起来，构建出整个加密和解密的结构。

其中<img src="http://latex.codecogs.com/gif.latex?K_1" /> ，<img src="http://latex.codecogs.com/gif.latex?K_2" /> ，...，<img src="http://latex.codecogs.com/gif.latex?K_n" /> 是由子密钥生成算法对约定的Key进行扩展获得的。这些子密钥分别对应用于不同轮的轮函数F中，也就是说，轮数是多少，就有多少个子密钥。

将原文分块以后，每次给一个原文块进行多轮的加密操作。首先这个原文块被分作L和R两个等长的部分。然后每一轮中经过轮函数获得新一轮的L和R：

<img src="http://latex.codecogs.com/gif.latex?L_{i+1} = R_i" />
<img src="http://latex.codecogs.com/gif.latex?R_{i+1} = L_i \oplus F(R_i, K_i)" />

**最后一轮加密**得到的L和R**对换后**就得到对应的密文块了。

对应地解密时：

<img src="http://latex.codecogs.com/gif.latex?R_i = L_{i+1}" />
<img src="http://latex.codecogs.com/gif.latex?L_i = R_{i+1} \oplus F(R_i, K_i)" />

结构图中右边部分其实是左边部分的镜面，RD对应LE，LD对应RE，这个注意不要弄混淆了。也因为这样，**最后一轮解密**后，还得把所得的LD和RD**对换**才能得到正确的原文块。

### 数据加密标准（DES）

![4](https://raw.githubusercontent.com/familyld/Network_Security/master/graph/DES.png)

DES算法的**块大小为64bit**，**轮数为16**，原**密钥长度56bit**，使用子密钥生成算法扩展为**16个48bit的子密钥**，用于每一轮的加密：

1. 输入一个64bit的原文块
2. 进行置换
3. 进行16轮的Feistel network加密
4. 得到结果后再进行一次置换
5. 输出置换后得到的64bit密文块

其中，置换是一种**非线性变换**，可以**增大暴力破解的时间**。

单轮的结构图：

![5](https://raw.githubusercontent.com/familyld/Network_Security/master/graph/DES2.png)

#### 缺陷

- Key的位数不够长，加解密也相对比较简单，
给暴力破解带来了可能
- 改进：用每一轮不同的key进行多轮加密
    - 3DES

#### 3DES

![6](https://raw.githubusercontent.com/familyld/Network_Security/master/graph/3DES.png)

Q：为什么3DES中首先进行加密算法，然后再进行解密算法，最后再进行加密算法，而非连续3次加密呢？

A：

- 只要加密算法和解密算法用到的key不同，两者就**不互为逆运算**<br>
在这三次DES中采用的是不同的key，加密解密算法不会抵消，所以能有效提高破解的复杂度。

- 不采用连续三次加密的原因是**为了保证和1DES的机器兼容**<br>
只要让其中一次加密算法的key和解密算法的key相同，抵消之后就相当于普通的1DES。<br>
这样一套机器就可以同时兼容3DES的加解密和1DES的加解密，而无须更换新的硬件了。

### 高级加密标准（AES）

[AES](https://zh.wikipedia.org/wiki/%E9%AB%98%E7%BA%A7%E5%8A%A0%E5%AF%86%E6%A0%87%E5%87%86)是DES的升级版，但不同于DES，它使用的不是Feistel Network架构，而是一种[代换-置换网络](https://zh.wikipedia.org/wiki/%E4%BB%A3%E6%8D%A2-%E7%BD%AE%E6%8D%A2%E7%BD%91%E7%BB%9C)。

AES算法的**块大小为128bit**，**密钥长度**可选128，192，或256bit。**轮数视密钥长度而定**，可以为10，12，或14。

AES加密过程是在一个4×4的**字节矩阵** (<img src="http://latex.codecogs.com/gif.latex?4*4*8=128bit" /> ) 上运作的，这个矩阵又称为**体（state）**，其初值就是一个明文区块（矩阵中一个元素大小就是明文区块中的一个Byte）。

#### 五个基本操作

1. 秘钥扩展(expand key)
    - 将秘钥变长（更准确地说，是扩展出多轮的子密钥）
    - 以10轮的AES为例，这一步把16bytes (128bit) 的初始key扩展为176bytes，即11个子密钥。
    - 最后一轮AES后还要进行一次秘钥加，所以要生成**轮数+1个子密钥**。

2. 秘钥加(add round key)

    - 将矩阵中的每一个字节都与该轮的子密钥做XOR运算
    - 这是关键步骤，因为2，3，4这三步大家都是一样的。

3. 代换(substitute bytes)
    - 将原文进行字节转换，起到非线性变换的作用

4. 置换(shift rows)
    - 为了把密码变得更复杂更难破解

5. 混合(mix column)
    - 为了把密码变得更复杂更难破解

其中2，3，4，5这四个操作是每一轮AES加密都会做的。

流程如图：

![7](https://raw.githubusercontent.com/familyld/Network_Security/master/graph/AES.png)

### 块密码的工作模式（modes of operation）

块密码的工作模式（mode of operation）允许使用同一个块密码密钥对多于一块的数据进行加密，并保证其安全性。

块密码本身只能加密长度等于密码块长度的单块数据，若要加密变长数据，则数据必须先被划分为多个数据块。通常而言，**最后一块数据需要使用合适填充方式将数据扩展到符合密码块大小的长度**。工作模式描述的是**加密每一数据块的过程**，并常常使用基于一个通常称为**初始化向量**的附加输入值以进行随机化，以保证安全。

工作模式主要用来进行加密和认证。对加密模式的研究**曾经包含数据的完整性保护**，即在某些数据被修改后的情况下密码的**误差传播**特性。后来的研究则将完整性保护作为另一个完全不同的，与加密无关的密码学目标。部分现代的工作模式用有效的方法将加密和认证结合起来，称为认证加密模式。

详细参考[分组密码的工作模式](http://blog.163.com/kevinlee_2010/blog/static/16982082020113853451308/)或者wiki的[块密码的工作模式](https://zh.wikipedia.org/wiki/%E5%9D%97%E5%AF%86%E7%A0%81%E7%9A%84%E5%B7%A5%E4%BD%9C%E6%A8%A1%E5%BC%8F)。

#### ECB（电子密码本）模式

![](https://upload.wikimedia.org/wikipedia/commons/c/c4/Ecb_encryption.png)

![](https://upload.wikimedia.org/wikipedia/commons/6/66/Ecb_decryption.png)

ECB是最简单的加密模式，直接把消息按块大小分成若干块，然后用同样的Key分别进行加密。

缺点：

1. **相同的平文块会被加密成相同的密文块**，因此无法提供严格的机密性。

2. 不能提供数据完整性保护，**易受到重放攻击的影响**，因为每个块是以完全相同的方式解密的。
    - 比方说，游戏中作弊者可以通过重复发送加密的“杀死怪物”的消息包以非法地快速获取经验值。

#### CBC（密码分组链接）模式

![](https://upload.wikimedia.org/wikipedia/commons/d/d3/Cbc_encryption.png)

![](https://upload.wikimedia.org/wikipedia/commons/6/66/Cbc_decryption.png)

在CBC模式中，每个平文块**先与前一个密文块进行异或，然后再进行加密**。在这种方法中，每个密文块都依赖于它前面的所有平文块。同时，**为了保证每条消息的唯一性，第一个块要先跟初始化向量IV异或再加密**。每条消息发送后IV自增1，这样就能使重放攻击无效了。

若第一个块下标为1，<img src="http://latex.codecogs.com/gif.latex?C_{0}=IV" /> ，则CBC模式的加密过程为:

<img src="http://latex.codecogs.com/gif.latex? C_{i}=E_{K}(P_{i} \oplus C_{i-1})" />

而其解密过程则为:

<img src="http://latex.codecogs.com/gif.latex? P_{i}=D_{K}(C_i) \oplus C_{i-1}" />

缺点：

1. 加密过程是串行的，无法被并行化（速度慢）。
    - 注：解密过程是可以并行化的。

2. 消息必须被填充到块大小的整数倍。

#### CFB（密码反馈）模式

![](https://upload.wikimedia.org/wikipedia/commons/f/fd/Cfb_encryption.png)

![](https://upload.wikimedia.org/wikipedia/commons/7/75/Cfb_decryption.png)

若第一个块下标为1，<img src="http://latex.codecogs.com/gif.latex?C_{0}=IV" /> ，则CFB模式的加密过程为:

<img src="http://latex.codecogs.com/gif.latex? C_{i}=E_{K}(C_{i-1}) \oplus P_i" />

而其解密过程则为:

<img src="http://latex.codecogs.com/gif.latex? P_{i}=E_{K}(C_{i-1}) \oplus C_{i}" />

**一定要注意！**这里**无论是加密还是解密对Key执行的操作都是加密操作**！ 因为密文对应的原文并没有加密，而是与前一段密文的加密结果进行了异或！所以解出对应原文时，按照异或特性，把密文再与前一段密文的加密结果进行异或就可以了。

类似CBC，因为前后块关联，所以**加密无法并行化**，但解密是可以的。 另外，**CFB和OFB，CTR的流模式相似**，有流加密的特征，只需要把块密码和原文块进行异或就可以实现加密，且**消息无需进行填充**。

#### OFB（输出反馈）模式

![](https://upload.wikimedia.org/wikipedia/commons/a/a9/Ofb_encryption.png)

![](https://upload.wikimedia.org/wikipedia/commons/8/82/Ofb_decryption.png)

OFB和CFB的区别就是它不是对前一块的密文进行加密，而是对前一块加密算法所得的块进行加密。由于异或操作的对称性，**OFB的加密和解密操作完全相同**！

<img src="http://latex.codecogs.com/gif.latex? C_{i}=P_{i}\oplus O_{i} " />
<img src="http://latex.codecogs.com/gif.latex? P_{i}=C_{i}\oplus O_{i} " />
<img src="http://latex.codecogs.com/gif.latex? O_{i}=\ E_{K}(O_{i-1}) " />
<img src="http://latex.codecogs.com/gif.latex? O_{0}=\ {\mbox{IV}} " />

类似CFB，这里**无论是加密还是解密对Key执行的操作都是加密操作**！

因为OFB需要对前一块加密所得的块进行加密，所以不能并行化处理。但是！由于平文和密文都只在最终的异或过程中使用，因此！**可以事先对IV进行加密**，得到所有用于和平文或密文进行异或的块，最后再**并行地进行异或操作**。

### CTR（计数器模式）模式

![](https://upload.wikimedia.org/wikipedia/commons/3/3f/Ctr_encryption.png)

![](https://upload.wikimedia.org/wikipedia/commons/3/34/Ctr_decryption.png)

Again！ CTR也是**无论是加密还是解密对Key执行的操作都是加密操作**！

CTR模式是比较常用的一个模式，它采用递增的计数器产生连续的密匙流。**Nonce可以理解为一个随机数**，用来和Counter进行拼接/相加/异或等操作得到IV，从而**避免被轻易猜到**。在CTR中，前后两个块是没有联系的，但是可以通过Counter来判
断顺序，并且因为 Counter是依次加一的，这样**每次通信都有唯一标识，杜绝了包被重放的可能**。


### 其他

拿到一个字符串，分块后各字节的排列是这样的：

```c
[00, 04, 08, 12]
[01, 05, 08, 13]
[02, 06, 10, 14]
[03, 07, 11, 15]
```

注意它是**按列排序**，而非我们习惯的按行排序。

### 填充

特别注意CBC和CTR，这两种模式是考点。

CBC的填充方式有好几种，假设采用PKCS7，这种方式**填充的每个byte数值上等于需要填充的byte的数目**。

举个例子：

块大小16bytes，而密文有36bytes，需要填充12bytes才能成为块大小的整数倍，则这12个需要填充的byte都被填充为`0x0c`。如果需要填充4bytes，则这4个填充的byte都填充为`0x04`。

CTR的填充就更简单了，缺少的bytes全部填为0x00可以了。

### 为什么需要IV？

增强机密性，IV不同情况下原文相同结果不同。

### 误差传播

密文传输中，若一个数据块出现了错误，对于不同的工作模式引发的错误是不同的。如：

- 采用ECB模式，解密出的平文中，对应的块会出现错误。

- 采用CBC模式，则会导致对应平文块和下一个块的错误。

比方说这样问：

1. C1在发送端到接收端的传输过程中出现问题，多少密文的值受到了影响？影响了多少的原文的恢复？

2. C1在加密过程中错了一位，多少密文的值受到了影响？影响了多少的原文的恢复？

**注意：**这两个问题的不同是，前者是**传输过程中出错**，所以在加密是后面的加密都没错；后者是**加密过程中出错**，所以如果后一块需要用到前一块的加密结果，那么后面的密文块就全错了。另外！**采用什么模式对解密出的原文影响是很大的**。即使前者只有一个密文块出错，解密结果也不见得只有一个原文块出错；相对地，后者虽然后面的密文块都错了，解密出的原文也不见得都错。

## 密钥分发与密钥管理

对于对称秘钥加密系统来说，双方都要持有同样的秘钥。那怎么保证双方都能拿到同样一份秘钥而且又神不知鬼不觉呢。于是就出现了秘钥分发和秘钥管理：

- 秘钥分发（Key distribution）：

    由第三方机构完成秘钥的生成与分发（可以通过物理形式发送，也可以通过秘密信道进行发送）

- 秘钥管理（Key agreement）：

    由发送双方自己协商秘钥（可以通过物理形式发送，也可以通过用现有秘钥加密新秘钥后再发送）

### 秘钥分配中心（KDC）

KDC是用于进行秘钥分发的可信的第三方机构。

#### 请求流程

- A向KDC请求与B进行会话的秘钥
- KDC收到请求后告诉B是否和A进行会话
- B同意后就生成A，B的秘钥，然后分给他们

#### 会话密钥

- 会话秘钥和私钥（secret key）有所不同，私钥是只有私人知道的，而会话秘钥是当两个人建立会话有会话请求时才会用到的，然后发给会话的双方，双方用此秘钥进行加密。**随着会话的结束，该秘钥不再被使用**（对于对称秘钥加密而言重复使用秘钥将导致危险）。

- 在下面的内容中可以暂且将私钥理解成某一方自己持有的一套密码，并不一定是指非对称加密系统里面的私钥（也就是说内部有可能用的也是对称秘钥加密，但是**内部用的**对称秘钥加密的**秘钥和进行会话用的会话秘钥是不一样的**）。

#### KDC的秘钥分发

最简单的一种方式如下：

![8](https://raw.githubusercontent.com/familyld/Network_Security/master/graph/KDC1.png)

第一步：A向KDC发送请求

- A用明文告知KDC会话的双方，请求KDC为双方分配秘钥
- 可以明文发送，当然也可以加密后发送，只要要保证KDC解得出来

第二步：KDC响应请求

- KDC收到请求后会产生票据(ticket)，票据里面包含会话双方的信息和分发的秘钥
    - 票据通过B的私钥进行加密，也就是A是不能解出里面的东西的…
    - 然后票据在加密后再加上**会话秘钥**<img src="http://latex.codecogs.com/gif.latex?K_{AB}" /> ，再用A的私钥进行加密后发送给A
    - A收到以后进行解密就得到了会话秘钥
    - 在这一步KDC完成了对A的验证（因为只有A才能解开上面的信息）

第三步：AB间秘钥交换

- A将自己解密后剩下的东西（也就是票据）发送给B
    - B用自己的私钥进行解密就可以知道谁要和他进行会话，会话秘钥是什么了
    - 在这一步中相当于KDC通过A对B完成了验证，因为只有B能解出票据。
    - 由此，在KDC的牵线搭桥下，AB双方都顺利得到了会话秘钥

##### 缺陷

- KDC为什么要先发给A再通过A发给B，而不直接发给B？

> 因为可能存在攻击者C冒充A，给KDC发一个A和B建立会话的请求。假设这时C截断了A和KDC的信道，则A无法更新KDC发来的会话密钥，而B却更新了会话密钥，并且深信此时是A在和他建立会话。这样就间接地阻断了A和B正在进行的会话了，因为A原来用的会话密钥失效了。 但如果先发给A然后再通过A发给B就可以解决这个问题，验证了A的身份。

- 重放攻击

> 攻击者C无需知道A和B各自的私钥，无需知道如何解密包含会话密钥的包。比方说攻击B，C只要把用B的私钥加密过的票据copy下来，在下一次A和B建立会话时，重放这个copy的包，则B刚刚更新的会话密钥又会被这个重放包中的会话密钥覆盖掉，从而使得A，B的会话密钥不一致，无法正常通讯。

- 其他缺陷

> 无法确保B收到了A发给他的包，如果中途被拦截了，则也会造成会话密钥不一致的问题。

#### 更安全的KDC秘钥分发协议

可以理解为加入了时间戳以及类似三次握手的程序，流程如下：

![9](https://raw.githubusercontent.com/familyld/Network_Security/master/graph/KDC2.png)

第一步：A向B提出会话请求

- A告知B要对B进行会话
- B在此时可以决定是否和A进行对话，是则然后将自己的时间戳<img src="http://latex.codecogs.com/gif.latex?R_B" /> 用自己的私钥加密后交给A

第二步：A向KDC申请会话秘钥

- A将自己的时间戳<img src="http://latex.codecogs.com/gif.latex?R_A" /> 连同会话双方的信息发送给KDC。KDC照旧对票据做两层加密后给到A。
- A再进行第一层解密得到会话秘钥，同时得到自己的时间戳。比对时间戳后即可得知时间信息（避免黑客进行重放攻击而造成影响）

第三步：A向B发送票据与验证信息

- 票据中有会话秘钥
- 验证信息包括
    - B一开始给A发送的时间戳<img src="http://latex.codecogs.com/gif.latex?R_B" /> （以应对重放攻击）
    - A用新的会话秘钥加密一个新的时间戳<img src="http://latex.codecogs.com/gif.latex?R_1" />

第四步：最终验证

- 类似于TCP
- 连续握手，确保对方收到的是共同的会话秘钥
- B发回用新会话密钥加密的包，包含A发来的时间戳-1以及新的时间戳<img src="http://latex.codecogs.com/gif.latex?R_2" />
- A再发送一个用新会话密钥加密的包，包含B发来的时间戳-1，此时确认握手成功，正式建立会话。

### 秘钥管理（Key Agreement）

- 由于不经过第三方，所以A直接和B进行交互。在这种情况下A要么通过物理方式密送给B，要么就要进行特殊加密

- 一种方法就是利用本原根（Primitive root）
    - 对于一个质数p来说，当一个整数<img src="http://latex.codecogs.com/gif.latex?\alpha" /> 称为p的本原根时，其通过下式生成的a**各不相同**

    <img src="http://latex.codecogs.com/gif.latex?a = \alpha^i mod \ p, \ where \ 0 \leq i \leq p-2" />

**注意区分<img src="http://latex.codecogs.com/gif.latex?a" /> 和阿尔法<img src="http://latex.codecogs.com/gif.latex?\alpha" /> **，前者是模运算的结果，后者是质数p的本原根。

比方说2就是11的本原根：

![10](https://raw.githubusercontent.com/familyld/Network_Security/master/graph/primitive_root.png)

可以看到<img src="http://latex.codecogs.com/gif.latex?2^0" /> 到<img src="http://latex.codecogs.com/gif.latex?2^{11-2}" /> 模11所得的结果皆不相同。

> **性质: 每个质数都存在本原根**

#### 利用本原根进行加密

 本原根的好处就在于，即使给出a，<img src="http://latex.codecogs.com/gif.latex?\alpha" /> 和p，**要求出i依然很难**。

由此引申出了Diffle-Hellman加密方法：

![11](https://raw.githubusercontent.com/familyld/Network_Security/master/graph/Diffle-Hellman.png)

它计算Key的方式很巧妙，这样算出来的两个key是相等的。

<img src="http://latex.codecogs.com/gif.latex?k = (Y_B)^{X_A} \ mod \ p =  (\alpha ^{X_B} \ mod \ p)^{X_A} \ mod \ p" />

<img src="http://latex.codecogs.com/gif.latex?k = (Y_A)^{X_B} \ mod \ p =  (\alpha ^{X_A} \ mod \ p)^{X_B} \ mod \ p" />

即使攻击者知道了<img src="http://latex.codecogs.com/gif.latex?Y_A" /> ，<img src="http://latex.codecogs.com/gif.latex?Y_B" /> ，<img src="http://latex.codecogs.com/gif.latex?\alpha" /> ，和 <img src="http://latex.codecogs.com/gif.latex?p" /> ，依然无法猜出<img src="http://latex.codecogs.com/gif.latex?X_A" /> 和<img src="http://latex.codecogs.com/gif.latex?X_B" /> ，因此无法破解出密钥。

但是Diffle-Hellman也不是没有问题，最大的问题在于**不能确定对方的身份**。

**Man-In-the-Middle（中间人）攻击**

![12](https://raw.githubusercontent.com/familyld/Network_Security/master/graph/ManInTheMiddle.png)

攻击者在A和B建立通讯时，截断A和B的信道，然后连向自己。这样通讯在一开始就被劫持了。

A以为自己是在跟B协商密钥，B也以为自己是在跟A协商密钥。而实际上他们的密钥都是跟攻击者协商的。

这样通讯时，攻击者可以解密出双方的所有信息，因为他持有正确的密钥。只要他愿意，完全可以用和A协商的密钥窥探A发给B的信息，然后再用和B协商的密钥加密发给B。这样通讯双方依然能正常通讯，却不知到自己的信息全部暴露了。

怎么解决这个问题呢！？**用数字签名**！
