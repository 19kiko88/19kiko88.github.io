---
title: 在VirtualBox上安裝Unbuntu
date: 2025-09-23 11:34:30
categories: 
  - Ubuntu
tags:
  - Docker
  - WSL2
  - Ubuntu  
---
##### 最近在學習Docker。想用.net 8 + mssql + kestrel + angular來建立一個容器化的CRUD網站，原本想說用WSL2來建立Linux環境來建立Docker環境，但在一開始的安裝Docker Engine時就出現問題了，上網爬了一下文章後，大部分人是說WSL2跟Docker常常會遇到一些不可預期的問題，趁著還在建置前置環境的階段，果斷從WSL2改成在VM上建立一個native Linux環境，避免夜長夢多!
<!-- more -->
#
#
#
#
---
### VMWare新增Ubuntu
#
#
#
#
##### 到Ubuntu網站下載最新LST版本的映象檔[Ubuntu 24.04.3 LTS](https://ubuntu.com/download/server).
{% asset_image get_ubuntu_server.jpg get_ubuntu_server %}
#
#
#
#
##### ISO Image選擇剛剛下載好的映像檔
{% asset_image vb_setting_1.jpg vb_setting_1 %}
#
#
#
#
##### 如果無人值守安裝(Processed with Unattended Installation)如果有勾選的話，就要進行此步驟，先輸入要登入Ubuntu的帳號密碼，到時就會自動幫你建立帳號
{% asset_image vm_unattended.jpg vm_unattended %}
#
#
#
#

##### 記憶體 & CPU設定
{% asset_image vb_setting_2.jpg ram_cpu_setting %}
#
#
#
#
##### 硬碟設定
{% asset_image vb_setting_3.jpg hd_setting %}
#
#
#
#
##### 接著再重新啟動一次VM，就會開始進行安裝程序，會花些時間，請耐心等待。
{% asset_image vb_setting_4.jpg star_installing %}
#
#
#
#
##### 如果安裝過程中，卡在這個步驟太久的話，就先卡網路設定改成橋接模式(Bridge)，然後再重新啟動一次VM。
{% asset_image vb_setting_5.jpg enp0s3 %}
#
#
#
#
{% asset_image bridge.jpg bridge %}
#
#
#
#
##### 安裝完成，用剛剛無人值守安裝的帳號登入Ubuntu看看。
{% asset_image login.jpg login %}
---

