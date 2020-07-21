# ECR
### docker/docker-compose install
```
$ sudo yum update -y
$ sudo amazon-linux-extras | grep docker
$ sudo amazon-linux-extras install -y docker

$ docker -v
$ sudo systemctl start docker
$ sudo systemctl status docker
$ sudo systemctl enable docker
$ sudo systemctl is-enabled docker

$ sudo gpasswd -a $USR docker
$ sudo systemctl restart docker
$ sudo systemcltl status docker
$ getent group docker

$ sudo curl -L https://github.com/docker/compose/releases/download/1.26.0/docker-compose-$(uname -s)-$(uname -m) -o /usr/local/bin/docker-compose
$ sudo chmod +x /usr/local/bin/docker-compose
$ docker-compose --version 
```

### aws config
```
$ which aws
$ aws --version

$ aws configure
  Access key ID: ...
  Secret key ID: ...
  reagion: ...
  output format: ...
$ aws s3 ls
```

### ECR full access Policy attache to EC2
```
Make Role for access to ECR
Attache Role to EC2
```

### ECR Login
```
$ aws ecr get-login --region ap-northeast-1 --no-include-email
  ...copy!

$ ...paste!(docker login -u AWS -p ...)
  ...Login Succeeded
```

### Image push to ECR
```
$ vi Dockerfile
 
$ aws ecr create-repository --repository-name myk-test --region ap-northeast-1
  ...Json output check!

$ sudo docker build -t 662251661090.dkr.ecr.ap-northeast-1.amazonaws.com/myk-test:latest .

$ docker push 662251661090.dkr.ecr.ap-northeast-1.amazonaws.com/myk-test:latest


$ aws ecr describe-repositories --region ap-northeast-1
  ...Json output check!

$ aws ecr describe-images --repository-name myk-test
  ...Json output check!

---------------------------------------------------------------------------
  66251...      : Registory ID            ap-northeast-1: Region Name
  myk-test      : Repository Name         latest        : Image Tag
```

### How to use by ECS
```
<ECR/Registory/Repository/Image>
 - Copy image URI

<ECS/Task Definition/Container Definition/Image>
 - Set  image URI
```

### Update ECR
```
$ vi Dockerfile

# Only change <Tag name>
$ docker build -t <Registory ID>.dkr.ecr.<Region>.amazon.com/<Repository Name>:<Tag Name> .

$ docker push <Registory ID>.dkr.ecr.<Region>.amazon.com/<Repository Name>:<Tag Name>
```

### Update ECS
```
<Task Definition/New Revision/Container Definition>
 - image: New image URI

<Cluster/Service/Update>
 - Revision: latest

   task           ： 4
   min health rate： 100             min health rate： 50
   max rate       ： 200 => max 8    max rate       ： 100 => max 4、change 2 at a time

 - Service Auto Scalling/Scalling Policy
   - Set Config
     Target Policy(https://docs.aws.amazon.com/ja_jp/autoscaling/ec2/userguide/as-scaling-target-tracking.html)
     Step   Policy(Normal)
```

# 参考
### AWS ECR Page
https://docs.aws.amazon.com/ecr/index.html/

### DockerHubとの料金比較
https://tracpath.com/works/devops/docker_hub_and_amazon_ec2_container_registry/






