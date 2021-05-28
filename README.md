# 스트림의 새로운 표준 - 리액티브 스트림

## 모두를 위한 반응성

이 장에서는 리액티브 스트림 스펙을 따라 구현된 라이브러리를 이용해야 하는 이유에 대해 설명한다.

### API 불일치 문제

비동기 논블로킹 작업을 위한 여러가지 라이브러리가 있는데, 하나의 작업단위에 여러 라이브러리가 같이 존재하는 경우 복잡해진다.

예를 들어, 자바 코어 라이브러리에 존재하는 비슷한 개념의 인터페이스가 아래와 같이 2가지가 존재하고 하나의 코드에서 사용하는 경우가 있다고 하자.

* CompletionStage
* ListenableFuture

둘다 비동기 작업 완료 후 작업 처리를 위해 제공되는 인터페이스로, spring 4 에서는 하위호환성을 위해 Future 인터페이스를 확장한 ListenableFuture 인터페이스를 제공하였고 비동기 rest api 호출 용도의 라이브러리인 AsyncRestTemplate 클래스의 리턴 타입으로도 사용된다.

ListenableFuture 인터페이스는 성공, 실패에 대한 콜백을 등록하여 사용하는데 결과에 대한 처리를 위해 콜백 코드를 등록해야하고 콜백 코드가 추가되면서 콜백지옥을 경험하게 될 수도 있다.

조금 더 간단하고 확장성을 위해 \(콜백 지옥을 피하기 위해\) CompletionStage 인터페이스가 추가되었는데, 이 인터페이스는 함수형 스타일로 코드를 작성할 수 있다. 

다음 비교 코드를 보자. \(간단히 비교를 위해 어떤 블로그에서 참조한 코드.\)

**ListenableFutre 를 이용한 코드.**

```java
ListenableFutureTask listenableFutureTask = new ListenableFutureTask(task);
        listenableFutureTask.addCallback(new ListenableFutureCallback() {
            @Override
            public void onFailure(Throwable throwable) {
                System.out.println("exception occurred!!");
            }

            @Override
            public void onSuccess(Object o) {
                ListenableFutureTask listenableFuture = new ListenableFutureTask(task);
                listenableFuture.addCallback(new ListenableFutureCallback() {
                    @Override
                    public void onFailure(Throwable throwable) {
                        System.out.println("exception occurred!!");
                    }

                    @Override
                    public void onSuccess(Object o) {
                        System.out.println("all tasks completed!!");
                    }
                });
                listenableFuture.run();
            }
        });
        listenableFutureTask.run();
```

**CompletionStage 인터페이스를 이용한 코드.**

```java
 CompletableFuture
                .runAsync(task)
                .thenCompose(aVoid -> CompletableFuture.runAsync(task))
                .thenAcceptAsync(aVoid -> System.out.println("all tasks completed!!"))
                .exceptionally(throwable -> {
                    System.out.println("exception occurred!!");
                    return null;
                });
```

api 불일치 문제에 대한 내용을 이해하기 위해 ListenableFuture, CompletionStage 에 대해 잠시 살펴보았고 다시 불일치 문제로 돌아오면, spring 4 에서는 비동기 api 통신을 위해 AsyncRestTemplate 클래스를 제공해주었는데 해당 클래스의 리턴타입이 **ListenableFutre 인터페이스이고 위에서 확인했다시피 콜백코드 때문 코드가 지저분해지게 된다**. 그래서 보통 CompletionStage 인터페이스로 변환하여 사용하는 경우가 많다.

이렇게 변환하여 사용하는 경우에 서로 다른 인터페이스 2가지를 사용하게 되고 api 리턴 타입의 불일치가 발생하므로 이를 해결하기 위해 새로운 유틸성 클래스를 생성해주어야 한다.

**AsyncAdapters \(api 불일치를 해결하기 위한 아답터 클래스\)**

```java
public final class AsyncAdapters {
    // rest api 완료 후 completaionStage 로 변환하는 코드
    public static <T> CompletionStage<T> toCompletion(ListenableFuture<T> future) {

        CompletableFuture<T> completableFuture = new CompletableFuture<>();

        future.addCallback(completableFuture::complete,
                completableFuture::completeExceptionally);

        return completableFuture;
    }

    // controller에서 작업 리턴을 위해 completionStage 코드를 LinstenableFuture 로 변환하는 코드
    public static <T> ListenableFuture<T> toListenable(CompletionStage<T> stage) {
        SettableListenableFuture<T> future = new SettableListenableFuture<>();

        stage.whenComplete((v, t) -> {
            if (t == null) {
                future.set(v);
            }
            else {
                future.setException(t);
            }
        });

        return future;
    }
}
```

위와 같이 유틸성 클래스를 추가하는 경우, 사용자가 직접 코드를 작성하게 되는데 이때 버그가 유입될 가능성이 크고 자신도 모르는 사이에 문제점이 발생할 여지가 있다.

### 풀 방식과 푸시 방식

> 푸시 방식 : 퍼블리셔가 자신을 구독하고 있는 구독자에게 직접 데이터를 전송해주는 방식 \(생산자가 소비자에게 내\)
>
> 풀 방식 : 구독자가 퍼블리셔에게서 데이터를 요청해서 받아오는 방식  \(실제 주문 요청이 있을때 생산하여 제공. 소비자가 생성자에게로 요청\)

이 절은 예제코드로 풀 방식과 푸시 방식의 차이점에 대해 설명한다.

위 풀, 푸시 방식의 비교 개념을 이해하고 순수한 풀 방식의 경우를 예제 코드로 살펴보자.

**예제코드1** \(순수한 풀 방식\)

```java
public class Puller {

	final AsyncDatabaseClient dbClient = ...

	// count = 10 으로 호출 (ex. puller.list(10); )
 	public CompletionStage<Queue<Item>> list(int count) {
		BlockingQueue<Item> storage = new ArrayBlockingQueue<>(count);
		CompletableFuture<Queue<Item>> result = new CompletableFuture<>();

		pull("1", storage, result, count);
		
		sout("비동기 논블로킹으로 db 를 호출하였으므로 여기가 젤 먼저 출력됨");
		return result;
	}

	void pull(String elementId,
			Queue<Item> queue,
			CompletableFuture resultFuture,
			int count) {
		dbClient.getNextAfterId(elementId)
		        .thenAccept(item -> {
							// getNextAfterId() 의 결과가 리턴될때까지 기다린 후 실행되며, 10까지 +1 씩 상승/반
			        if (isValid(item)) {
				        queue.offer(item);

				        if (queue.size() == count) {
					        resultFuture.complete(queue);
					        return;
				        }
			        }

			        pull(item.getId(), queue, resultFuture, count);
		        });
	}

	boolean isValid(Item item) {...}
}
```

서비스\(Puller\) 에서 데이터베이스\(dbClient\) 를 호출할때는 비동기 논블로킹으로 호출하였지만, count\(10\) 를 모두 처리할때는 +1씩 올리면서 코드를 반복한다. 해당 과정은 thenAccept 부분에서 블로킹 되므로 **서비스에서 데이터베이스로의 요청\(pull\) 시간** 때문에 비효율적인 코드이다. 

**예제코드2** \(순수한 풀 방식 개선\)

```java
public class Puller {

	final AsyncDatabaseClient dbClient = ...

	public CompletionStage<Queue<Item>> list(int count) {
		BlockingQueue<Item> storage = new ArrayBlockingQueue<>(count);
		CompletableFuture<Queue<Item>> result = new CompletableFuture<>();

		pull("1", storage, result, count);

		return result;
	}

	void pull(String elementId,
			Queue<Item> queue,
			CompletableFuture resultFuture,
			int count) {
		// 서비스가 데이터베이스를 요청할때 10 개를 한번에 처리하도록 함.
		dbClient.getNextBatchAfterId(elementId, count)
		        .thenAccept(items -> {
							// 완료된 items 를 for 문안에서 한번에 처리한다. (이없으면 10개)
			        for (Item item : items) {
				        if (isValid(item)) {
					        queue.offer(item);

					        if (queue.size() == count) {
						        resultFuture.complete(queue);
						        return;
					        }
				        }
			        }

							// 위 작업 완료될때까지 대기중...
			        pull(items.get(items.size() - 1)
			                  .getId(), queue, resultFuture, count);
		        });
	}

	boolean isValid(Item item) {...}
}

```

첫 예제의 순수한 풀 방식과의 차이점은 10개를 한번에 배치처리 한다는 점이다. 하지만 데이터베이스에서 10개를 모두 처리할때까지 thenAccept 함수에서 블로킹 되며 여전히 클라이언트는 대기시간하는 존재한다.

**예제코드3** \(푸시방식\)

```java
public class DelayedFakeAsyncDatabaseClient implements AsyncDatabaseClient {

	@Override
	public Observable<Item> getStreamOfItems() {
		return Observable.range(1, Integer.MAX_VALUE)
		                 .map(i -> new Item("" + i))
		                 .delay(50, TimeUnit.MILLISECONDS)
		                 .delaySubscription(100, TimeUnit.MILLISECONDS)
	}
}

public class Puller {

	final AsyncDatabaseClient dbClient = ...

	public Observable<Item> list(int count) {
		return dbClient.getStreamOfItems()
		               .filter(this::isValid)
		               .take(count);
	}

	boolean isValid(Item item) {...}
}

```

count 를 모두 처리할때까지 블로킹 되는 구간이 없다. 데이터베이스에서 처리가 완료되는 즉시 서비스로 푸시를 해주고 서비스에서는 원하는 count 개수가 나올때까지 데이터베이스가 전송해주는 값을 받아오므로, **데이터베이스에게 데이터를 요청하는 요청 시간이 추가적으로 들지 않는다.**

#### 흐름 제어

#### 해결책

### 리액티브 스트림의 기본 스펙

#### 리액티브 스트림 동작해보기





참고

[https://www.hungrydiver.co.kr/bbs/detail/develop?id=2&scroll=comment](https://www.hungrydiver.co.kr/bbs/detail/develop?id=2&scroll=comment)

