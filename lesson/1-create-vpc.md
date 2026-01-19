
```
aws cloudformation create-stack \
  --stack-name ray-vpc \
  --template-body file://template/vpc.yaml \
  --parameters ParameterKey=KeyPairName,ParameterValue=ap-northeast-1 \
  --capabilities CAPABILITY_IAM \
  --tags Key=Project,Value=ray-on-aws
```
