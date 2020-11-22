#!/bin/bash

clear

echo

echo -e  "                     ENTER YOUR AWS Access Key and Secret Key - \n "

aws configure


echo -e  "\n -----------------------     AWS CLI CONFIGURATION DONE SUCCESSFULLY     ----------------------- "

echo -e "\n  ----------------------          CREATING KEY PAIR      ----------------------- "         

echo -e -n  "\n   Enter the Key-name :  "

read key_name

aws ec2 create-key-pair --key-name "$key_name"  --query KeyMaterial --output text > "$key_name.pem"

chmod 400 $key_name.pem


echo -e "\n  ------------------------       KEY-PAIR CREATION DONE SUCCESSFULLY -------------------------- "

echo -e "\n  ----------------------          CREATING SECURITY GROUP FOR THE INSTANCE      ----------------------- "

echo -e -n "\n      Enter the Security-Group name : "

read sg_name

export sg_id=$( aws ec2 create-security-group --description "security-group-from-cli"   --group-name "$sg_name"  | jq ".GroupId" ) 

export sg_id=$( echo $sg_id | xargs )

aws ec2 create-tags  --resources $sg_id  --tags Key=Name,Value=SecurityGroup


aws ec2 authorize-security-group-ingress --group-id "$sg_id"   --protocol tcp   --port 22    --cidr 0.0.0.0/0
aws ec2 authorize-security-group-ingress --group-id "$sg_id"   --protocol tcp   --port 80    --cidr 0.0.0.0/0

echo -e "\n  ------------------------       SECURITY GROUP CREATION DONE SUCCESSFULLY -------------------------- "

echo -e "\n  ----------------------         LAUNCHING AWS INSTANCE      ----------------------- "

echo -e -n "\n      Enter the AWS Image ID : "

read image_id

echo -e -n "\n      Enter the Instance Type : "

read instance_type

echo -e -n "\n      Enter the Number of instances : "

read instance_count

echo -e -n "\n      Enter the Subnet ID : "

read subnet_id

export instance_id=$( aws ec2 run-instances  --image-id "$image_id" --instance-type "$instance_type" --key-name "$key_name" --security-group-ids "$sg_id" --count  "$instance_count" --subnet-id  "$subnet_id" | jq ".Instances[0].InstanceId " )

export instance_id=$( echo $instance_id | xargs )

aws ec2 create-tags --resources "$instance_id"  --tags  Key=Name,Value=CLI-instance

sleep 20

echo -e "\n  ----------------------         AWS INSTANCE LAUNCHED SUCCESSFULLY     ----------------------- "

echo -e "\n  ----------------------         CREATING EBS VOLUME      ----------------------- "

echo -e -n "\n      Enter the Availability Zone  : "

read az

echo -e -n "\n      Enter the EBS Volume Size ( in GiB )  : "

read volume_size

echo -e -n "\n      Enter the EBS Volume Type ( gp2 / io1 / io2 / sc1 / st1 / standard )  : "

read volume_type

export volume_id=$( aws ec2 create-volume --availability-zone "$az" --size "$volume_size" --volume-type "$volume_type" | jq ".VolumeId" )

export volume_id=$(echo $volume_id | xargs )

aws ec2 create-tags --resources "$volume_id" --tags Key=Name,Value=volume-from-cli

echo -e "\n                   EBS VOLUME CREATION IN PROGRESS     ----------------------- "

sleep 20

echo -e "\n  ----------------------         EBS VOLUME CREATED SUCCESSFULLY     ----------------------- "

echo -e "\n                   ATTACHING THE EBS VOLUME TO THE INSTANCE     ----------------------- "

aws ec2 attach-volume --device "/dev/xvdh" --instance-id "$instance_id" --volume-id "$volume_id"

sleep 20

echo -e "\n  ----------------------         CREATING S3 BUCKET     ----------------------- "

echo -e -n "\n      Enter the Region  : "

read region

echo -e -n "\n      Enter the Bucket Name  : "

read bucket_name

aws s3api create-bucket  --bucket "$bucket_name" --create-bucket-configuration LocationConstraint="$region" --region "$region"

sleep 20

echo -e "\n  ----------------------         S3 BUCKET CREATION WAS SUCCESSFUL     ----------------------- "

echo -e -n "\n      Enter the File to upload  : "

read object

aws s3api put-object --acl public-read-write  --bucket "$bucket_name"  --key "$object" --body "$object"   --content-type "image/jpeg" --content-disposition "inline; filename=$object"

sleep 10

echo -e "\n  ----------------------         FILE UPLOAD WAS SUCCESSFUL     ----------------------- "

export public_dns=$( aws ec2 describe-instances --instance-ids "$instance_id" | jq ".Reservations[0].Instances[0].PublicDnsName" )

export public_dns=$( echo $public_dns | xargs )

export public_ip=$( aws ec2 describe-instances --instance-ids "$instance_id" | jq ".Reservations[0].Instances[0].PublicIpAddress" )

export public_ip=$( echo $public_ip | xargs )

export domain_name=$( aws cloudfront create-distribution  --origin-domain-name "$bucket_name.s3.$region.amazonaws.com" | jq ".Distribution.DomainName" )

export domain_name=$( echo $domain_name | xargs )

echo  "<img src=http://$domain_name/$object  alt=image  height=500   width=500 />"  >> index.html

echo -e "\n  -------------- Creating the High Availability Architecture ------------------------ "

sleep 15

echo -e "\n"

ssh -i   $key_name.pem     ec2-user@$public_dns    sudo yum install httpd -y

echo -e "\n"

ssh -i   $key_name.pem     ec2-user@$public_dns    sudo fdisk /dev/xvdh

echo -e "\n"

ssh -i   $key_name.pem     ec2-user@$public_dns    sudo mkfs.ext4 /dev/xvdh1

echo -e "\n"

ssh -i   $key_name.pem     ec2-user@$public_dns    sudo mount /dev/xvdh1 /var/www/html


scp -i   $key_name.pem   index.html    ec2-user@$public_dns:/home/ec2-user


ssh -i   $key_name.pem   ec2-user@$public_dns    sudo cp index.html /var/www/html

ssh -i   $key_name.pem   ec2-user@$public_dns    sudo  setenforce 0

sleep 20

ssh -i   $key_name.pem   ec2-user@$public_dns    sudo systemctl start httpd

sleep 20

echo -e "\n  -------------- Opening the Website  ------------------------ "

sleep 15

echo -e "\n"

firefox "http://$public_ip/index.html" &


