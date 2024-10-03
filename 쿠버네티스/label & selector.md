1. label
- label은 특정 오브젝트에 첨부된 key:value pair
- 오브젝트의 특성을 식별하는데 있어서 사용자에겐 중요, 그러나 코어 시스템에게 직접적인 영향은 없다
- 오브젝트 생성할때나 이후에 언제든지 수정가능

2. 왜 사용하는가?
- 사용자가 시스템 오브젝트와 자신들의 org 구조를 느슨하게 매핑할 수 있다.
```
ex.
"release" : "stable", "release" : "canary"
"environment" : "dev", "environment" : "qa", "environment" : "production"
"tier" : "frontend", "tier" : "backend", "tier" : "cache"
```

3. 규칙
- 63자 이하(공백일 수 있다)
- 시작과 끝은 알파벳과 숫자
- 값 중간에 -,_,. 포함 가능
```
ex.
apiVersion: v1
kind: Pod
metadata:
  name: label-demo
  labels:
    environment: production
    app: nginx
spec:
  containers:
  - name: nginx
    image: nginx:1.14.2
    ports:
    - containerPort: 80
```

4. selector
- 서로 다른 오브젝트에 동일한 레이블을 가질 수 있다. ex) environment
- label selector는 이러한 label을 가진 오브젝트를 선택하기 위해 사용

5. select 기준 - Equality-based(일치)
- = / == => 일치를 판단
- != => 불일치 판단
```
ex. 
environment = production
tier != frontend
```
- env가 production인 label을 가진 오브젝트만
- tier가 frontend가 아닌 label을 가진 오브젝트만 (이때 tier키의 값이 공백인것들도 포함)
- 최종적으로 두 개의 조건 and 연산

6. select 기준 - Set-based(포함)
- in / not-in => exsits 여부
```
environment in (production, qa)
tier notin (frontend, backend)
partition
!partition
```
- env key의 값이 production,qa 에 포함되는
- tier key의 값이 (frontend,backend)에 포함되지 않는
- 값은 상관없이 partition key가 존재하는
- 값은 상관없이 partition key가 존재하지 않는
- 이 조건들 역시 n개는 and로 동작

7. 사용 예시
- LIST / WATCH API
- `kubectl get pods -l environment=production,tier=frontend`
- `kubectl get pods -l 'environment in (production),tier in (frontend)'`
- or 조건 같은 경우 set-based를 활용할 수 있다
  - `kubectl get pods -l 'environment in (production, qa)'`
```
selector:
  matchLabels:
    component: redis
  matchExpressions:
    - {key: tier, operator: In, values: [cache]}
    - {key: environment, operator: NotIn, values: [dev]}
```
