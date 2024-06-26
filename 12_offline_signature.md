# 12.オフライン署名

ロック機構の章で、アナウンスしたトランザクションをハッシュ値指定でロックして、  
複数の署名（オンライン署名）を集めるアグリゲートトランザクションを紹介しました。    
この章では、トランザクションを事前に署名を集めてノードにアナウンスするオフライン署名について説明します。  

## 手順

Aliceが起案者となりトランザクションを作成し、署名します。  
次にBobが署名してAliceに返します。  
最後にAliceがトランザクションを結合してネットワークにアナウンスします。  


## 12.1 トランザクション作成

#### v2

```js
bob = sym.Account.generateNewAccount(networkType);

innerTx1 = sym.TransferTransaction.create(
    undefined,
    bob.address, 
    [],
    sym.PlainMessage.create("tx1"),
    networkType
);

innerTx2 = sym.TransferTransaction.create(
    undefined,
    alice.address, 
    [],
    sym.PlainMessage.create("tx2"),
    networkType
);

aggregateTx = sym.AggregateTransaction.createComplete(
    sym.Deadline.create(epochAdjustment),
    [
      innerTx1.toAggregate(alice.publicAccount),
      innerTx2.toAggregate(bob.publicAccount)
    ],
    networkType,
    [],
).setMaxFeeForAggregate(100, 1);

signedTx =  alice.sign(aggregateTx,generationHash);
signedHash = signedTx.hash;
signedPayload = signedTx.payload;

console.log(signedPayload);
```

#### v3

```js
bob = facade.createAccount(sdkCore.PrivateKey.random());

// アグリゲートTxに含めるTxを作成
innerTxDescriptor1 = new sdkSymbol.descriptors.TransferTransactionV1Descriptor(  // Txタイプ:転送Tx
  bob.address,  // Bobへの送信
  [],
  '\0tx1'       // メッセージ
);
innerTx1 = facade.createEmbeddedTransactionFromTypedDescriptor(
  innerTxDescriptor1, // トランザクション Descriptor 設定
  alice.publicKey,    // Aliceから
);

innerTxDescriptor2 = new sdkSymbol.descriptors.TransferTransactionV1Descriptor(  // Txタイプ:転送Tx
  alice.address,  // Aliceへの送信
  [],
  '\0tx2'         // メッセージ
);
innerTx2 = facade.createEmbeddedTransactionFromTypedDescriptor(
  innerTxDescriptor2, // トランザクション Descriptor 設定
  bob.publicKey,      // Bobから
);

embeddedTransactions = [
  innerTx1,
  innerTx2
];

// アグリゲートTx作成
aggregateDescriptor = new sdkSymbol.descriptors.AggregateCompleteTransactionV2Descriptor(
  facade.static.hashEmbeddedTransactions(embeddedTransactions),
  embeddedTransactions
);
aggregateTx = facade.createTransactionFromTypedDescriptor(
  aggregateDescriptor,  // トランザクション Descriptor 設定
  alice.publicKey,      // 署名者公開鍵
  100,                  // 手数料乗数
  60 * 60 * 2,          // Deadline:有効期限(秒単位)
  1                     // 連署者数
);

// 署名
sig = alice.signTransaction(aggregateTx);
jsonPayload = facade.transactionFactory.static.attachSignature(aggregateTx, sig);

signedHash = facade.hashTransaction(aggregateTx).toString();
signedPayload = JSON.parse(jsonPayload).payload;

console.log(signedPayload);
```

###### 出力例
```js
>580100000000000039A6555133357524A8F4A832E1E596BDBA39297BC94CD1D0728572EE14F66AA71ACF5088DB6F0D1031FF65F2BBA7DA9EE3A8ECF242C2A0FE41B6A00A2EF4B9020E5C72B0D5946C1EFEE7E5317C5985F106B739BB0BC07E4F9A288417B3CD6D26000000000198414100AF000000000000D4641CD902000000306771D758886F1529F9B61664B0450ED138B27CC5E3AE579C16D550EDEE5791B00000000000000054000000000000000E5C72B0D5946C1EFEE7E5317C5985F106B739BB0BC07E4F9A288417B3CD6D26000000000198544198A1BE13194C0D18897DD88FE3BC4860B8EEF79C6BC8C8720400000000000000007478310000000054000000000000003C4ADF83264FF73B4EC1DD05B490723A8CFFAE1ABBD4D4190AC4CAC1E6505A5900000000019854419850BF0FD1A45FCEE211B57D0FE2B6421EB81979814F629204000000000000000074783200000000
```

署名を行い、signedHash,signedPayloadを出力します。  
signedPayloadをBobに渡して署名を促します。  

## 12.2 Bobによる連署


Aliceから受け取ったsignedPayloadでトランザクションを復元します。

#### v2

```js
tx = sym.TransactionMapping.createFromPayload(signedPayload);
console.log(tx);
```
###### 出力例
```js
> AggregateTransaction
    cosignatures: []
    deadline: Deadline {adjustedValue: 12197090355}
  > innerTransactions: Array(2)
      0: TransferTransaction {type: 16724, networkType: 152, version: 1, deadline: Deadline, maxFee: UInt64, …}
      1: TransferTransaction {type: 16724, networkType: 152, version: 1, deadline: Deadline, maxFee: UInt64, …}
    maxFee: UInt64 {lower: 44800, higher: 0}
    networkType: 152
    payloadSize: undefined
    signature: "4999A8437DA1C339280ED19BE0814965B73D60A1A6AF2F3856F69FBFF9C7123427757247A231EB89BB8844F37AC6F7559F859E2FDE39B8FA58A57F36DDB3B505"
    signer: PublicAccount
      address: Address {address: 'TBXUTAX6O6EUVPB6X7OBNX6UUXBMPPAFX7KE5TQ', networkType: 152}
      publicKey: "D4933FC1E4C56F9DF9314E9E0533173E1AB727BDB2A04B59F048124E93BEFBD2"
    transactionInfo: undefined
    type: 16705
    version: 1
```

#### v3

```js
tx = facade.transactionFactory.static.deserialize(sdkCore.utils.hexToUint8(JSON.parse(jsonPayload).payload));
console.log(tx);
```
###### 出力例
```js
> AggregateBondedTransactionV2 
    _aggregateTransactionHeaderReserved_1: 0
    _cosignatures: []
    _deadline: Timestamp {size: 8, isSigned: false, value: 22883170004n}
    _entityBodyReserved_1: 0
    _fee: Amount {size: 8, isSigned: false, value: 44800n}
    _network: NetworkType {value: 152}
    _signature: Signature {bytes: Uint8Array(64)}
    _signerPublicKey: PublicKey {bytes: Uint8Array(32)}
  > _transactions: Array(2)
      0: EmbeddedTransferTransactionV1 {_signerPublicKey: PublicKey, _version: 1, _network: NetworkType, _type: TransactionType, _recipientAddress: UnresolvedAddress, …}
      1: EmbeddedTransferTransactionV1 {_signerPublicKey: PublicKey, _version: 1, _network: NetworkType, _type: TransactionType, _recipientAddress: UnresolvedAddress, …}
      length: 2
    _transactionsHash: Hash256 {bytes: Uint8Array(32)}
    _type: TransactionType {value: 16961}
    _verifiableEntityHeaderReserved_1: 0
    _version: 2
```

念のため、Aliceがすでに署名したトランザクション（ペイロード）かどうかを検証します。

#### v2

```js
Buffer = require("/node_modules/buffer").Buffer;
res = tx.signer.verifySignature(
    tx.getSigningBytes([...Buffer.from(signedPayload,'hex')],[...Buffer.from(generationHash,'hex')]),
    tx.signature
);
console.log(res);
```

#### v3

```js
res = facade.verifyTransaction(tx, tx.signature);
console.log(res);
```

###### 出力例
```js
> true
```

ペイロードがsigner、つまりAliceによって署名されたものであることが確認できました。
次にBobが連署します。

#### v2

```js
bobSignedTx = sym.CosignatureTransaction.signTransactionPayload(bob, signedPayload, generationHash);
bobSignedTxSignature = bobSignedTx.signature;
bobSignedTxSignerPublicKey = bobSignedTx.signerPublicKey;
```

#### v3

```js
bobCosignature = bob.cosignTransaction(tx, true);
bobSignedTxSignature = bobCosignature.signature;
bobSignedTxSignerPublicKey = bobCosignature.signerPublicKey;
```

CosignatureTransactionで署名を行い、bobSignedTxSignature,bobSignedTxSignerPublicKeyを出力しAliceに返却します。  
Bobが全ての署名を揃えられる場合は、Aliceに返却しなくてもBobがアナウンスすることも可能です。

## 12.3 Aliceによるアナウンス

AliceはBobからbobSignedTxSignature,bobSignedTxSignerPublicKeyを受け取ります。  
また事前にAlice自身で作成したsignedPayloadを用意します。  

#### v2

```js
signedHash = sym.Transaction.createTransactionHash(signedPayload,Buffer.from(generationHash, 'hex'));
cosignSignedTxs = [
    new sym.CosignatureSignedTransaction(signedHash,bobSignedTxSignature,bobSignedTxSignerPublicKey)
];

recreatedTx = sym.TransactionMapping.createFromPayload(signedPayload);

cosignSignedTxs.forEach((cosignedTx) => {
    signedPayload += cosignedTx.version.toHex() + cosignedTx.signerPublicKey + cosignedTx.signature;
});

size = `00000000${(signedPayload.length / 2).toString(16)}`;
formatedSize = size.substr(size.length - 8, size.length);
littleEndianSize = formatedSize.substr(6, 2) + formatedSize.substr(4, 2) + formatedSize.substr(2, 2) + formatedSize.substr(0, 2);

signedPayload = littleEndianSize + signedPayload.substr(8, signedPayload.length - 8);
signedTx = new sym.SignedTransaction(signedPayload, signedHash, alice.publicKey, recreatedTx.type, recreatedTx.networkType);

await txRepo.announce(signedTx).toPromise();
```

後半部分の連署を追加する部分がPayload(サイズ値)を直接操作しているので少し難しいかもしれません。
Aliceの秘密鍵で再度署名できる場合はcosignSignedTxsを生成した後、以下のように連署済みトランザクションを生成することも可能です。

```js
resignedTx = recreatedTx.signTransactionGivenSignatures(alice, cosignSignedTxs, generationHash);
await txRepo.announce(resignedTx).toPromise();
```

#### v3

```js
recreatedTx = facade.transactionFactory.static.deserialize(sdkCore.utils.hexToUint8(JSON.parse(jsonPayload).payload));

// 連署者の署名を追加
cosignature = new sdkSymbol.models.Cosignature();
signTxHash = facade.hashTransaction(aggregateTx);
cosignature.parentHash = signTxHash;
cosignature.version = 0n;
cosignature.signerPublicKey = bobSignedTxSignerPublicKey;
cosignature.signature = bobSignedTxSignature;
recreatedTx.cosignatures.push(cosignature);

signedPayload = sdkCore.utils.uint8ToHex(recreatedTx.serialize());

// アナウンス
await fetch(
  new URL('/transactions', NODE),
  {
    method: 'PUT',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({"payload": signedPayload}),
  }
)
.then((res) => res.json())
.then((json) => {
  return json;
});
```

## 12.4 現場で使えるヒント

### マーケットプレイスレス
ボンデッドトランザクションと異なりハッシュロックのための手数料(10XYM)を気にする必要がありません。    
ペイロードを共有できる場が存在する場合、売り手は考えられるすべての買い手候補に対してペイロードを作成して交渉開始を待つことができます。
（複数のトランザクションが個別に実行されないように、1つしか存在しない領収書NFTをアグリゲートトランザクションに混ぜ込むなどして排他制御をしてください）。
この交渉に専用のマーケットプレイスを構築する必要はありません。
SNSのタイムラインをマーケットプレイスにしたり、必要に応じて任意の時間や空間でワンタイムマーケットプレイスを展開することができます。  

ただ、オフラインで署名を交換するため、なりすましのハッシュ署名要求には気を付けましょう。  
（必ず検証可能なペイロードからハッシュを生成して署名するようにしてください）  
