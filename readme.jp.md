roap
====
__AOP__ + __Ruby__<br>
<br>

Overview　(roapとAOPについて)
----
`roap`はルビーで __Aspect Oriented Programming__を可能にするgemです. <br>
__AOP__についてには下の簡単な例で説明するとします。
```rb
def some_dangerous_method p
  raise RuntimeError if p % 2 == 0
  puts "you're lucky" 
end
```
上記のようなメソードがある状況を想定します。<br>
どんなメソードが得な条件でExceptionを発生することを知っていますが、Exceptionが発生してもプログラムの進みには障りが無いようにする時もあります。
こんな状況には普通`try~catch`を使用するのが一般的ですが。。。

```rb
begin
  some_dangerous_method rand 10 
rescue
end
```
メソードをコールする場合が多くなれば多くなるほど不必要な`try~catch`も一緒に多くになります。<br>
<br>
こんな問題を解決するため`no_longer_dangerous_method`メソッドを作るのも考えることもできます。

```rb
def no_longer_dangerous_method p
  begin
    some_dangerous_method p 
  rescue
  end
end
```
しかし、このような方法はとてもバカみたいし、他人に見せたくない仕方です。さらに`危険`なメソードが増えたら増えるほど１つずつラッピングメソードを作るのも良い方法では考えられません。<br>
<br>
__roap__はこんなシチュエーションですっきりなソルションになります。<br>
下の例は__roap__を使ってメソードを`安全`なメソードに変形する方を示します。

```rb
#--supress-error
def some_dangerous_method p
  # ...method-body...
end
```
コールスタックの上にあふれて流れないようにします。

ただ、メソードを上で`#--supress-error`を書くことだけでメソードで発生したエラーがコールスタックの上に溢れないようになります。<br>
これは__roap__の__AOP__機能を利用し、`supress-error`と名付けた__Aspect__を生成して該当メソードに適用したことであります。
이는 __roap__의 __AOP__ 기능을 이용하여, `supress-error`라는 __Aspect__를 만들고 해당 메소드에 적용시킨 결과입니다. 코드는 훨신 간결해졌으며, 심지어 메소드 바로위에 설명하듯이 __Aspect__를 기술하므로 다른사람이 보기에도 '이 메소드는 익셉션을 내지 않는다' 는 사실을 단박에 알아챌 수 있습니다.
(但し、コードを読む人もroapとAOPに対する理解がある仮定で。)<br>
<br>
もちろんAspectに付けた名も変わることもいつでも出来ます。もし`supress-error`が
물론 Aspect의 이름을 바꾸는것도 얼마든지 가능합니다. 만약 `supress-error`라는 이름이 읽는 사람으로 하여금 이 메소드가 안전하다는걸 한눈에 알아챌 수 없을 것 같다면 `IM EXTERMELY SAFE, SO I WILL NEVER THROW AN EXCETION` 등의 맘에 드는 이름 (마치 문장처럼) 으로 변경하는것 또한 자유입니다.<br>
また`roap`は __[YARD](http://yardoc.org/)__と連係して使うこともできます。 코드로부터 자동 생성되는 document에도 `이 메소드는 (존나) 안전해요` 라는 사실을 알려줄 수 있다는 뜻입니다. 이는 추후에 자세히 다루도록 하겠습니다.    


Aspectを作る方
----
(少し長かった) 概要を通じて__AOP__と__Aspect__はなにものかを調べてみました。これからは__roap__で実際にAspectを作る方を説明するとします。<br>
아래는 `on 정규식` 메소드를 통해 Aspect를 선언하고, 생성된 Aspect를 실제로 클래스에 적용하는 방법을 보여줍니다.

```rb
module MyFirstRoapModule
  extend Roap::AttributeBase

  # runtime
  on /^print-hello()$/ do |_super, md, *args|
    puts "hello world"
    _super.call *args
  end
end
```
위의 코드와 같이 `on 정규식 do ~ 코드블럭 end` 를 이용하면, `정규식`에 매칭되는 Aspect가 있는 메소드가 실행될때마다 지정한 `코드블럭`으로 제어가 넘어오게됩니다.<br>
이 코드블럭에서 할 수 있는 일의 예시를 들어보면 아래와 같습니다.

* 条件によって `_super`をコールするとかコールしない
* `*args`でアクセスしてパラメータを操作する
* `_super`実行 __前__ または __後__に他のメソードをコールする (chainning) 

作った__Aspect__を実際に使用するコードを作成するとします。

```rb
class Foo
  #--print-hello
  def sum a, b
    return a + b
  end

  include MyFirstRoapModule
end

puts Foo.new.sum(1,2)
```

Static Aspect
----
今までの`runtime`Aspectは全てランタイムで実行されます。　すなわち、Aspectが適用されたメソードがコールすれば`on`の下でコードが実行されます。<br>
しかし、時によってはメソードがコールされる前にも作業したいと思う時もありあす。こんな時には`# runtime`代わりに`# static`を使って下さい。<br>
`# static`を使う例は下にあります。　`# runtime`を使った時とコードブロックに渡されるパラメーターが違うことに注意してください。
```rb
# static
on /^static-aspect()$/ do |klass, method, md|
  # klass : Aspect가 적용된 메소드를 정의하는 클래스입니다.
  # method : Aspect가 적용된 메소드입니다.
  # md : Regex MatchData
end
```

全てのエクステンションincludeさせる
----
今までのガイドには`Aspect`を使用するclassに一々どんなエクステンションを使うのかを指定しましたが、　（下の例のように）

```rb
class Foo
  include MyFirstExt
  include MyLastExt
  include MyLastFinalExt
  include MyLastFinalLastExt
end
```
이는  `roap`의 `Aspect`검출이 정규식 기반이므로 의도하지 않은 주석에 의해 우연히 정규식이 매칭되어 생각지도 못했던 동작이 발생함을 방지하기 위함이었습니다.<br>
하지만 이러한 배려는 필요 없고 그냥 편리하게 쓰고싶은 경우에는 아래와 같이 사용해 주세요.

```rb
include Roap::AllExtensions
```
こうすれば作成した全てのエクステンションを使用するを意味します。


コメントへアクセスする
----
`roap`의 기본 원리는 주석을 이용한 `AOP`입니다.<br>
내부적으로 메소드의 주석을 가져와서 파싱하는 동작을 수행하며, 이를 굳이 닫아놓을 필요는 없는것 같아서 API로도 제공하고 있습니다.<br>
<br>
__全体コメントを取得__<br>
メソードの内部で `__comments___`を使えばメソードに上の全体コメントを取得することができます。 
```rb
#Hello World
def will_return_hello_world
  __comments__
end
```
<br>
__항목별 가져오기__<br>
`__ELEM-NAME__`과 같이 접근하여 개별 항목에 대한 주석을 가져올 수 있습니다.
```rb
# @description
#  i'm rini.
# @hates
#  pjc0247, stupid
def about_rini
  puts "Hi, #{__description__}"
  puts "I hate following things : #{__hates__}"
end
```

基本または外部moduleとして提供されるAspectたち
----
__Roap::DigestExtension__
```rb
class AuthHandler
  #--sha256-digested password
  def login id, password
    # ここではpasswordが自動的にSHA256-HASINGされた状況です。
  end

  include Roap::DigestExtension
end
```

__Roap::LogExtension__<br>
メソードコール以前/以後に自動的にログを出力することができます。
```rb
class AuthHandler
  #--log-before I got a login request.
  def login id, password
    # ....
  end

  include Roap::LogExtension
end
```

__Roap::ThreadSafeExtension__
```rb
class TSCounter
  #--thread-safe
  def increase
    @n ++;
  end

  include Roap::ThreadSafeExtension
end
```

__Roap::TestHelper__<br>
`TestHelper`를 사용하면 별도의 TestCase클래스를 작성하지 않고도 메소드 위에 바로 테스트를 선언할 수 있습니다.<br>
__YARD__스타일의 주석을 지원하기 때문에 document로 빌드되는 예제 코드가 정상적으로 동작하는지 또한 점검할 수 있도록 해줍니다.<br>
(별도의 테스트 유닛을 만들어 테스트하는 방법도 가능합니다. `TestHelper`확장은 테스트 도구의 기본적인 기능인 `Setup/Cleanup` 또한 지원하고 있습니다.)
```rb
class MyMath
  #--파라미터와 리턴값을 지정할 수 있습니다.
  #--test-me 10, 5 => 15
  #--test-me 20, 1 => 21
  def self.sum a,b
    a + b
  end

  #--또는 yardスタイル
  # @example
  #   a = 10, b = 6;
  #   MyMath.sub(a, b) #=> 4
  def self.sub a, b
    a - b
  end

  include Roap::TestHelper
end
```
