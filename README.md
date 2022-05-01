# vpc-cni
Custom networking with the AWS VPC CNI plug-in

Steps:
=======

1. Attach secondary CIDR to existing VPC where EKS cluster is created
2. Create Subnets within secondary CIDR range
3. Create eniconfig for each AZ and apply it
4. Add env variables (AWS_VPC_K8S_CNI_CUSTOM_NETWORK_CFG,ENI_CONFIG_LABEL_DEF, AWS_VPC_K8S_CNI_EXTERNALSNAT) for aws-node daemonset 


          VPC          CIDR	             Subnet A	    Subnet B	    Subnet C
          Primary	192.168.0.0/16	192.168.0.0/19	192.168.32.0/19	192.168.64.0/19
          Secondary	100.64.0.0/16	100.64.0.0/19	100.64.32.0/19	100.64.64.0/19


1. Attach secondary CIDR to existing VPC where EKS cluster is created


          VPC_ID=aws ec2 describe-vpcs --filters Name=tag:Name,Values=*eksdemo-qa* --query 'Vpcs[].VpcId' --output text
          aws ec2 associate-vpc-cidr-block --vpc-id $VPC_ID --cidr-block 100.64.0.0/16

2. Create Subnets within secondary CIDR range


I have 3 instances and using 3 subnets in my environment. For simplicity, we will use the same AZ’s and create 3 secondary CIDR subnets but you can certainly customize according to your networking requirements.


          export POD_AZS=($(aws ec2 describe-instances --filters "Name=tag-key,Values=eks:cluster-name" "Name=tag-value,Values=*eksdemo-qa*" --query 'Reservations[*].Instances[*].[Placement.AvailabilityZone]' --output text | sort | uniq))
          CGNAT_SNET1=$(aws ec2 create-subnet --cidr-block 100.64.0.0/19 --vpc-id $VPC_ID --availability-zone us-east-2a --query 'Subnet.SubnetId' --output text)
          CGNAT_SNET2=$(aws ec2 create-subnet --cidr-block 100.64.32.0/19 --vpc-id $VPC_ID --availability-zone us-east-2b --query 'Subnet.SubnetId' --output text)
          CGNAT_SNET3=$(aws ec2 create-subnet --cidr-block 100.64.64.0/19 --vpc-id $VPC_ID --availability-zone us-east-2c --query 'Subnet.SubnetId' --output text)

we need to associate three new subnets into a route table. Again for simplicity, we chose to add new subnets to the Public route table that has connectivity to Internet Gateway

          SNET1=$(aws ec2 describe-subnets --filters Name=cidr-block,Values=192.168.0.0/19 --query 'Subnets[].SubnetId' --output text)
          RTASSOC_ID=$(aws ec2 describe-route-tables --filters Name=association.subnet-id,Values=$SNET1 --query 'RouteTables[].RouteTableId' --output text)
          aws ec2 associate-route-table --route-table-id $RTASSOC_ID --subnet-id $CGNAT_SNET1
          aws ec2 associate-route-table --route-table-id $RTASSOC_ID --subnet-id $CGNAT_SNET2
          aws ec2 associate-route-table --route-table-id $RTASSOC_ID --subnet-id $CGNAT_SNET3

sg-0917c42587e776c96
sg-0dbb27eab2c02382a

kubectl set env daemonset aws-node -n kube-system AWS_VPC_K8S_CNI_CUSTOM_NETWORK_CFG=true ENI_CONFIG_LABEL_DEF=failure-domain.beta.kubernetes.io/zone AWS_VPC_K8S_CNI_EXTERNALSNAT=false
