---
description: buffer 와 window 의 차이는 무엇인가?
---

# 번외 - buffer vs window

일단 둘다 데이터를 하나하나 처리/실행하지 않고 한번에 데이터셋을 모아 실행시킬때 사용한다.

buffer : flux 데이터를 list 에 모은다.

window : flux 데이터를 flux 로 모은다.



## Reactor buffer

### Use case 1

buffer 는 보통 이런 상황에서 사용을 고려할 수 있다.   
매우 처리량이 높은 애플리케이션에서 발생한 이벤트를 받아서 DB 에 넣을때, 하나하나 처리를 하게 될 경우 전체 처리시간에 영향을 주므로 매우 비효율적이다.

buffer 를 이용하면 100개의 데이터를 받아서 한번에 DB 에 넣도록 bulk 작업을 할 수 있으므로 훨씬 효율적이다.  
유의할점은 100개의 데이터가 모두 들어오기를 기다리면 안된다는 것이다.  
만약, 99개의 데이터를 받은 상태에서 마지막 1개의 데이터가 들어오지 않으면 영원히 기다리게 되므로  
시간을 지정해두는게 좋다.

예를들어, 10초 동안 최대 100개의 아이템을 받아서 처리하는 식으로 처리할 수 있다.



### Use case 2

credit card 사용이 2초 동안 동시에 3번 실행되는 경우를 상상해봐라. 

실제로 이런 경우는 없겠지만 이 경우 카드가 악용된 사용이다. 그러므로 이런 경우를 캐치해서 블락을 할 수 있도록\(동시에 사용이 안되도록\) 처리해야 한다.   
간단해보이지만 실제 구현은 그렇게 쉽지 않다.

```java
Flux.range(1,20)
        .delayElements(Duration.ofMillis(500))
        .buffer(5)  // collect the items in batches of 5
        .subscribe(l -> System.out.println("Received :: " + l));
        
// output:
Received :: [1, 2, 3, 4, 5]
Received :: [6, 7, 8, 9, 10]
Received :: [11, 12, 13, 14, 15]
Received :: [16, 17, 18, 19, 20]
```

각 원소들을 500 ms 대기 시간 후에 5개 모은다.



## Reactor window

window 는 buffer 와 유사하지만, window 는 데이터를 list 로 모으는 대신 새로운 flux 로 만든다.

buffer -&gt; Flux&lt;List&lt;T&gt;&gt;

windwo -&gt; Flux&lt;Flux&lt;T&gt;&gt;

```java
Flux.range(1, 10)
   .window(5)
   .doOnNext(flux -> flux.collectList().subscribe(l -> System.out.println("Received :: " + l)))
   .subscribe();

// output
Received :: [1, 2, 3, 4, 5]
Received :: [6, 7, 8, 9, 10]
```

window 는 buffer 와 유사한 메소드들을 대부분 제공한다.



## Reference

{% embed url="https://www.vinsguru.com/reactor-buffer-vs-window/" %}

