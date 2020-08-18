*****
{"path": "4/iOSにもスナックバーをください.md", "img": "https://raw.githubusercontent.com/yogita109/ueshun_blog_repository/master/articles/4/choko_bar.png", "tag": "iOS", "date": "2020-08-18"}
*****

## 概要
Snackbarとは、Material Designで定義されている画面下部に簡単なメッセージを提供するためのUIコンポーネントです。
Androidでは、Snackbarが提供されているのですが、iOSでは提供されていません。
そこで今回はiOSでスナックバーを実装するための方法を調査していきます。

## Snackbarのメリット
Snackbarの利点は、簡単なメッセージをユーザーのインタラクションを必要とせず提供できることだと思います。
例えば、位置情報を取得して住所を自動入力しておくようにする仕様があるとします。
自動入力はオプションなので、位置情報の取得に失敗しても住所を手動で入力できるようにしなければなりません。

仮に位置情報の取得に失敗した場合、何もユーザーに伝えずに住所が空の状態にしておくことは良いとは言えません。
なぜなら普段なら自動入力されているはずの住所が空になっていたら、バグと思われるかもしれないためです。

では位置情報の取得に失敗したことをユーザーにどのように伝えれば良いでしょうか。

iOSではUIAlertControllerというアラートダイアログを表示するためのUIコンポーネントが提供されており、これを用いてユーザーに通知してみましょう。

![alert_dialog_ios.gif](https://raw.githubusercontent.com/yogita109/ueshun_blog_repository/master/articles/4/alert_dialog_ios.gif)

位置情報の取得に失敗したことを伝えることはできましたが、
ダイアログを消すためにユーザーがタップアクションを起こさなければならない、
という副作用が発生します。

また個人的にはアラートダイアログは、サービスを続行するのが困難になるエラーの場合に表示すべきかなと考えています。

## UIAlertControllerのデメリット
アラートダイアログはユーザーを操作を中断させてしまうことで、UXが落ちると考えています。

またHIGにも、以下のように記述されています。
>  Alerts disrupt the user experience and should only be used in important situations like confirming purchases and destructive actions (such as deletions), or notifying people about problems. The infrequency of alerts helps ensure that people take them seriously. Ensure that each alert offers critical information and useful choices.

重要な状況以外でも通知すると、ユーザーはアラートダイアログで通知された内容を真剣に受け止めなくなってしまう、とあります。
アラートダイアログは、ユーザー操作を中断させてまでも伝えてたいことがある場合使用するほうが良さそうです。

ではUIAlertControllerを使うほど重要な通知ではないが、ユーザーに通知したい場合どうしたらよいでしょうか。
ここでSnackbarの登場です。Snackbarは、ユーザー操作を中断させずにメッセージを通知できるので、今回の要件にぴったりです。

ということでSnackbarの実装方法について見ていきましょう。

## Snackbarの実装
Snackbarを実装するといっても実装自体は単純で、UIViewを下から上にアニメーションし、一定時間後上から下にアニメーションするだけです。

まずは、SnackbarのView部分を作っていきます。

```swift
import UIKit

final class Snackbar: UIView {
    private var message: String?
    private var label: UILabel?

    private var topConstraint: NSLayoutConstraint?
    private var leadingConstraint: NSLayoutConstraint?
    private var trailingConstraint: NSLayoutConstraint?

    required init?(coder: NSCoder) {
        // コード上から使用することのみを想定しているためIBから使うことを禁止しています
		    fatalError("init(coder:) has not benn implemented")
	  }

    init(message: String) {
        super.init(frame: .zero)
        self.message = message

        label = UILabel()
        label?.text = message
        label?.textColor = .white
        label?.font = .systemFont(ofSize: 15.0, weight: .semibold)
        label?.translatesAutoresizingMaskIntoConstraints = false
        layer.cornerRadius = 3.0
        backgroundColor = .darkGray
        translatesAutoresizingMaskIntoConstraints = false

        addSubview(label!)

        // Snackbarに制約をつける.
        topConstraint = topAnchor.constraint(equalTo: parent.layoutMarginsGuide.bottomAnchor, constant: 0.0)
        leadingConstraint = leadingAnchor.constraint(equalTo: parent.layoutMarginsGuide.leadingAnchor)
        trailingConstraint = trailingAnchor.constraint(equalTo: parent.layoutMarginsGuide.trailingAnchor)
        heightAnchor.constraint(equalToConstant: 44.0).isActive = true

        parent.addSubview(self)

        topConstraint?.isActive = true
        leadingConstraint?.isActive = true
        trailingConstraint?.isActive = true

        // Snackbarの子ビューであるラベルに制約をつける.
        label?.leadingAnchor.constraint(equalTo: leadingAnchor, constant: 16.0).isActive = true
        label?.trailingAnchor.constraint(equalTo: trailingAnchor, constant: 16.0).isActive = true
        label?.topAnchor.constraint(equalTo: topAnchor).isActive = true
        label?.bottomAnchor.constraint(equalTo: bottomAnchor).isActive = true

        parent.layoutIfNeeded()
    }
}
```

次にアニメーションを付けていきます。
今回は複雑なアニメーションではないのでUIKitのアニメーションを使用します。

まず下から上に表示するアニメーションをつけます。
```swift
extension Snackbar {
    func show() {
        topConstraint?.constant = -44.0

        UIView.animate(
            withDuration: 1.0,
            delay: 0,
            usingSpringWithDamping: 1.0,
            initialSpringVelocity: 4.0,
            options: .allowUserInteraction,
            animations: { [weak self] in
                self?.superview?.layoutIfNeeded()
            }
        )
    }
}
```

そして上から下に非表示になるアニメーションをつけます。
```swift
extension Snackbar {
    func show() {
      Timer.scheduledTimer(timeInterval: 3.0, target: self, selector: #selector(dismiss), userInfo: nil, repeats: false)

      topConstraint?.constant = -44.0
      ・・・
    }

    @objc func dismiss() {
        topConstraint?.constant = 44.0
        UIView.animate(
            withDuration: 2.0,
            delay: 0,
            usingSpringWithDamping: 1.0,
            initialSpringVelocity: 4.0,
            options: .allowUserInteraction,
            animations: { [weak self] in
                self?.superview?.layoutIfNeeded()
            },
            completion: { [weak self] _ in
              self?.removeFromSuperview()
            }
        )
    }
}

これでSnackbarの完成です。あとはViewControllerから以下のように呼べばスナックバーが表示されます。
```swift
let snackbar = Snackbar(message: LocationError.failureGetLocation.message, parent: view)
snackbar.show()
```

![snackbar_ios.gif](https://raw.githubusercontent.com/yogita109/ueshun_blog_repository/master/articles/4/snackbar_ios.gif)

## まとめ
iOSではユーザーに何かを伝える手段がUIAlertControllerしかありませんが、UIAlertControllerを頻繁に利用するのは得策ではなさそうです。
サービスの利用には差し支えないなど重要ではないが、ユーザーに伝えたほうが良いことがある場合はスナックバーで手軽に伝えるのが良いのかなと思います。