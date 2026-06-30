---
title: n8n with Git
date: 2026-06-30 15:10:53
categories:
  - AI應用
tags:
  - AI
  - n8n
  - docker
---
##### 這篇文章記錄如何用GitHub來對n8n的workflow進行版控
<!-- more -->
#
#
#
---
### 1.確認/調整 docker-compose 掛載
``` yaml
version: '3.8'

services:
  n8n:
    image: docker.n8n.io/n8nio/n8n:latest
    container_name: n8n_local
    restart: always
    network_mode: "host" # 讓容器直接使用 WSL/Windows 的 localhost
    #ports:
    #  - "5678:5678"
    environment:
      - TZ=Asia/Taipei # 設定你的時區
      - N8N_SECURE_COOKIE=false # 地端測試關閉安全 Cookie 限制
      - NODE_FUNCTION_ALLOW_BUILTIN=fs  # ✅ 開啟 fs 模組
    volumes:
      - ~/n8n_local/n8n_data:/home/node/.n8n
      - /mnt/d/Homer/Project_MyGit/mcp-camera/photos:/photos  # ✅ 掛載照片資料夾
      - ./n8n-workflows-backup:/backup/workflows   # WSL路徑 : Container路徑
      - /mnt/c/Users/homer_chen/OneDrive - ASUS/Pictures/Camera Roll:/photos
```
##### 修改好docker-compose.yml後，記得執行下面指令重啟n8n。
``` bash
docker compose down && docker compose up -d
```
#
#
#
##### 另外特別說明一下這行，冒號左邊 ./n8n-workflows-backup 是主機(WSL)端的路徑,右邊 /backup/workflows 是container 內部的路徑。所以更準確的註解應該寫成「WSL 主機路徑 : Container 內路徑」。
``` yaml
- ./n8n-workflows-backup:/backup/workflows
```
#
#
#
##### <span style="color:red;">Docker 在掛載 bind mount 時,如果 WSL 主機端 ~/n8n_local/n8n-workflows-backup 資料夾不存在,Docker 會自動建立它,所以理論上你什麼都不做,執行 docker compose up -d 之後它就會自動生成。</span>
#
#
#
---
### 2.設定 git 身份、初始化 repo、連結 remote

1. ##### 在GitHub建立Repo
#
#
#
2. ##### 在 WSL 產生 SSH key 並加到 GitHub
  * ##### 先確認有無ssh檔案
  ``` bash
  ls -la ~/.ssh
  ```
  {% asset_image check-ssh-exist.jpg check-ssh-exist  %}
#
#
#
  * ##### 如果沒看到 id_ed25519 之類的檔案，產生一個新的(*ed25519為加密演算法)。過程中按 Enter 全部用預設值即可(也可以設密碼，但設了之後每次 push 會要你輸入)：
  ``` bash
  ssh-keygen -t ed25519 -C "你的email@example.com"
  ```
  {% asset_image gen-ssh-file.jpg gen-ssh-file  %}
#
#
#   
  * ##### 複製 public key(ssh-ed25519 {KEY} {Mail}全部都要複製)：
  ``` bash
  cat ~/.ssh/id_ed25519.pub
  ```
  {% asset_image ssh-content.jpg ssh-content  %}
#
#
#
  * ##### 到 GitHub → 右上角頭像 → Settings → SSH and GPG keys → New SSH key，把剛剛複製的內容貼進去並存檔。測試連線是否成功：
  ``` bash
  ssh -T git@github.com
  ```
  * ##### 如果一切設定正確（金鑰已經上傳到 GitHub 帳號），會看到：
  ``` text
  Hi 你的GitHub帳號名稱! You've successfully authenticated, but GitHub does not provide shell access.
  ```
  {% asset_image connet-to-github.jpg connet-to-github  %}  
#
#
#  
3. ##### 設定 git 身份(只需做一次，全域生效)。可以用任意名字和 email，但建議跟你 GitHub 帳號的 email 一致，方便之後 commit 記錄能對應到你的 GitHub 帳號，這兩行只是設定本地git的身分標籤。
``` bash
  git config --global user.email "你的email@example.com"
  git config --global user.name "Homer Chen"
```
#
#
#
4. ##### 進入備份資料夾，初始化 git：
  * ##### 在 WSL terminal輸入下面指令:
``` bash
cd ~/n8n_local/n8n-workflows-backup
git init
git remote add origin git@github.com:{你的帳號}/{repo_name}.git
git branch -M main
```
#
#
#
---
### 3.寫一個匯出 + commit + push 腳本

1. ##### 建立 ~/n8n_local/backup-workflows.sh:
``` bash
nano backup-workflows.sh
```
#
#
#
##### CONTAINER_NAME. BACKUP_DIR_HOST. BACKUP_DIR_CONTAINER要記得更改。
``` bash
#!/bin/bash
set -e

CONTAINER_NAME="n8n_local"
BACKUP_DIR_HOST="$HOME/n8n-project/n8n-workflows-backup"
BACKUP_DIR_CONTAINER="/backup/workflows"

# 在 container 內匯出,--separate 讓每個 workflow 變成獨立檔案,方便看 diff
docker exec -u node "$CONTAINER_NAME" n8n export:workflow --all --separate --output="$BACKUP_DIR_CONTAINER/"

cd "$BACKUP_DIR_HOST"

# 只有真的有變化才 commit
if [[ -n "$(git status --porcelain)" ]]; then
  git add -A
  git commit -m "Auto backup: $(date '+%Y-%m-%d %H:%M:%S')"
  git push origin main
  echo "Workflows pushed to GitHub."
else
  echo "No changes detected."
fi
```
#
#
#
2. ##### 給執行權限:
``` bash
chmod +x ~/n8n_local/backup-workflows.sh
```
#
#
#
3. ##### 手動跑一次測試:
``` bash
cd ~/
cd n8n_local
bash backup-workflows.sh
```
#
#
#
  * ##### 成功匯出了2 個 workflow
  {% asset_image execute-backup-workflows.jpg execute-backup-workflows  %}
#
#
#
  * ##### 在n8n workflow裡面隨便插入一個測試node。  
#
#
#
   * ##### 重新手動執行腳本，成功把workflow的json檔push到GitHub遠端了!!
  {% asset_image push-success.jpg push-success  %}    
---
### 補充
##### 後來又想到，如果能把container的/backup/workflows裡面的東西由原本的WSL改掛載到(Win 11)Host，我就能用SourceTree之類的工具看Git歷程了。
#
#
#
* ##### 先把原本的掛載註解。這樣 container 內寫入 /backup/workflows 的東西，實際上會存到 Windows 的 D:\Homer\Project_MyGit\n8n-smart-mirrow-workflow\，所以 SourceTree（跑在 Windows）就能直接打開這個資料夾看到 git 歷程。
``` yaml
version: '3.8'

services:
  n8n:
    image: docker.n8n.io/n8nio/n8n:latest
    container_name: n8n_local
    restart: always
    network_mode: "host" # 讓容器直接使用 WSL/Windows 的 localhost
    #ports:
    #  - "5678:5678"
    environment:
      - TZ=Asia/Taipei # 設定你的時區
      - N8N_SECURE_COOKIE=false # 地端測試關閉安全 Cookie 限制
      - NODE_FUNCTION_ALLOW_BUILTIN=fs  # ✅ 開啟 fs 模組
    volumes:
      - ~/n8n_local/n8n_data:/home/node/.n8n
      - /mnt/d/Homer/Project_MyGit/mcp-camera/photos:/photos  # ✅ 掛載照片資料夾
      #- ./n8n-workflows-backup:/backup/workflows   # WSL路徑 : Container路徑
      - /mnt/d/Homer/Project_MyGit/n8n-workflows-backup:/backup/workflows   # 改成 Windows D槽路徑
      - /mnt/c/Users/homer_chen/OneDrive - ASUS/Pictures/Camera Roll:/photos
```
#
#
#
* ##### 把原本舊路徑 ~/n8n_local/n8n-workflows-backup 裡已經 commit 過的 .git 資料夾和檔案，搬到新位置（如果你想保留歷史紀錄）：
``` bash
cp -r ~/n8n_local/n8n-workflows-backup/. /mnt/d/Homer/Project_MyGit/n8n-workflows-backup/
```
  ##### 執行完上面指令後，原本空的Host資料夾多出新的檔案了。
{% asset_image copy-git.jpg copy-git  %}
#
#
#
* ##### 改完 docker-compose.yml 後重啟：
``` bash
cd ~/n8n_local
docker compose down
docker compose up -d
```
#
#
#
* ##### 更新 backup-workflows.sh 裡的 BACKUP_DIR_HOST 變數：
``` bash
cd ~
cd ~/n8n_local
nano backup-workflows.sh
```

```bash
#!/bin/bash
set -e

CONTAINER_NAME="n8n_local"
#BACKUP_DIR_HOST="$HOME/n8n_local/n8n-workflows-backup"
BACKUP_DIR_HOST="/mnt/d/Homer/Project_MyGit/n8n-workflows-backup"
BACKUP_DIR_CONTAINER="/backup/workflows"

# 在 container 內匯出,--separate 讓每個 workflow 變成獨立檔案,方便看 diff
docker exec -u node "$CONTAINER_NAME" n8n export:workflow --all --separate --output="$BACKUP_DIR_CONT>

cd "$BACKUP_DIR_HOST"

# 只有真的有變化才 commit
if [[ -n "$(git status --porcelain)" ]]; then
  git add -A
  git commit -m "Auto backup: $(date '+%Y-%m-%d %H:%M:%S')"
  git push origin main
  echo "Workflows pushed to GitHub."
else
  echo "No changes detected."
fi
```
