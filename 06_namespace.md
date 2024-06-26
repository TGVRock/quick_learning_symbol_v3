# 6.ネームスペース

Symbolブロックチェーンではネームスペースをレンタルしてアドレスやモザイクに視認性の高い単語をリンクさせることができます。
ネームスペースは最大64文字、利用可能な文字は a, b, c, …, z, 0, 1, 2, …, 9, _ , - です。

## 6.1 手数料の計算

ネームスペースのレンタルにはネットワーク手数料とは別にレンタル手数料が発生します。
ネットワークの活性度に比例して価格が変動しますので、取得前に確認するようにしてください。

ルートネームスペースを365日レンタルする場合の手数料を計算します。

#### v2

```js
nwRepo = repo.createNetworkRepository();

rentalFees = await nwRepo.getRentalFees().toPromise();
rootNsperBlock = rentalFees.effectiveRootNamespaceRentalFeePerBlock.compact();
rentalDays = 365;
rentalBlock = rentalDays * 24 * 60 * 60 / 30;
rootNsRenatalFeeTotal = rentalBlock * rootNsperBlock;
console.log("rentalBlock:" + rentalBlock);
console.log("rootNsRenatalFeeTotal:" + rootNsRenatalFeeTotal);
```

#### v3

```js
rentalFees = await fetch(
  new URL('/network/fees/rental', NODE),
  {
    method: 'GET',
    headers: { 'Content-Type': 'application/json' },
  }
)
.then((res) => res.json())
.then((json) => {
  return json;
});

rootNsperBlock = Number(rentalFees.effectiveRootNamespaceRentalFeePerBlock);
rentalDays = 365;
rentalBlock = rentalDays * 24 * 60 * 60 / 30;
rootNsRenatalFeeTotal = rentalBlock * rootNsperBlock;
console.log("rentalBlock:" + rentalBlock);
console.log("rootNsRenatalFeeTotal:" + rootNsRenatalFeeTotal);
```

###### 出力例

```js
> rentalBlock:1051200
> rootNsRenatalFeeTotal:210240000 //約210XYM
```

期間はブロック数で指定します。1ブロックを30秒として計算しました。
最低で30日分はレンタルする必要があります（最大で1825日分）。

サブネームスペースの取得手数料を計算します。

#### v2

```js
childNamespaceRentalFee = rentalFees.effectiveChildNamespaceRentalFee.compact()
console.log(childNamespaceRentalFee);
```

#### v3

```js
childNamespaceRentalFee = Number(rentalFees.effectiveChildNamespaceRentalFee);
console.log(childNamespaceRentalFee);
```

###### 出力例
```js
> 10000000 //10XYM
```

サブネームスペースに期間指定はありません。ルートネームスペースをレンタルしている限り使用できます。

## 6.2 レンタル

ルートネームスペースをレンタルします(例:xembook)

#### v2

```js

tx = sym.NamespaceRegistrationTransaction.createRootNamespace(
    sym.Deadline.create(epochAdjustment),
    "xembook",
    sym.UInt64.fromUint(86400),
    networkType
).setMaxFee(100);
signedTx = alice.sign(tx,generationHash);
await txRepo.announce(signedTx).toPromise();
```

#### v3

```js
// ネームスペース設定
name = "xembook";  // 作成するルートネームスペース

// Tx作成
descriptor = new sdkSymbol.descriptors.NamespaceRegistrationTransactionV1Descriptor(  // Txタイプ:ネームスペース登録Tx
  new sdkSymbol.models.NamespaceId(sdkSymbol.generateNamespaceId(name)),  // ネームスペースID
  sdkSymbol.models.NamespaceRegistrationType.ROOT,  // 登録タイプ : ルートネームスペース
  new sdkSymbol.models.BlockDuration(86400n),       // duration:レンタル期間
  undefined,                                        // 親ネームスペース (ルートの場合は省略可)
  name                                              // ネームスペース
);
tx = facade.createTransactionFromTypedDescriptor(
  descriptor,       // トランザクション Descriptor 設定
  alice.publicKey,  // 署名者公開鍵
  100,              // 手数料乗数
  60 * 60 * 2       // Deadline:有効期限(秒単位)
);

// 署名とアナウンス
sig = alice.signTransaction(tx);
jsonPayload = facade.transactionFactory.static.attachSignature(tx, sig);
await fetch(
  new URL('/transactions', NODE),
  {
    method: 'PUT',
    headers: { 'Content-Type': 'application/json' },
    body: jsonPayload,
  }
)
.then((res) => res.json())
.then((json) => {
  return json;
});
```

サブネームスペースをレンタルします(例:xembook.tomato)

#### v2

```js
subNamespaceTx = sym.NamespaceRegistrationTransaction.createSubNamespace(
    sym.Deadline.create(epochAdjustment),
    "tomato",  //作成するサブネームスペース
    "xembook", //紐づけたいルートネームスペース
    networkType,
).setMaxFee(100);
signedTx = alice.sign(subNamespaceTx,generationHash);
await txRepo.announce(signedTx).toPromise();
```

#### v3

```js
// ネームスペース設定
parentNameId = sdkSymbol.generateNamespaceId("xembook"); // 紐づけたいルートネームスペース
name = "tomato";  // 作成するサブネームスペース
subNamespaceDescriptor = new sdkSymbol.descriptors.NamespaceRegistrationTransactionV1Descriptor(  // Txタイプ:ネームスペース登録Tx
  new sdkSymbol.models.NamespaceId(sdkSymbol.generateNamespaceId(name, parentNameId)),            // ネームスペースID
  sdkSymbol.models.NamespaceRegistrationType.CHILD,   // 登録タイプ : サブネームスペース
  undefined,                                          // duration:レンタル期間 (サブの場合は省略可)
  parentNameId,                                       // 親ネームスペースID
  name                                                // ネームスペース
);
subNamespaceTx = facade.createTransactionFromTypedDescriptor(
  subNamespaceDescriptor, // トランザクション Descriptor 設定
  alice.publicKey,        // 署名者公開鍵
  100,                    // 手数料乗数
  60 * 60 * 2             // Deadline:有効期限(秒単位)
);

// 署名とアナウンス
sig = alice.signTransaction(subNamespaceTx);
jsonPayload = facade.transactionFactory.static.attachSignature(subNamespaceTx, sig);
await fetch(
  new URL('/transactions', NODE),
  {
    method: 'PUT',
    headers: { 'Content-Type': 'application/json' },
    body: jsonPayload,
  }
)
.then((res) => res.json())
.then((json) => {
  return json;
});
```

2階層目のサブネームスペースを作成したい場合は
例えば、xembook.tomato.morningを定義したい場合は以下のようにします。

#### v2

```js
subNamespaceTx = sym.NamespaceRegistrationTransaction.createSubNamespace(
    ,
    "morning",  //作成するサブネームスペース
    "xembook.tomato", //紐づけたいルートネームスペース
    ,
)
```

#### v3

```js
// ネームスペース設定
rootNameId = sdkSymbol.generateNamespaceId("xembook"); // ルートネームスペース
parentNameId = sdkSymbol.generateNamespaceId("tomato", rootNameId);  // 紐づけたい1階層目のサブネームスペース
name = "morning";  // 作成するサブネームスペース

// 以下はサブネームスペース作成と同じ
```

### 有効期限の計算

レンタル済みルートネームスペースの有効期限を計算します。

#### v2

```js
nsRepo = repo.createNamespaceRepository();
chainRepo = repo.createChainRepository();
blockRepo = repo.createBlockRepository();

namespaceId = new sym.NamespaceId("xembook");
nsInfo = await nsRepo.getNamespace(namespaceId).toPromise();
lastHeight = (await chainRepo.getChainInfo().toPromise()).height;
lastBlock = await blockRepo.getBlockByHeight(lastHeight).toPromise();
remainHeight = nsInfo.endHeight.compact() - lastHeight.compact();

endDate = new Date(lastBlock.timestamp.compact() + remainHeight * 30000 + epochAdjustment * 1000)
console.log(endDate);
```

#### v3

```js
namespaceIds = sdkSymbol.generateNamespacePath("xembook"); // ルートネームスペース
namespaceId = namespaceIds[namespaceIds.length - 1];

nsInfo = await fetch(
  new URL('/namespaces/' + namespaceId.toString(16).toUpperCase(), NODE),
  {
    method: 'GET',
    headers: { 'Content-Type': 'application/json' },
  }
)
.then((res) => res.json())
.then((json) => {
  return json.namespace;
});

chainInfo = await fetch(
  new URL('/chain/info', NODE),
  {
    method: 'GET',
    headers: { 'Content-Type': 'application/json' },
  }
)
.then((res) => res.json())
.then((json) => {
  return json;
});
lastHeight = chainInfo.height;

lastBlock = await fetch(
  new URL('/blocks/' + lastHeight, NODE),
  {
    method: 'GET',
    headers: { 'Content-Type': 'application/json' },
  }
)
.then((res) => res.json())
.then((json) => {
  return json.block;
});
remainHeight = nsInfo.endHeight - lastHeight;

endDate = new Date(Number(lastBlock.timestamp) + remainHeight * 30000 + epochAdjustment * 1000);
console.log(endDate);
```

ネームスペース情報の終了ブロックを取得し、現在のブロック高から差し引いた残ブロック数に30秒(平均ブロック生成間隔)を掛け合わせた日時を出力します。
テストネットでは設定した有効期限よりも1日程度更新期限が猶予されます。メインネットはこの値が30日となっていますのでご留意ください

###### 出力例
```js
> Tue Mar 29 2022 18:17:06 GMT+0900 (日本標準時)
```
## 6.3 リンク

### アカウントへのリンク

#### v2

```js
namespaceId = new sym.NamespaceId("xembook");
address = sym.Address.createFromRawAddress("TBIL6D6RURP45YQRWV6Q7YVWIIPLQGLZQFHWFEQ");
tx = sym.AliasTransaction.createForAddress(
    sym.Deadline.create(epochAdjustment),
    sym.AliasAction.Link,
    namespaceId,
    address,
    networkType
).setMaxFee(100);
signedTx = alice.sign(tx,generationHash);
await txRepo.announce(signedTx).toPromise();
```

#### v3

```js
// リンクするネームスペースとアドレスの設定
namespaceIds = sdkSymbol.generateNamespacePath("xembook");
namespaceId = new sdkSymbol.models.NamespaceId(namespaceIds[namespaceIds.length - 1]);
address = new sdkSymbol.Address("TBIL6D6RURP45YQRWV6Q7YVWIIPLQGLZQFHWFEQ");

// Tx作成
descriptor = new sdkSymbol.descriptors.AddressAliasTransactionV1Descriptor(  // Txタイプ:アドレスエイリアスTx
  namespaceId,      // ネームスペースID
  address,          // リンク先アドレス
  sdkSymbol.models.AliasAction.LINK
);
tx = facade.createTransactionFromTypedDescriptor(
  descriptor,       // トランザクション Descriptor 設定
  alice.publicKey,  // 署名者公開鍵
  100,              // 手数料乗数
  60 * 60 * 2       // Deadline:有効期限(秒単位)
);

// 署名とアナウンス
sig = alice.signTransaction(tx);
jsonPayload = facade.transactionFactory.static.attachSignature(tx, sig);
await fetch(
  new URL('/transactions', NODE),
  {
    method: 'PUT',
    headers: { 'Content-Type': 'application/json' },
    body: jsonPayload,
  }
)
.then((res) => res.json())
.then((json) => {
  return json;
});
```

リンク先のアドレスは自分が所有していなくても問題ありません。

### モザイクへリンク

#### v2

```js
namespaceId = new sym.NamespaceId("xembook.tomato");
mosaicId = new sym.MosaicId("3A8416DB2D53xxxx");
tx = sym.AliasTransaction.createForMosaic(
    sym.Deadline.create(epochAdjustment),
    sym.AliasAction.Link,
    namespaceId,
    mosaicId,
    networkType
).setMaxFee(100);
signedTx = alice.sign(tx,generationHash);
await txRepo.announce(signedTx).toPromise();
```

#### v3

```js
// リンクするネームスペースとモザイクの設定
namespaceIds = sdkSymbol.generateNamespacePath("xembook.tomato");
namespaceId = new sdkSymbol.models.NamespaceId(namespaceIds[namespaceIds.length - 1]);
mosaicId = new sdkSymbol.models.MosaicId(0x3A8416DB2D53xxxxn);

// Tx作成
descriptor = new sdkSymbol.descriptors.MosaicAliasTransactionV1Descriptor(  // Txタイプ:モザイクエイリアスTx
  namespaceId,      // ネームスペースID
  mosaicId,         // リンク先モザイク
  sdkSymbol.models.AliasAction.LINK
);
tx = facade.createTransactionFromTypedDescriptor(
  descriptor,       // トランザクション Descriptor 設定
  alice.publicKey,  // 署名者公開鍵
  100,              // 手数料乗数
  60 * 60 * 2       // Deadline:有効期限(秒単位)
);

// 署名とアナウンス
sig = alice.signTransaction(tx);
jsonPayload = facade.transactionFactory.static.attachSignature(tx, sig);
await fetch(
  new URL('/transactions', NODE),
  {
    method: 'PUT',
    headers: { 'Content-Type': 'application/json' },
    body: jsonPayload,
  }
)
.then((res) => res.json())
.then((json) => {
  return json;
});
```

モザイクを作成したアドレスと同一の場合のみリンクできるようです。


## 6.4 未解決で使用

送信先にUnresolvedAccountとして指定して、アドレスを特定しないままトランザクションを署名・アナウンスします。
チェーン側で解決されたアカウントに対しての送信が実施されます。

#### v2

```js
namespaceId = new sym.NamespaceId("xembook");
tx = sym.TransferTransaction.create(
    sym.Deadline.create(epochAdjustment),
    namespaceId, //UnresolvedAccount:未解決アカウントアドレス
    [],
    sym.EmptyMessage,
    networkType
).setMaxFee(100);
signedTx = alice.sign(tx,generationHash);
await txRepo.announce(signedTx).toPromise();
```

#### v3

```js
// UnresolvedAccount 導出
namespaceIds = sdkSymbol.generateNamespacePath("xembook");
namespaceId = new sdkSymbol.models.NamespaceId(namespaceIds[namespaceIds.length - 1]);
unresolvecAccount = sdkSymbol.Address.fromNamespaceId(namespaceId, networkType);

// Tx作成
descriptor = new sdkSymbol.descriptors.TransferTransactionV1Descriptor(  // Txタイプ:転送Tx
  unresolvecAccount,  // UnresolvedAccount:未解決アカウントアドレス
  [],
  new Uint8Array()    // メッセージ
);
tx = facade.createTransactionFromTypedDescriptor(
  descriptor,       // トランザクション Descriptor 設定
  alice.publicKey,  // 署名者公開鍵
  100,              // 手数料乗数
  60 * 60 * 2       // Deadline:有効期限(秒単位)
);

// 署名とアナウンス
sig = alice.signTransaction(tx);
jsonPayload = facade.transactionFactory.static.attachSignature(tx, sig);
await fetch(
  new URL('/transactions', NODE),
  {
    method: 'PUT',
    headers: { 'Content-Type': 'application/json' },
    body: jsonPayload,
  }
)
.then((res) => res.json())
.then((json) => {
  return json;
});
```

送信モザイクにUnresolvedMosaicとして指定して、モザイクIDを特定しないままトランザクションを署名・アナウンスします。

#### v2

```js
namespaceId = new sym.NamespaceId("xembook.tomato");
tx = sym.TransferTransaction.create(
    sym.Deadline.create(epochAdjustment),
    address, 
    [
        new sym.Mosaic(
          namespaceId,//UnresolvedMosaic:未解決モザイク
          sym.UInt64.fromUint(1) //送信量
        )
    ],
    sym.EmptyMessage,
    networkType
).setMaxFee(100);
signedTx = alice.sign(tx,generationHash);
await txRepo.announce(signedTx).toPromise();
```

#### v3

```js
// address = new sdkSymbol.Address("*****");
namespaceIds = sdkSymbol.generateNamespacePath("xembook.tomato");
namespaceId = namespaceIds[namespaceIds.length - 1];

// Tx作成
descriptor = new sdkSymbol.descriptors.TransferTransactionV1Descriptor(  // Txタイプ:転送Tx
  address,          // 受取アドレス
  [
    new sdkSymbol.descriptors.UnresolvedMosaicDescriptor(
      new sdkSymbol.models.UnresolvedMosaicId(namespaceId), // UnresolvedMosaic:未解決モザイク
      new sdkSymbol.models.Amount(1n)                       // 送信量
    )
  ],
  new Uint8Array()  // メッセージ
);
tx = facade.createTransactionFromTypedDescriptor(
  descriptor,       // トランザクション Descriptor 設定
  alice.publicKey,  // 署名者公開鍵
  100,              // 手数料乗数
  60 * 60 * 2       // Deadline:有効期限(秒単位)
);

// 署名とアナウンス
sig = alice.signTransaction(tx);
jsonPayload = facade.transactionFactory.static.attachSignature(tx, sig);
await fetch(
  new URL('/transactions', NODE),
  {
    method: 'PUT',
    headers: { 'Content-Type': 'application/json' },
    body: jsonPayload,
  }
)
.then((res) => res.json())
.then((json) => {
  return json;
});
```

XYMをネームスペースで使用する場合は以下のように指定します。

#### v2

```js
namespaceId = new sym.NamespaceId("symbol.xym");
```
```js
> NamespaceId {fullName: 'symbol.xym', id: Id}
    fullName: "symbol.xym"
    id: Id {lower: 1106554862, higher: 3880491450}
```

Idは内部ではUint64と呼ばれる数値 `{lower: 1106554862, higher: 3880491450}` で保持されています。

#### v3

```js
namespaceIds = sdkSymbol.generateNamespacePath("symbol.xym");
namespaceId = namespaceIds[namespaceIds.length - 1];
```
```js
> 16666583871264174062n
```

## 6.5 参照

アドレスへリンクしたネームスペースの参照します

#### v2

```js
nsRepo = repo.createNamespaceRepository();

namespaceInfo = await nsRepo.getNamespace(new sym.NamespaceId("xembook")).toPromise();
console.log(namespaceInfo);
```
###### 出力例
```js
NamespaceInfo
    active: true
  > alias: AddressAlias
        address: Address {address: 'TBIL6D6RURP45YQRWV6Q7YVWIIPLQGLZQFHWFEQ', networkType: 152}
        mosaicId: undefined
        type: 2 //AliasType
    depth: 1
    endHeight: UInt64 {lower: 500545, higher: 0}
    index: 1
    levels: [NamespaceId]
    ownerAddress: Address {address: 'TBIL6D6RURP45YQRWV6Q7YVWIIPLQGLZQFHWFEQ', networkType: 152}
    parentId: NamespaceId {id: Id}
    registrationType: 0 //NamespaceRegistrationType
    startHeight: UInt64 {lower: 324865, higher: 0}
```

#### v3

```js
namespaceIds = sdkSymbol.generateNamespacePath("xembook");
nameId = namespaceIds[namespaceIds.length - 1];
namespaceInfo = await fetch(
  new URL('/namespaces/' + nameId.toString(16).toUpperCase(), NODE),
  {
    method: 'GET',
    headers: { 'Content-Type': 'application/json' },
  }
)
.then((res) => res.json())
.then((json) => {
  return json;
});
console.log(namespaceInfo);
```
###### 出力例
```js
> {meta: {…}, namespace: {…}, id: '6492E0242F7CE156B00D7580'}
    id: "6492E0242F7CE156B00D7580"
  > meta: 
      active: true
      index: 9
  > namespace: 
    > alias: 
        address: "981CFF6235B50FEAB020C824D5E0EBD4894F8D605DD2DF93"
        type: 2 //AliasType
      depth: 1
      endHeight: "1432950"
      level0: "A43415EB268C7385"
      ownerAddress: "981CFF6235B50FEAB020C824D5E0EBD4894F8D605DD2DF93"
      parentId: "0000000000000000"
      registrationType: 0 // NamespaceRegistrationType
      startHeight: "393270"
      version: 1
```

AliasTypeは以下の通りです。
```js
{0: 'None', 1: 'Mosaic', 2: 'Address'}
```

NamespaceRegistrationTypeは以下の通りです。
```js
{0: 'RootNamespace', 1: 'SubNamespace'}
```

モザイクへリンクしたネームスペースを参照します。

#### v2

```js
nsRepo = repo.createNamespaceRepository();

namespaceInfo = await nsRepo.getNamespace(new sym.NamespaceId("xembook.tomato")).toPromise();
console.log(namespaceInfo);
```
###### 出力例
```js
NamespaceInfo
  > active: true
    alias: MosaicAlias
        address: undefined
        mosaicId: MosaicId
        id: Id {lower: 1360892257, higher: 309702839}
        type: 1 //AliasType
    depth: 2
    endHeight: UInt64 {lower: 500545, higher: 0}
    index: 1
    levels: (2) [NamespaceId, NamespaceId]
    ownerAddress: Address {address: 'TBIL6D6RURP45YQRWV6Q7YVWIIPLQGLZQFHWFEQ', networkType: 152}
    parentId: NamespaceId {id: Id}
    registrationType: 1 //NamespaceRegistrationType
    startHeight: UInt64 {lower: 324865, higher: 0}
```

#### v3

```js
namespaceIds = sdkSymbol.generateNamespacePath("xembook.tomato");
nameId = namespaceIds[namespaceIds.length - 1];
namespaceInfo = await fetch(
  new URL('/namespaces/' + nameId.toString(16).toUpperCase(), NODE),
  {
    method: 'GET',
    headers: { 'Content-Type': 'application/json' },
  }
)
.then((res) => res.json())
.then((json) => {
  return json;
});
console.log(namespaceInfo);
```
###### 出力例
```js
> {meta: {…}, namespace: {…}, id: '6492E0242F7CE156B00D758B'}
    id: "6492E0242F7CE156B00D758B"
  > meta: 
      active: true
      index: 9
  > namespace: 
    > alias: 
        mosaicId: "663B178E904CADB8"
        type: 1 //AliasType
      depth: 1
      endHeight: "1432950"
      level0: "A43415EB268C7385"
      level1: "FA547FD28C836431"
      ownerAddress: "981CFF6235B50FEAB020C824D5E0EBD4894F8D605DD2DF93"
      parentId: "A43415EB268C7385"
      registrationType: 1 // NamespaceRegistrationType
      startHeight: "393270"
      version: 1
```

### 逆引き

アドレスに紐づけられたネームスペースを全て調べます。

#### v2

```js
nsRepo = repo.createNamespaceRepository();

accountNames = await nsRepo.getAccountsNames(
  [sym.Address.createFromRawAddress("TBIL6D6RURP45YQRWV6Q7YVWIIPLQGLZQFHWFEQ")]
).toPromise();

namespaceIds = accountNames[0].names.map(name=>{
  return name.namespaceId;
});
console.log(namespaceIds);
```

#### v3

```js
body = {
  "addresses": [
    "TBIL6D6RURP45YQRWV6Q7YVWIIPLQGLZQFHWFEQ"
  ]
};
accountNames = await fetch(
  new URL('/namespaces/account/names', NODE),
  {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify(body),
  }
)
.then((res) => res.json())
.then((json) => {
  return json.accountNames;
});

namespaceIds = accountNames[0].names;
console.log(namespaceIds);
```

モザイクに紐づけられたネームスペースを全て調べます。

#### v2

```js
nsRepo = repo.createNamespaceRepository();

mosaicNames = await nsRepo.getMosaicsNames(
  [new sym.MosaicId("72C0212E67A08BCE")]
).toPromise();

namespaceIds = mosaicNames[0].names.map(name=>{
  return name.namespaceId;
});
console.log(namespaceIds);
```

#### v3

```js
body = {
  "mosaicIds": [
    "72C0212E67A08BCE"
  ]
};
mosaicNames = await fetch(
  new URL('/namespaces/mosaic/names', NODE),
  {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify(body),
  }
)
.then((res) => res.json())
.then((json) => {
  return json.mosaicNames;
});

namespaceIds = mosaicNames[0].names;
console.log(namespaceIds);
```

### レシートの参照

トランザクションに使用されたネームスペースをブロックチェーン側がどう解決したかを確認します。

#### v2

```js
receiptRepo = repo.createReceiptRepository();
state = await receiptRepo.searchAddressResolutionStatements({height:179401}).toPromise();
```
###### 出力例
```js
data: Array(1)
  0: ResolutionStatement
    height: UInt64 {lower: 179401, higher: 0}
    resolutionEntries: Array(1)
      0: ResolutionEntry
        resolved: Address {address: 'TBIL6D6RURP45YQRWV6Q7YVWIIPLQGLZQFHWFEQ', networkType: 152}
        source: ReceiptSource {primaryId: 1, secondaryId: 0}
    resolutionType: 0 //ResolutionType
    unresolved: NamespaceId
      id: Id {lower: 646738821, higher: 2754876907}
```

ResolutionTypeは以下の通りです。
```js
{0: 'Address', 1: 'Mosaic'}
```

#### v3

v3 では、ネームスペースを紐づけている対象によってアクセスする際のURLが異なります。

##### アドレスの場合

```js
query = new URLSearchParams({
  "height": 179401  // 6.4 で作成したTxが取り込まれたブロック高
});
state = await fetch(
  new URL('/statements/resolutions/address?' + query.toString(), NODE), // アドレス用のURI
  {
    method: 'GET',
    headers: { 'Content-Type': 'application/json' },
  }
)
.then((res) => res.json())
.then((json) => {
  return json.data;
});
```

##### モザイクの場合

```js
query = new URLSearchParams({
  "height": 179401  // 6.4 で作成したTxが取り込まれたブロック高
});
state = await fetch(
  new URL('/statements/resolutions/mosaic?' + query.toString(), NODE), // モザイク用のURI
  {
    method: 'GET',
    headers: { 'Content-Type': 'application/json' },
  }
)
.then((res) => res.json())
.then((json) => {
  return json.data;
});
```

###### 出力例

```js
> data: Array(1)
  > 0: 
      id: "64B6A3742F7CE156B0108826"
    > meta: 
        timestamp: "22440529194"
    > statement: 
        height: "646246"
        resolutionEntries: Array(1)
        > 0: 
            resolved: "986A4B98552007D13947E6F4F1202604BD26CC4018BACACC"
          > source: 
              primaryId: 1
              secondaryId: 0
        unresolved: "99534C3F83755AD3D9000000000000000000000000000000"
```

#### 注意事項
ネームスペースはレンタル制のため、過去のトランザクションで使用したネームスペースのリンク先と
現在のネームスペースのリンク先が異なる可能性があります。
過去のデータを参照する際などに、その時どのアカウントにリンクしていたかなどを知りたい場合は
必ずレシートを参照するようにしてください。

## 6.6 現場で使えるヒント

### 外部ドメインとの相互リンク

ネームスペースは重複取得がプロトコル上制限されているため、
インターネットドメインや実世界で周知されている商標名と同一のネームスペースを取得し、
外部(公式サイトや印刷物など)からネームスペース存在の認知を公表することで、
Symbol上のアカウントのブランド価値を構築することができます
(法的な効力については調整が必要です)。
外部ドメイン側のハッキングあるいは、Symbol側でのネームスペース更新忘れにはご注意ください。


#### ネームスペースを取得するアカウントについての注意
ネームスペースはレンタル期限という概念をもつ機能です。
今のところ、取得したネームスペースは放棄か延長の選択肢しかありません。
運用譲渡などが発生する可能性のあるシステムでネームスペース活用を検討する場合は
マルチシグ化(9章)したアカウントでネームスペースを取得することをおすすめします。

