# 스트림의 새로운 표준 - 리액티브 스트림

## 모두를 위한 반응성

이장에서는 리액티브 스트림 스펙을 따라 구현된 라이브러리를 이용해야 하는 이유에 대해 설명한다.

### API 불일치 문제

비동기 논블로킹 작업을 위한 여러가지 라이브러리가 있는데, 하나의 작업단위에 여러 라이브러리가 같이 존재하는 경우 복잡해진다.

예를 들어, 자바 코어 라이브러리에 존재하는 비슷한 개념의 인터페이스가 아래와 같이 2가지가 존재하고 하나의 코드에서 사용하는 경우가 있다고 하자.

* CompletionStage
* ListenableFuture

둘다 비동기 작업이 완료된 후 처리를 위해 제공된 인터페이스로, spring 4 에서는 하위호환성을 위해 Future 인터페이스를 확장한 ListenableFuture 인터페이스를 제공하였고, 비동기 rest api 용도의 라이브러리인 AsyncRestTemplate 클래스의 리턴 타입으로도 사용된다.

ListenableFuture 인터페이스는 성공, 실패에 대한 콜백을 등록하여 사용되는데 결과에 대한 처리를 위해 콜백 코드를 등록하면서 콜백지옥을 경험하게 된다.

콜백 지옥을 피하기 위해 CompletionStage 인터페이스가 추가되었는데, 해당 인터페이스는 함수형 스타일로 코드를 작성할 수 있다. 다음 비교 코드를 보자.

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



api 불일치 문제에 대한 내용을 이해하기 위해 ListenableFuture, CompletionStage 에 대해 잠시 살펴보았고 다시 불일치 문제로 돌아오면, spring 4 에서는 비동기 api 통신을 위해 AsyncRestTemplate 클래스를 제공해주었는데 해당 클래스의 리턴타입이 **ListenableFutre 인터페이스이고 위에서 확인했다시피 콜백지옥으로 코드가 지저분해지게 된다**. 그래서 보통 CompletionStage 인터페이스로 변환하여 사용하는 경우가 많다. 

이렇게 변환하여 사용하는 경우에 서로 다른 인터페이스 2가지를 사용하게 되고 api 리턴 타입의 불일치가 발생하므로 이를 해결하기 위해 주간에 새로운 유틸성 클래스를 생성해주어야 한다.

**AsyncAdapters \(api 불일치를 해결하기 위한 아답터 클래스\)**

```java
public final class AsyncAdapters {

	public static <T> CompletionStage<T> toCompletion(ListenableFuture<T> future) {

		CompletableFuture<T> completableFuture = new CompletableFuture<>();

		future.addCallback(completableFuture::complete,
				completableFuture::completeExceptionally);

		return completableFuture;
	}

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

위와 같이 유틸성 클래스를 추가하는 경우, 사용자가 직접 코드를 작성하게 되는데 이때 버그가 유입될 가능성이 크고 어떤 문제점이 발생할 여지가 있다.





#### 풀 방식과 푸시 방식

#### 흐름 제어

#### 해결책

### 리액티브 스트림의 기본 스펙

#### 리액티브 스트림 동작해보기



참고

[https://www.hungrydiver.co.kr/bbs/detail/develop?id=2&scroll=comment](https://www.hungrydiver.co.kr/bbs/detail/develop?id=2&scroll=comment)

