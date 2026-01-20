로컬 PC 에서 워크샵을 다운로드 받는다.  
```
cd ~
git clone https://github.com/gnosia93/ray-on-ec2.git
cd ~/ray-on-ec2
```

## vpc 생성 ##
```
export AWS_REGION="ap-northeast-1"
export KEYPAIR_NAME="aws-kp-1"
cd ~/ray-on-ec2
pwd

AMI=$(aws ssm get-parameters --names /aws/service/ami-amazon-linux-latest/al2023-ami-kernel-default-x86_64 \
  --region ${AWS_REGION} --query "Parameters[0].Value" --output text)
MY_IP="$(curl -s https://checkip.amazonaws.com)""/32"
echo ${AMI} ${MY_IP}

#sed -i "s/\${AMI}/$AMI/g" $(pwd)/cf/ray-vpc.yaml
sed -i "" "s|\${AMI}|$AMI|g" $(pwd)/cf/ray-vpc.yaml
sed -i "" "s|\${MY_IP}|$MY_IP|g" $(pwd)/cf/ray-vpc.yaml
```
vpc 를 생성한다.
```
aws cloudformation create-stack \
  --region ${AWS_REGION} \
  --stack-name ray-vpc \
  --template-body file://$(pwd)/cf/ray-vpc.yaml \
  --parameters ParameterKey=KeyPairName,ParameterValue=${KEYPAIR_NAME} \
  --capabilities CAPABILITY_IAM \
  --tags Key=Project,Value=ray-on-aws
```
vpc 생성 진행 과정을 조회하고 완료될때 까지 대기한다. 
```
aws cloudformation describe-stacks --stack-name ray-vpc --query "Stacks[0].StackStatus"
```

생성 결과를 출력한다. 
```
aws cloudformation describe-stacks \
  --stack-name ray-vpc \
  --query "Stacks[0].Outputs[?OutputKey=='BastionDNS' || OutputKey=='VSCodeURL'].OutputValue" \
  --output text
```

* 참고 - 스택 삭제하기
```
aws cloudformation delete-stack --stack-name ray-vpc
```

