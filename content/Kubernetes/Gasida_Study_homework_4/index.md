---
emoji: 🔮
title: Gasida Kubernetes Study 4주차
date: '2023-02-08 22:00:00'
author: KORWOO
tags: Kubernetes
categories: Kubernetes
---


가시다님 스터디 : https://www.notion.so/gasidaseo/23-7635cc4f02c04954a3260b317588113e

4주차 Harbor를 이용한 이미지 저장소 구축 및 ArgoCD

## 1.ArgoCD??

어느덧 가시다님의 쿠버네티스 스터디도 4주차에 접어들었습니다.
이번주의 주제는 Harbor 구축 및 ArgoCD 입니다.
Harbor의 경우 제가 근무하고 있는 회사에서 한때 사용하였던 경험이 있어 나름 익숙한 Container Registry 였지만
ArgoCD같은 경우 여기저기서 많이 들어는봤지만 생소한 개념이었습니다..ㅜㅜ
그래서 ArgoCD가 무엇인지에 대해 설명하고 넘어가겠습니다.

ArgoCD에 대해 설명하기 위해서는 먼저 GitOps에 대해 알아야합니다.
GitOps는 어렵게 생각할것 없이 Git에서 사용하는 Devops의 실천 방법 중 하나입니다.
그중에서도 CD(Continous Deployment)에 초점을 두고 있습니다.

GitOps의 핵심 아이디어는
1. 배포에 관련된 모든 것을 선언형 기술서(Declarative Descriptions)형태로 작성하여 Config Repostiry에서 관리
2. Config Repositry의 선언형 기술서와 운영 환경 간 상태차이가 없도록 유지시키는 자동화 시스템 구성

즉 GitOps는 K8S Manifest 파일을 Git에서 관리하고 배포할때에도 Git에 저장된 Manifest로 Cluster에 배포하는 과정입니다.
Manifest 파일에 배포된 상태가 어떤 모양을 가져아 할지 선언되어 있는 방식으로 정의하여 단일 진실의 원천(SSOT) 를 지킵니다.
GitOps를 사용하면 배포상태의 모습을 항상 Manifest에 정의된 대로 원천가 동일하게 맞추며 현재 배포환경의 상태를 쉽게 파악할 수 있습니다.

GitOps에 대해서 설명드릴 내용이 좀 더 많이 있으나 GitOps가 메인이 아닌지라 기회가 된다면 다음 포스팅에서 GitOps에 대해 다뤄볼수 있도록 하겠습니다.
이 GitOps를 구현체가 바로 ArgoCD 입니다!!

서론이 길었습니다.

바로 실습으로 넘어가도록 하겠습니다.


## 2. Harbor Container Registry 구축
Cluster 구성 과정은 지난번 포스팅을 참고하여 주시고 바로 진행하겠습니다.

먼저 지나번과 같이 Control Plane, Worker Node에 AWS LoadBalancer, ExternalDNS 에 대한 권한을 부여하겠습니다.


```bash
aws iam attach-role-policy --policy-arn arn:aws:iam::$ACCOUNT_ID:policy/AWSLoadBalancerControllerIAMPolicy --role-name masters.$KOPS_CLUSTER_NAME
aws iam attach-role-policy --policy-arn arn:aws:iam::$ACCOUNT_ID:policy/AWSLoadBalancerControllerIAMPolicy --role-name nodes.$KOPS_CLUSTER_NAME
aws iam attach-role-policy --policy-arn arn:aws:iam::$ACCOUNT_ID:policy/AllowExternalDNSUpdates --role-name masters.$KOPS_CLUSTER_NAME
aws iam attach-role-policy --policy-arn arn:aws:iam::$ACCOUNT_ID:policy/AllowExternalDNSUpdates --role-name nodes.$KOPS_CLUSTER_NAME
```
다음 kops cluster 를 업데이트 해주도록하겠습니다.
```bash
kops edit cluster
-----
spec:
  certManager:
    enabled: true
  awsLoadBalancerController:
    enabled: true
  externalDns:
    provider: external-dns
-----

kops update cluster --yes && echo && sleep 3 && kops rolling-update cluster
```

다음은 이미지 저장소인 Harbor를 설치해주도록 하겠습니다.
Harbor 설치는 Helm Chart를 통해 설치를 진행하도록하겠습니다.

```bash
helm repo add harbor https://helm.goharbor.io
helm fetch harbor/harbor --untar
```

다음 value.yaml에 대한 수정이 필요합니다.
```bash
vim ~/harbor/values.yaml

expose.tls.certSource=none  # 19줄
expose.ingress.hosts.core=harbor.<각자자신의도메인>    # 36줄
expose.ingress.hosts.notary=notary.<각자자신의도메인>  # 37줄
expose.ingress.hosts.core=harbor.gasida.link    
expose.ingress.hosts.notary=notary.gasida.link  
expose.ingress.controller=alb                      # 44줄
expose.ingress.className=alb                       # 47줄
expose.ingress.annotations=alb.ingress.kubernetes.io/scheme: internet-facing
expose.ingress.annotations=alb.ingress.kubernetes.io/target-type: ip
expose.ingress.annotations=alb.ingress.kubernetes.io/listen-ports: '[{"HTTPS":443}, {"HTTP":80}]'
expose.ingress.annotations=alb.ingress.kubernetes.io/certificate-arn: ${CERT_ARN}
externalURL=https://harbor.<각자자신의도메인>
externalURL=https://harbor.gasida.link             # 131줄
```
위의 yaml 파일 내용수정 중 
51~54번째줄에 해당하는 내용은 주석처리를 해주시면 됩니다.
추가적으로 expore.ingress.annotations=alb.ingress.kubernetes.io/certificate-arn: ${CERT_ARN} 부분은
문자 그대로 넣어주시는것이 아니라
터미널에서 echo "alb.ingress.kubernetes.io/certificate-arn: $CERT_ARN" 명령어를 입력하여 나온 값을
${CERT_ARN} 값 대신에 넣어줄수 있도록 합니다.
단순히 ${CERN_ARN} 값을 넣으면 인증키 값이 꼬일수도 있기 떄문입니다.

![harbor_yaml.png](harbor_yaml.png)

수정이 완료 되었다면
아래의 명령어를 입력하여 helm install을 진행해주도록 하겠습니다.

```bash
kubectl create ns harbor
helm install harbor harbor/harbor -f ~/harbor/values.yaml --namespace harbor --version 1.11.0
```
헬름 인스톨 이후 
harbor.자신의도메인 에 접속하시면 Harbor에 접속하실 수 있습니다.
이 작업또한 시간이 조금 필요하기에 느긋하게 5분정도 후에 접속해보시면 되겠습니다!

![route53.png](route53.png)
![LB.png](LB.png)


만약 접속이 잘 안되실 경우 IAM에서 역할 할당이 잘 되어있는지,
Route53에 harbor.도메인이 잘 등록되어 있는지,
LB는 잘 생성되어 있는지를 확인해주시면 되겠습니다.

정상적으로 접속이 되었습니다.

![harbor_conn.png](harbor_conn.png)

ID PW 는 기본
ID : admin
PW : Harbor12345로 설정이 되어있습니다.

접속이 되었다면 신규 프로젝트를 생성해주도록하겠습니다.

![harbor_project.png](harbor_project.png)

이미지 레지스트리가 잘 생성 되었으므로 한번 이미지저장소에 이미지를 Push해보도록 하겠습니다.
아래 명령어를 통해 임의의 이미지를 내려받고 태그를 달아주도록 하겠습니다.
```bash
docker pull nginx
docker pull busybox
docker tag busybox harbor.$KOPS_CLUSTER_NAME/pkos/busybox:0.1
```

아래 명령어는 insecure 설정 명령어입니다. 진행해주도록 하겠습니다. 진행 후 daemor reload 및 docker 재시작을 하겠습니다.
```bash
cat <<EOT> /etc/docker/daemon.json
{
    "insecure-registries" : ["harbor.$KOPS_CLUSTER_NAME"]
}
EOT

systemctl daemon-reload && systemctl restart docker
```

다음 아래 명령어를 통하여 Harbor에 로그인하고 이미지를 Push하겠습니다.
```bash
docker login harbor.$KOPS_CLUSTER_NAME -u admin -p Harbor12345
docker push harbor.$KOPS_CLUSTER_NAME/pkos/busybox:0.1
```

명령어 수행 후 Harbor에서 Image가 정상적으로 Push 된것을 확인하실 수 있습니다.

![image_push.png](image_push.png)

다음은 YAML 파일의 컨테이너 이미지 저장소 주소를 Local Harbor로 변경해보겠습니다.

```bash
curl -s -O https://raw.githubusercontent.com/junghoon2/kube-books/main/ch13/busybox-deploy.yml
sed -i "s|harbor.myweb.io/erp|harbor.$KOPS_CLUSTER_NAME/pkos|g" busybox-deploy.yml
```

위의 명령어의 yaml 파일을 조회하시면 image가 저희가 방금 저장하였던 local Harbor의 이미지로 지정이 되어있습니다.
kubectl apply -f busybox-deply.yml 을 입력하시면
해당 이미지가 Pulling 되어 Pod가 기동될것입니다.

##3. GitLab 을 이용한 Local Git 소스 저장소 구축

Gitops 실습에 앞서 소스 저장소 구축을 위해 GitLab 이용해 Local Git 저장소를 구축해보겠습니다.
참고로 GitLab은 최소 요구사항이 제법 높은편입니다.(https://docs.gitlab.com/ee/install/requirements.html)
그래서 실습할때 높은 사양의 EC2를 생성해주는것을 권장드립니다.
아래의 명령어를 순차적으로 입력해주겠습니다.
```bash
kubectl create ns gitlab
helm repo add gitlab https://charts.gitlab.io/
helm repo update
helm fetch gitlab/gitlab --untar
```



다음 yaml 파일을 아래와 같이 수정해주시면 되겠습니다.
```bash
vim ~/gitlab/values.yaml
----------------------
global:
  hosts:
    domain: <각자자신의도메인>             # 52줄
    https: true

  ingress:                  # 66줄
    configureCertmanager: false
    provider: aws
    class: alb
    annotations:
      alb.ingress.kubernetes.io/scheme: internet-facing
      alb.ingress.kubernetes.io/target-type: ip
      alb.ingress.kubernetes.io/listen-ports: '[{"HTTPS":443}, {"HTTP":80}]'
      alb.ingress.kubernetes.io/certificate-arn: ${CERT_ARN}
      alb.ingress.kubernetes.io/success-codes: 200-399
    tls:
      enabled: false

certmanager:           # 834줄
  installCRDs: false
  install: false
  rbac:
    create: false

nginx-ingress:
  enabled: false

prometheus:    #904줄
  install: false

gitlab-runner:       #1129줄
  install: false
----------------------

helm install gitlab gitlab/gitlab -f ~/gitlab/values.yaml --namespace gitlab --version 6.8.1
```

수정 참고용 이미지 첨부하겠습니다.
![GitLab.png](GitLab.png)

헬름 인스톨이 완료되었다면 아래 명령어를 통해 Gitlab 접속 비밀번호를 출력하실 수 있습니다.
```bash
kubectl get secrets -n gitlab gitlab-gitlab-initial-root-password --template={{.data.password}} | base64 -d ;echo
```

다음 gitlab.자신의 도메인으로 접속하시고
ID : root
PW : 위의 명령어의 출력값
을 입력하셔서 접속테스트를 해보시면 되겠습니다.

![gitlab_login.png](gitlab_login.png)

Gitlab에 접속하신 후 신규 유저를 생성하겠습니다.
권한은 Admin으로 설정해주시면 되겠습니다.

계정 생성 후 상단의 Edit 버튼을 눌러 비밀번호를 설정 해주도록 합니다.

![gitlab_passwd.png](gitlab_passwd.png)

비밀번호 설정후 Impersonation Token 을 클릭하고 표시된 모든 scopes를 선택하여 Token을 생성해주도록 하겠습니다.

![gitlab_token.png](gitlab_token.png)














아래 명령어를 통해 인스토어 스토어 볼륨이 있는 c5모든 타입의 스토리지 크기를 조회하실 수 있습니다.
```bash
aws ec2 describe-instance-types \ --filters "Name=instance-type,Values=c5*" "Name=instance-storage-supported,Values=true" \ --query "InstanceTypes[].[InstanceType, InstanceStorageInfo.TotalSizeInGB]" \ --output table
```

저희가 배포한 c5d.large는 50Gib 만큼의 크기를 제공하고 있습니다.

먼저 각 워커노드의 스토리지를 확인해 보도록 하겠습니다.
```bash
ssh -i ~/.ssh/id_rsa ubuntu@$W1PIP sudo apt install -y nvme-cli
ssh -i ~/.ssh/id_rsa ubuntu@$W2PIP sudo apt install -y nvme-cli
ssh -i ~/.ssh/id_rsa ubuntu@$W1PIP sudo nvme list
ssh -i ~/.ssh/id_rsa ubuntu@$W2PIP sudo nvme list
```

nvm list를 통해서 보면 2개의 스토리지를 확인하실 수 있습니다.

![nvm_list.png](nvm_list.png)

하지만 AWS Management Console를 통해서 보면 EC2에 할당된 디스크중에 50Gib짜리 디스크는 찾아보실 수 없습니다.

![EC2_Storage.png](EC2_Storage.png)

인스턴스 스토어에 대한 정보는 스토리지 정보에 출력이 되지 않는다는 점을 확인하실 수 있습니다.

아래의 명령어를 통해 파일시스템 생성 및 /data를 마운트 시킬 수 있습니다.
```bash
ssh -i ~/.ssh/id_rsa ubuntu@$W1PIP sudo mkfs -t xfs /dev/nvme1n1
ssh -i ~/.ssh/id_rsa ubuntu@$W2PIP sudo mkfs -t xfs /dev/nvme1n1
ssh -i ~/.ssh/id_rsa ubuntu@$W1PIP sudo mkdir /data
ssh -i ~/.ssh/id_rsa ubuntu@$W2PIP sudo mkdir /data
ssh -i ~/.ssh/id_rsa ubuntu@$W1PIP sudo mount /dev/nvme1n1 /data
ssh -i ~/.ssh/id_rsa ubuntu@$W2PIP sudo mount /dev/nvme1n1 /data
ssh -i ~/.ssh/id_rsa ubuntu@$W1PIP df -hT -t ext4 -t xfs
ssh -i ~/.ssh/id_rsa ubuntu@$W2PIP df -hT -t ext4 -t xfs
```

## 2.Ingress

지난주에 진행했던 실습과 비슷한 실습입니다.
각 EC2에 LB생성 권한을 부여해 LB를 생성하고 여기에 배포를 하여 외부에 오픈해주는 작업을 진행해 보겠습니다.

각 컨트롤플레인, 워커노드에 LB 생성 권한을 부여하겠습니다.
```bash
aws iam attach-role-policy --policy-arn arn:aws:iam::$ACCOUNT_ID:policy/AWSLoadBalancerControllerIAMPolicy --role-name nodes.$KOPS_CLUSTER_NAME
aws iam attach-role-policy --policy-arn arn:aws:iam::$ACCOUNT_ID:policy/AWSLoadBalancerControllerIAMPolicy --role-name nodes.$KOPS_CLUSTER_NAME
```


아래 명령어를 통해 IAM 권한또한 부여하겠습니다.
```bash
aws iam attach-role-policy --policy-arn arn:aws:iam::$ACCOUNT_ID:policy/AllowExternalDNSUpdates --role-name masters.$KOPS_CLUSTER_NAME
aws iam attach-role-policy --policy-arn arn:aws:iam::$ACCOUNT_ID:policy/AllowExternalDNSUpdates --role-name nodes.$KOPS_CLUSTER_NAME ```\n
```
 

다음 kops cluster edit을 통해

spec.cerManager.enable:ture
spec.LoadBalancerController.enable:true
spec.externalDns.provider:external-dns


를 추가해준 후 업데이트를 진행해주도록 하겠습니다.

```bash
kops update cluster --yes && echo && sleep 3 && kops rolling-update cluster
```


실습을 진행하기 전 항상 IAM에 해당 권한이 정상적으로 부여가 되었는지 확인 후 진행하기를 권장드립니다.
권한이 부여되어있지 않다면 LB가 생성되지 않습니다..ㅜㅜ
![IAM_Check.png](IAM_Check.png)

IAM 권한을 확인하였다면 POD를 배포해보도록 하겠습니다.
아래 명령어를 통해 POD를 배포해보도록 하겠습니다.

```bash
kubectl apply -f ~/pkos/3/ingress1.yaml
```

배포 후 AWS Management Console 에서 LB가 생성된것을 확인하실 수 있습니다.

![Load_Balancer.png](Load_Balancer.png)

아래 명렁어를 통해 Ingress를 확인하실 수 있습니다.

```bash
kubectl describe ingress -n game-2048 ingress-2048
```

아래 명렁어를 통해 외부에서 접속 가능한 게임의 URL 주소가 생성이 됩니다. 해당 URL에 접속해보도록 하겠습니다.

```bash
kubectl get ingress -n game-2048 ingress-2048 -o jsonpath={.status.loadBalancer.ingress[0].hostname} | awk '{ print "Game URL = http://"$1 }'
```

정상적으로 접속 됩니다!

![game_url.png](game_url.png)

명령어를 통해 POD의 IP를 조회해보겠습니다.
```bash
kubectl get pod -n game-2048 -owide
```

![LB_BP.png](LB_BP.png)


위의 POD의 IP가 LB의 백엔드풀에 정상적으로 등록된것을 확인하실 수 있습니다.

![LB_BP2.png](LB_BP2.png)

만약 해당 POD의 갯수를 늘려준다면 백엔드풀에 등록된 IP또한 계속해서 늘어나게 됩니다.

이후 아래 명령어를 통해 실습 리소스를 제거해주도록 합니다.

```bash
kubectl delete ingress ingress-2048 -n game-2048
kubectl delete svc service-2048 -n game-2048 && kubectl delete deploy deployment-2048 -n game-2048 && kubectl delete ns game-2048
```


다음 실습으로 넘어가겠습니다.

아래의 명렁어를 입력하여 본인이 지정한 도메인에 pod를 배포해보도록 하겠습니다.

```bash
WEBDOMAIN=albweb.korwoo.net
WEBDOMAIN=$WEBDOMAIN envsubst < ~/pkos/3/ingress2.yaml | kubectl apply -f -
```


AWS Management Console에서 본인이 지정한 도메인이 등록되어있는것을 확인하실 수 있습니다.


![route53.png](route53.png)

저는 제 모바일기기를 통하여 해당 도메인에 접속을하였고 접속이 성공적으로 되었습니다.

![ExternalDomain.jpg](ExternalDomain.jpg)

과제1. Ingress를 이용하여 /mario에는 mario 게임 접속, /tetris에는 tetris게임 접속되게 설정 및 SSL 적용 하기


![mario.png](mario.png)
![tetris.png](tetris.png)
![route53_2.png](route53_2.png)

현재 화면에서 보시면 mario는 SSL이 적용되어 있으나, tetris는 SSL 이 적용되어 있지 않습니다..
현재 원인 분석 중 입니다..ㅜㅜ


## 3. K8S 스토리지

K8S 스토리지에서 가장 중요한 부분을 뽑자면 파드 내부의 데이터는 파드가 정지되면 모두 제거가 된다는 점 입니다.
따라서 보존이 필요한 중요한 데이터가 있다면 PV같은 Stateful 애플리케이션으로 데이터를 보존해야된다는 점을 기억해주시면 되겠습니다.

간단한 실습으로 데이터의 보존 여부를 체크해보겠습니다.
아래의 명령어를 통해 10초마다 1번씩 데이터를 기록하는 POD를 배포해보겠습니다.

```bash
kubectl apply -f ~/pkos/3/date-busybox-pod.yaml
```

해당 POD를 배포하면 주기적으로 데이터를 기록하게 되는데

```bash
kubectl delete pod busybox
kubectl apply -f ~/pkos/3/date-busybox-pod.yaml
```


위의 명령어를 통해서 POD를 제거하였다가 다시 생성하였을 시 이전에 기록한 데이터가 남아있는지 확인합니다.

당연하게도 이전의 데이터는 모두 제거가 되게 됩니다.

과제2. HostPath 실습 및 문제점 확인과 성능 측정

먼저 HostPath를 사용하는 PV/PVC 스토리지 클래스를 배포하도록 하겠습니다!
local path 정의 파일 다운로드

```bash
curl -s -O https://raw.githubusercontent.com/rancher/local-path-provisioner/v0.0.23/deploy/local-path-storage.yaml
```

vim local-path-storage.yaml 을 통해 각자 Control Plane의 이름 입력해줍니다.


![InputMaster.png](InputMaster.png)


아래 명령어를 입력해서 배포해보도록 하겠습니다.

```bash
kubectl apply -f local-path-storage.yaml
```


다음으로 PV/PVC를 사용하는 POD를 생성해보도록 하겠습니다.

```bash
kubectl apply -f ~/pkos/3/localpath1.yaml
kubectl get pvc
kubectl describe pvc
```

를 입력해서 PVC를 확인해보도록 하겠습니다.

![check_pvc.png](check_pvc.png)

위에서 진행하였던 실습과 똑같이 10초마다 1번씩 데이터를 입력하는 POD를 배포해보도록 하겠습니다.

```bash
kubectl apply -f ~/pkos/3/localpath2.yaml
```

각 워커노드에 Tool을 설치해주도록 하겠습니다.

```bash
ssh -i ~/.ssh/id_rsa ubuntu@$W1PIP sudo apt install -y tree jq sysstat
ssh -i ~/.ssh/id_rsa ubuntu@$W2PIP sudo apt install -y tree jq sysstat
```


배포된 POD에는 out.txt라는 파일이 있을것입니다. 아래 명령어로 확인 가능합니다.

```bash
ssh -i ~/.ssh/id_rsa ubuntu@$W1PIP tree /data
```

![out.png](out.png)

이 파일은 10초에 1번씩 입력되는 데이터를 저장하는 경로 입니다.
이제 POD를 제거하면 과연 데이터가 살아있을지 확인해보도록 하겠습니다.

```bash
kubectl delete pod app
kubectl get pod,pv,pvc
```



![data_alive.png](data_alive.png)

POD가 죽었음에도 PV가 살아있으니 데이터또한 같이 살아 있습니다!

다음으로 kubestr & sar을 활용하여 모니터링 및 성능측정 확인을 진행해보도록 하겠습니다.

아래 명령어로 kubestr를 다운로드 받아주도록 하겠습니다.

```bash
wget https://github.com/kastenhq/kubestr/releases/download/v0.4.36/kubestr_0.4.36_Linux_amd64.tar.gz
tar xvfz kubestr_0.4.36_Linux_amd64.tar.gz && mv kubestr /usr/local/bin/ && chmod +x /usr/local/bin/kubestr
```



아래 명령어로 모니터링을 걸어주도록 하겠습니다.

```bash
ssh -i ~/.ssh/id_rsa ubuntu@$W1PIP iostat -xmdz 1 -p nvme1n1
ssh -i ~/.ssh/id_rsa ubuntu@$W2PIP iostat -xmdz 1 -p nvme1n1
```



모니터링에서 표시되는 정보 관련 내용입니다.
rrqm/s : 초당 드라이버 요청 대기열에 들어가 병합된 읽기 요청 횟수
wrqm/s : 초당 드라이버 요청 대기열에 들어가 병합된 쓰기 요청 횟수
r/s : 초당 디스크 장치에 요청한 읽기 요청 횟수
w/s : 초당 디스크 장치에 요청한 쓰기 요청 횟수
rMB/s : 초당 디스크 장치에서 읽은 메가바이트 수
wMB/s : 초당 디스크 장치에 쓴 메가바이트 수
await : 가장 중요한 지표, 평균 응답 시간. 드라이버 요청 대기열에서 기다린 시간과 장치의 I/O 응답시간을 모두 포함 (단위: ms)

아래 명령어를 입력하여 측정을 시작하겠습니다.(5~10분정도 소요됩니다.)
```bash
curl -s -O https://raw.githubusercontent.com/wikibook/kubepractice/main/ch10/fio-read.fio
kubestr fio -f fio-read.fio -s local-path --size 10G
```


![kubestr.png](kubestr.png)


```bash
curl -s -O https://raw.githubusercontent.com/wikibook/kubepractice/main/ch10/fio-write.fio
kubestr fio -f fio-write.fio -s local-path --size 10G
```

![kubestr2.png](kubestr2.png)


과제3. AWS EBS를 PVC로 사용 후 온라인 볼륨 증가 해보기

아래 명령어를 입력하여 aws-ebs-csi를 확인해봅니다. 미리 설치되어 있다고합니다.(친절한 가시다님ㅜㅜ)
```bash
kubectl get pod -n kube-system -l app.kubernetes.io/instance=aws-ebs-csi-driver
```

스토리지 클래스 확인

```bash
kubectl get sc kops-csi-1-21 kops-ssd-1-17
kubectl describe sc kops-csi-1-21 | grep Parameters
kubectl describe sc kops-ssd-1-17 | grep Parameters
```


아래 명령어를 입력해 워커노드의 EBS 볼륨을 확인해봅시다.
```bash
aws ec2 describe-volumes --filters Name=tag:k8s.io/role/node,Values=1 --output table
```

![image_volume.png](image_volume.png)

아래 명령어는 Pod에 추가한 EBS 볼륨을 확인하는 명령어 입니다.
```bash
aws ec2 describe-volumes --filters Name=tag:ebs.csi.aws.com/cluster,Values=true --output table
```


![ebs_volume.png](ebs_volume.png)

아래 명령어로 PVC,POD를 생성해주도록 하겠습니다.

```bash
kubectl apply -f ~/pkos/3/awsebs-pvc.yaml
kubectl apply -f ~/pkos/3/awsebs-pod.yaml
```


다음 아래 명령어를 통해 볼륨 정보를 확인하실 수 있습니다.
```bash
kubectl exec -it app -- sh -c 'df -hT --type=ext4'
```

![volume_check.png](volume_check.png)

또한 AWS Management Console -> EC2 -> 볼륨으로 들어가면 4Gi의 볼륨이 생성된것을 확인하실 수 있습니다.

![console_ebs.png](console_ebs.png)

아래의 명령어를 통해 4Gi -> 10Gi 로 용량을 증가시켜 준다.
```bash
kubectl patch pvc ebs-claim -p '{"spec":{"resources":{"requests":{"storage":"10Gi"}}}}'
```

용량이 증가된것을 확인하실 수 있습니다.

![console_ebs2.png](console_ebs2.png)

혹은 아래 명렁어를 통해서도 확인하실 수 있습니다!

```bash
kubectl exec -it app -- sh -c 'df -hT --type=ext4'
```


과제4. AWS Volume Snapshot 실습

kops edit 을 통해 아래 표시된 부분을 추가해준후 업데이트 해주도록 하겠습니다.

![snapshotcontroller.png](snapshotcontroller.png)

아래 명령어를 통해 업데이트가 되었는지 확인해보겠습니다.

```bash
kubectl get crd | grep volumesnapshot
```

![volume_snapshot.png](volume_snapshot.png)

아래 명령어를 통해 vsclass 생성해주도록 하겠습니다.

```bash
kubectl apply -f https://raw.githubusercontent.com/kubernetes-sigs/aws-ebs-csi-driver/master/examples/kubernetes/snapshot/manifests/classes/snapshotclass.yaml
```

PVC, POD 생성 명령어
```bash
kubectl apply -f ~/pkos/3/awsebs-pvc.yaml
kubectl apply -f ~/pkos/3/awsebs-pod.yaml
```


아래 명령어를 통해 볼륨 스냅샷을 생성해주도록 하겠습니다.
```bash
kubectl apply -f ~/pkos/3/ebs-volume-snapshot.yaml
```


볼륨 스냅샷을 확인하는 명령어 입니다.
```bash
kubectl get volumesnapshot
```


![volume_snapshot_2.png](volume_snapshot_2.png)

완료가 되었다면 복원을 해보겠습니다.
임의로 app & pvc를 제거하여 이슈를 만들어보겠습니다.
```bash
kubectl delete pod app && kubectl delete pvc ebs-claim
```


이후 스냅샷에서 PVC로 복원을 해보겠습니다.

```bash
kubectl apply -f ~/pkos/3/ebs-snapshot-restored-claim.yaml
kubectl apply -f ~/pkos/3/ebs-snapshot-restored-pod.yaml
```


![snapshot_recovery.png](snapshot_recovery.png)

완료되어 이전에 입력되었던 데이터가 그대로 살아있는것을 확인하실 수 있습니다.




```toc

```


















```toc

```
