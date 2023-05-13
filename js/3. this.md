```
!! this 바인딩은 어디서 한다?
- 실행 컨텍스트가 활성화될 때 한다.
- 실행 컨텍스트는 함수가 호출될때 생성된다
- 즉 함수가 호출될때 this가 바인딩된다.
```
- <img width="940" alt="스크린샷 2023-05-13 오후 5 55 58" src="https://github.com/skarltjr/study/assets/62214428/fee21f90-b28c-4957-9dfc-92cea8e71ac2">

### this binding
- this는 함수가 호출될때 바인딩된다.
- 즉 동적 바인딩이며, 함수 호출 방식에 따라 this가 다르다.
- <img width="581" alt="스크린샷 2023-05-13 오후 5 59 41" src="https://github.com/skarltjr/study/assets/62214428/6db55747-0ebc-400f-866c-62ed1ac769c0">

### 전역 공간에서의 this
- 전역 공간을 실행하는 객체가 바로 전역 객체
- 전역 공간 내부에서의 this는 전역 객체를 가리킨다.
```
브라우저에서는 window 객체
node js에서는 global 객체를 가리킨다.
즉 자바 스크립트가 실행되는 환경 (런타임) 에 따라 달라진다.
```
- <img width="1200" alt="스크린샷 2023-05-13 오후 6 05 16" src="https://github.com/skarltjr/study/assets/62214428/839724b7-7432-4a91-99ee-819eb89e6423">

### 함수 공간에서의 this
- 이 부분은 굉장히 헷갈린다.
- 함수 공간 내부에서의 this도 전역 공간과 마찬가지로 전역 객체
- <img width="1232" alt="스크린샷 2023-05-13 오후 6 07 55" src="https://github.com/skarltjr/study/assets/62214428/ef4256c9-82ba-49eb-82df-24a1c8b82e68">
```
b()의 경우 함수를 실행한 주체가 전역 공간이기 때문에 b 함수 내부에서의 this는 전역 객체라고 받아들일 수 있다.
그런데 c() 또한 호출한 주체가 전역 객체로 판단되어 c() 내부에서의 this는 전역객체.
이건 일종의 버그아닌 버그로 이를 대체하기위해 arrow function이 나왔다.
arrow function의 경우 상위 컨텍스트의 this를 그대로 가져다 쓴다. = this binding이 x
```

### 매서드 공간에서의 this
- <img width="435" alt="스크린샷 2023-05-13 오후 6 12 19" src="https://github.com/skarltjr/study/assets/62214428/72bad017-b6b3-4c77-b643-f5c051566650">
```
여기서 객체 d의 매서드 e()를 호출할때 호출한 대상은 d.
따라서 e() 내부에서의 this는 d 객체를 가리킨다.

!!! 그런데 e 내부에서의 f() 함수 내부에서의 this는? 
앞에서봤듯이 함수 공간에서의 this는 전역 객체!!
```
```
주의
a.b => a['b'] 이 경우도 마찬가지로 b에서의 this는 a

person.info.getName()에서 getName 내부에서의 this는 person.info
```

### 잠깐 생각해보자
- <img width="515" alt="스크린샷 2023-05-13 오후 6 18 42" src="https://github.com/skarltjr/study/assets/62214428/fb81f207-57f6-47d1-a8e8-5c48e92044a7">
```
obj.b()가 호출되었을때
b()와 c()에서의 this.a 결과는??

b : b를 호출한 주체인 obj = 20이나온다
a : 전역 공간의 10이 나온다
```
- 이건 분명하게 이상한 문제점이다.
- 이런 문제를 우회하는 방법으론
- <img width="478" alt="스크린샷 2023-05-13 오후 6 22 27" src="https://github.com/skarltjr/study/assets/62214428/f835b49e-4b56-4c4c-a143-c119d8ec9656">
```
b()에서의 self = this. 즉 여기서의 this는 obj
그리고 c()에서 self.a = obj.a = 20
```

### 콜백 함수에서의 this
- 먼저 call, apply, bind 함수를 살펴보자
- <img width="1269" alt="스크린샷 2023-05-13 오후 6 29 10" src="https://github.com/skarltjr/study/assets/62214428/740fe92b-b9e9-4f53-a9f8-b3f43bf7d55b">
```
[]는 생략가능하다
출력 결과는 아래와 같다.
```
- <img width="1062" alt="스크린샷 2023-05-13 오후 6 31 01" src="https://github.com/skarltjr/study/assets/62214428/5ffebedc-3734-49b7-810f-c23faf792df3">
```
그럼 아래에서는?
```
- <img width="530" alt="스크린샷 2023-05-13 오후 6 33 53" src="https://github.com/skarltjr/study/assets/62214428/785707e4-0838-49ac-8f2a-7c218f83327d">
```
어쨋든 cb()가 함수로써 호출된다.
그럼 cb()내부에서의 this는? 전역객체다.
```
```
그럼 만약 아래는?
```
- <img width="606" alt="스크린샷 2023-05-13 오후 6 36 59" src="https://github.com/skarltjr/study/assets/62214428/c6c41e83-e4fb-4aca-807e-2406f8d81724">
```
여기 cb.call(this)에서 넘겨준 this는 자신을 호출한 객체인 obj가 되고 
당연히 obj가 출력된다.
```
### 즉 callback 함수에서의 this는 지정하는바에서 그때그때 달라진다.
### 생성자에서의 this
- !!! 만들어질 인스턴스 자체가 this가 된다.
- 여기서 놓치면안될 부분이 있다.
- <img width="1190" alt="스크린샷 2023-05-13 오후 6 41 13" src="https://github.com/skarltjr/study/assets/62214428/fbc75661-0a9f-4fcf-af41-cd464b5007cd">
```
new 키워드가 없는건 결국 생성자를 호출하는게 아니다
그렇기 때문에 여기서 this는 전역 객체를 가리키고  전역객체.age가 a값을 할당한다.
```
- <img width="914" alt="스크린샷 2023-05-13 오후 6 42 59" src="https://github.com/skarltjr/study/assets/62214428/8fd422fd-7f42-4a5e-b2fa-3f77fb450f49">
```
new를 통한 생성자 호출시 this는 생성자로 생성될 인스턴스 즉 Person 인스턴스인 roy
```
