---
title: "Symbolブロックチェーン入門"
emoji: "⛓"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["javascript", "typescript", "nodejs", "blockchain", "symbol"]
published: false
---

Symbolブロックチェーンのユースケースとして、Symbolブロックチェーンの基軸トークンであるXYMを用いた決済があります。この記事ではその際に必要となる以下の要素技術についてTypeScriptでのサンプルコードで解説します。

1. XYM価格情報取得
2. 請求QRコード生成
3. 着金トランザクション検知

## 前提

- Symbolブロックチェーンのテストネットでの実装を想定
- 支払いを受ける側は日本円ベースでの請求を行いたい(=日本円にしていくら分のXYMを支払ってもらえばいいかを知りたい)
  - 例として1000円のXYMを払ってもらいたい状況を想定
  - 支払いを受ける側のアドレスは`TARDV42KTAIZEF64EQT4NXT7K55DHWBEFIXVJQY`とする
  - 着金トランザクションの識別のため請求側でUUIDを発行しトランザクションにメッセージとして付与してもらう
- 支払いを行う側はSymbol公式モバイルウォレット[https://play.google.com/store/apps/details?id=nem.group.symbol.wallet](https://play.google.com/store/apps/details?id=nem.group.symbol.wallet)やArcana[https://play.google.com/store/apps/details?id=com.shu.software.symbol_arcana](https://play.google.com/store/apps/details?id=com.shu.software.symbol_arcana)等のモバイルウォレットでQRコードを読み込んで支払いを行いたい状況を想定
- (説明の簡略化のため)支払いを行う側のアカウントはマルチシグアカウントではないものと仮定

## 環境構築

Node.jsでTypeScriptを実行できる環境を構築します。

```shell
# この記事の内容を試すためのディレクトリ作成
mkdir introduction-to-symbol-blockchain
cd introduction-to-symbol-blockchain

# Node.js初期化
npm init -y

# TypeScript関連のパッケージをインストール
npm install -D typescript ts-node

# 「(1) XYM価格情報取得」の際に利用するaxiosをインストール
npm install axios

# 「(2) 請求QRコード生成」の際に利用するsymbol-sdk, rxjs, symbol-qr-libraryをインストール
npm install symbol-sdk@2 rxjs symbol-qr-library

```

`.gitignore`ファイルを作成し、以下のように`node_modules`を除外しておきます。

```gitignore
node_modules/

```

## (1) XYM価格情報取得

日本円ベースでのXYMでの支払い請求を行いたい場合、1[XYM]=?[JPY]なのかという価格情報が必要になります。XYM価格の取得方法の一例として暗号資産取引所Zaifの公開API[https://zaif-api-document.readthedocs.io/ja/latest/PublicAPI.html#id22](https://zaif-api-document.readthedocs.io/ja/latest/PublicAPI.html#id22)を用いてXYM価格を取得する方法を解説します。

通貨ペアXYM/JPYの価格情報を取得するには[https://api.zaif.jp/api/1/ticker/xym_jpy](https://api.zaif.jp/api/1/ticker/xym_jpy)にGETリクエストを送信すればOKです。

`index.ts`ファイルを作成し以下のような実装を行い`npx ts-node index.ts`コマンドで実行してみましょう。

```ts:index.ts
import axios from "axios";

const jpyAmount = 1000;
const recipientRawAddress = "TARDV42KTAIZEF64EQT4NXT7K55DHWBEFIXVJQY";
const uuid = crypto.randomUUID(); // トランザクションの識別のためにUUIDを生成(UUIDである必要は特になく適当な文字列で検証可能であればなんでもOK)

(async () => {
  // (1) XYM価格取得
  // Zaifの公開APIを利用してXYMの価格を取得
  const xymPriceResponse = await axios.get(
    "https://api.zaif.jp/api/1/ticker/xym_jpy"
  );
  const xymPriceData = xymPriceResponse.data;
  const xymPriceLast = xymPriceData.last;

  // XYM価格情報と日本円での請求金額からXYMの請求金額を算出
  // XYMの最小単位は10^(-6)[XYM]であるため小数点以下6桁までの値を算出する≒可分性6
  const xymAbsoluteAmount = Math.round((jpyAmount / xymPriceLast) * 1000000); // システム的には10^6倍した整数値で扱う(絶対値としての数量表記=μXYM単位)
  const xymRelativeAmount = xymAbsoluteAmount / 1000000; // ユーザー向けには絶対値を10^6で割った値を表示する(相対値としての数量表記=XYM単位)

  console.log({
    xymPriceData,
    xymPriceLast,
    xymAbsoluteAmount,
    xymRelativeAmount,
  });
})();

```

以下のような結果が表示されれば成功です。

```shell
{
  xymPriceData: {
    last: 5.4529,
    high: 5.6012,
    low: 5.4529,
    vwap: 5.5421,
    volume: 1871411.8,
    bid: 5.4529,
    ask: 5.4809
  },
  xymPriceLast: 5.4529,
  xymAbsoluteAmount: 183388656,
  xymRelativeAmount: 183.388656
}

```

## (2) 請求QRコードの生成

(1)で結果的に得られたXYM数量と支払いを受ける側のアドレスと入金検知用のUUIDのメッセージを指定して、請求するトランザクションのデータを作成し、そのデータをQRコードに変換することで請求QRコードを生成します。

`index.ts`ファイルを以下のように追記修正し、`npx ts-node index.ts`コマンドで実行してみましょう。

```ts:index.ts
import crypto from "crypto";
import axios from "axios";
import { firstValueFrom } from "rxjs";
import {
  Address,
  Deadline,
  PlainMessage,
  RepositoryFactoryHttp,
  TransferTransaction,
} from "symbol-sdk";
import { QRCodeGenerator, TransactionQR } from "symbol-qr-library";

const jpyAmount = 1000;
const recipientRawAddress = "TARDV42KTAIZEF64EQT4NXT7K55DHWBEFIXVJQY";
const uuid = crypto.randomUUID(); // トランザクションの識別のためにUUIDを生成(UUIDである必要は特になく適当な文字列で検証可能であればなんでもOK)

(async () => {
  // (1) XYM価格取得
  // Zaifの公開APIを利用してXYMの価格を取得
  const xymPriceResponse = await axios.get(
    "https://api.zaif.jp/api/1/ticker/xym_jpy"
  );
  const xymPriceData = xymPriceResponse.data;
  const xymPriceLast = xymPriceData.last;

  // XYM価格情報と日本円での請求金額からXYMの請求金額を算出
  // XYMの最小単位は10^(-6)[XYM]であるため小数点以下6桁までの値を算出する≒可分性6
  const xymAbsoluteAmount = Math.round((jpyAmount / xymPriceLast) * 1000000); // システム的には10^6倍した整数値で扱う(絶対値としての数量表記=μXYM単位)
  const xymRelativeAmount = xymAbsoluteAmount / 1000000; // ユーザー向けには絶対値を10^6で割った値を表示する(相対値としての数量表記=XYM単位)

  console.log({
    xymPriceData,
    xymPriceLast,
    xymAbsoluteAmount,
    xymRelativeAmount,
  });

  // (2) 請求QRコード生成
  // Symbolブロックチェーンの情報を取得するためのノードを指定(テストネットのノードリストはhttps://symbolnodes.org/nodes_testnet/)
  const nodeUrl = "https://sym-test-04.opening-line.jp:3001";
  // ノードのURLを指定してREST APIやWebSocket APIを利用するためのクラスを生成
  const repositoryFactoryHttp = new RepositoryFactoryHttp(nodeUrl);

  // ノードに対してREST APIを利用してネットワーク固有の設定値(epochAdjustment, networkCurrencies, networkType)を取得
  const epochAdjustment = await firstValueFrom(
    repositoryFactoryHttp.getEpochAdjustment()
  ); // ブロックチェーンの時刻とシステム時刻の基準値の違いを補正するための値を取得
  const networkCurrencies = await firstValueFrom(
    repositoryFactoryHttp.getCurrencies()
  ); // ブロックチェーンネットワークの基軸通貨の情報を取得
  const networkType = await firstValueFrom(
    repositoryFactoryHttp.getNetworkType()
  ); // ブロックチェーンネットワークの種類(メインネット/テストネット等の識別)を取得
  const generationHash = await firstValueFrom(
    repositoryFactoryHttp.getGenerationHash()
  ); // ブロックチェーンネットワークのハッシュ値(ネットワーク固有の値)を取得

  // XYMでの支払いを受け付けるためのトランザクションにセットするために必要なデータを生成
  const deadline = Deadline.create(epochAdjustment); // トランザクションの有効期限を生成(デフォルトだと現在時刻から2時間後)
  const recipientAddress = Address.createFromRawAddress(recipientRawAddress); // トランザクションの送金先アドレスを生成
  const xym = networkCurrencies.currency.createAbsolute(xymAbsoluteAmount); // トランザクションの送金額を含めたトークンのデータを生成
  const mosaics = [xym]; // トランザクションの送金額を含めたトークンのデータを配列に格納
  const message = PlainMessage.create(uuid); // トランザクションのメッセージを生成(支払い確認時にトランザクションを識別できるようUUIDをセット)
  const feeMultiplier = 100; // トランザクションの手数料率を指定(各ノード毎に設定値が異なりデフォルトでは100で多くのノードが100以下なので100を指定)

  // トランザクションデータを生成
  const transferTransaction = TransferTransaction.create(
    deadline,
    recipientAddress,
    mosaics,
    message,
    networkType
  ).setMaxFee(feeMultiplier);

  // トランザクションデータからQRコードのデータを生成
  const transactionQR: TransactionQR = QRCodeGenerator.createTransactionRequest(
    transferTransaction,
    networkType,
    generationHash
  );

  // QRコードのデータをJSON文字列形式に変換
  const transactionQRJsonString = transactionQR.toJSON();
  console.log({ transactionQRJsonString });

  // HTMLのimgタグに埋め込むためのBase64エンコードされたpng画像データに変換
  const transactionQRBase64 = await firstValueFrom(transactionQR.toBase64());
  console.log({ transactionQRBase64 });

  // 公式デスクトップウォレット向けにトランザクションURIを生成
  const transactionPayload = JSON.parse(transactionQRJsonString).data.payload;
  const transactionURI = `web+symbol://transaction?data=${transactionPayload}`;
  console.log({ transactionURI });
})();

```

(1)の結果に加えて以下のような結果が表示されれば成功です。

```shell
{
  transactionQRJsonString: '{"v":3,"type":3,"network_id":152,"chain_id":"49D6E1CE276A85B70EAFE52349AACCA389302E7A9754BCF1221E79494FC665A4","data":{"payload":"D500000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000198544134530000000000005735034F0200000098223AF34A98119217DC2427C6DE7F577A33D8242A2F54C32500010000000000CE8BA0672E21C0721FC7EF0A000000000065333036376263342D656437622D346361382D396639372D643338353161663766383365"}}'
}
{
  transactionQRBase64: 'data:image/png;base64,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'
}
{
  transactionURI: 'web+symbol://transaction?data=D500000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000198544134530000000000005735034F0200000098223AF34A98119217DC2427C6DE7F577A33D8242A2F54C32500010000000000CE8BA0672E21C0721FC7EF0A000000000065333036376263342D656437622D346361382D396639372D643338353161663766383365'
}

```

## (3) 着金トランザクション検知

最後に(2)のQRコードを支払いを行うユーザー側で読み取ってトランザクションが送信された際に、その着金トランザクションを検知するための処理を解説します。

WebSocketでブロックチェーンの状態をモニタリングしてトランザクションがネットワークにアナウンスされたイベントや、トランザクションがブロックに含まれたイベントを受け取って、トランザクションの内容を確認することで、着金トランザクション検知を行います。

`index.ts`ファイルを以下のように追記修正し、`npx ts-node index.ts`コマンドで実行し、Symbol公式デスクトップウォレットの「トランザクションURIをインポート」機能を使ってトランザクションを送信し、着金トランザクション検知が意図通りできていることを確認してみましょう。

```ts:index.ts
import crypto from "crypto";
import axios from "axios";
import { firstValueFrom } from "rxjs";
import {
  Address,
  Deadline,
  MessageType,
  PlainMessage,
  RepositoryFactoryHttp,
  TransferTransaction,
} from "symbol-sdk";
import { QRCodeGenerator, TransactionQR } from "symbol-qr-library";

const jpyAmount = 1000;
const recipientRawAddress = "TARDV42KTAIZEF64EQT4NXT7K55DHWBEFIXVJQY";
const uuid = crypto.randomUUID(); // トランザクションの識別のためにUUIDを生成(UUIDである必要は特になく適当な文字列で検証可能であればなんでもOK)

(async () => {
  // (1) XYM価格取得
  // Zaifの公開APIを利用してXYMの価格を取得
  const xymPriceResponse = await axios.get(
    "https://api.zaif.jp/api/1/ticker/xym_jpy"
  );
  const xymPriceData = xymPriceResponse.data;
  const xymPriceLast = xymPriceData.last;

  // XYM価格情報と日本円での請求金額からXYMの請求金額を算出
  // XYMの最小単位は10^(-6)[XYM]であるため小数点以下6桁までの値を算出する≒可分性6
  const xymAbsoluteAmount = Math.round((jpyAmount / xymPriceLast) * 1000000); // システム的には10^6倍した整数値で扱う(絶対値としての数量表記=μXYM単位)
  const xymRelativeAmount = xymAbsoluteAmount / 1000000; // ユーザー向けには絶対値を10^6で割った値を表示する(相対値としての数量表記=XYM単位)

  console.log({
    xymPriceData,
    xymPriceLast,
    xymAbsoluteAmount,
    xymRelativeAmount,
  });

  // (2) 請求QRコード生成
  // Symbolブロックチェーンの情報を取得するためのノードを指定(テストネットのノードリストはhttps://symbolnodes.org/nodes_testnet/)
  const nodeUrl = "https://sym-test-04.opening-line.jp:3001";
  // ノードのURLを指定してREST APIやWebSocket APIを利用するためのクラスを生成
  const repositoryFactoryHttp = new RepositoryFactoryHttp(nodeUrl);

  // ノードに対してREST APIを利用してネットワーク固有の設定値(epochAdjustment, networkCurrencies, networkType)を取得
  const epochAdjustment = await firstValueFrom(
    repositoryFactoryHttp.getEpochAdjustment()
  ); // ブロックチェーンの時刻とシステム時刻の基準値の違いを補正するための値を取得
  const networkCurrencies = await firstValueFrom(
    repositoryFactoryHttp.getCurrencies()
  ); // ブロックチェーンネットワークの基軸通貨の情報を取得
  const networkType = await firstValueFrom(
    repositoryFactoryHttp.getNetworkType()
  ); // ブロックチェーンネットワークの種類(メインネット/テストネット等の識別)を取得
  const generationHash = await firstValueFrom(
    repositoryFactoryHttp.getGenerationHash()
  ); // ブロックチェーンネットワークのハッシュ値(ネットワーク固有の値)を取得

  // XYMでの支払いを受け付けるためのトランザクションにセットするために必要なデータを生成
  const deadline = Deadline.create(epochAdjustment); // トランザクションの有効期限を生成(デフォルトだと現在時刻から2時間後)
  const recipientAddress = Address.createFromRawAddress(recipientRawAddress); // トランザクションの送金先アドレスを生成
  const xym = networkCurrencies.currency.createAbsolute(xymAbsoluteAmount); // トランザクションの送金額を含めたトークンのデータを生成
  const mosaics = [xym]; // トランザクションの送金額を含めたトークンのデータを配列に格納
  const message = PlainMessage.create(uuid); // トランザクションのメッセージを生成(支払い確認時にトランザクションを識別できるようUUIDをセット)
  const feeMultiplier = 100; // トランザクションの手数料率を指定(各ノード毎に設定値が異なりデフォルトでは100で多くのノードが100以下なので100を指定)

  // トランザクションデータを生成
  const transferTransaction = TransferTransaction.create(
    deadline,
    recipientAddress,
    mosaics,
    message,
    networkType
  ).setMaxFee(feeMultiplier);

  // トランザクションデータからQRコードのデータを生成
  const transactionQR: TransactionQR = QRCodeGenerator.createTransactionRequest(
    transferTransaction,
    networkType,
    generationHash
  );

  // QRコードのデータをJSON文字列形式に変換
  const transactionQRJsonString = transactionQR.toJSON();
  console.log({ transactionQRJsonString });

  // HTMLのimgタグに埋め込むためのBase64エンコードされたpng画像データに変換
  const transactionQRBase64 = await firstValueFrom(transactionQR.toBase64());
  console.log({ transactionQRBase64 });

  // 公式デスクトップウォレット向けにトランザクションURIを生成
  const transactionPayload = JSON.parse(transactionQRJsonString).data.payload;
  const transactionURI = `web+symbol://transaction?data=${transactionPayload}`;
  console.log({ transactionURI });

  // (3) 着金トランザクション検知
  // WebSocketでのトランザクション検知用のリスナーを生成
  const listener = repositoryFactoryHttp.createListener();

  // ノードに接続してリスナーでのモニタリングを開始
  await listener.open();

  // 新しいブロックの生成をモニタリングする処理を追加 ... この情報は不要だがリスナーに情報が流れてこない時間が続くと接続がタイムアウトしてしまうのでそうならないようにこの処理を追加しておく
  listener.newBlock().subscribe((newBlock) => {
    console.log("new block");
    console.dir({ newBlock }, { depth: null });
  });

  // トランザクションがUnconfirmed(=ネットワークに届いたが未だブロックに組み込まれていない状態)になったイベントをモニタリングする処理を追加
  listener
    .unconfirmedAdded(recipientAddress)
    .subscribe((unconfirmedTransaction) => {
      console.log("unconfirmed transaction");
      console.dir({ unconfirmedTransaction }, { depth: null });

      const transactionHash = unconfirmedTransaction.transactionInfo?.hash;
      console.log(
        `unconfirmed transaction https://testnet.symbol.fyi/transactions/${transactionHash}`
      );

      const isTransferTransaction =
        unconfirmedTransaction instanceof TransferTransaction; // 転送トランザクションか
      if (!isTransferTransaction) {
        console.log("not transfer transaction");
        return;
      }

      const hasPlainMessage =
        unconfirmedTransaction.message.type === MessageType.PlainMessage; // トランザクションのメッセージが平文メッセージか
      if (!hasPlainMessage) {
        console.log("not plain message");
        return;
      }

      const message = unconfirmedTransaction.message.payload;
      console.log({ message });
      const hasTargetMessage = message === uuid; // トランザクションのメッセージにUUIDが含まれているか
      if (!hasTargetMessage) {
        console.log("not target message");
        return;
      }

      const mosaics = unconfirmedTransaction.mosaics;
      const hasXym = mosaics.some(
        (mosaic) =>
          mosaic.id.toHex() ===
            networkCurrencies.currency.mosaicId?.id.toHex() &&
          mosaic.amount.toString() === xymAbsoluteAmount.toString()
      ); // トランザクションに指定金額のXYMが含まれているか
      if (!hasXym) {
        console.log("does not have xym");
        return;
      }

      // 着金トランザクションがネットワークに届いたことをここで検知できる
      console.log("\n\ntarget transaction unconfirmed\n\n");
    });

  // トランザクションがConfirmed(=ブロックに組み込まれた状態)になったイベントをモニタリングする処理を追加
  listener.confirmed(recipientAddress).subscribe((confirmedTransaction) => {
    console.log("confirmed transaction");
    console.dir({ confirmedTransaction }, { depth: null });

    const transactionHash = confirmedTransaction.transactionInfo?.hash;
    console.log(
      `confirmed transaction https://testnet.symbol.fyi/transactions/${transactionHash}`
    );

    const isTransferTransaction =
      confirmedTransaction instanceof TransferTransaction; // 転送トランザクションか
    if (!isTransferTransaction) {
      console.log("not transfer transaction");
      return;
    }

    const hasPlainMessage =
      confirmedTransaction.message.type === MessageType.PlainMessage; // トランザクションのメッセージが平文メッセージか
    if (!hasPlainMessage) {
      console.log("not plain message");
      return;
    }

    const message = confirmedTransaction.message.payload;
    const hasTargetMessage = message === uuid; // トランザクションのメッセージにUUIDが含まれているか
    if (!hasTargetMessage) {
      console.log("not target message");
      return;
    }

    const mosaics = confirmedTransaction.mosaics;
    const hasXym = mosaics.some(
      (mosaic) =>
        mosaic.id.toHex() === networkCurrencies.currency.mosaicId?.id.toHex() &&
        mosaic.amount.toString() === xymAbsoluteAmount.toString()
    ); // トランザクションの送金額にXYMが含まれているか
    if (!hasXym) {
      console.log("does not have xym");
      return;
    }

    // 着金トランザクションがブロックに組み込まれたことをここで検知できる
    console.log("\n\ntarget transaction confirmed\n\n");
    // リスナーを閉じる
    listener.close();
  });

  // 上記のリスナーのモニタリング開始後、
  // (2)のQRコードをユーザーに表示し、
  // トランザクションを送信してもらい、
  // 着金を待つ

  // デバッグの際には
  // QRコードのJSON文字列データの中のpayloadを使って
  // トランザクションURIを生成し、
  // デスクトップウォレットの
  // 「トランザクションURIをインポート」機能で
  // 読み込んでトランザクションを送信すると便利
})();

```

(1)(2)の結果に加えて以下のような結果が表示されたら成功です。

```shell
unconfirmed transaction
{
  unconfirmedTransaction: TransferTransaction {
    type: 16724,
    networkType: 152,
    version: 1,
    deadline: Deadline { adjustedValue: 9916232611 },
    maxFee: UInt64 { lower: 21300, higher: 0 },
    signature: 'D78CEDA71EDEAB9F146F3AC12A316CCB9CD59B0C33EEDB657AD876D1B1D74C8C858E6A39DA23918CEB5079F3FAB18DE71E1D681C04C455AB677D83EBBAD27800',
    signer: PublicAccount {
      publicKey: 'AC833EBF1216557CBCAD7906A293220B0A7D3F007EFDD6B1F8942E220E6DFF3F',
      address: Address {
        address: 'TDOUWYN6V7NWYQEJXSCORC3PLPASIEEWRQEAHYI',
        networkType: 152
      }
    },
    transactionInfo: TransactionInfo {
      height: UInt64 { lower: 0, higher: 0 },
      index: undefined,
      id: undefined,
      timestamp: UInt64 { lower: 0, higher: 0 },
      feeMultiplier: 0,
      hash: 'B66D14675E6A3368B280C41974E3BC77E443ABD1C4D4AD2ECA1DA757F82A466A',
      merkleComponentHash: 'B66D14675E6A3368B280C41974E3BC77E443ABD1C4D4AD2ECA1DA757F82A466A'
    },
    payloadSize: undefined,
    recipientAddress: Address {
      address: 'TARDV42KTAIZEF64EQT4NXT7K55DHWBEFIXVJQY',
      networkType: 152
    },
    mosaics: [
      Mosaic {
        id: MosaicId { id: Id { lower: 1738574798, higher: 1925194030 } },
        amount: UInt64 { lower: 182815356, higher: 0 }
      }
    ],
    message: PlainMessage {
      type: 0,
      payload: 'f96ae857-ca0d-4728-830d-811b2b091acb'
    }
  }
}
unconfirmed transaction https://testnet.symbol.fyi/transactions/B66D14675E6A3368B280C41974E3BC77E443ABD1C4D4AD2ECA1DA757F82A466A
{ message: 'f96ae857-ca0d-4728-830d-811b2b091acb' }


target transaction unconfirmed


new block
{
  newBlock: NewBlock {
    hash: 'B07030045621CB4333283DBD6F275BD11D5A9DEDA70DF44806E196CDAE06BBF6',
    generationHash: 'FA9795BD72473951A39FD308B0F7BC6C64D60AC79443702CCFC198AA469072A7',
    signature: '2E38DF9502E78FBDF0719C93877DEB03049183071756EBC235F40CEEEB0AC22C008549E8CE4B6271D59430ACCF40303230B1FEA1A7BE8A702EDCF415B675BD0D',
    signer: PublicAccount {
      publicKey: '62FBFB9D386BBCE69CAF24ADA1803412601EABD3CF8EFC2B58386286CCBD0757',
      address: Address {
        address: 'TACTCZ5YC4FC6FZQR3ATFORA3R3FCJIEIG4QKWY',
        networkType: 152
      }
    },
    networkType: 152,
    version: 1,
    type: 33091,
    height: UInt64 { lower: 245403, higher: 0 },
    timestamp: UInt64 { lower: 1319106905, higher: 2 },
    difficulty: UInt64 { lower: 1316134912, higher: 2328 },
    feeMultiplier: 100,
    previousBlockHash: '251391BC7033237D4597A300D49F4906CE76E2EBF27F4405E4C0BD07B2C1496F',
    blockTransactionsHash: 'B66D14675E6A3368B280C41974E3BC77E443ABD1C4D4AD2ECA1DA757F82A466A',
    blockReceiptsHash: '06EF2E144843FB0D83513FB466064E06AD63CE28380415430BECDD4EE0E91C11',
    stateHash: '2B0D1AD794D04A589F7A06CAE1F096A3F4C9B0EDD95C2D7C7F5DA43C360900F3',
    proofGamma: 'A143F494A61812C65BEE92E1FB34FAA60C1C49F58506146DB33010E1F7DEBA20',
    proofScalar: 'E8E65236E8310F83DDFB58FA8FB039D6D75A352EE105A5D4B5CCFF627A31A30A',
    proofVerificationHash: 'B2117DBAB61F1FCEE7AEB78A481B58B0',
    beneficiaryAddress: Address {
      address: 'TCU27PSZNAHJIN6E3BEXWEPGNREP6OPRMWW4CIY',
      networkType: 152
    }
  }
}
confirmed transaction
{
  confirmedTransaction: TransferTransaction {
    type: 16724,
    networkType: 152,
    version: 1,
    deadline: Deadline { adjustedValue: 9916232611 },
    maxFee: UInt64 { lower: 21300, higher: 0 },
    signature: 'D78CEDA71EDEAB9F146F3AC12A316CCB9CD59B0C33EEDB657AD876D1B1D74C8C858E6A39DA23918CEB5079F3FAB18DE71E1D681C04C455AB677D83EBBAD27800',
    signer: PublicAccount {
      publicKey: 'AC833EBF1216557CBCAD7906A293220B0A7D3F007EFDD6B1F8942E220E6DFF3F',
      address: Address {
        address: 'TDOUWYN6V7NWYQEJXSCORC3PLPASIEEWRQEAHYI',
        networkType: 152
      }
    },
    transactionInfo: TransactionInfo {
      height: UInt64 { lower: 245403, higher: 0 },
      index: undefined,
      id: undefined,
      timestamp: UInt64 { lower: 0, higher: 0 },
      feeMultiplier: 0,
      hash: 'B66D14675E6A3368B280C41974E3BC77E443ABD1C4D4AD2ECA1DA757F82A466A',
      merkleComponentHash: 'B66D14675E6A3368B280C41974E3BC77E443ABD1C4D4AD2ECA1DA757F82A466A'
    },
    payloadSize: undefined,
    recipientAddress: Address {
      address: 'TARDV42KTAIZEF64EQT4NXT7K55DHWBEFIXVJQY',
      networkType: 152
    },
    mosaics: [
      Mosaic {
        id: MosaicId { id: Id { lower: 1738574798, higher: 1925194030 } },
        amount: UInt64 { lower: 182815356, higher: 0 }
      }
    ],
    message: PlainMessage {
      type: 0,
      payload: 'f96ae857-ca0d-4728-830d-811b2b091acb'
    }
  }
}
confirmed transaction https://testnet.symbol.fyi/transactions/B66D14675E6A3368B280C41974E3BC77E443ABD1C4D4AD2ECA1DA757F82A466A


target transaction confirmed


```

## まとめ

Symbolブロックチェーンの基軸トークンXYMを使った決済処理を行う際に必要となるであろう以下の要素技術についてサンプルコードを用いて解説しました。

1. XYM価格情報取得
2. 請求QRコード生成
3. 着金トランザクション検知

本記事のサンプルコードをベースにほんのわずかに応用を加えるだけで、様々なサービスに柔軟にSymbol決済を組み込むことが可能だと思います。

もしご興味持っていただけましたら、ぜひ、ご自身のサービスにSymbol決済を組み込んでみてくださいますと幸いです。幅広いサービスでSymbol決済が使用できる、そんな未来の実現を願っています。

## 参考資料・リンク

もし何か技術的な質問等ございましたら、以下のDiscord等にてお気軽にコメント頂いたり、SymbolコミュニティメンバーによるSymbolブロックチェーン情報まとめサイト等をご参照くださいますと幸いです。

- nem Japan UserGroup ... [https://discord.gg/CmptF383Wd](https://discord.gg/CmptF383Wd)
- Symbol Community Web ... [https://symbol-community.com/ja](https://symbol-community.com/ja)
