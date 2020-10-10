*****
{"path": "5/iOSにおける定期購読について.md", "img": "https://raw.githubusercontent.com/yogita109/ueshun_blog_repository/master/articles/5/shopping_shiharai_businesswoman.png", "tag": "iOS", "date": "2020-10-10"}
*****

## 目次
- In-App Purchasesの概要
- 購入処理に必要な主なStoreKitのAPIについて
    - SKPaymentQueue
    - SKTransactionObserver
    - SKPaymentTransaction
- 実際に購入処理を実装してみる
- レシートについて
    - レシートの検証
    - レシートの管理
- まとめ

## In-App Purchasesの概要
In-App PurchasesとはApp内課金のことであり、アプリ内でプレミアムコンテンツや、デジタル商品、サブスクリプションなどの追加コンテンツは追加機能を、App内で直接ユーザーに提供できます。

In-App Purchasesには、以下の４種類があります。
- 消耗型
- 非消耗型
- 自動更新サブスクリプション
- 非更新サブスクリプション

詳細については[ここ](https://developer.apple.com/jp/in-app-purchase/)を参照してください。

今回は自動更新サブスクリプションの実装方法について見ていきます。
(が他の種類の実装を行うにしても、基本的な考え方は同じだと思います。)

## 購入処理に必要な主なStoreKitのAPIについて
アプリ内コンテンツの購入処理を実装する際、StoreKitというフレームワークを使用します。
ここではStoreKitが提供する主なAPIについて説明していきます。

### SKPaymentQueue
`SKPaymentQueue`は、AppStoreと通信しコンテンツの購入処理のためのインターフェースを提供するクラスです。
`SKPaymentQueue`にエンキューするオブジェクトは、`SKPayment`という購入処理を実行するためのオブジェクトです。
`SKPayment`をエンキューするたびに、`SKPaymentQueue`は購入するためのトランザクション(`SKPaymentTransaction`)を作成します。

### SKTransactionObserver
トランザクションの状態を監視するためのAPIを提供するプロトコルです。
トランザクションが発生すると、`SKPaymentTransactionObserver`から通知されます。
このObserverはApp内課金の課金の柱です。

<b>★Tips★</b><br>
他のクラスでグローバルに参照するために、共有インスタンスとして作成することを検討してください。共有インスタンスは、オブジェクトの生存期間も保証し、`SKPaymentTransactionObserver`を介したコールバックが常に同じインスタンスによって処理されるようにします。

### SKPaymentTransaction
購入するために必要な一連の処理を表すオブジェクトです。
トランザクションの状態は5種類あります。以下はその種類と適当な処理についてです。

|ステータス(SKPaymentTransactionState)|適当な処理|
|:--|:--|
|.purchasing|何もしない。トランザクションのステータスが変化するのを待ちます。|
|.purchased|ユーザーにコンテンツを提供し、`finishTransaction`を呼びます。|
|.failed|エラーを検知し必要なハンドリングを行います。その後`finishTransaction`を呼びます。|
|.restored|ユーザーにコンテンツを提供し、`finishTransaction`を呼びます。|
|.defferd|基本何もしません。|

## 実際に購入処理を実装してみる
ここで上記で説明したAPIを用いて、実際に購入処理を実装してみます。
サンプルコードは[github](https://github.com/yogita109/InAppPurchaseSample/tree/main/InAppPurchaseSample)にあげているので、よければ参考にしてください。

### 1.トランザクションの監視処理を登録する
App内課金を実装する上で重要なことは、トランザクションの状態に応じて適切な処理を行うことです。ここをしっかりしなければユーザーに不利益をもたらしたり、困惑させたりする原因となってしまいます。なので、まずはトランザクションの状態を監視できるようにします。

監視処理の登録タイミングは、AppDelegateの`application(_:didFinishLaunchingWithOptions:)`が望ましいです。
このタイミングで、`SKPaymentQueue`の` add(_:)`を使用して監視処理を登録します。

```swift
@UIApplicationMain
class AppDelegate: UIResponder, UIApplicationDelegate {

    func application(_ application: UIApplication, didFinishLaunchingWithOptions launchOptions: [UIApplication.LaunchOptionsKey: Any]?) -
        // Attach an observer to the payment queue.
        SKPaymentQueue.default().add(PurchaseProduct.shared)
    }
    
    // Called when the application is about to terminate.
    func applicationWillTerminate(_ application: UIApplication) {
        // Remove the observer.
        SKPaymentQueue.default().remove(PurchaseProduct.shared)
    }
}
```

### 2.課金アイテムをAppStoreから取得する
購入するためにはどの商品を購入するかを決めなければなりません。
まず購入可能な商品を取得します。商品は、`SKProduct`というクラスで表現されています。

```swift
/// 課金アイテムを取得する責務を持つクラス
final class DownloadProduct: NSObject {
    // Singleton Instance
    static let shared = DownloadProduct()
	private override init() { }
	private var productsRequest: SKProductsRequest?
    
    func callAsFunction() {
        productsRequest = SKProductsRequest(productIdentifiers: Set(productIds))
		productsRequest?.delegate = self
        // 課金アイテムを取得を開始する.
		productsRequest?.start()
    }
}

extension DownloadProduct: SKProductsRequestDelegate {
    // 課金アイテムの取得結果を受け取る.
	func productsRequest(_ request: SKProductsRequest, didReceive response: SKProductsResponse) {
		guard response.products.isNotEmpty else {
			print("No Product")
			return
		}
		
		guard response.invalidProductIdentifiers.isEmpty else {
			print("Invalid Product")
			return
		}
		print(response.products)
	}
}

```

### 3.課金アイテムを購入する
購入可能なアイテムを取得したら次はそれを購入します。
`PaymentQueue`にenqueueすると、購入のためのトランザクションが生成されます。<br>
トランザクションの状態は、`SKPaymentTransactionObserver`で監視でき、状態に応じて適切な処理を行います。

```swift
/// 課金アイテムを購入する責務を持つクラス
final class PurchaseProduct: NSObject {
	
	private override init() { }
	static let shared = PurchaseProduct()
	
	weak var delegate: PurchasedResultNotification?
	
	func callAsFunction(product: SKProduct) {
		let payment = SKPayment(product: product)
		// キューに追加することでトランザクションが生成される.
		SKPaymentQueue.default().add(payment)
	}
}

extension PurchaseProduct: SKPaymentTransactionObserver {
    // トランザクション(購入処理)の状態が通知される.
	func paymentQueue(_ queue: SKPaymentQueue, updatedTransactions transactions: [SKPaymentTransaction]) {
		transactions.forEach { transaction in
			switch transaction.transactionState {
				case .purchased, .restored:
					print("complete")
                    // ここでレシートデータの取得&レシート検証を行う.
                    SKPaymentQueue.default().finishTransaction(transaction)
				case .failed:
					print("failed")
                    SKPaymentQueue.default().finishTransaction(transaction)
				case .purchasing, .deferred:
                    print("nothing")
				@unknown default:
					break
			}
		}
	}
}
```

購入処理が完了したらレシートが発行されるので、そのレシート(トランザクション)が有効なものか検証します。まずはレシートデータを取得していきます。

### 4.レシートデータを取得する
レシートデータは端末内に保存され、`Bundle.main.appStoreReceiptURL`で取得できるパスに保存されます。

```swift
if let appStoreReceiptURL = Bundle.main.appStoreReceiptURL,
    FileManager.default.fileExists(atPath: appStoreReceiptURL.path) {
    do {
        let receiptData = try Data(contentsOf: appStoreReceiptURL, options: .alwaysMapped)
        let receiptString = receiptData.base64EncodedString(options: [])

        // この後にレシート検証を行う.
    } catch {
        print("Couldn't read receipt data with error: " + error.localizedDescription)
    }
}
```

### 5.サーバー上でレシートを検証する
レシートを取得できたら、次にレシートの有効性を確認するためにレシートの検証を行います。<br>
レシートの検証方法には２つあります。
- デバイス上での検証
    - アプリ内で購読状態を更新する
- サーバー上での検証
    - AppStoreにリクエストを投げるオンライン上での検証。エンコードされたレシートデータをサーバー経由でApp Storeに送信します。するとApp Storeが検証を行います。
    - 自身のサーバー内で購読状態を更新する

[※ デバイス上でオンライン検証(AppStoreにリクエストを投げること)は行わないでください](https://developer.apple.com/jp/documentation/storekit/in-app_purchase/validating_receipts_with_the_app_store/)

上記２つの方法を比較してみます。
|指標|デバイス上での検証|サーバー上での検証|
|:--|:--|:--|
|レシートの信憑性の検証|○|○|
|領収書に更新取引が含まれているか|○|○|
|購読情報の追加(購読の自動更新など)|×|○|
|情報を常に更新（マルチプラットフォーム展開に重要）|×|○|
|デバイスの時刻の変更に影響されない(ローカル検証した場合、デバイスの時刻を過去にされると逆らえない。そのため無料お試しなどに悪用される。)|×|○|
|暗号化処理の不必要性|×|○|

オンライン接続が必要ないアプリケーション(一つの端末内で完結するアプリ)なら、ローカル検証でも問題無そうです。
(とはいえモダンなアプリはネットワーク接続必須なものが多いので、多くの場合はリモート検証になるかと思います。)

```swift
switch transaction.transactionState {
    case .purchased, .restored:
        if let appStoreReceiptURL = Bundle.main.appStoreReceiptURL,
            FileManager.default.fileExists(atPath: appStoreReceiptURL) {
                let rawReceiptData = Data(contentsOf: appStoreReceiptURL.path)
                let recepitData = rawReceiptData.base64EncodedString(options: _)

                // 自身のサーバーにレシート文字列を送信する
                currentUser.processTransaction(recepitData) { isValid in
                    if isValid {
                        // トランザクションを終了させる
                        queue.finishTransaction(transaction)
                    }
                }
            }
}
```

`finishTransaction(_:)`は、トランザクションが終了したことをAppStoreに通知するAPIです。
このAPIはアイテム(ゲームだとジェム)の付与やレシート検証など、<b>必要な処理がすべて完了した後</b>に呼び出してください。

ここまでは、アプリ内で必要な処理についてでした。<br>
次は、サーバー上でどのように検証すれば良いかについてとレシートデータの管理について説明していきます。

## レシートについて
レシートはAppStoreで発行され、決済が完了したかどうか検証に使うものです。
これはお店のレシートと同じで、ユーザーが購入したことを証明します。

レシートの特徴は以下のとおりです。
- AppStoreによって発行される、アプリ内購入に関する信頼できる購入記録
- レシートは各デバイスに保存される
- 証明としてAppleが署名するので、発行元がAppleだと確認できる
- レシートはデバイス固有のもの(同じユーザーでも、所持しているデバイスごとに少しずつ内容が変わる)

### レシートの検証
レシートの発行元がAppleかどうか、レシートの有効性を検証します。
サーバーからAppStore(`https://buy.itunes.apple.com/verifyReceipt`)にレシートデータを送信すると、AppStoreから以下のようなJSON形式のペイロートが返却されます。

```json
{  status: 0
   receipt: {
       bundle_id: "com.your.app",
       in_app: [{ // in_appオブジェクトにはトランザクションの一覧が入っている。
           transaction_id: "1234567890",
           product_id: "com.your.product.id",
           original_transaction_id: "1133557799",
           expires_date: "2018-09-17"
           ・・・
       }]
   }
}
```

返却されたJSONデータを確認し、正しい情報か検証します。
1. statusが0ならAppleが発行しており、レシートのコンテンツ(receiptオブジェクト内のデーター。これはレシート検証のために復元されたバイナリデータ)を確認できる。
2. bundle_idが一致するか確認する
3. product_idがアプリケーションと関連づいているか検証する

すべて一致していたら、このユーザーの定期購読を認証できます。

次に購読状態を更新します。

### レシートの管理
AppStoreから返却されたペイロード内の`original_transaction_id`と`expires_date`をDBのテーブルに保存します。
そして先程デバイスからサーバーへのリクエストのレスポンスとして、trueを返します。

定期購読を購入したユーザーには、トランザクションID(`original_transaction_id`)が割り当てられ、それを保存しました。
このIDが<b>サブスクリプションID</b>です。更新時に必ず表示される重要なものです。

仕組みを見ていきます。
更新を検証するとしたらトランザクションエンドポイント(`https://buy.itunes.apple.com/verifyReceipt`)で行います。
トランザクションが有効かを検証し、ユーザーの購読状態を更新します。

更新なので過去の複数のトランザクションがあります。

サーバーで保存している有効期限のレコードが`2018-07-08`と期限を過ぎてしましました。
そこで現在も購読しているかレシートデータを確認します。
```json
receipt: {
       in_app: [{ // in_appオブジェクトにはトランザクションの一覧が入っている。
           transaction_id: "1234567890",
           product_id: "com.your.product.id",
           original_transaction_id: "1133557799",
           expires_date: "2018-07-08"
       }, {
           transaction_id: "2244668800",
           product_id: "com.your.product.id",
           original_transaction_id: "1133557799",
           expires_date: "2018-08-08"
       }]
   }
```

レシートデータから判断する方法は、
1. `original_transaction_id`でフィルタリングする
2. 最新の`expires_date`でtransactionを探す
3. 現在の日付を過ぎていたら、ユーザーは購読していません
4. 現在の日付を過ぎておらず、未来の日付だったらユーザーは購読中です。そしてサーバー上の有効期限のフィールドを更新します。

これでユーザーの購読期間が延長されました。

## まとめ
ここまでiOSアプリにおける、自動更新サブスクリプションのIn-App Purchasesの実装方法、レシートの検証と管理方法についてざっくり述べてきました。
In-App Purchaseを初めて実装する前は難しそうなイメージでしたが、実際にやってみるとそこまで難しくないと感じました。

## 参考URL
- https://developer.apple.com/jp/in-app-purchase/
- https://developer.apple.com/documentation/storekit/skpaymentqueue
- https://developer.apple.com/videos/play/wwdc2018/704
- https://developer.apple.com/videos/play/wwdc2018/705
- https://developer.apple.com/documentation/storekit/in-app_purchase/