1. go의 runtime은 go 프로그램 실행을 위한 핵심 라이브러리
- GC
- Concurrency
- Stack Management
- etc... system call, errors

2. go runtime vs java
- go는 jvm같은 가상 머신을 사용하지 않는다.
- 대신 go는 컴파일러가 바로 기계어로 컴파일하여 native machine code로 실행
  - java의 경우 컴파일러가 소스 코드를 플랫폼과 상관없는 바이트코드로 변환하고 jvm을 통해 가상머신에서 한 번 더 해석하여 실행
- 그래서 go는 성능의 이점이 있지만 플랫폼에 따라 지정하여 컴파일이 필요, jvm 기반은 플랫폼에 독립적
