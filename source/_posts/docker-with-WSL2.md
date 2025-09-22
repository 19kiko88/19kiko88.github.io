---
title: docker with WSL2
date: 2025-09-22 18:00:00
categories:
  - Docker
tags:
  - Docker
  - WSL2
  - Ubuntu  
---
##### 最近在學習Docker。想用.net 8 + mssql + kestrel + angular來建立一個容器化的CRUD網站，順便把過程記錄下來。
<!-- more -->
#
#
#
#
---
### 安裝WSL2(Windows Subsystem for Linux 2)
#
#
#
#
##### WSL2簡單的說明就是可以讓你在 Windows 上直接跑 Linux 環境（像 Ubuntu、Debian 等），不用安裝雙系統，也不用開虛擬機器的架構。
##### 但由於我在先前已經有安裝過了，裡面已經有安裝過Docker了，這次為了重頭到尾完整記錄，決定把原本的WSL2內容全部刪除並重新安裝，100%的重頭開始。
#
#
#
#
* #####  清空現有的 WSL2 環境：
    1. ##### 關閉所有 WSL2 執行個體。
    ``` bash
    wsl --shutdown
    ``` 

    2. ##### 解除註冊 (Unregister) WSL2 發行版。
    ##### 在 PowerShell 中執行以下命令，解除註冊您想要清空的 WSL2 發行版。 請將 <DistroName> 替換為您要解除註冊的發行版的名稱 (例如 Ubuntu、Debian 等)。 您可以使用 wsl --list 或 wsl -l 命令來查看已安裝的發行版名稱。
    ``` bash
    wsl --unregister <DistroName>
    ``` 

    3. ##### (可選)刪除 WSL2 虛擬硬碟 (VHDX) 檔案：
        ##### 解除註冊後，WSL2 使用的虛擬硬碟檔案 (VHDX) 仍然存在於您的硬碟上。 如果您想要完全清除所有痕跡，可以手動刪除 VHDX 檔案。
        1. ##### 找到 VHDX 檔案的位置。 預設情況下，VHDX 檔案位於以下路徑： 
        ``` text
        %USERPROFILE%\AppData\Local\Packages\<DistroPackageName>\LocalState\ext4.vhdx
        ```
        ##### <DistroPackageName> 是發行版的套件名稱。 您可以在 Microsoft Store 中找到發行版的套件名稱。 例如，Ubuntu 的套件名稱是 CanonicalGroupLimited.UbuntuonWindows_79rhkp1fndgsc。

        2. ##### 關閉所有 WSL2 執行個體 (再次確認)。
        3. ##### 刪除 VHDX 檔案。
#
#
#
#        
* #####  重新安裝 WSL2：
  ##### 安裝WSL2可以參考MSDN的 [如何使用 WSL 在 Windows 上安裝 Linux](https://learn.microsoft.com/zh-tw/windows/wsl/install).

  1. #####  以系統管理員模式開啟 PowerShell，並輸入下面指令
  ``` bash
  wsl --install
  ```
        
  ##### 回應訊息顯示[已安裝 Windows 子系統 Linux 版]，並且下面列出很多Linux的發行版。這個訊息代表 WSL (Windows Subsystem for Linux) 已經安裝完成，但並不代表已經有 Linux 發行版可以直接用。
  {% asset_image wsl2_distro.jpg wsl2_distro %}

  2. ##### 可以再用下面指令查詢查看已安裝的 Linux 發行版確認。如果沒有任何發行版，會顯示 尚未安裝任何發行版。
  ``` bash
  wsl --list --verbose
  ```
  {% asset_image wsl2_verbose.jpg wsl2_verbose %}

  2. ##### 安裝 Linux 發行版。如果不特別加上版本號，會自動拉最新的 LTS (Long Term Support) 版本。WSL預設安裝的會是Ubuntu Server。
  ``` bash
  wsl --install -d Ubuntu-24.04
  ```
  {% asset_image wsl2_install_ubuntu_verbose.jpg wsl2_install_ubuntu_verbose %}

  3. ##### 安裝完畢後，會要你輸入登入者的帳號 & 密碼，輸入完成後可以看到初始畫面。
  {% asset_image wsl2_set_id_pwd.jpg wsl2_set_id_pwd %}
  #
  #
  #
  #
  {% asset_image wsl2_ubuntu_init.jpg wsl2_ubuntu_init %}

  4. ##### 設定完成。執行下面指令確認WSL狀態 & 目前Ubuntu發行版本。在\\AppData資料夾下也可以看到新建的.vhdx虛擬硬碟檔。
  ``` bash
  wsl --status
  ```
  {% asset_image wsl2_status.jpg wsl2_status %}
  #
  #
  #
  #
  ``` bash
  wsl --list --verbose
  ```
  {% asset_image wsl2_list_verbose.jpg wsl2_list_verbose %}
  #
  #
  #
  #
  {% asset_image wsl2_vhdx.jpg wsl2_vhdx %}
  #
  #
  #
  #
  5. ##### 開啟WSL
  {% asset_image wsl2_open.jpg wsl2_open %}  
#
#
#
#
---  
### Docker
* ##### 以下是在 WSL2 環境中直接安裝 Docker Engine 的步驟：
    1. ##### 更新套件索引(APT(Advanced Package Tool)為Ubuntu的套件管理系統)：
    ``` bash
    sudo apt update
    ```
    2. ##### 安裝必要的套件：
    ``` bash
    sudo apt install apt-transport-https ca-certificates curl gnupg lsb-release
    ```




