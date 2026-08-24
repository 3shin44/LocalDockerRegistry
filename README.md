# 內網 Docker Registry 部署流程（HTTP 方案）

> 適用場景：純內網環境，開發機（Windows）與部署機（Debian）分離，透過內部 Docker Registry 傳遞 image。採用 HTTP（`insecure-registries`）以降低維護成本，**僅限內網使用，勿對外開放**。

## Quick CMD

- Build For multi-platform

    `docker buildx build --platform linux/amd64,linux/arm64 -t $Registry/$ImageName:$Version  --push .`

- Usage

    ```
    # docker-compose.yaml
    services:
    app-name:
        image: 192.168.1.100:5000/IMG_NAME:TAG
    ```

- 查詢介面

    `Docker-Registry-API.postman_collection.json` 提供POSTMAN呼叫接口, 對Registry進行維護


## 目錄

1. [開發環境（Windows）設置](#1-開發環境windows設置)
2. [開發完成後如何 Push Image](#2-開發完成後如何-push-image)
3. [部署環境如何建置 Docker Registry](#3-部署環境如何建置-docker-registry)
4. [部署環境如何拉取 Image 並部署（docker compose）](#4-部署環境如何拉取-image-並部署docker-compose)
5. [維護腳本：定期清除舊版本（每個 Image 僅保留 3 版）](#5-維護腳本定期清除舊版本每個-image-僅保留-3-版)

---

## 1. 開發環境（Windows）設置

### 1.1 前置需求

- 已安裝 Docker Desktop（含 WSL2 backend）
- 內網可連線到 Registry 主機（確認 `ping {{registry.internal}}` 或直接用 IP 可通）

### 1.2 設定 insecure-registries

打開 **Docker Desktop → Settings → Docker Engine**，在 JSON 設定中加入：

```json
{
  "insecure-registries": ["{{registry.internal:5000}}"]
}
```

> 若內網無 DNS，直接改用 Registry 主機 IP，例如 `192.168.1.100:5000`。
> ⚠️ WSL2 backend 模式下，此設定務必在 **Docker Desktop GUI** 修改，而非 WSL 內部的 `daemon.json`，否則會被 Docker Desktop 覆蓋。

點擊 **Apply & Restart** 套用設定。

### 1.3 （選用）一鍵設定腳本

`setup-insecure-registry.ps1`（需以系統管理員身分執行 PowerShell）：

```powershell
$configPath = "$env:ProgramData\Docker\config\daemon.json"
$registry = "{{registry.internal:5000}}"

if (Test-Path $configPath) {
    $json = Get-Content $configPath -Raw | ConvertFrom-Json
} else {
    $json = @{}
}

if (-not $json.'insecure-registries') {
    $json | Add-Member -NotePropertyName 'insecure-registries' -NotePropertyValue @($registry)
} elseif ($json.'insecure-registries' -notcontains $registry) {
    $json.'insecure-registries' += $registry
}

$json | ConvertTo-Json -Depth 5 | Set-Content $configPath
Write-Host "已更新，請至 Docker Desktop 重啟服務使設定生效"
```

> 建議仍以 GUI 貼上 JSON 為主，腳本改檔案較容易格式出錯，此腳本僅供批次部署新機器時參考。

### 1.4 驗證設定

```powershell
docker login {{registry.internal:5000}}
```

若跳出帳密輸入且無 TLS 錯誤，代表設定成功。

---

## 2. 開發完成後如何 Push Image

### 2.1 Push Script（PowerShell）

儲存為 `push.ps1`，放在專案根目錄（與 `Dockerfile` 同層）：

```powershell
param(
    [Parameter(Mandatory=$true)][string]$Version,
    [string]$Registry = "{{registry.internal:5000}}",
    [string]$ImageName = "your-app"
)

$ErrorActionPreference = "Stop"
$FullImage = "$Registry/$ImageName`:$Version"

Write-Host "=== Login ===" -ForegroundColor Cyan
docker login $Registry

Write-Host "=== Build ===" -ForegroundColor Cyan
docker build -t $FullImage .

Write-Host "=== Push ===" -ForegroundColor Cyan
docker push $FullImage

Write-Host "=== Update VERSION file ===" -ForegroundColor Cyan
Set-Content -Path "VERSION" -Value $Version -NoNewline

Write-Host "Done. Pushed $FullImage" -ForegroundColor Green
Write-Host "別忘了 commit VERSION 檔並 push 到 deploy repo" -ForegroundColor Yellow
```

### 2.2 使用方式

```powershell
.\push.ps1 -Version 1.2.3
```

> 版本號建議採用明確語意化版號（semver）或 git commit hash，**不要使用 `latest`**，以確保部署端能精確控制版本、方便退版。

### 2.3 （選用）自動同步版本紀錄至 Git

若 `VERSION` 檔案由 Git repo 管理（GitOps 風格），可在 `push.ps1` 最後加入：

```powershell
git add VERSION
git commit -m "release: $Version"
git push
```

---

## 3. 部署環境如何建置 Docker Registry

整體流程分兩份 compose 檔案管理，職責分開：

| 檔案                          | 用途                       | 執行方式                 |
| ----------------------------- | -------------------------- | ------------------------ |
| `docker-compose.htpasswd.yml` | 一次性建立/更新帳密檔案    | `run --rm`（跑完即結束） |
| `docker-compose.yml`          | 常駐啟動 Registry 服務本體 | `up -d`（長駐執行）      |

### 3.1 目錄結構

於 Debian 主機**手動建立**：

```bash
mkdir -p /opt/registry/{data,auth,certs}
cd /opt/registry
```

將以下檔案放入 `/opt/registry/` 目錄。

### 3.2 帳密設定：.env

帳號密碼透過 `.env` 管理，不寫死在 compose 檔案內，避免密碼進版控。

`.env.example`（範本，可提交到 Git）：

```bash
# 複製此檔為 .env 並填入實際帳密，.env 不應提交到 Git（加入 .gitignore）
REGISTRY_USER=admin
REGISTRY_PASSWORD=YourStrongPassword
```

`.gitignore`：

```
.env
```

實際使用：

```bash
cp .env.example .env
vim .env   # 填入實際帳密
```

### 3.3 docker-compose.htpasswd.yml（建立帳密檔案）

```yaml
services:
  htpasswd:
    image: httpd:2
    entrypoint: htpasswd
    # -B: bcrypt 加密, -b: 從參數讀取密碼(非互動)
    # 首次建立檔案需加 -c，後續新增/更新帳號請移除 -c（避免覆蓋既有帳號）
    command:
      ["-Bbc", "/auth/htpasswd", "${REGISTRY_USER}", "${REGISTRY_PASSWORD}"]
    volumes:
      - ./auth:/auth
```

執行：

```bash
docker compose -f docker-compose.htpasswd.yml run --rm htpasswd
```

**新增第二組帳號**（例如給 CI/CD 用的獨立帳號）：

1. 修改 `.env` 內容為新的一組帳密（或另存 `.env.ci` 並用 `--env-file .env.ci` 指定）
2. 將 `command` 內的 `-Bbc` 改成 `-Bb`（**移除 `-c`**，避免覆蓋既有帳號檔案）
3. 重新執行 `run --rm`

### 3.4 docker-compose.yml（Registry 本體服務）

```yaml
services:
  registry:
    image: registry:2
    restart: always
    ports:
      - "5000:5000"
    environment:
      REGISTRY_AUTH: htpasswd
      REGISTRY_AUTH_HTPASSWD_REALM: Registry Realm
      REGISTRY_AUTH_HTPASSWD_PATH: /auth/htpasswd
      REGISTRY_STORAGE_DELETE_ENABLED: "true" # 允許之後清除舊版本
    volumes:
      - ./data:/var/lib/registry
      - ./auth:/auth
```

> 此服務本身不需要帳密環境變數，驗證是直接讀取 `./auth/htpasswd` 檔案內容（由 3.3 步驟產生）。

### 3.5 啟動 Registry

```bash
docker compose -f docker-compose.yml up -d
```

### 3.6 Debian 主機（Registry host 本身）也需設定 insecure-registries

```bash
sudo tee /etc/docker/daemon.json <<'EOF'
{
  "insecure-registries": ["{{registry.internal:5000}}"]
}
EOF
sudo systemctl restart docker
```

> 若 Registry 主機本身也要 `docker pull` 測試，同樣需要此設定，否則會遇到 `http: server gave HTTP response to HTTPS client`。

### 3.7 防火牆限制（重要）

僅允許內網網段連線 5000 port，避免對外曝露：

```bash
sudo ufw allow from 192.168.0.0/16 to any port 5000
sudo ufw deny 5000
```

（依實際內網 CIDR 調整）

---

## 4. 部署環境如何拉取 Image 並部署（docker compose）

### 4.1 部署機（Debian，部署 your-app 的主機）設定 insecure-registries

同 3.6 步驟，每台需要 `pull` image 的 Debian 主機都要設定：

```bash
sudo tee /etc/docker/daemon.json <<'EOF'
{
  "insecure-registries": ["{{registry.internal:5000}}"]
}
EOF
sudo systemctl restart docker
```

### 4.2 部署專案目錄結構

```
deploy-repo/
├── docker-compose.yml
├── docker-compose.dev.yml
├── docker-compose.prod.yml
├── .env.dev
├── .env.prod
├── VERSION          # 內容為目前要部署的 image tag，例如 "1.2.3"
└── deploy.sh
```

### 4.3 docker-compose.yml（共用設定）

```yaml
services:
  your-app:
    image: {{registry.internal:5000}}/your-app:${IMAGE_TAG}
    restart: unless-stopped
    env_file:
      - .env.${DEPLOY_ENV}
```

### 4.4 docker-compose.dev.yml / docker-compose.prod.yml（啟動參數差異）

```yaml
# docker-compose.dev.yml
services:
  your-app:
    environment:
      - LOG_LEVEL=debug
      - DEBUG=true
```

```yaml
# docker-compose.prod.yml
services:
  your-app:
    environment:
      - LOG_LEVEL=info
      - DEBUG=false
```

### 4.5 deploy.sh（部署腳本）

```bash
#!/bin/bash
set -e
ENV=${1:-dev}   # 傳入 dev 或 prod
cd /opt/deploy-repo
git pull

export IMAGE_TAG=$(cat VERSION)
export DEPLOY_ENV=$ENV

docker login {{registry.internal:5000}}
docker pull {{registry.internal:5000}}/your-app:$IMAGE_TAG

docker compose -f docker-compose.yml -f docker-compose.$ENV.yml \
  --env-file .env.$ENV up -d

echo "Deployed your-app:$IMAGE_TAG to $ENV"
```

賦予執行權限並使用：

```bash
chmod +x deploy.sh
./deploy.sh prod
```

### 4.6 退版方式（GitOps 風格）

```bash
# 找到要退回的 commit
git log --oneline -- VERSION

# revert 該次版本更新
git revert <commit-hash>
git push

# 重新部署
./deploy.sh prod
```

---

## 5. 維護腳本：定期清除舊版本（每個 Image 僅保留 3 版）

Docker Registry 的刪除是兩階段：**先刪 manifest，再執行 garbage collect** 才會真正釋放磁碟空間。

### 5.1 清理腳本 cleanup-registry.sh

放在 Registry 主機（`/opt/registry/`）：

```bash
#!/bin/bash
set -e

REGISTRY_HOST="localhost:5000"
KEEP=3   # 每個 image 保留的版本數
REGISTRY_DATA_DIR="/opt/registry"

echo "=== 取得所有 repository 清單 ==="
REPOS=$(curl -s "http://$REGISTRY_HOST/v2/_catalog" | jq -r '.repositories[]')

for REPO in $REPOS; do
  echo "--- 處理 $REPO ---"

  # 取得所有 tag，依照建立時間排序（用 tag 本身若為 semver 排序，或改用 config 內的 created 時間）
  TAGS=$(curl -s "http://$REGISTRY_HOST/v2/$REPO/tags/list" | jq -r '.tags[]' | sort -V)
  TAG_COUNT=$(echo "$TAGS" | wc -l)

  if [ "$TAG_COUNT" -le "$KEEP" ]; then
    echo "  版本數 $TAG_COUNT <= 保留數 $KEEP，略過"
    continue
  fi

  DELETE_TAGS=$(echo "$TAGS" | head -n $(($TAG_COUNT - $KEEP)))

  for TAG in $DELETE_TAGS; do
    echo "  刪除 $REPO:$TAG"

    # 取得該 tag 的 digest
    DIGEST=$(curl -s -I \
      -H "Accept: application/vnd.docker.distribution.manifest.v2+json" \
      "http://$REGISTRY_HOST/v2/$REPO/manifests/$TAG" \
      | grep -i Docker-Content-Digest | awk '{print $2}' | tr -d '\r')

    if [ -n "$DIGEST" ]; then
      curl -s -X DELETE \
        -H "Accept: application/vnd.docker.distribution.manifest.v2+json" \
        "http://$REGISTRY_HOST/v2/$REPO/manifests/$DIGEST"
    else
      echo "  警告：找不到 $REPO:$TAG 的 digest，略過"
    fi
  done
done

echo "=== 執行 garbage collect（釋放實際磁碟空間） ==="
docker compose -f "$REGISTRY_DATA_DIR/docker-compose.yml" exec -T registry \
  bin/registry garbage-collect /etc/docker/registry/config.yml

echo "清理完成"
```

> 需要 `jq`：`sudo apt install -y jq`
> 排序依 tag 字串（`sort -V`，version sort），若版號並非嚴格 semver（例如 git hash 或日期戳），建議改用 image config 內的 `created` 時間戳排序，可另外調整腳本邏輯。

### 5.2 設定排程（每週執行一次）

```bash
chmod +x /opt/registry/cleanup-registry.sh
sudo crontab -e
```

加入：

```
0 3 * * 0 /opt/registry/cleanup-registry.sh >> /var/log/registry-cleanup.log 2>&1
```

（每週日凌晨 3 點執行一次）

### 5.3 注意事項

- `REGISTRY_STORAGE_DELETE_ENABLED: "true"` 務必已在 Registry 的 `docker-compose.yml` 設定，否則刪除 API 會回傳 405
- Garbage collect 執行期間建議暫停 push 操作，避免競爭寫入（可安排在離峰時段執行）
- 執行前建議先備份 `data/` 目錄，避免誤刪重要版本

---

## 附錄：完整安全提醒

| 項目     | 說明                                                |
| -------- | --------------------------------------------------- |
| 傳輸加密 | HTTP 模式無加密，僅限內網信任網段使用               |
| 存取控制 | 已設 htpasswd 帳密驗證，仍建議搭配防火牆限制來源 IP |
| 對外曝露 | 務必確認 5000 port 未對外網開放                     |
| 版本管理 | 一律使用明確 tag，禁止使用 `latest`，以利追蹤與退版 |
