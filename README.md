# vpc-cni
Custom networking with the AWS VPC CNI plug-in

Steps:
=======

1. Attach secondary CIDR to existing VPC where EKS cluster is created
2. Create Subnets within secondary CIDR range
3. Create eniconfig for each AZ and apply it
4. Add env variables (AWS_VPC_K8S_CNI_CUSTOM_NETWORK_CFG,ENI_CONFIG_LABEL_DEF, AWS_VPC_K8S_CNI_EXTERNALSNAT) for aws-node daemonset 

          VPC	      CIDR	          Subnet A	      Subnet B	      Subnet C
          Primary	  192.168.0.0/16	192.168.0.0/19	192.168.32.0/19	192.168.64.0/19
          Secondary	100.64.0.0/16	  100.64.0.0/19	  100.64.32.0/19	100.64.64.0/19

<img width="494" alt="image" src="https://user-images.githubusercontent.com/74225291/166148858-be892757-ba31-4b21-bf32-eaf6c9890e28.png">




1. Attach secondary CIDR to existing VPC where EKS cluster is created

aws ec2 associate-vpc-cidr-block --vpc-id vpc-0c7ca59f1ae98501a  --cidr-block 100.64.0.0/16

2. Create Subnets within secondary CIDR range

export VPC_ID=vpc-0c7ca59f1ae98501a
export POD_AZS=us-east-2a,us-east-2b,us-east-2c
CGNAT_SNET1=$(aws ec2 create-subnet --cidr-block 100.64.0.0/19 --vpc-id $VPC_ID --availability-zone us-east-2a --query 'Subnet.SubnetId' --output text)
CGNAT_SNET2=$(aws ec2 create-subnet --cidr-block 100.64.32.0/19 --vpc-id $VPC_ID --availability-zone us-east-2b --query 'Subnet.SubnetId' --output text)
CGNAT_SNET3=$(aws ec2 create-subnet --cidr-block 100.64.64.0/19 --vpc-id $VPC_ID --availability-zone us-east-2c --query 'Subnet.SubnetId' --output text)

sg-0917c42587e776c96
sg-0dbb27eab2c02382a

kubectl set env daemonset aws-node -n kube-system AWS_VPC_K8S_CNI_CUSTOM_NETWORK_CFG=true ENI_CONFIG_LABEL_DEF=failure-domain.beta.kubernetes.io/zone AWS_VPC_K8S_CNI_EXTERNALSNAT=false
