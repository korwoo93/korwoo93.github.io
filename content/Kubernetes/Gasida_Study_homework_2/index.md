---
emoji: 🔮
title: Gasida Kubernetes Study 2주차
date: '2022-08-29 22:00:00'
author: KORWOO
tags: Kubernetes
categories: Kubernetes
---


가시다님 스터디 : https://www.notion.so/gasidaseo/23-7635cc4f02c04954a3260b317588113e

가시다님의 쿠버네티스 스터디 2주차 쿠버네티스 네트워크 입니다.
이번주차는 저를 포함한 이번 스터디를 수행하는 모든 문들이 어려움을 많이 느꼈던것으로 알고 있습니다.
저역시 쉽지 않아서 원래대로 라면 지난주에 포스팅해야했을것을 과제만 우선 제출하고 현재 포스팅을 하고 있습니다.ㅜㅜ



## 1. KOPS 원클릭 배포
지난번 1주차에서 스터디장님인 가시다님이 제공해주신 CloudFormation을 통해 EC2등을 배포하고
거기서 aws configure 등의 작업을 진행해 주었습니다.
하지만 가시다님의 노력으로 인하여 저희는 몇줄의 명령어 만으로 쉽게 배포할 수 있게 되었습니다.

kops 원클릭 배포

원클릭 배포를 위해서는 먼저 SSH 키페어, IAM 계정 키, S3 버킷이 필요합니다.

위에서 말씀드린 것들은 1주차에서 생성하였으므로 이번 포스팅에서는 생략하겠습니다.

먼저 cloudformation에서 사용할 yaml 파일이 필요합니다.
``` curl -O https://s3.ap-northeast-2.amazonaws.com/cloudformation.cloudneta.net/K8S/kops-oneclick.yaml ```

해당 파일을 다운받았다면 아래 명렁어를 통하여 배포해줍니다.

``` aws cloudformation deploy --template-file kops-oneclick.yaml --stack-name mykops --parameter-overrides KeyName=본인의 키페어 이름 SgIngressSshCidr=$(curl -s ipinfo.io/ip)/32  MyIamUserAccessKeyID=액세스키 MyIamUserSecretAccessKey='시크릿키' ClusterBaseName='korwoo.net' S3StateStore='hellokorwoo' MasterNodeInstanceType=c5d.large WorkerNodeInstanceType=c5d.large --region ap-northeast-2```

![OneClick_Deploy.png](OneClick_Deploy.png)

5~10분 정도의 시간이 지나면 위에 화면과 같이 컨트롤플레인 및 워커노드가 모두 생성이 되어있는것을 확인하실 수 있습니다.

다시한번 가시다님의 노고에 감사드립니다ㅜㅜ



## 2. AWS VPC CNI

`C`ontainer `N`etwork `I`nterface

CNI는 공식 Github(https://github.com/containernetworking/cni) 에 들어가서 내용을 확인하실 수 있습니다.
간단하게 말하자면 컨테이너 간 네트워킹을 제어할 수 있는 플러그인을 만들기 위한 도구 정도라고 정리가 됩니다.

지원되는 플러그인은 (https://kubernetes.io/docs/concepts/cluster-administration/addons/#networking-and-network-policy) 에서 확인 가능합니다.

가시다님의 스터디에서 소개되는 내용은 AWS VPC CNI에 대해서 소개가 되었습니다.

AWS VPC CNI의 강조되는 특징중 하나는 POD와 NODE의 네트워크 대역이 같다는 점 입니다.

![Network_Diff](Network_Diff.png)

같은 네트워크 대역대를 사용함으로서 노드와 파드간의 소통이 좀더 원활하다는 장점을 가지게 됩니다.

간단하게 내용들을 확인해보겠습니다.

```kubectl describe daemonset aws-node --namespace kube-system | grep Image | cut -d "/" -f 2```
명령어를 입력하게 되면

![CNI_Info](CNI_Info.png)

다음과 같이 CNI 정보를 확인하실 수 있습니다.


```  aws ec2 describe-instances --query "Reservations[*].Instances[*].{PublicIPAdd:PublicIpAddress,PrivateIPAdd:PrivateIpAddress,InstanceName:Tags[?Key=='Name']|[0].Value,Status:State.Name}" --filters Name=instance-state-name,Values=running --output table ```

명령어를 입력하면

![Node_IP.png](Node_IP.png)

다음과 같이 노드의 IP를 확인할 수 있습니다.

``` kubectl get pod -n kube-system -o=custom-columns=NAME:.metadata.name,IP:.status.podIP,STATUS:.status.phase ```

명령어를 입력하여 Pod의 IP를 확인해보도록합시다.

![POD_IP.png](POD_IP.png)

위에서 확인했던 노드의 Private IP 대역대와 파드의 IP 대역대가 일치하는것을 확인하실 수 있습니다.

##3. 노드 간 파드 통신

![Node&Pod](Node&Pod.png)

![Pod_Commu](Pod_Commu.png)

파드간 통신 흐름 : 별도의 오버레이 통신 기술 없이, VPC Native 하게 파드간 직접 통신이 가능하다.



``` kubectl apply -f ~/pkos/2/netshoot-2pods.yaml ```

먼저 위 명령어를 통해 테스트용 POD를 배포해주도록 하겠습니다.

![get_pod.png](get_pod.png)

배포한 POD를 보면 각각 다른 Node에 배포가 되어 있는것을 확인할 수 있습니다.

이때 이 POD 끼리의 통신을 테스트해보도록 하겠습니다.

먼저 각 워커 노드에 접속하여
``` ssh -i ~/.ssh/id_rsa ubuntu@$W1PIP ```
``` ssh -i ~/.ssh/id_rsa ubuntu@$W2PIP ```

아래 명령어를 통하여 패킷 덤프를 확인하는 명령어를 실행시켜 두도록 하겠습니다.
``` sudo tcpdump -i any -nn icmp ``` 
``` sudo tcpdump -i ens5 -nn icmp ```
``` sudo tcpdump -i ens6 -nn icmp ```

``` POD1=$(kubectl get pod pod-1 -o jsonpath={.status.podIP}) ```
``` POD2=$(kubectl get pod pod-2 -o jsonpath={.status.podIP}) ```
``` kubectl exec -it pod-1 -- ping -c 2 $POD2 ```
pod 1 shell 에서 pod2 로 ping 테스트
``` kubectl exec -it pod-2 -- ping -c 2 $POD1 ```
pod 2 shell 에서 pod1 로 ping 테스트

먼저 any 에서의 패킷 덤프는 확인이 되었습니다.
![tcpdump_any.png](tcpdump_any.png)

다음 ens5에서의 패킷 덤프또한 확인이 되었습니다.
![tcpdump_ens5](tcpdump_ens5.png)

하지만 ens6에서는 패킷 덤프가 확인이 되지 않습니다.
pod간의 통신은 무조건 첫번째 ENI로만 빠져나가게끔 설정이 되어 있어서 ens6 로의 패킷 덤프는 확인이 되지 않는것입니다.

![ENI.png](ENI.png)




## 4. Pod에서 외부 통신

![Pod_ExtCommu](Pod_ExtComm.png)

파드에서 외부 통신 흐름 : iptable에 SNAT을 통하여 노드의 eth0 IP로 변경되어 외부와 통신합니다.

VPC CNI의 External source network address translation 설정에 따라, 외부 통신 시 SNAT 하거나 혹은 SNAT 없이 통신 가능합니다.


작업용 EC2에서 pod-1 shell에서 외부로 ping을 날려보겠습니다.
``` kubectl exec -it pod-1 -- ping -c 1 www.google.com ```

다음 Pod-1에 접속해서 tcpdump를 해주겠습니다.
``` sudo tcpdump -i any -nn icmp ```







```toc

```


















```toc

```
