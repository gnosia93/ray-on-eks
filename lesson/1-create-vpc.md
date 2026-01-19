
### 1. vpc 생성 ###
```
export AWS_REGION="ap-northeast-1"
export KEYPAIR_NAME="aws-kp-1"
cd ~/ray-on-aws
pwd

AMI=$(aws ssm get-parameters --names /aws/service/ami-amazon-linux-latest/al2023-ami-kernel-default-x86_64 \
  --region ${AWS_REGION} --query "Parameters[0].Value" --output text)
echo ${AMI}
sed -i "s/\${AMI}/$AMI_ID/g" $(pwd)/lesson/template/ray-vpc.yaml

aws cloudformation create-stack \
  --region ${AWS_REGION} \
  --stack-name ray-vpc \
  --template-body file://$(pwd)/lesson/template/ray-vpc.yaml \
  --parameters ParameterKey=KeyPairName,ParameterValue=${KEYPAIR_NAME} \
  --capabilities CAPABILITY_IAM \
  --tags Key=Project,Value=ray-on-aws
```
결과를 조회한다. 
```
aws cloudformation describe-stacks --stack-name ray-vpc --query "Stacks[0].StackStatus"
```

* 참고 - 스택 삭제하기
```
aws cloudformation delete-stack --stack-name ray-vpc
```

### 2. Role 생성 ###
Ray Head 노드가 Worker 노드들을 생성/삭제할 수 있도록 ray-instance-profile을 생성한다.
```
cat <<EOF > ray-trust-policy.json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": { "Service": "ec2.amazonaws.com" },
      "Action": "sts:AssumeRole"
    }
  ]
}
EOF

aws iam create-role \
  --role-name ray-autoscaling-role \
  --assume-role-policy-document file://ray-trust-policy.json
cat <<EOF > ray-autoscaling-policy.json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "ec2:RunInstances",
        "ec2:TerminateInstances",
        "ec2:DescribeInstances",
        "ec2:DescribeSubnets",
        "ec2:CreateTags"
      ],
      "Resource": "*"
    }
  ]
}
EOF

aws iam put-role-policy \
  --role-name ray-autoscaling-role \
  --policy-name ray-autoscaling-policy \
  --policy-document file://ray-autoscaling-policy.json

# 인스턴스 프로파일 생성
aws iam create-instance-profile --instance-profile-name ray-instance-profile

# 프로파일에 역할 추가
aws iam add-role-to-instance-profile \
  --instance-profile-name ray-instance-profile \
  --role-name ray-autoscaling-role
```
