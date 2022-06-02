# Sequence builder

Kotlin 의 Sequence 는 List 와 Set 과 비슷하지만 원소가 필요할때 평가된다 (**Lazy evaluate**). 처음에는 이게 왜 코루틴 책에 나올까 의문이였지만 원리를 알고 나면 이야기가 달라진다.
일단 Sequence 의 경우 아래 예시 코드를 보면 어떻게 동작하는지 알 수 있다.

```kotlin
val seq = sequence {
    yield(1)
    yield(2)
    yield(3)
}
```

위와 같이 yield 되는 순간이 바로 sequence 에 들어간다.

``` kotlin
val seq = sequence { 
    println("Generating first") 
    yield(1) 
    println("Generating second") 
    yield(2) 
    println("Generating third") 
    yield(3)
    println("Done")
}

fun main() {
    for (num in seq) {
        println("The next number is $num")  
    }
}
```

이 코드를 보면 왜 코루틴책에 Sequence 가 나오는지 알 수 있는데, **바로 실행 흐름이 역전**되기 때문이다.
메인의 코드 흐름을 보면 아래와 같은 이미지이다.

<img width="552" alt="image" src="https://user-images.githubusercontent.com/57784077/171399827-114674d9-82a0-4fc4-986d-f93a1f2ca387.png">

여기서 중요한 것은 seq.next() 가 호출될때 `print()` 가 호출되고, num 이 평가되려는 순간 yield(1) 이 되며 실행 flow 가 역전된다.
책에서는 이를 이렇게 표현했다.
> Then something crucial happens: execution jumps to the place where we previously stopped to find another number. 

여기서 책에서 적은 표현을 보면 알 수 있듯이, **previously stopped** 라는 말이 나온다. 이게 무슨 의미냐면 **suspend** 를 의미한다. 즉 값의 평가를 멈춰두고, 평가가 필요한 순간 **resume** 할 수 있는 이유는 **suspension mechanism** 때문이다. 이 메커니즘 덕분에 우리는 main thread -> sequnce generator 쪽으로의 controll flow 역전이 가능했다. 만약 Coroutines 없이 이 Flow 를 역전시키기 위해서는 어떻게 했을까? 아마도 Thread 를 사용할 것이다. 실험해보면 알겠지만 Thread 는 Coroutines 에 비해 훨씬 무겁고 비용이 크다.

## Suspension Work

앞에서 계속 언급했던 **Suspension Work (Suspension mechanism)** 는 어떻게 동작하는걸까? 책에서는 아주 이해하기 쉬운 설명을 하는데 
suspension work 란, 우리가 게임을 하다가 check point 에서 세이브를 하고 컴퓨터를 끄고 난뒤에도 **나중에 다시 컴퓨터로 들어오던 닌텐도로 들어오던 해당 체크포인트에서 게임을 이어나갈 수 있다. 이게 바로 suspension mechanism** 이다.

**Coroutines 이 suspend 되면 Continuation 이라는 객체를 리턴**하는데, 이게 게임으로 비유하면 **체크포인트에서 세이브를 했어!** 이런 느낌이다. 이게 왜 좋냐면, Thread 를 공부해보면 알겠지만 **Thread 에서 suspend 를 하는 것은 block** 을 거는것과 같다. 이건 매우 큰 비용이다. 하지만 Coroutines 에서는 Thread 의 block 없이 가능하다. 코루틴은 suspend 된 후에 다시 resume 될때 다른 스레드로 이어서 실행될 수 있다.

```kotlin
suspend fun main() {
    println("Before")

    suspendCoroutine<Unit> { }

    println("After")
}
```

이 코드가 어떻게 실행될것 같은가? 아마 실행시키면 "After" 를 볼수 없을 것이다. 왜냐하면 suspend 된 상태로 resume 될 수 없기 때문이다. 실행시키려면 아래와 같이 코드를 작성하면 된다.

```kotlin
suspend fun main() {
    println("Before")

    suspendCoroutine<Unit> { continuation ->
        println(continuation) // SafeContinuation for Continuation at coroutines_book.Example03Kt.main(example03.kt:9)
        continuation.resume(Unit)
    }

    println("After")
}

```

위의 코드를 보면 알 수 있듯이 **continuation 에서 resume** 를 호출해줘야 다시 재 실행된다. 

```kotlin
suspend fun main() {
val i: Int = suspendCoroutine<Int> { cont ->
       cont.resume(42)
   }
   println(i) // 42
val str: String = suspendCoroutine<String> { cont -> cont.resume("Some text")
   }
   println(str) // Some text
val b: Boolean = suspendCoroutine<Boolean> { cont -> cont.resume(true)
}
   println(b) // true
}
```

**resume 할때 return** 하는 값은 위와 같이 리턴되어 값이 된다. 여기서 Async 내부도 대략적으로 어떻게 되어 있겠구나. 감이 왔다. 근데 아닐수도 있긴한데 아래와 같은 구조일것 같았다.

```kotlin

suspend fun requestUser(): User {
    return suspendCoroutine<User> { cont ->
        requestUser { user ->
            cont.resume(user)
        } 
    }
}
suspend fun main() {
    println("Before")
    val user = requestUser()
    println(user)
    println("After")
}
```

근데 저자는 suspendCoroutine 을 사용하는 것보다 **suspendCancellableCoroutine** 을 사용하라고 한다.


