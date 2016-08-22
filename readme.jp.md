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
上記のようなメソッドがある状況を想定します。<br>
어떠한 메소드가 특정 조건에서 익셉션을 발생시키는것을 알고는 있지만, 익셉션이 발생해도 진행에 지장이 없도록 하고 싶은 경우가 있습니다.
이럴때는 보통 `try~catch`를 사용해서 아래와 같이 작성하는것이 일반적이지만,,,

```rb
begin
  some_dangerous_method rand 10 
rescue
end
```
메소드를 호출하는곳이 많아지면 많아질수록 불필요한 `try~catch`는 점점 늘어나게 됩니다.<br>
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
하지만 이러한 방법은 굉장히 무식해 보이며, 남에게 보여주기 부끄럽습니다. 게다가 `위험한` 메소드가 늘어나면 늘어날때마다 한개씩 래핑 메소드를 만들수도 없는노릇입니다.<br>
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

ただ、メソードを上で`#--supress-error`を書くことだけでメソードで発生したエーラがコールスタックの上に溢れないようになります。<br>
これは__roap__の__AOP__機能を利用し、`supress-error`と名付けた__Aspect__を生成して該当メソードに適用したことであります。
이는 __roap__의 __AOP__ 기능을 이용하여, `supress-error`라는 __Aspect__를 만들고 해당 메소드에 적용시킨 결과입니다. 코드는 훨신 간결해졌으며, 심지어 메소드 바로위에 설명하듯이 __Aspect__를 기술하므로 다른사람이 보기에도 '이 메소드는 익셉션을 내지 않는다' 는 사실을 단박에 알아챌 수 있습니다.
(물론 읽는 사람도 roap과 AOP에 대한 이해가 있다는 가정 하에)<br>
<br>
물론 Aspect의 이름을 바꾸는것도 얼마든지 가능합니다. 만약 `supress-error`라는 이름이 읽는 사람으로 하여금 이 메소드가 안전하다는걸 한눈에 알아챌 수 없을 것 같다면 `IM EXTERMELY SAFE, SO I WILL NEVER THROW AN EXCETION` 등의 맘에 드는 이름 (마치 문장처럼) 으로 변경하는것 또한 자유입니다.<br>
심지어 `roap`은 __[YARD](http://yardoc.org/)__과 연계하여 사용할 수 있습니다. 코드로부터 자동 생성되는 document에도 `이 메소드는 (존나) 안전해요` 라는 사실을 알려줄 수 있다는 뜻입니다. 이는 추후에 자세히 다루도록 하겠습니다.    


Aspectを作る方
----
(다소 긴) 개요를 통해 __AOP__와 __Aspect__란 무었인가를 알아보았으니, 이번에는 __roap__에서 실제로 Aspect를 만들어보는 방법을 설명합니다.<br>
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

* 조건부에 따라 `_super` 실행 또는 실행하지 않기
* `*args`에 접근하여 파라미터 조작하기
* `_super` 실행 __전__ 또는 __후__에 다른 메소드 실행하기 

이제 우리가 작성한 첫번째 __Aspect__를 실제로 사용하는 코드를 작성해보도록 하겠습니다.
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
지금까지 다룬 `runtime` Aspect들은 모두 런타임에 실행됩니다. 즉, Aspect가 적용된 메소드가 호출되면 그제서야 `on` 아래의 코드블럭이 실행되고, 제어가 가능한 상태가 됩니다.<br>
하지만 때때로는 메소드가 호출되기 전에도 작업하고 싶을때가 있습니다. 이럴때에는 `# runtime` 대신에 `# static`을 적용합니다.<br>
예제 코드는 아래와 같습니다. `on`의 코드블럭에 넘겨지는 파라미터들이 `runtime`과는 다름에 주의해주세요.  
```rb
# static
on /^static-aspect()$/ do |klass, method, md|
  # klass : Aspect가 적용된 메소드를 정의하는 클래스입니다.
  # method : Aspect가 적용된 메소드입니다.
  # md : Regex MatchData
end
```

모든 확장을 include 시키기
----
위의 가이드에서는 `Aspect`를 사용할 클래스에 각각 어떠한 확장 모듈을 사용할것인지를 일일히 지정했습니다.

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
이렇게 작성하면 모든 확장을 사용함을 의미합니다.


주석에 접근하기
----
`roap`의 기본 원리는 주석을 이용한 `AOP`입니다.<br>
내부적으로 메소드의 주석을 가져와서 파싱하는 동작을 수행하며, 이를 굳이 닫아놓을 필요는 없는것 같아서 API로도 제공하고 있습니다.<br>
<br>
__전체 주석에 접근하기__<br>
메소드 내부에서 `__comments___`를 사용하면 해당 메소드 위에 달려있는 전체 주석이 반환됩니다. 
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

기본 또는 모듈로 제공되는 Aspect들
----
__Roap::DigestExtension__
```rb
class AuthHandler
  #--sha256-digested password
  def login id, password
    # 이곳에서 password는 자동으로 해싱된 상태입니다.
  end

  include Roap::DigestExtension
end
```

__Roap::LogExtension__<br>
해당 메소드 호출 이전/이후에 자동으로 로그를 남길 수 있습니다.
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

  #--또는 yard 스타일
  # @example
  #   a = 10, b = 6;
  #   MyMath.sub(a, b) #=> 4
  def self.sub a, b
    a - b
  end

  include Roap::TestHelper
end
```
