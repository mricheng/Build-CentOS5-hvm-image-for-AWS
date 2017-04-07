# Build-CentOS5-hvm-image-for-AWS

This project is to build CentOS 5 hvm image by kickstart on KVM for AWS.

1. setup a build host with KVM and http server, install AWS CLI from
 https://aws.amazon.com/amazon-linux-ami/

2. setup a local respository on the build host, copy those dependents packages to the http server directory and run the following command to setup local respository

>sudo mkdir /var/www/html/centos5-ks

>sudo cp kickstart-cfg/* /var/www/html/centos5-ks

>sudo mkdir /var/www/html/centos5-cloud

>sudo cp respository/* /var/www/html/centos5-cloud

>cd /var/www/html/centos5-cloud

>sudo createrepo -s sha1 .

3. install a VM by kickstart as ami-centos-5.11-base-hvm-x86.img

>sudo qemu-img create -f raw /var/lib/libvirt/images/ami-centos-5.11-base-hvm-x86.img 10G

>sudo virt-install --name ami-centos-5.11-base-hvm \
     --ram 1024 --vcpus=2 --os-variant=rhel5 \
     --location=http://mirror.optus.net/centos/5/os/x86_64/  \
     --extra-args="ks=http://localhost/centos5-ks/aws-base-hvm-ks5.cfg" \
     --disk path=/var/lib/libvirt/images/ami-centos-5.11-base-hvm-x86.img,size=10,format=raw,bus=virtio \
     --bridge=br0,model=virtio

4. upload the image to bucket on AWS storage 
>aws s3 cp ami-centos-5.11-base-hvm-x86.img s3://my-ami

5. create trust policy
>vim trust-policy.json

{
   "Version":"2012-10-17",
   "Statement":[
      {
         "Sid":"",
         "Effect":"Allow",
         "Principal":{
            "Service":"vmie.amazonaws.com"
         },
         "Action":"sts:AssumeRole",
         "Condition":{
            "StringEquals":{
               "sts:ExternalId":"vmimport"
            }
         }
      }
   ]
}

6. create iam role for vminport 
>aws iam create-role --role-name vmimport --assume-role-policy-document file://trust-policy.json

7. create role policy
>vim role-policy.json
{
   "Version":"2012-10-17",
   "Statement":[
      {
         "Effect":"Allow",
         "Action":[
            "s3:ListBucket",
            "s3:GetBucketLocation"
         ],
         "Resource":[
            "arn:aws:s3:::my-ami"
         ]
      },
      {
         "Effect":"Allow",
         "Action":[
            "s3:GetObject"
         ],
         "Resource":[
            "arn:aws:s3:::my-ami/*"
         ]
      },
      {
         "Effect":"Allow",
         "Action":[
            "ec2:ModifySnapshotAttribute",
            "ec2:CopySnapshot",
            "ec2:RegisterImage",
            "ec2:Describe*"
         ],
         "Resource":"*"
      }
   ]
}

8. apply the role policy

>aws iam put-role-policy --role-name vmimport --policy-name vmimport --policy-document file://role-policy.json 

9. create import.json

>vim import.json
{
    "Description": "Import image",
    "DiskContainers": [
        {
            "Description": "import raw image",
            "UserBucket": {
                "S3Bucket": "my-ami",
                "S3Key": "ami-centos-5.11-base-hvm-x86.img"
            }
        }
    ]
}

10. run import image task 
>aws ec2 import-image --cli-input-json file://import.json

11. check import image task
>aws ec2 describe-import-image-tasks --import-task-ids "import-ami-fg5cffht"

"ImportImageTasks": [
        {
            "Status": "completed", 
            "LicenseType": "BYOL", 
            "Description": "Import image", 
            "ImageId": "ami-c8111fab", 
            "Platform": "Linux", 
            "Architecture": "x86_64", 
            "SnapshotDetails": [
                {
                    "DeviceName": "/dev/sda1", 
                    "Description": "ami centos 5.11 hvm image", 
                    "Format": "RAW", 
                    "DiskImageSize": 2147483648.0, 
                    "SnapshotId": "snap-02fa81816e5b41473", 
                    "UserBucket": {
                        "S3Bucket": "my-ami", 
                        "S3Key": "ami-centos-5.11-hvm-x86.img"
                    }
                }
            ], 
            "ImportTaskId": "import-ami-fg5cffht"
        }
    ]
}


12. Copy the imported image to a image with the normal name
>aws ec2 copy-image --source-image-id ami-c8111fab --source-region ap-southeast-2 --name "ami-centos-5.11-base-hvm-x86" --description "CentOS 5.11 base hvm"

13. deregister the imported image
>aws ec2 deregister-image --image-id ami-c8111fab
