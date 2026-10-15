# ArgoCD 오류 해결을 위한 Metrics Addon 교체

## kubectl 버전 업그레이드
~~~
cd ~
curl -O https://s3.us-west-2.amazonaws.com/amazon-eks/1.36.4/2026-07-05/bin/linux/amd64/kubectl
chmod +x ./kubectl
mkdir -p $HOME/bin && cp ./kubectl $HOME/bin/kubectl && export PATH=$HOME/bin:$PATH
echo 'export PATH=$HOME/bin:$PATH' >> ~/.bashrc
~~~

## 
