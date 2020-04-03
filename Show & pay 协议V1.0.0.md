
# Show & pay 协议 V1.0.0

| brfc         | title             | authors     | **version** |
| ------------ | ----------------- | ----------- | ----------- |
| 3065510ee0dd | Show&Pay protocol | ShowPayTeam | 1           |

​   当前市面上数字货币钱包的的支付大部分是收款方展示二维码，由支付方获得收款方地址之后输入支付金额，再由支付方完成交易生成，并且广播，收款方通过区块上收到具体转账结果，完成整个钱包资产转移的过程。

​   这个过程中看似没有问题，实际上有很多安全隐患，对于收款方（商家）来说 每次支付都需要直接的暴露自己的收款地址，具有安全隐患，更大的问题是对于bsv 链来说，由于可以接受小额零支付，对于支付方广播的方式，对自己的自身安全权益更是更大的问题，毕竟支付方链接的广播连接的矿池可能并没有收款方自己选择的广播来的安全，因为需要对零确认安全负责。 

​   因为bsv 0确认支付带来的无与伦比的体验感 和 广播方的更好的为自己安全负责，我们在[**bip270**](https://github.com/moneybutton/bips/blob/master/bip-0270.mediawiki)基础上设计了一个安全的便捷的线下支付流程，我们称之为**show&pay**方案。

​   首先定义一下角色，分别为支付方钱包，收款方POS设备，为收款方POS设备提供服务的POS-server，还有支付方钱包连接的wallet-server。

#### **一个理想的支付场景是这样子的：**

> 钱包用户给出自己的支付二维码，商家使用pos设备扫描支付二维码，支付方确认/小额授权无须确认，就完成了支付

​   是不是特别符合当前我们的消费习惯，这就是bsv0确认给我们支付场景带来的体验感。

#### **实现流程：**

![image](https://github.com/showMoneyapp/protocol/blob/master/3991585827011_timing_diagram.jpg):

##### 1.首先付款方钱包生成自己的支付二维码，编码内容是

```
featureCode,paymentUrl,walletId,tokenIndex,random,hash
```
字段定义分别是：

|             |                                                              |
| ----------- | ------------------------------------------------------------ |
| featureCode | 特征码 show&pay:                                             |
| paymentUrl  | 付款方服务器URL                                              |
| walletId    | 付款方钱包id (用来给wallet-service识别walletId)              |
| tokenIndex  | 约定的token索引，默认0为bsv（1个字节） |
| random      | wallet带的的随机码（防止两次交易）字符串长度为6              |
| hash        | SHA256(featureCode,paymentUrl,walletId,tokenIndex,random)  摘要的前6字节对应字符串 |

一个例子是

```
show&pay:https://www.showpay.com,wallet1,0,HU8N09,61598386dc54
```

##### 2.当POS设备接收到具体信息之后，生成一个output的数据

```
Output {
  amount      // number. required.
  script      // string. required. hexadecimal script.
  description // string. optional. must not have JSON string length of greater than 100.
  tokenIndex  // number token_index. default 0, 0 mean bsv 
}
```
字段定义分别是：

|             |                                           |
| ----------- | ----------------------------------------- |
| amount      | 要支付的聪数（0.00000001 BTC）。          |
| script      | 支付的 TxOut 脚本，格式为十六进制字符串。 |
| description | 可选描述，最大长度为100个字符。            |
| tokenIndex  | tokenIndex                                |

##### 3.pos设备将创建生成PaymentRequest 返回给支付方服务器，同时在pos-server上生成对应的订单信息。

```
PaymentRequest {
    network           // string. required. always set to "bitcoin".
    outputs           // an array of outputs. required, but can have zero elements.
    walletId          // string.  required. wallet-device  
    deviceId          // string.  required. pos-device
    creationTimestamp // number. required.
    expirationTimestamp // number. optional.
    memo                // string. optional.
    paymentUrl          // string. required.
    merchantData        // string. optional.
    qrcodeLabelData     // string. optional.
}
```

|                     |                                                              |
| ------------------- | ------------------------------------------------------------ |
| network             | 此字段是必填字段，并且始终设置为“ bitcoin”。如果将其设置为除“比特币”以外的任何其他值，则任何钱包都不应处理付款。出于测试目的，可以将此字段设置为“测试”，以确保没有钱包会意外地将付款发送到可能无效的地址或测试地址。 |
| outputs             | 一个或多个比特币交易输出。                                   |
| creationTimestamp   | 创建PaymentRequest时的Unix时间戳。                           |
| expirationTimestamp | Unix时间戳，之后应将PaymentRequest视为无效。                 |
| memo                | 备忘文字，解释此PaymentRequest的用途。最大长度为50个字符。   |
| paymentUrl          | 支付服务器url以获得PaymentACK。最大长度为4000个字符。        |
| merchantData        | 付款主机可以用来标识PaymentRequest的任意数据。如果付款主机不需要将Payments与PaymentRequest关联，或者将每个PaymentRequest与单独的付款地址关联，则可以省略。最大长度为10000个字符。 |
| walletId            | 钱包 ID标识  (用来给wallet-service识别walletId)              |
| deviceId            | 设备 ID标识  (用来给POS-service识别的deviceId)               |
| qrcodeLabelData     | random string in qrcode                                      |

##### 4.Payment返回给POS设备服务器

支付方在验证之后（大额需用用户手动确认，保证安全）

签名完整的交易，之后将签名完成的的tx，组装的Payment返回给pos设备服务器。

```
Payment {
  walletId          // string.  required. wallet-device  
  deviceId          // string.  required. pos-device
  merchantData  // string. optional.
  transaction   // a hex-formatted (and fully-signed and valid) transaction. required.
  refundTo      // string. paymail to send a refund to. optional.
  memo          // string. optional.
}
```

|              |                                                              |
| ------------ | ------------------------------------------------------------ |
| walletId     | 钱包 ID标识  (用来给wallet-service识别walletId)              |
| deviceId     | 设备 ID标识  (用来给POS-service识别的deviceId)               |
| merchantData | 付款主机可以用来标识PaymentRequest的任意数据。如果付款主机不需要将Payments与PaymentRequest关联，或者将每个PaymentRequest与单独的付款地址关联，则可以省略。最大长度为10000个字符。 |
| transaction  | 有效的，已签名的比特币交易，可全额支付PaymentRequest。十六进制编码，并且不以“ 0x”为前缀。 |
| refundTo     | 如果需要退款，退款paymail。最大长度为100个字符。             |
| memo         | 从客户到付款主机的备注。                                     |


##### 5.POS广播交易

pos 设备收到交易之后广播，并且将结果通过PaymentACK反馈给支付钱包，同时通知等待的pos设备提示支付结果。

```
PaymentACK {
    payment // Payment. required.
    memo    // string. optional.
    error   // number. optional.
}
```

|         |                                                              |
| ------- | ------------------------------------------------------------ |
| payment | 触发此PaymentACK的付款消息的副本。如果客户实施将付款与PaymentACKs关联的另一种方式，则可能会忽略。 |
| memo    | 显示并提供交易状态                                           |
| error   | 一个数字，指示为什么不接受该交易。 0或未定义表示没有错误。 1或任何其他正整数表示错误。这些错误暂时未定义。建议仅使用“ 1”，并在备忘录中填写有关为何在定义和标准化更多数字之前不接受交易的文字说明。 |

##### 6. pos服务器回调wallet

最终pos将广播结果返回给支付客户端。

```
PaymentResult {
  code            // int.
  transaction // broadcast txid.
  error           // broadcast error message  
}
```


|                 |                               |
| --------------- | ----------------------------- |
| code            | 广播状态码 0为成功 非0为失败. |
| transactionHash | 如果成功，返回hash            |
| error           | 如果失败，返回error message   |

通过BIP271指定，通过HTTPS发送付款消息的钱包,必须设置适当的Content-Type和Accept：

```
Content-Type: application/bitcoinsv-payment
Accept: application/bitcoinsv-paymentack
```

当POS主机的服务器收到“付款”消息时，它必须确定交易是否满足付款条件。当且仅当他们这样做时，它才应将交易广播到比特币p2p网络。


#### References

[BIP 0071](https://github.com/bitcoin/bips/blob/master/bip-0071.mediawiki) : Payment Protocol mime types

[BIP 0072](https://github.com/bitcoin/bips/blob/master/bip-0072.mediawiki) : Payment Protocol bitcoin: URI extensions

[BIP 0270](https://github.com/moneybutton/bips/blob/master/bip-0270.mediawiki) : Payment Protocol

[BIP 0270](https://github.com/moneybutton/bips/blob/master/bip-0270.mediawiki) : Payment Protocol

#### Reference implementation

[ShowMoneyPOSService](https://github.com/showMoneyapp/ShowMoneyPOSService.git
): POSService Simple Code

[ShowMoneyWalletService](https://github.com/showMoneyapp/ShowMoneyWalletService.git):WalletService Simple Code
