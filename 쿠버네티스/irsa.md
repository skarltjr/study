1. IRSA란
- IAM Roles for Service Accounts는 쿠버네티스 service account에 AWS IAM role을 설정하는 기능
- pod는 service account에 설정된 IAM role을 사용
- pod가 IAM role을 사용할 때,  AssumeRoleWithWebIdentity API가 호출되어 assume role이 수행
- 이래서 이 롤로 assuem할때 신뢰관계에 AssumeRoleWithWebIdentity action을 추가해줘야함
```
apiVersion: v1
kind: ServiceAccount
metadata:
  name: my-service-account
  namespace: default
  annotations:
    eks.amazonaws.com/role-arn: arn:aws:iam::123:role/my-role

```

2. irsa 선행 조건
> EKS OIDC idp 생성
- eks 생성시 자동 생성
```

ops-dev eks cluster OpenID Connect 공급자 URL
https://oidc.eks.ap-northeast-2.amazonaws.com/id/7398A2FE7F18173AC9B75F09C9288F0B
```
- 해당 oidc provider를 aws에 등록은 필요
> role trust relationship 설정
- pod에서 assume role 할 수 있도록 신뢰 관계 설정
- 이때 action은 sts:AssumeRoleWithWebIdentity
```
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Principal": {
                "Federated": "arn:aws:iam::781170117707:oidc-provider/oidc.eks.ap-northeast-2.amazonaws.com/id/7398A2FE7F18173AC9B75F09C9288F0B"
            },
            "Action": "sts:AssumeRoleWithWebIdentity",
            "Condition": {
                "StringEquals": {
                    "oidc.eks.ap-northeast-2.amazonaws.com/id/7398A2FE7F18173AC9B75F09C9288F0B:sub": "system:serviceaccount:int-policy-checker-dev:int-policy-checker"
                }
            }
        }
    ]
}
```

3. 동작 방식
- oidc provider가 왜 필요한가부터 살펴보면
  - aws는 assumerole을 수행하고자하는 pod가 믿을 수 있는 pod인지 확인부터 필요하다.
  - 즉 인증 정보가 필요한데, access key를 활용하는게 아니라면 쿠버네티스 자체적으로 이런 인증정보를 전달할 수 없다.
  - 따라서 믿을 수 있는 제 3자인 openid connector를 통해 인증정보를 전달받는다
  - 참고로 oidc는 인증 정보 제공이 목적이고, oidc는 idp에게 oauth 방식으로 인증 정보를 받아오는데
    - 기본적으로 oauth의 목적은 인증 정보 제공이 아니라 임시 권한을 가진 access token 제공이다.
    - oidc는 이를 우회적으로 활용하여 인증 정보를 요구하는 "권한 요쳥"을 통해 access token과 user identity(jwt)를 전달받는다.
  - 이제 oidc를 통해 user identity(jwt) 정보를 받았으니 이를 aws에게 전달하고, aws는 jwt를 decode하여 정보를 확인하고 이를 바탕으로 assuem role을 진행한다.
