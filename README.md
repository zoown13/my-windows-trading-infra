# my-windows-trading-infra

## 배포 예시

> ⚠️ `AllowedRdpIp=본인공인IP/32` 입력은 **필수**입니다.
> 
> `0.0.0.0/0`은 보안상 매우 위험하므로 **운영 환경에서는 사용 금지**이며, 디버깅을 위한 임시 용도로만 제한적으로 사용하세요.

```bash
aws cloudformation deploy \
  --template-file template.yaml \
  --stack-name my-windows-trading-infra \
  --capabilities CAPABILITY_NAMED_IAM \
  --parameter-overrides AllowedRdpIp=본인공인IP/32
```
