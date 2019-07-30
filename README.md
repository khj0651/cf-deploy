## Pre-Reqs
```
sudo apt update
```






# bosh-lite (CLI v2)

- [Pre-Reqs (Ubuntu)](#pre-reqs_(ubuntu))   
- [Install CLI v2](#install-cli-v2)   
- [Install Director VM](#install-director-vm)   
- [Alias and log into the Director](#alias-and-log-into-the-director)   
- [Local Route](#local-route)   
- [Update cloud config](#update-cloud-config)   
- [Upload stemcell](#upload-stemcell)   
- [Deploy CF](#deploy-cf)  
- [CredHub-CLI](#credhub-cli)  
- [Release Modify](#release-modify)  
- [Commands](#commands)

## Pre-Reqs (Ubuntu)
```
sudo apt-get update
```
만약 http://kr.archive.ubuntu.com에 연결할 수 없다는 오류 발생 시 아래 파일 수정
```
sudo vi /etc/apt/sources.list

:%s/http:\/\/kr.archive.ubuntu.com\/ubuntu/http:\/\/ftp.daumkakao.com\/ubuntu/g
```

### VirtualBox
https://www.virtualbox.org/wiki/Downloads

### openjdk
```
sudo apt-get install openjdk-8-jdk
```

### go
```
wget https://storage.googleapis.com/golang/go1.8.3.linux-amd64.tar.gz
sudo tar -C /usr/local -xzf go1.8.3.linux-amd64.tar.gz
echo 'export GOROOT=/usr/local/go' >> $HOME/.myenv
echo 'export PATH=$PATH:$GOROOT/bin' >> $HOME/.myenv

. $HOME/.myenv
```

### git & curl
```
sudo apt install git curl
```
### ruby2.3
```
sudo apt-add-repository ppa:brightbox/ruby-ng
sudo apt update

sudo apt install ruby2.3
```

## Install CLI v2
### bosh
#### ubuntu
```
wget https://github.com/cloudfoundry/bosh-cli/releases/download/v5.5.1/bosh-cli-5.5.1-linux-amd64
chmod +x bosh-cli-*
sudo mv bosh-cli-* /usr/local/bin/bosh
```
#### Mac
```
brew install cloudfoundry/tap/bosh-cli
```
check
```
$ bosh -v
version 5.5.1-7850ac98-2019-05-21T22:28:39Z

Succeeded
```
### cf
#### ubuntu
```
wget -q -O - https://packages.cloudfoundry.org/debian/cli.cloudfoundry.org.key | sudo apt-key add -
echo "deb https://packages.cloudfoundry.org/debian stable main" | sudo tee /etc/apt/sources.list.d/cloudfoundry-cli.list
sudo apt-get update
sudo apt-get install cf-cli
```
#### Mac
```
brew tap cloudfoundry/tap
brew install cf-cli
```
check
```
$ cf -v
cf version 6.45.0+5f9ff16f9.2019-06-03
```

## Install Director VM
```
git clone https://github.com/cloudfoundry/bosh-deployment ~/workspace/bosh-deployment
vi ~/workspace/bosh-deployment/virtualbox/cpi.yml
    memory: 8196

mkdir -p ~/deployments/vbox
cd ~/deployments/vbox
bosh create-env ~/workspace/bosh-deployment/bosh.yml \
  --state ./state.json \
  -o ~/workspace/bosh-deployment/virtualbox/cpi.yml \
  -o ~/workspace/bosh-deployment/virtualbox/outbound-network.yml \
  -o ~/workspace/bosh-deployment/bosh-lite.yml \
  -o ~/workspace/bosh-deployment/bosh-lite-runc.yml \
  -o ~/workspace/bosh-deployment/uaa.yml \
  -o ~/workspace/bosh-deployment/credhub.yml \
  -o ~/workspace/bosh-deployment/jumpbox-user.yml \
  --vars-store ./creds.yml \
  -v director_name=bosh-lite \
  -v internal_ip=192.168.50.6 \
  -v internal_gw=192.168.50.1 \
  -v internal_cidr=192.168.50.0/24 \
  -v outbound_network_name=NatNetwork
```

## Alias and log into the Director
```
bosh alias-env vbox -e 192.168.50.6 --ca-cert <(bosh int ./creds.yml --path /director_ssl/ca)
export BOSH_CLIENT=admin
export BOSH_CLIENT_SECRET=`bosh int ./creds.yml --path /admin_password`

$ bosh -e vbox env
Using environment '192.168.50.6' as '?'

Name: ...
User: admin

Succeeded
```

## Local Route
```
sudo route add -net 10.244.0.0/16     192.168.50.6 # Mac OS X
sudo ip route add   10.244.0.0/16 via 192.168.50.6 # Linux (using iproute2 suite)
route add           10.244.0.0/16     192.168.50.6 # Windows
```

## Update cloud config
```
git clone https://github.com/cloudfoundry/cf-deployment ~/workspace/cf-deployment
cd ~/workspace/cf-deployment
bosh -e vbox update-cloud-config iaas-support/bosh-lite/cloud-config.yml
```
### Upload a runtime-config
```
bosh -e vbox update-runtime-config ~/workspace/bosh-deployment/runtime-configs/dns.yml --name dns
```
>https://github.com/cloudfoundry/cf-deployment/issues/648  
>https://github.com/cloudfoundry/cf-deployment/blob/master/deployment-guide.md#upload-a-runtime-config

## Upload stemcell
stemcell 버전은 cf-deployment.yml 파일의 "stemcells" 버전을 입력한다.
```
export IAAS_INFO=warden-boshlite
export STEMCELL_VERSION=$(bosh interpolate cf-deployment.yml --path=/stemcells/alias=default/version)
bosh -e vbox upload-stemcell https://bosh.io/d/stemcells/bosh-$IAAS_INFO-ubuntu-xenial-go_agent?v=$STEMCELL_VERSION
```

## CredHub CLI
#### Ubuntu
Download release file
> https://github.com/cloudfoundry-incubator/credhub-cli/releases

and install
```
tar -xvf credhub-linux-2.2.0.tgz
mv credhub /usr/local/bin
```
#### Mac
```
brew install cloudfoundry/tap/credhub-cli
```
### Set CredHub
```
export BOSH_CA_CERT="$(bosh interpolate ~/deployments/vbox/creds.yml --path /director_ssl/ca)"

export CREDHUB_SERVER=https://192.168.50.6:8844
export CREDHUB_CLIENT=credhub-admin
export CREDHUB_SECRET=$(bosh interpolate ~/deployments/vbox/creds.yml --path=/credhub_admin_client_secret)
export CREDHUB_CA_CERT="$(bosh interpolate ~/deployments/vbox/creds.yml --path=/credhub_tls/ca )"$'\n'"$( bosh interpolate ~/deployments/vbox/creds.yml --path=/uaa_ssl/ca)"
```

and retrieve credentials in CredHub
```
credhub login -s https://192.168.50.6:8844 --skip-tls-validation
credhub find
credhub find -n cf_admin_password
credhub get -n <FULL_CREDENTIAL_NAME>
credhub get -n /bosh-lite/cf/cf_admin_password
```

## Deploy CF

```
bosh -e vbox -d cf deploy cf-deployment.yml \
   -o operations/bosh-lite.yml \
   -v system_domain=bosh-lite.com
```
https://github.com/cloudfoundry/cf-deployment/blob/master/deployment-guide.md
```
cf api https://api.bosh-lite.com --skip-ssl-validation
export CF_ADMIN_PASSWORD=$(bosh int <(credhub get -n /bosh-lite/cf/cf_admin_password --output-json) --path /value)
cf auth admin $CF_ADMIN_PASSWORD
```
```
cf create-org org
cf create-space space -o org
cf target -o org -s space

mkdir -p test-php; cd test-php
echo "<?php phpinfo(); ?>" > index.php
cf push test-php -b php_buildpack
```

## Release Modify
```
git clone https://github.com/cloudfoundry/diego-release ~/workspace/diego-release
cf ~/workspace/diego-release
git checkout tags/v2.22.0
bosh -e vbox releases
bosh -e vbox create-release --force
bosh -e vbox upload-release
```
modify cf-deployment.yml
```
releases:
- name: diego
#  url: https://bosh.io/d/github.com/cloudfoundry/diego-release?v=2.22.0
  version: 2.22.0+dev.3
  sha1: 0cd26089729d4e526dd27c47191cc3b9cb3189c6
```
deploy
```
cd ~/workspace/cf-deployment
bosh -e vbox -d cf deploy cf-deployment.yml \
   -o operations/bosh-lite.yml \
   -v system_domain=bosh-lite.com
```
