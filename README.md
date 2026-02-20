# my-windows-trading-infra

CloudFormation 배포 전에 사용 가능한 가용 영역(AZ)을 먼저 확인하고 `AvailabilityZone` 파라미터에 입력하세요.

```bash
aws ec2 describe-availability-zones --query 'AvailabilityZones[].ZoneName' --output table
```

예시:

```bash
aws cloudformation deploy \
  --template-file template.yaml \
  --stack-name my-windows-trading-infra \
  --parameter-overrides AvailabilityZone=ap-northeast-2a
```
