---
description: 리액터 프로젝트 심화학습
---

# 리액터 프로젝트 - 리액티브 앱의 기초

## 리액티브 스트림의 수명 주기

### 조립 단계

리액터에서는 빌더패턴과 비슷한 형태로 연산자들을 조립할 수 있도록 제공한다.

![&#xBE4C;&#xB354;&#xD328;&#xD134;&#xACFC; &#xBE44;&#xC2B7;&#xD55C; &#xD615;&#xD0DC;&#xB85C; &#xC870;&#xB9BD;&#xD558;&#xC5EC; &#xC0AC;&#xC6A9;](.gitbook/assets/2021-06-20-5.09.08.png)

빌더 패턴은 하나의 객체에 대해 새로운 메서드가 추가됨과 동시에  객체의 데이터가 변경되지만,  
리액터에서는 불변으로 메서드\(연산자\)가 추가되어도 객체의 내용이 변하지  않고 새로운 객체가 생성된다.

![&#xC870;&#xB9BD;&#xB2E8;&#xACC4; &#xC608;&#xC2DC;&#xCF54;&#xB4DC;](.gitbook/assets/2021-06-14-10.28.30.png)

각 연산과정에서 도출된 sourceFlux, mapFlux, filterFlux 는 모두 새로운 객체로 생성된다.



조립의 또 다른 예시. \(내부적으로 최적화가 진행되는 경우\)

concatWith 연산자로 각 변수들을 조합\(조립\)하여 출력한다.

![](.gitbook/assets/2021-06-14-10.39.11.png)



concnatWith 연산자의 내부코드를 보면 변수의 타입이 FluxConcatArray 인지 여부에 따라 다른 함수를 사용하고 있다. 

![concatWith &#xD568;&#xC218;](.gitbook/assets/2021-06-20-5.05.28.png)

![concat &#xD568;&#xC218;](.gitbook/assets/2021-06-14-10.46.55.png)

concatAdditionalSourceLast\(\) 함수에서는 FluxConcnatArray 로 변환시키면서 se1, se2, se3 각 데이터를 모두 하나의 array 에 담아서 리턴.

FluxConcatArray\(FluxConcatArray\(FluxA, FluxB\), FluxC\) -&gt; **FluxConcatArray\(FluxA, FluxB, FluxC\)**



또한 조립 단계에서 각종 훅을 이용하면 중요한 기능들을 추가적으로 사용할 수 있다.

![](.gitbook/assets/2021-06-14-10.48.14.png)

### 

### 구독 단계

Publisher 를 구독할 때 발생.

![](.gitbook/assets/2021-06-15-11.44.41.png)



publisher 들\(sourceFlux, mapFlux, filterFlux\) 연결되어 subscriber 가 전달되는 형식.

![publisher &#xAC00; &#xC804;&#xB2EC;&#xB428;.](.gitbook/assets/2021-06-15-11.49.34.png)

구독 순서는 다음과 같다. 

filterFlux publisher 를 subscribe\(\) 하면

1. mapFlux publisher 의 subscribe\(\) 메소드가 호출되고 \(실제 publisher는 FluxFilterFuseable\)
2. sourceFlux publisher 의 subscribe\(\) 메소드가 호출 \(FluxMapFuseable publisher\)
3. Flux publisher 의 subscribe\(\) 메소드가 호출 \(FluxArray publisher\)



결국 arrayFlux 가 모두를 포함한 형태.

![filter -&amp;gt; map -&amp;gt; array &#xC21C;&#xC73C;&#xB85C; subscriber &#xC804;&#xD30C;](.gitbook/assets/2021-06-16-12.07.03.png)

 최종적으로 마지막 연산자까지 subscriber 를 전달 완료한 경우에 데이터를 송신을 위한 데이터 요청 메소드를\(request\) 호출한다.



### 런타임 단계

구독 단계에서 subscriber 가 filter -&gt; map -&gt; source 순으로 전파된다고 했다.

sourceFlux 까지 subscriber 가 전파되면 onSubscriber\(\) 메소드가 차례대로 호출되고 \(source -&gt; map -&gt; filter\)

onSubscribe\(\) 메소드가 호출된 후 부터 request\(\) 메소드가 다시 역순으로 호출 된다 \(filter -&gt; map -&gt; source\) 

![](.gitbook/assets/2021-06-20-7.30.47.png)

ArraySubscription 의 request 까지 호출되면 실제 데이터 전송을 시작한다.





아래 코드와 같이 각 단계\(Subscriber\) 마다 특정 조건을 거치면서 데이터가 흐른다.

FilterSubscriber\(...\).onNext\("1"\) 부분에서 필터 처리 및 데이터 추가 호출\(request\(1\)\)

FilterSubscriber\(...\).onNext\("20"\) 부분에서는 구독자에게 20 데이터 전송.



![](.gitbook/assets/2021-06-20-7.33.57.png)



## 리액터에서 스레드 스케줄링 모델

## 프로젝트 리액터의 내부 구조

## 요약 



