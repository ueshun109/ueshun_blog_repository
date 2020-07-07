# LiveDataをObserveする際の注意点

みんなLiveDataって使ってますよね！LiveData便利ですよね！
でも意識していないと意図しない動作になってしまうことがあります。
本稿ではそんな落とし穴になり得る書き方の一つを紹介します。

## TLDR
- LiveDataの`observe`メソッドを呼び出した際、LiveDataに値があればそれを返す。
- LiveDataには、Hot/Coldという性質がある。

## LiveDataの頻出パターン
まずはLiveDataを使う上での頻出パターンを紹介します。

まずViewModelにLiveDataに値を流すための処理を書きます。

``` ViewModel
class FirstViewModel: ViewModel() {
    // 外から値を流せないようにするために、mutableLiveDataはアクセス修飾子をprivateにしている。
    private val mutableLiveData = MutableLiveData<String>)()
    val liveData: LiveData<String> = mutableLiveData

    fun tappedButton() {
      mutableLiveData.value = "Tapped Button!!"
    }
}
```

次にFragmentでViewModelから流れてきた値を受け取るための処理を書きます。

``` Fragment
class FirstFragment : Fragment() {

    private val viewModel: FirstViewModel by lazy {
        ViewModelProvider.NewInstanceFactory().create(FirstViewModel::class.java)
    }

    override fun onCreateView(
        inflater: LayoutInflater, container: ViewGroup?,
        savedInstanceState: Bundle?
    ): View? {
        return inflater.inflate(R.layout.first_fragment, container, false)
    }

    override fun onViewCreated(view: View, savedInstanceState: Bundle?) {
        super.onViewCreated(view, savedInstanceState)

        // 次の画面に遷移するボタン.
        val nextButton: Button = view.findViewById(R.id.nextButton)
        nextButton.setOnClickListener {
            findNavController().navigate(R.id.from_first_to_second)
        }

        // ViewModelのLiveDataから値を流してもらうボタン.
        val liveButton: Button = view.findViewById(R.id.liveButton)
        liveButton.setOnClickListener {
            viewModel.tappedButton()
        }

        // ここでViewModelから流れてきた値を受け取る.
        viewModel.liveData.observe(viewLifecycleOwner, Observer {
            Snackbar.make(view, it, Snackbar.LENGTH_LONG).show()
        })
    }
}
```

以下のような仕様がある前提で本稿を進めていきます。
```
LIVEボタンをタップした時にだけスナックバーを表示する。
```

念の為LiveDataで流れてきた値が受け取れるか確認しておきます。

// ここにgifを挿入する

SnackBarが表示されたので、きちんとLiveDataの値を受け取れていることが確認できますね。

## LiveDataの落とし穴その１

では次に、以下のようにViewModelのMutableLiveDataにデフォルト値を入れてみます。

```
private val mutableLiveData = MutableLiveData<String>)("Not yet tapped button!!")
```

これで再度アプリを起動してみます。

// ここにgifを挿入する

少し分かりづらいですがLIVEボタンをタップしていないにも関わらずスナックバーが表示されてしまっています。
これは仕様とは異なる動作です。。ぴえん。。

## LiveDataの落とし穴その2

では次に、Liveボタンを押したあとにFirstFragmentからSecondFragmentに遷移し、SecondFragmentからFirstFragmentに戻ってみます。

// ここにgifを挿入する

FirstFragmentに戻ってきた際、LIVEボタンをタップしていないにも関わらずまたｍたスナックバーが表示されてしまっています。
これも仕様とは異なる動作です。。

## 解決方法
どうやら`observe`した際、LiveDataにイベントが保持されていればそのイベントを即座に処理してしまうようです。
上記の様な挙動は想定仕様のようで、設計上の問題として対処しなければいけないようです。

以降は前述した問題を解決するためのいくつかの方法について見ていくことにします。

## 解決方法1~イベントを処理したか否かの状態を持っておく~
まず思いつくのは解決方法は、イベントを処理したか否かの状態をフラグで管理する、というものかなと思います。

今回はFragment側で状態を持つようにしておきます。

```
class FirstFragment: Fragment() {
    private var hasBeenHandled = fasle

    override fun onViewCreated(view: View, savedInstanceState: Bundle?) {
        super.onViewCreated(view, savedInstanceState)
        ・
        ・
        ・
        viewModel.liveData.observe(viewLifecycleOwner, Observer {
            if (hasBeenHandled.not()) {
                hasBeenHandled = true
                Snackbar.make(view, it, Snackbar.LENGTH_LONG).show()
            }
        })
    }
}
```

もしくは、

```
class FirstFragment: Fragment() {
    private var hasBeenHandled = fasle

    override fun onViewCreated(view: View, savedInstanceState: Bundle?) {
        super.onViewCreated(view, savedInstanceState)
        ・
        ・
        ・
        if (hasBeenHandled.not) {
            hasBeenHandled = true
            viewModel.liveData.observe(viewLifecycleOwner, Observer {  
                Snackbar.make(view, it, Snackbar.LENGTH_LONG).show()
            }
        }
    }
}
```
ただこのフラグで状態をもつ方法には、
- 管理するコストがかかってしまったり、うっかりフラグの状態を書きかえてしまう可能性がある。
- 一度処理されたあとは永遠にtrueになったフラグを持ち続けなければならない
というようなデメリットがあると思います。

## 解決方法2~イベントを流したあと、nullを流しておく~
LiveDataの型を`String`から`String?`のnullableに変更し、値を流した直後にnullを流す、という方法もあるかと思います。

``` ViewModel
class FirstViewModel: ViewModel() {
    // 外から値を流せないようにするために、mutableLiveDataはアクセス修飾子をprivateにしている。
    private val mutableLiveData = MutableLiveData<String?>)()
    val liveData: LiveData<String?> = mutableLiveData

    fun tappedButton() {
      mutableLiveData.value = "Tapped Button!!"
      mutableLiveData.value = null
    }
}
```

``` fragment
class FirstFragment: Fragment() {

    override fun onViewCreated(view: View, savedInstanceState: Bundle?) {
        super.onViewCreated(view, savedInstanceState)
        ・
        ・
        ・
        viewModel.liveData.observe(viewLifecycleOwner, Observer {
            if (it != null) {
                Snackbar.make(view, it, Snackbar.LENGTH_LONG).show()
            }
        })
    }
}
```

この方法では、
- いちいちnullを流してリセットしなければならない
- リセットするためだけにLiveDataのジェネリック型をString?にしなければならない。(nullableは値がオプションのときに使用するためだと考えているためです。)

## 解決方法3~SingleLiveEventを使用する~

以下のような。イベントを流したときに１度だけobserveするクラスを作ります。

```
class SingleLiveEvent<T>: MutableLiveData<T>() {

    private val tag = "SingleLiveEvent"

    private var isPending = AtomicBoolean(false)

    override fun observe(owner: LifecycleOwner, observer: Observer<in T>) {

        if (hasActiveObservers()) {
            Log.w(tag, "Multiple observers registered but only one will be notified of changes.");
        }

        super.observe(owner, Observer<T?> {
            if (isPending.compareAndSet(true, false)) {
                observer.onChanged(it)
            }
        })
    }

    override fun setValue(value: T?) {
        isPending.set(true)
        super.setValue(value)
    }

    fun call(value: T?) {
        this.value = value
    }
}
```

```viewModel
class FirstViewModel : ViewModel() {

    private val mutableLiveData = SingleLiveEvent<String>()
    val liveData: LiveData<String> = mutableLiveData

    fun tappedButton() {
        mutableLiveData.call("Tapped Button!!")
    }
}
```

```fragment
viewModel.liveData.observe(viewLifecycleOwner, Observer {
    Snackbar.make(view, it, Snackbar.LENGTH_LONG).show()
})
```

このようにすると、イベントを流したときのみスナックバーが表示されるようになります。

ただこの方法には一つのストリームを複数observeできない、というデメリットがあります。

## 解決方法4~イベントのラッパークラスを使用する~

最後に以下のような流したいイベントをラップするクラスを使用することです。

```
class Event<out T>(private val content: T) {
    var hasBeenHandled = false
        private set

    val contentIfNotHandled: T?
        get() {
            return if (hasBeenHandled) {
                null
            } else {
                hasBeenHandled = true
                content
            }
        }

    val peekContent: T = content
}
```

``` viewModel
class FirstViewModel : ViewModel() {

    private val mutableLiveData = MutableLiveData<Event<String>>()
    val liveData: LiveData<Event<String>> = mutableLiveData

    fun tappedButton() {
        mutableLiveData.value = Event("Tapped Button!!")
    }
}
```

``` fragment
viewModel.liveData.observe(viewLifecycleOwner, Observer { event ->
    event.contentIfNotHandled?.let {
        Snackbar.make(view, it, Snackbar.LENGTH_LONG).show()
    }
})
```

内部で行っていることは解決方法1とほぼ同じことですが、Eventでラップするだけで状態を管理するコストが無くなるのでこの方法がベストなのかなと考えています。

## まとめ
`LiveData`は便利で簡単に扱えますが、気をつけていないと意図しない挙動が発生することがあるので少しだけ注意して使ったほうが良さそうです。

## 参考URL
- https://medium.com/androiddevelopers/livedata-with-snackbar-navigation-and-other-events-the-singleliveevent-case-ac2622673150
- https://github.com/android/architecture-samples/blob/dev-todo-mvvm-live/todoapp/app/src/main/java/com/example/android/architecture/blueprints/todoapp/SingleLiveEvent.java