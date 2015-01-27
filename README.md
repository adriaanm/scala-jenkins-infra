# Scala's Jenkins Cluster 
The idea is to use chef to configure EC2 instances for both the master and the slaves. The jenkins config will be captured in chef recipes. Everything is versioned, with server and workers not allowed to maintain state.

This is inspired by https://erichelgeson.github.io/blog/2014/05/10/automating-your-automation-federated-jenkins-with-chef/


# Get some tools
```
brew cask install awscli cord
```

# One-time EC2/IAM setup
## Create security group (ec2 firewall)

CONFIGURATOR_IP is the ip of the machine running knife to initiate the bootstrap -- it can be removed once chef-client is running.


```
aws ec2 create-security-group --group-name "Master" --description "Remote access to the Jenkins master" 
aws ec2 authorize-security-group-ingress --group-name "Master" --protocol tcp --port 22 --cidr $CONFIGURATOR_IP/32 # ssh bootstrap
aws ec2 authorize-security-group-ingress --group-name "Master" --protocol tcp --port 8080 --cidr 0.0.0.0/0
```

Type             | Protocol | Port Range | Source        |
----------------------------------------------------------
HTTP             |  TCP  |  80   | 0.0.0.0/0             |
HTTPS            |  TCP  |  443  | 0.0.0.0/0             |
Custom TCP Rule  |  TCP  |  8888 | 0.0.0.0/0             |
All traffic      |  All  |  All  | sg-ecb06389 (Workers) |
All traffic      |  All  |  All  | sg-1dec3d78 (Windows) |
SSH              |  TCP  |  22   | $CONFIGURATOR_IP/32   |


```
aws ec2 create-security-group --group-name "Windows" --description "Remote access to Windows instances" 
aws ec2 authorize-security-group-ingress --group-name "Windows" --protocol tcp --port 5985 --cidr $CONFIGURATOR_IP/32 # allow WinRM from the machine that will execute `knife ec2 server create` below
aws ec2 authorize-security-group-ingress --group-name "Windows" --protocol tcp --port 0-65535 --source-group Master
```

Type            | Protocol | Port Range | Source             |
--------------------------------------------------------------
All TCP            | TCP | 0 - 65535 |  sg-7afd2d1f (Master) |
Custom TCP Rule    | TCP | 445       |  $CONFIGURATOR_IP/32  |
RDP                | TCP | 3389      |  $CONFIGURATOR_IP/32  |
SSH                | TCP | 22        |  $CONFIGURATOR_IP/32  |
Custom TCP Rule    | TCP | 5985      |  $CONFIGURATOR_IP/32  |


```
aws ec2 create-security-group --group-name "Workers" --description "Jenkins workers nodes" 
aws ec2 authorize-security-group-ingress --group-name "Workers" --protocol tcp --port 22 --cidr $CONFIGURATOR_IP/32 # ssh bootstrap
aws ec2 authorize-security-group-ingress --group-name "Workers" --protocol tcp --port 0-65535 --source-group Master
```

Type        | Protocol | Port Range |        Source        |
------------------------------------------------------------
All TCP     |   TCP    | 0 - 65535  | sg-7afd2d1f (Master) |
SSH         |   TCP    | 22         | $CONFIGURATOR_IP/32  |


## Instance profiles
This avoids passing credentials for instances to use aws services.

### Create instance profiles
An instance profile must be passed when instance is created. It must contain one role. Can attach more policies to that role at any time.


Based on http://domaintest001.com/aws-iam/

```
aws iam create-instance-profile --instance-profile-name JenkinsMaster
aws iam create-instance-profile --instance-profile-name JenkinsWorkerPublish
aws iam create-instance-profile --instance-profile-name JenkinsWorker

aws iam create-role --role-name jenkins-master         --assume-role-policy-document file:///Users/adriaan/git/scala-jenkins-infra/chef/ec2-role-trust-policy.json
aws iam create-role --role-name jenkins-worker         --assume-role-policy-document file:///Users/adriaan/git/scala-jenkins-infra/chef/ec2-role-trust-policy.json
aws iam create-role --role-name jenkins-worker-publish --assume-role-policy-document file:///Users/adriaan/git/scala-jenkins-infra/chef/ec2-role-trust-policy.json

aws iam add-role-to-instance-profile --instance-profile-name JenkinsMaster        --role-name jenkins-master
aws iam add-role-to-instance-profile --instance-profile-name JenkinsWorker        --role-name jenkins-worker
aws iam add-role-to-instance-profile --instance-profile-name JenkinsWorkerPublish --role-name jenkins-worker-publish
```

### Attach policies to roles:

```
aws iam put-role-policy --role-name jenkins-master --policy-name jenkins-ec2-start-stop --policy-document file:///Users/adriaan/git/scala-jenkins-infra/chef/jenkins-ec2-start-stop.json
aws iam put-role-policy --role-name jenkins-master --policy-name jenkins-dynamodb --policy-document file:///Users/adriaan/git/scala-jenkins-infra/chef/dynamodb.json

// TODO: once https://github.com/sbt/sbt-s3/issues/14 is fixed, remove s3credentials from nodes and use IAM profile instea
aws iam put-role-policy --role-name jenkins-worker-publish --policy-name jenkins-s3-upload --policy-document file:///Users/adriaan/git/scala-jenkins-infra/chef/jenkins-s3-upload.json

aws iam put-role-policy --role-name jenkins-worker TODO
```

NOTE: if you get syntax errors, check the policy doc URL
pass JenkinsWorker as the iam profile to knife bootstrap


## Create an Elastic IP for each node
TODO: attach to elastic IPs 


# Install chef/knife

```
brew cask install chefdk
eval "$(chef shell-init zsh)" # set up gem environment
gem install knife-ec2 knife-windows knife-github-cookbooks chef-vault
```

## Get credentials for the typesafe-scala chef.io organization
```
export CHEF_ORG="typesafe-scala"
```

download from https://manage.chef.io/organizations/typesafe-scala:
  - user key (to `.chef/config/#{ENV['USER']}.pem` -- name it like this to use the `.chef/knife.rb` in this repo)
  - org validation key (to `.chef/config/#{ENV['CHEF_ORG']}-validator.pem`)

## Get cookbooks

```
git init .chef/cookbooks
cd .chef/cookbooks
g commit --allow-empty -m"Initial" 
```

- knife cookbook site install wix 1.0.2 # newer versions don't work for me; also installs windows
- knife cookbook site install aws
- knife cookbook site install git
- knife cookbook site install git_user
  - knife cookbook site install partial_search
 
- move to unreleased versions on github:
  - knife cookbook github install opscode-cookbooks/windows    # fix nosuchmethoderror (#150)
  - knife cookbook github install adriaanm/java/windows-jdk1.6 # jdk 1.6 installer barfs on re-install -- wipe its INSTALLDIR
  - knife cookbook github install adriaanm/jenkins/fix305      # ssl fail on windows
  - knife cookbook github install adriaanm/scala-jenkins-infra
  - knife cookbook github install adriaanm/chef-sbt
  - knife cookbook github install gildegoma/chef-sbt-extras

- knife cookbook upload --all

## Cache installers locally
- they are tricky to access, might disappear,...
- checksum is computed with `shasum -a 256`
- TODO: host them on an s3 bucket (credentials are available automatically)


# Configuring the jenkins cluster


## Secure data (one-time setup, can be done before bootstrap)

from http://jtimberman.housepub.org/blog/2013/09/10/managing-secrets-with-chef-vault/

NOTE: the JSON must not have a field "id"!!!

### Chef user with keypair for jenkins cli access
```
eval "$(chef shell-init zsh)" # use chef's ruby, which has the net/ssh gem
ruby chef/keypair.rb > ~/Desktop/chef-secrets/config/keypair.json
ruby chef/keypair.rb > ~/Desktop/chef-secrets/config/scabot-keypair.json

# extract private key to ~/Desktop/chef-secrets/config/scabot.pem

knife vault create master scala-jenkins-keypair \
  --json ~/Desktop/chef-secrets/config/keypair.json \
  --search 'name:jenkins*' \
  --admins adriaan

knife vault create master scabot-keypair \
  --json ~/Desktop/chef-secrets/config/scabot-keypair.json \
  --search 'name:jenkins-master' \
  --admins adriaan

knife vault create master scabot \
  --json ~/Desktop/chef-secrets/config/scabot.json \
  --search 'name:jenkins-master' \
  --admins adriaan

```

### For github oauth

https://github.com/settings/applications/new --> https://github.com/settings/applications/154904
 - Authorization callback URL = https://scala-ci.typesafe.com/securityRealm/finishLogin

```
knife vault create master github-api \
  '{"client-id":"<Client ID>","client-secret":"<Client secret>"}' \
  --search 'name:jenkins-master' \
  --admins adriaan
```

### For nginx ssl

```
knife vault create master scala-ci-key \
  --json scalaci-key.json \
  --search 'name:jenkins-master' \
  --admins adriaan
```


### Workers that need to publish
```
knife vault create worker-publish sonatype \
  '{"user":"XXX","pass":"XXX"}' \
  --search 'name:jenkins-worker-ubuntu-publish' \
  --admins adriaan

knife vault create worker-publish private-repo \
  '{"user":"XXX","pass":"XXX"}' \
  --search 'name:jenkins-worker-ubuntu-publish' \
  --admins adriaan

knife vault create worker-publish s3-downloads \
  '{"user":"XXX","pass":"XXX"}' \
  --search 'name:jenkins-worker-windows OR name:jenkins-worker-ubuntu-publish' \
  --admins adriaan

knife vault create worker-publish chara-keypair \
  --json chara-keypair.json \
  --search 'name:jenkins-worker-ubuntu-publish' \
  --admins adriaan

knife vault create worker-publish gnupg \
  --json /Users/adriaan/Desktop/chef-secrets/gnupg.json \
  --search 'name:jenkins-worker-ubuntu-publish' \
  --admins adriaan

knife vault create worker private-repo-public-jobs \
  '{"user":"XXX","pass":"XXX"}' \
  --search 'name:jenkins-worker-behemoth-*' \
  --admins adriaan

```

#  Dev machine convenience
This is how I set up my desktop to make it easier to connect to the EC2 nodes.
The README assumes you're using this as well.

## /etc/hosts
Note that the IPs are stable by allocating elastic IPs and associating them to nodes.

```
54.67.111.226 jenkins-master
54.67.33.167  jenkins-worker-ubuntu-publish
54.183.156.89 jenkins-worker-windows
54.153.2.9    jenkins-worker-behemoth-1
54.153.1.99   jenkins-worker-behemoth-2
```

## ~/.ssh/config
```
Host jenkins-worker-ubuntu-publish
  IdentityFile ~/Desktop/chef-secrets/config/chef.pem
  User ubuntu

Host jenkins-worker-behemoth-1
  IdentityFile ~/Desktop/chef-secrets/config/chef.pem
  User ec2-user

Host jenkins-worker-behemoth-2
  IdentityFile ~/Desktop/chef-secrets/config/chef.pem
  User ec2-user

Host jenkins-master
  IdentityFile ~/Desktop/chef-secrets/config/chef.pem
  User ec2-user

Host scabot
  HostName jenkins-master
  IdentityFile ~/Desktop/chef-secrets/config/scabot.pem
  User scabot

Host jenkins-worker-windows
  IdentityFile ~/Desktop/chef-secrets/jenkins-chef
  User jenkins
```


# Launch instance on EC2
## Create (ssh) key pair
```
echo $(aws ec2 create-key-pair --key-name chef | jq .KeyMaterial) | perl -pe 's/"//g' > ~/git/scala-jenkins-infra/.chef/config/chef.pem
chmod 0600 ~/git/scala-jenkins-infra/.chef/config/chef.pem
```

make sure `knife[:aws_ssh_key_id] = 'chef'` matches `--identity-file ~/git/scala-jenkins-infra/.chef/config/chef.pem`


## Selected AMIs

current windows: ami-cfa5b68a Windows_Server-2012-R2_RTM-English-64Bit-Base-2014.12.10
current linux-publisher: ami-b11b09f4 ubuntu/images-testing/hvm/ubuntu-trusty-daily-amd64-server-20141212

ami-5956491c ubuntu/images-testing/hvm/ubuntu-utopic-daily-amd64-server-20150106

current linux (master/worker): ami-4b6f650e Amazon Linux AMI 2014.09.1 x86_64 HVM EBS


## Bootstrap
NOTE:

  - name is important (used to allow access to vault etc); it can't be changed later, and duplicates aren't allowed (can bite when repeating knife ec2 create)
  - can't access the vault on bootstrap (see After bootstrap below)



```
knife ec2 server create -N jenkins-master \
   --region us-west-1 --flavor t2.small -I ami-4b6f650e \
   -G Master --ssh-user ec2-user \
   --iam-profile JenkinsMaster \
   --identity-file .chef/config/chef.pem \
   --run-list "scala-jenkins-infra::master-init"

knife ec2 server create -N jenkins-worker-windows \
   --region us-west-1 --flavor c3.xlarge -I ami-45332200 \
   -G Windows --user-data chef/userdata/win2012.txt --bootstrap-protocol winrm \
   --iam-profile JenkinsWorkerPublish \
   --identity-file .chef/config/chef.pem \
   --run-list "scala-jenkins-infra::worker-init"

// NOTE: c3.large is much slower than c3.xlarge (scala-release-2.11.x-build takes 2h53min vs 1h40min )
knife ec2 server create -N jenkins-worker-ubuntu-publish \
   --region us-west-1 --flavor c3.xlarge -I ami-5956491c \
   -G Workers --ssh-user ubuntu \
   --iam-profile JenkinsWorkerPublish \
   --identity-file .chef/config/chef.pem \
   --user-data chef/userdata/linux-2-ephemeral-one-home \
   --run-list "scala-jenkins-infra::worker-init"

knife ec2 server create -N jenkins-worker-behemoth-1 \
   --region us-west-1 --flavor c4.2xlarge -I ami-4b6f650e \
   -G Workers --ssh-user ec2-user \
   --iam-profile JenkinsWorker \
   --identity-file .chef/config/chef.pem \
   --run-list "scala-jenkins-infra::worker-init"

knife ec2 server create -N jenkins-worker-behemoth-2 \
   --region us-west-1 --flavor c4.2xlarge -I ami-4b6f650e \
   -G Workers --ssh-user ec2-user \
   --iam-profile JenkinsWorker \
   --identity-file .chef/config/chef.pem \
   --run-list "scala-jenkins-infra::worker-init"

```

TODO: use /mnt/ephemeral1 for something during build?

NOTE: userdata.txt must be one line, no line endings (mac/windows issues?)
`<script>winrm quickconfig -q & winrm set winrm/config/service @{AllowUnencrypted="true"} & winrm set winrm/config/service/auth @{Basic="true"} & netsh advfirewall firewall set rule group="remote administration" new enable=yes & netsh advfirewall firewall add rule name="WinRM Port" dir=in action=allow protocol=TCP  localport=5985</script>`



## After bootstrap (or when nodes are added)
### Attach eips
```
aws ec2 associate-address --allocation-id eipalloc-df0b13bd --instance-id i-94adaa5e  # jenkins-master
aws ec2 associate-address --allocation-id eipalloc-1cc6de7e --instance-id i-a100026b  # jenkins-worker-windows
aws ec2 associate-address --allocation-id eipalloc-c2abb3a0 --instance-id i-0c3c3cc6  # jenkins-worker-ubuntu-publish
aws ec2 associate-address --allocation-id eipalloc-b8574dda --instance-id i-fa4e7032  # jenkins-worker-behemoth-1
aws ec2 associate-address --allocation-id eipalloc-bb574dd9 --instance-id i-7b563db8  # jenkins-worker-behemoth-1
```

### Update access to vault
```
knife vault update master scala-jenkins-keypair --search 'name:jenkins*'

knife vault update worker private-repo-public-jobs  --search 'name:jenkins-worker-behemoth-*'

knife vault update master github-api            --search 'name:jenkins-master'

knife vault update worker-publish sonatype      --search 'name:jenkins-worker-ubuntu-publish'
knife vault update worker-publish private-repo  --search 'name:jenkins-worker-ubuntu-publish'
knife vault update worker-publish chara-keypair --search 'name:jenkins-worker-ubuntu-publish'
knife vault update worker-publish gnupg         --search 'name:jenkins-worker-ubuntu-publish'
knife vault update worker-publish s3-downloads  --search 'name:jenkins-worker-windows OR name:jenkins-worker-ubuntu-publish'
```

### Add run-list items that need the vault after bootstrap
```
knife node run_list set jenkins-master    "scala-jenkins-infra::master-init,scala-jenkins-infra::master-config"
for w in jenkins-worker-windows jenkins-worker-ubuntu-publish jenkins-worker-behemoth-1 jenkins-worker-behemoth-2
  do knife node run_list set jenkins-worker-behemoth-2  "scala-jenkins-infra::worker-init,scala-jenkins-infra::worker-config"
done
```

### Re-run chef manually

- windows:
```
PASS=$(aws ec2 get-password-data --instance-id i-a100026b --priv-launch-key ~/Desktop/chef-secrets/config/chef.pem | jq .PasswordData | xargs echo)
knife winrm jenkins-worker-windows chef-client -m -P $PASS
```

- ubuntu:  `ssh jenkins-worker-ubuntu-publish sudo chef-client`
- amazon linux: `ssh jenkins-worker-behemoth-1`, and then `sudo chef-client`



# Misc

## Testing locally using vagrant

http://blog.gravitystorm.co.uk/2013/09/13/using-vagrant-to-test-chef-cookbooks/:

```
Vagrant.configure("2") do |config|
  config.vm.box = "precise64"
  config.vm.provision :chef_solo do |chef|
    chef.cookbooks_path = "/home/andy/src/toolkit-chef/cookbooks"
    chef.add_recipe("toolkit")
  end
  config.vm.network :forwarded_port, guest: 80, host: 11180
end
```

## If connections hang
Make sure security groups allow access...

## SSL cert
```
$ openssl genrsa -out scala-ci.key 2048
```
and

```
$ openssl req -new -out scala-ci.csr -key scala-ci.key -config ssl-certs/scalaci.openssl.cnf
```

Send CSR to SSL provider, receive scalaci.csr. Store scala-ci.key securely in vault master scala-ci-key (see above).

Incorporate the cert into an ssl chain for nginx:
```
(cd ssl-certs && cat 00\ -\ scala-ci.crt 01\ -\ COMODORSAOrganizationValidationSecureServerCA.crt 02\ -\ COMODORSAAddTrustCA.crt 03\ -\ AddTrustExternalCARoot.crt > ../files/default/scala-ci.crt)
```

For [forward secrecy](http://axiacore.com/blog/enable-perfect-forward-secrecy-nginx/):
```
openssl dhparam -out files/default/dhparam.pem 2048
```

Confirm values in the csr using:

```
$ openssl req -text -noout -in scala-ci.csr
```

## If the bootstrap didn't work at first, complete:
If it appears stuck at "Waiting for remote response before bootstrap.", the userdata didn't make it across 
(check C:\Program Files\Amazon\Ec2ConfigService\Logs) we need to enable unencrypted authentication:

```
aws ec2 get-password-data --instance-id $INST --priv-launch-key ~/git/scala-jenkins-infra/.chef/config/chef.pem

cord $IP, log in using password above and open a command line:

  winrm set winrm/config/service @{AllowUnencrypted="true"}
  winrm set winrm/config/service/auth @{Basic="true"}

knife bootstrap -V windows winrm $IP
```



## Alternative windows AMIs
too stripped down (bootstraps in 8 min, though): ami-23a5b666 Windows_Server-2012-R2_RTM-English-64Bit-Core-2014.12.10
userdata.txt: `<script>winrm quickconfig -q & winrm set winrm/config/service @{AllowUnencrypted="true"} & winrm set winrm/config/service/auth @{Basic="true"} & netsh advfirewall firewall set rule group="remote administration" new enable=yes & netsh advfirewall firewall add rule name="WinRM Port" dir=in action=allow protocol=TCP  localport=5985</script>`

older: ami-e9a4b7ac amazon/Windows_Server-2008-SP2-English-64Bit-Base-2014.12.10
userdata.txt: '<script>winrm quickconfig -q & winrm set winrm/config/service @{AllowUnencrypted="true"} & winrm set winrm/config/service/auth @{Basic="true"}</script>'

older: ami-6b34252e Windows_Server-2008-R2_SP1-English-64Bit-Base-2014.11.19
doesn't work: ami-59a8bb1c Windows_Server-2003-R2_SP2-English-64Bit-Base-2014.12.10
