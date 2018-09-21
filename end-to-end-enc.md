## 基于ONT ID的数据加解密服务

本文将描述一种基于Ont ID的数据加解密服务，采用**[混合加密方案](https://en.wikipedia.org/wiki/Hybrid_cryptosystem)**，支持多种公钥类型。

<!-- 安全存储服务模型，在该模型中，存储服务提供商包括两种服务器：数据存储服务器、存储区访问控制服务器。 -->

<!-- 
![架构](./architecture.png)
    
## 上传数据

提供方在数据交易成交后，将数据加密后存储在数据市场提供的存储平台上。在向存储平台上传数据之
前，首先需要向数据市场提交一个“存入请求”，该请求应当包括如下信息：
- 数据哈希值
- 数据文件大小
- 提供方的ONT ID
- 临时随机数（可选）
- 提供方的签名

数据市场验证该请求，包括文件大小是否超出最大允许范围、请求签名值是否合法等等。验证通过之后，数据市场应当返回一个“存入token”，该token的详细设计需遵循特定存储平台的要求。

## 下载数据
购买方完成资金交割之后，即具备权限来下载数据。从存储平台下载数据之前，需先向数据市场服务器提交一个“读取请求”，该请求应当包含如下信息：
- 数据文件的哈希值
- 数据文件大小
- 购买方的ONT ID
- 临时随机数（可选）
- 提供方的签名

数据市场服务器验证该请求，包括数据文件是否存在、签名是否合法、购买方是否具备读取权限等等。验证通过之后，数据市场应当返回一个“读取token”，该token的详细设计需遵循底层存储平台的要求。

-->

## 数据加密

数据的加密使用如下的算法，分为三个步骤：
1. 获取公钥：访问Ontology区块链，获取购买方的公钥；
2. 随机生成AES加密秘钥： 随机采样256比特数据，作为AES256加密密钥；
3. AES加密： 将AES256加密密钥用公钥加密算法进行加密，数据使用AES256算法GCM模式进行加密。

![数据加密](./end-to-end.png)

事实上，密文数据用JSON格式传输，具有如下形式：

```json
{
    "OntID": "",  
    "PkIndex": 1,
    "IV": "",
    "EncrypteAESKey": "", 
    "Ciphertext": "",
    "AuthTag": ""
}
```

各个字段的含义如下： 
 * OntID： 接收者的OntID
 * PkIndex： 公钥加密算法所使用的接受者公钥索引（4个字节）
 * IV： 用于AES256-GCM模式的初始向量，默认为12字节
 * EncryptedAESKey：被公钥加密后的AES秘钥
 * Ciphertext：密文
 * AuthTag: 认证标签，即MAC

其中，AES-GCM模式下的认证数据（AAD）为 `OntID || PkIndex`。

下面我们详细描述加密算法流程。

* 输入：
   * 接收者OntID 
   * 公钥索引PkIndex
   * 公钥pk
   * 待加密数据data 
   * 认证数据AAD
* 输出:
   * 密文JSON对象

* 算法流程：
   1. 随机采样12字节数据，称为IV；
   2. 随机采样32字节数据，称为key；
   3. 使用公钥pk加密key，计算得到encryptedAESKey，根据公钥类型的不同，采用不同的公钥加密算法，附录给出了ECIES公钥加密算法的伪代码；
   4. 使用key作为AES256-GCM模式（参考[The Galois/Counter Mode of Operation (GCM)](http://luca-giuzzi.unibs.it/corsi/Support/papers-cryptography/gcm-spec.pdf)
）加密算法的秘钥，IV作为初始向量，AAD作为认证数据，对数据data进行加密，得到密文`ciphertext`和认证标签`AuthTag`；
   5. 构造密文JSON对象并返回。

## 数据解密

数据的解密按如下三个步骤进行：
1. 根据Ont ID和PkIndex，从私钥管理模块中找到对应私钥；
2. 用私钥解密出AES对称密钥；
3. 用AES对称密钥，以AES256算法GCM模式解密数据。

下面我们详细描述加密算法流程。

* 输入：
   * 密文JSON对象
   * 私钥sk
* 输出:
   * 明文或“失败”

* 算法流程：
   1. 从密文JSON对象解出IV, AAD, ciphertext；
   2. 从密文JSON对象解出encryptedAESKey；
   3. 使用私钥sk解密encryptedAESKey，计算得到key，根据公钥类型的不同，采用不同的公钥解密算法，附录给出了ECIES公钥解密算法的伪代码；
   4. 使用key作为AES256-GCM模式（参考[The Galois/Counter Mode of Operation (GCM)](http://luca-giuzzi.unibs.it/corsi/Support/papers-cryptography/gcm-spec.pdf)
）解密算法的秘钥，IV作为初始向量，AAD作为认证数据，对ciphertext进行解密，若解密失败，则返回“失败”，否则返回解密结果。

<!-- 
## 附录 A

### A.1 ECIES 公钥加密算法

1.  密钥派生kdf

    密钥派生算法使用的哈希函数是`SHA256`, `DigestSize=32`.
 
    * 输入：seed, 派生密钥长度len(以bit作为计量单位)
    * 输出：长度为len的key(以bit作为计量单位)
    * 流程：
        1. 计算b = ceil(len / 8*digestSize)
        2. 初始化counter = 1, i = 0
        3. while (counter < b)
            - index = (counter-1)*digestSize
            - key[index:index+digestSize] = SHA256(seed || counter)
            - counter = counter + 1
        4. offset = len - (b-1)* digestSize
        5. key[b-1] = SHA256(seed || (b-1)) //前offset个比特

2. ECIES加密算法

    * 输入参数：公钥H，待加密的原文msg;

    * 输出：(IV, out, cxt)
    * 算法流程：
        1. 随机生成一个随机数r \in (1, n), n是椭圆曲线群基点的阶；
        2. 计算两个点：gTilde = [r]G, hTilde = [r]H；
        3. 计算out = encode(gTilde) = 04 || gTilde.x || gTilde.y；
        4. 计算PEH = hTilde.x；
        5. 用密钥派生函数kdf产生临时密钥key（用于AES加密）：key = kdf-sha256(out || PEH, 256);
        6. 随机生成一个16字节IV；
        7. 使用AES算法CBC模式加密原文，得到原文的密文：cxt = AES_CBC_256_encrypt(IV, key, msg)；
        8. 返回(IV, out, msg_cxt)。


3. ECIES解密算法

    * 输入参数： 私钥x，待解密的密文cxt = (IV, out, msg_cxt);
    * 输出：msg
    * 算法流程：
        1. 解码out得到一个椭圆曲线点gTilde；
        2. 计算hTilde = [x]gTilde；
        3. 计算PEH = hTilde.x；
        4. 使用密钥派生函数kdf，计算临时密钥（用于AES解密）：key = kdf-sha256(out || PEH, 256);
        5. 使用AES算法CBC模式解密密文，得到原文: msg = AES_CBC_256_encrypt(IV, key, msg_cxt)；
        6. 返回msg。


4. 参考实现

-->