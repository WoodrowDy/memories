---
title: CloudFront로 다른 망의 ALB에 ACM 인증서 붙이기
aliases: [CloudFront 오리진, 다중 클라우드 인증서]
created: 2026-09-04
updated: 2026-09-04
tags: [infra/aws, infra/cloudfront, infra/tls]
status: growing
---

# 다른 망에 인증서를 가져가지 않고 CloudFront로 붙이기 — 돈을 아껴야 하는 상황이라면

- **작업일**: 2026-09-04
- **결론**: CloudFront를 앞에 두고 오리진을 다른 클라우드의 ALB로 지정. ACM 무료 인증서로 해결

---

## 핵심

- 인증서는 도메인 단위가 아니라 **TLS를 종료하는 지점**에 필요하다. 종료 지점을 AWS로 당겨오면 다른 망에는 인증서를 넣을 필요가 없다.
- CloudFront 오리진은 AWS 리소스가 아니어도 된다. 공개적으로 도달 가능한 도메인이면 다른 클라우드의 ALB도 그대로 오리진이 된다.
- 그 결과 ACM 인증서(무료·자동갱신)를 쓰게 되고, 유효기간이 200일 → 100일 → 47일로 짧아져도 갱신 비용은 계속 0이다. 외부 CA 구매나 Exportable 재발급($86/회)이 필요 없다.
- 대신 감수하는 것: 오리진 구간이 HTTP 평문이라는 점, CloudFront 요청·전송 요금, AWS 종속.

---

## 1. 출발점

NCP 공공망에 올린 API 서버를 `api.○○○.or.kr`로 HTTPS 서비스해야 했다.

처음 생각한 구성은 단순했다. Route 53에 서브도메인을 만들고 NCP ALB 주소로 CNAME을 걸고, NCP ALB에 인증서를 붙이면 끝. 그런데 여기서 막혔다.

## 2. 첫 번째 벽 — ACM 인증서를 다른 망으로 가져갈 수 없다

기존 `○○○.or.kr` ACM 인증서를 NCP Certificate Manager에 등록하려 했으나 불가능했다.

ACM 인증서는 기본적으로 개인키를 내보낼 수 없다. 2025년 6월부터 '내보내기 가능(Exportable)' 옵션이 생겼지만, 이 옵션은 **발급 시점에만 지정할 수 있고 나중에 변경할 수 없다**.

```bash
aws acm describe-certificate --certificate-arn <ARN> --query 'Certificate.Options.Export'
```

이 값이 `ENABLED`가 아니면 방법이 없고, 같은 도메인으로 재발급받아야 한다.

## 3. 검토했던 선택지들

### 3-1. AWS에서 Exportable로 재발급

와일드카드 $79 + 루트 도메인 $7 = 회당 $86. 발급 시와 갱신 시마다 과금된다.

문제는 유효기간이다. 공인 인증서 최대 유효기간이 전 세계 공통으로 단축되는 중이다.

| 시점 | 최대 유효기간 |
| --- | --- |
| 2026-03-15~ | 200일 |
| 2027-03-15~ | 100일 |
| 2029-03-15~ | 47일 |

갱신 횟수가 늘어나는 만큼 비용이 그대로 곱해진다. 지금 발급하면 첫 갱신이 2027년 3월경이라 이미 100일 규칙에 걸린다.

### 3-2. 외부 CA에서 발급받아 각 망에 등록

인증서 파일을 직접 받아 어디든 설치할 수 있어 인프라 종속이 없다. 구독 기간 내 재발급이 무제한 무료인 상품을 쓰면 유효기간이 짧아져도 비용이 고정이다. 국내 업체 기준 DV 와일드카드가 연 30만 원 안팎.

### 3-3. 실제로 채택한 것 — CloudFront를 앞에 두기

두 선택지를 비교하다가 전제 자체가 틀렸다는 걸 발견했다.

인증서는 도메인 단위가 아니라 **TLS를 종료하는 지점마다** 필요하다. 트래픽을 CloudFront로 받으면 TLS가 AWS에서 끝나므로 ACM 인증서를 쓸 수 있고, 다른 망은 오리진 역할만 하면 된다.

```
api.○○○.or.kr
   ↓ Route 53 CNAME
CloudFront          ← 여기서 TLS 종료, ACM 인증서 (무료·자동갱신)
   ↓ 오리진 요청 (HTTP)
NCP 공공망 ALB
   ↓
API 서버
```

CloudFront는 오리진으로 AWS 리소스가 아닌 임의의 공개 도메인을 지정할 수 있다. 이걸 알고 나니 인증서 비용 문제 자체가 사라졌다.

## 4. 두 번째 벽 — 호스팅 영역이 다른 계정에 있었다

Route 53 콘솔에 `○○○.or.kr` 호스팅 영역이 보이지 않았다. Route 53 IAM 권한은 받은 상태였는데도.

IAM은 계정 경계를 넘지 못한다. 권한을 아무리 넓게 받아도 다른 AWS 계정의 리소스는 보이지 않는다.

네임서버를 조회해 확인했다.

```bash
dig NS ○○○.or.kr +short
# ns-xxx.awsdns-xx.com 외 3개 → Route 53에 있긴 함
```

```bash
aws route53 list-hosted-zones-by-name --dns-name ○○○.or.kr
# 알파벳상 다음 존이 반환됨 → 이 계정에 해당 존이 없음

aws sts get-caller-identity
```

호스팅 영역은 다른 계정에 있었다. 다만 ACM 인증서는 내 계정에 있었다. 검증 레코드만 상대 계정 존에 넣으면 계정이 달라도 발급되기 때문이다.

그래서 리소스는 내 계정에서 만들고, DNS 레코드만 요청하는 방식으로 진행했다.

## 5. 실제 작업 순서

### ① ACM 인증서 발급 (us-east-1)

CloudFront에 붙일 인증서는 **반드시 us-east-1(버지니아 북부)에서 발급해야 한다.** 다른 리전에서 받으면 CloudFront 드롭다운에 나타나지 않는다.

```bash
aws acm request-certificate \
  --region us-east-1 \
  --domain-name "*.○○○.or.kr" \
  --validation-method DNS
```

와일드카드로 받았다. ACM은 AWS 내부 사용 시 무료라 와일드카드도 비용이 같고, 앞으로 서브도메인이 늘어도 인증서를 다시 만들 필요가 없다.

검증 레코드를 요청하려 했는데 바로 `ISSUED`가 됐다. 기존 인증서 발급 때 넣은 검증 CNAME이 존에 남아 있었고, ACM이 같은 도메인 재요청 시 그 레코드를 재사용했기 때문이다.

### ② 오리진 ALB 구성

1. LB 전용 서브넷 확인
2. 타겟 그룹 생성 후 API 서버 등록, 헬스 체크 설정
3. ALB 생성, HTTP 80 리스너
4. API 서버 ACG 인바운드에 LB 서브넷 대역 허용 (누락 시 헬스 체크 실패로 503)

오리진이 퍼블릭에서 도달 가능한지 먼저 확인했다.

```bash
curl -I http://xxxx-prd-app-alb-xxxxxxxx.kr-gov.lb.naverncp.com/
# HTTP/1.1 200 OK, content-type: application/json
```

### ③ CloudFront 배포 생성

Origin type은 **Other**를 선택한다. 다른 클라우드의 ALB는 AWS 리소스가 아니라 외부 도메인이다.

| 항목 | 값 | 이유 |
| --- | --- | --- |
| Origin domain | 오리진 ALB 도메인 | 직접 입력 |
| Protocol | HTTP only | 오리진 구간 |
| Viewer protocol policy | Redirect HTTP to HTTPS | |
| Allowed HTTP methods | GET, HEAD, OPTIONS, PUT, POST, PATCH, DELETE | API라 전체 필요 |
| Cache policy | CachingDisabled | API 응답을 캐시하면 안 됨 |
| Origin request policy | AllViewer | 헤더·쿠키·쿼리 전달 |

권장 설정(Use recommended)을 그대로 두면 캐싱이 켜지고 메서드가 GET/HEAD로 제한될 수 있다. 반드시 Customize로 지정해야 한다.

### ④ 대체 도메인 이름 추가

이 콘솔은 배포를 먼저 만들고 도메인을 나중에 붙이는 흐름이다. 배포 상세에서 Add domain으로 `api.○○○.or.kr`을 추가하고 `*.○○○.or.kr` 인증서를 선택했다.

이걸 하지 않으면 CNAME을 걸어도 CloudFront가 403으로 거부한다.

### ⑤ DNS 레코드 요청

호스팅 영역이 다른 계정에 있어 담당자에게 요청했다.

```
유형: CNAME
이름: api.○○○.or.kr
값:   dxxxxxxxxxxxx.cloudfront.net
TTL:  300
```

### ⑥ 검증

CNAME이 들어가기 전에도 배포 도메인으로 확인할 수 있다.

```bash
curl -I https://dxxxxxxxxxxxx.cloudfront.net
# HTTP/2 200
# x-cache: Miss from cloudfront
# via: 1.1 xxxxx.cloudfront.net (CloudFront)
# x-amz-cf-pop: ICN57-P5
```

여기서 200이 나오면 CloudFront와 오리진 사이는 정상이고 남은 건 DNS뿐이다.

## 6. 알게 된 것

**인증서는 TLS를 끝내는 지점에 있어야 한다.** 도메인마다 하나씩 있어야 하는 게 아니다. 같은 도메인의 서브도메인들이 각각 다른 곳에서 TLS를 종료하면 인증서도 각각 다른 걸 쓰면 된다. 이 전제를 잡는 데 시간이 가장 많이 걸렸다.

**ACM의 내보내기 옵션은 발급 시점에만 정해진다.** 나중에 외부에서 쓸 가능성이 있으면 발급할 때 판단해야 한다.

**IAM 권한과 계정 경계는 다른 문제다.** "권한을 받았는데 안 보인다"면 대개 다른 계정이다.

**CloudFront 오리진은 AWS 밖이어도 된다.** 공개적으로 도달 가능한 도메인이면 다른 클라우드도 오리진이 될 수 있다. 이 하나로 인증서 구매 비용이 0이 됐다.

**인증서 교체는 무중단이다.** TLS 인증서는 핸드셰이크 시점에만 제시되므로, 리스너의 인증서를 바꿔도 기존 연결은 끊기지 않는다. 새벽 작업이 필요한 종류의 변경이 아니다. 다만 체인 누락이나 도메인 불일치는 즉시 드러나므로 교체 전 검증이 중요하다.

**와일드카드는 한 단계만 커버한다.** `*.○○○.or.kr`은 `api.○○○.or.kr`은 되지만 `a.b.○○○.or.kr`은 안 된다.
