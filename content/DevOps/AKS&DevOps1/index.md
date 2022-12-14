---
emoji: π§’
title: Azure Kubernetes Service μμ± & Test
date: '2022-08-28 21:00:00'
author: KORWOO
tags: AKS
categories: Devops
---

## π 1. AKS μκ°

AKS(AzureK Kubernetes Service)λ Azure μμ μ κ³΅νλ μ»¨νμ΄λ κΈ°λ° μΈνλΌ μ΄μ μλΉμ€μλλ€. Master μλ²λ₯Ό Azure μμ κ΄λ¦¬ν΄ μ£ΌκΈ°μ
μΈνλΌ κ΄λ¦¬ λΆλ΄μ μ€μ΄λ€κ³  Azure λ¦¬μμ€μμ λ°μ΄λ νΈνμ±μ κ°μ§κ³  μμ΅λλ€. λν Azure CNI(Container Network Interface) λ₯Ό νμ©νμ¬
AKSμμ μμ±λ Podμ Azure VNet μμ λ¦¬μμ€λ₯Ό μ§μ  ν΅μ μν¬ μ μμ΅λλ€.  π



> νΉμ λ§λμλ κ³Όμ μμ κΆκΈνμ  μ μ΄λ μ΄λ €μμ΄ μμΌμλ€λ©΄ [μ΄μ](https://github.com/zoomKoding/zoomkoding-gatsby-blog/issues/new)λ₯Ό ν΅ν΄ λ¬Έμ λ¨κ²¨μ£ΌμΈμ!  
> [μ€ν](https://github.com/zoomKoding/zoomkoding.com)λ λΈλ‘κ·Έ νλ§λ₯Ό μ§μμ μΌλ‘ λ°μ μν€λλ° ν° νμ΄ λ©λλ€!β­οΈ

## π μμνκΈ°

1.  Azure Portalμ λ‘κ·ΈμΈνμ¬ μλ¨μ κ²μλ°μμ Kubernetes & AKS λ₯Ό κ²μν©λλ€.
![AKS_1.png](AKS_1.png) 

2.  μλ¨ +λ§λ€κΈ° μμ Kubernetes ν΄λ¬μ€νΈ λ§λ€κΈ° λ₯Ό ν΄λ¦­ν©λλ€.
![AKS_2.png](AKS_2.png)

3.  AKS μμ± μμΈ λ©λ΄ μλλ€. λ¨Όμ  AKSλ₯Ό λ°°ν¬ν  λ¦¬μμ€ κ·Έλ£Ήμ μμ±ν΄μ€λλ€. λ¦¬μμ€ κ·Έλ£Ή λΆλΆμ μλ‘ λ§λ€κΈ°λ₯Ό ν΄λ¦­νμ¬ λ¦¬μμ€ κ·Έλ£Ή μ΄λ¦μ μ§μ ν΄μ€λλ€. λ¦¬μμ€ κ·Έλ£Ή μ΄λ¦μ κ·μΉμ± μκ² μ§μ ν΄μ£Όλ νΈμ΄ λμ€μ νκ°λ¦¬μ§ μμ΅λλ€.

![AKS_3.png](AKS_3.png)

ν΄λ¬μ€ν° μ¬μ  μ€μ  κ΅¬μ±μ AKSμ λͺ©μ μ λ°λΌ νμ€, κ°λ°/νμ€νΈ, λΉμ© μ΅μ ν, μΌκ΄ μ²λ¦¬, κ°νλ μ‘μΈμ€ λ‘ λλμ΄ μ§λλ€. 
μΌλ°μ μΈ μ€μ΅μ΄ λͺ©μ μ΄λΌλ©΄ νμ€ & κ°λ°/νμ€νΈ λ‘ μ νν΄μ€λλ€. λ³Έ μ€μ΅μμλ νμ€μΌλ‘ μ ννκ³  μ§ννκ² μ΅λλ€.

Kuberbetes Cluster μ μ΄λ¦μ κ΅¬λ λ΄μμ μ μΌν μ΄λ¦μΌλ‘ μ§μ ν΄μ€λλ€.

μ§μ­μ μ¬μ©μκ° μμΉν μ§μ­μ μλ§κ² μ€μ ν΄μ€λλ€.

κ°μ©μ± μμ­μ κΈ°λ³Έ 3κ°λ‘ μ§μ λμ΄μμ΅λλ€.

μ¬κΈ°μ κ°μ©μ μμ­μ΄λ μλ₯Όλ€μ΄ νμ¬ AKSλ₯Ό λ§λ€κ³ μ νλ νκ΅­ μ€λΆ λ¦¬μ μ μ§μ§,νμ λ± λ°μ΄ν° μΌν°μ λ¬Έμ κ° μκ²Όμ μμ λλΉνμ¬

λ¬Όλ¦¬μ μΌλ‘ κ΅¬λΆλμ΄ μ λ ₯μ μΌλ‘ λ°°μΉλ λ°μ΄ν° μ ν°λ₯Ό μ΅μ 3κ° μ΄μ μ°κ²°νλ μ§μ­ λ€νΈμν¬ μ€κ³λ₯Ό ν΅ν΄ λ¨μΌ μ€ν¨ μ§μ μ μ κ±°νλ λ°©μ μλλ€.

λΈλ ν¬κΈ°λ AKSλ₯Ό μν΄ μ‘΄μ¬νλ VMSS(Virtual Machine Scale Set) μ μ±λ₯(SKU)μ μλ―Έν©λλ€. μ΄ λν κΈ°λ³Έκ°μΌλ‘ μ§μ λ DS2 v2λ‘ μ§μ ν©λλ€.

4.  μΈμ¦ λ° κΆν λΆμ¬ λΆλΆμ κΈ°λ³Έμ μΌλ‘ μ€μ λμ΄ μλ Kubernetes RBACκ° μλ λ‘μ»¬ κ³μ μΌλ‘ μ νν©λλ€. μ΄κ²μ΄ μλ―Ένλ λ°λ 

![AKS_7.png](AKS_7.png)

Azure μμ μ κ³΅λλ λ¦¬μμ€μλ μ‘μΈμ€ μ μ΄(IAM) ν­μ΄ μμ΅λλ€. μ΄ IAMμ λͺμλμ΄ μλ μ¬μ©μλ€μκ² νν΄ μ κ·Ό κΆνμ λΆμ¬νλ€λ λ» μλλ€.

5.  λ€νΈμνΉ μ€μ  ν­ μλλ€. 

λ€νΈμν¬ κ΅¬μ±μ kubenet κ³Ό Azure CNI λ‘ λλμ΄ μ§λλ€. Kubenetμ κΈ°λ³Έ λ€νΈμνΉ Azure CNIλ κ³ κΈ λ€νΈμνΉ μλλ€.
κ°λ¨νκ²λ§ λ³΄λ©΄ kubenetμ AKSμ νμν κ°μ λ€νΈμν¬ λμ­ν­μ μλμΌλ‘ μ§μ ν΄μ£Όκ³ 
Azure CNIλ μ¬μ©μκ° μ§μ  κ°μλ€νΈμν¬μ λμ­ν­μ μ€μ ν΄μ€ μ μμ΅λλ€.

Azure CNIμ kubenetλ₯Ό λΉκ΅νλ ν μλλ€.

![AKS_Network.png](AKS_Network.png)

μ λ¦¬νμλ©΄

Kubenetμ΄ κΆμ₯λλ κ²½μ°
-> μ νλ IP μ£Όμ κ³΅κ°
-> λλΆλΆμ Pod ν΅μ μ΄ Cluster λ΄μ μμλ
-> κ°μ λΈλμ κ°μ κΈ°λ₯ μ¬μ© XX

Azure CNI κ° κΆμ₯λλ κ²½μ°
-> κ°μ λΈλ λλ λ€νΈμν¬ μ μ±κ³Ό κ°μ κ³ κΈ λ³΄μ κΈ°λ₯ νμ
-> Pod ν΅μ μ΄ Cluster μΈλΆμ λ¦¬μμ€μΌλ
-> μ§μ μ μΈ IP μ£Όμ ν λΉμ΄ νμν λ

λ‘ μ λ¦¬νμμ΅λλ€. λ³Έ μ€μ΅μμλ kubenetλ‘ μ§μ νκ² μ΅λλ€.

AKS μ λ°°ν¬λ μ νλ¦¬μΌμ΄μμ μ½κ² μ‘μΈμ€ νκΈ° μνμ¬ HTTP μ νλ¦¬μΌμ΄μ λΌμ°ν μ¬μ©μ μ²΄ν¬ν΄μ€λλ€.

AKS λ³΄μ μ€μ μ μν΄ Calico μ€μ μ ν  μ μμ΅λλ€. μ΄ μ€μ΅μμλ μ¬μ©νμ§ μκ² μ΅λλ€.

6. ν΅ν© ν­ μλλ€.

μΆν Azure Devopsμμ μ΄λ―Έμ§λ₯Ό Build & Deploy ν  ACR(Azure Container Registry)λ₯Ό μμ±νκ² μ΅λλ€.

![AKS_8.png](AKS_8.png)

μ»¨νμ΄λ λ μ§μ€νΈλ¦¬ ν­μμ μλ‘λ§λ€κΈ°λ₯Ό ν΄λ¦­ν ACRμ΄λ¦μ μ§μ νμ¬ μμ±ν΄μ€λλ€. μ΄λ ACRμ μ΄λ¦μ Azure μ μ²΄μμ μ μΌν μ΄λ¦μ΄μ΄μΌν©λλ€.

7. μ΅μ’

![AKS_9.png](AKS_9.png)

μ€μ ν λ΄μ©λ€μ μ΅μ’μ μΌλ‘ κ²ν νκ³  νλ¨μ λ§λ€κΈ°λ₯Ό ν΄λ¦­νμ¬ λ¦¬μμ€λ₯Ό μμ±ν©λλ€.

![AKS_10.png](AKS_10.png)

![AKS_11.png](AKS_11.png)

μμ±λ λ¦¬μμ€λ₯Ό νμΈν©λλ€.

8. AKS μ μ νμ€νΈ

![AKS_12.png](AKS_12.png)

Windowμ κ²½μ° PowerShellμμ Az login μ μλ ₯νμ¬ Azure κ³μ μΌλ‘ login ν©λλ€.
(Azure CLI μ€μΉ νμ!!)

- az account set --subscription subscriptionid λ₯Ό μλ ₯νμ¬ AKSκ° μμ±λ κ΅¬λ IDλ₯Ό μ§μ ν©λλ€.

- az aks get-credentials --resource-group RG-Kubernetes --name korwoo

λ₯Ό μλ ₯νμ¬ AKSμ Credentialλ₯Ό Get ν©λλ€.

- kubectl get namespaces λ±μ λͺλ Ήμ΄λ₯Ό μλ ₯νμ¬ AKSλ₯Ό νμ€νΈ ν©λλ€.

![AKS_13.png](AKS_13.png)


```toc

```
