# OpenShell 與 Codex：本機 sandbox 網路權限測試指南

本指南帶你從原始碼啟動 gateway、建置 Codex image、配置 ChatGPT 認證，最後驗證 agent 的網路存取會受到 OpenShell policy 限制。所有步驟使用自己的帳號與本機實例。

完成後應觀察到三個結果：未授權的 GitHub API 連線被拒絕、加入唯讀規則後 GET 成功、POST 仍被拒絕。這是預期安全行為的測試，不代表已找到漏洞。

操作順序為「環境 → gateway → image → provider → sandbox → 拒絕／核准測試」。已完成的階段可以跳過；先保持 gateway 運行，再執行管理命令。

## 1. 前置條件與範例名稱

| 項目 | 要求／範例 |
| --- | --- |
| 主機 | Windows，透過 Docker Desktop 與 WSL 2 執行 Linux workload |
| Linux 環境 | Ubuntu-24.04，WSL 2 |
| Checkout | 自己的 OpenShell checkout；建議放在 WSL 的 `~/src/OpenShell` |
| 開發工具 | 依 checkout 的 `mise.toml` 與 `mise.lock` 安裝 |
| Image 基底 | `node:24-bookworm-slim` |
| Codex | 此流程的驗證版本為 `0.163.0` |
| Gateway | `docker-dev`，開發腳本預設 `http://127.0.0.1:18080` |
| Workload image | `openshell-codex:local` |
| Provider profile / instance | `codex-lab` / `my-codex` |
| 測試 sandbox | `codex-play`，以 Bash 維持主程序 |

以上名稱都是可替換的範例名稱。名稱已存在時使用新名稱，並一致替換後續命令，不需刪除舊資源。

版本限制：以下固定 Codex 為已驗證的 `0.163.0`；原實驗使用未固定版本的安裝命令，取得同一版本。固定版本的 Dockerfile 尚未另行重跑驗證。Node base tag 不是 digest，checkout 與外部服務也可能改變；重現時自行記錄 `git rev-parse HEAD`、`docker version` 與 `codex --version`，不必公開帳號或裝置資訊。

## 2. 終端與元件分工

| 操作位置 | 工作 |
| --- | --- |
| Windows PowerShell | 管理 WSL 發行版 |
| WSL terminal A，repo 根目錄 | 啟動並保持 gateway 前景運行 |
| WSL terminal B，repo 根目錄 | 建置 image、管理 profile/provider/policy、查日誌 |
| Sandbox shell，提示字元類似 `app@...` | 執行 Codex、curl 等 workload |

流程：CLI 呼叫 gateway；gateway 的 Docker driver 配置容器；supervisor 取得 policy 並處理網路；sandbox runtime 執行受限 workload。

`mise.local.toml` 管理本機工具與任務。Dockerfile 管理 image 內容。Sandbox policy 控制可存取資源。Provider profile 定義認證欄位與存取規則；provider instance 保存實際認證。它們不是同一份設定。

## 3. Windows、WSL 與 Docker 前置條件

PowerShell 檢查：

```shell
wsl --list --verbose
```

需要新 WSL 2 環境時，可另外安裝 Ubuntu-24.04，不必轉換現有發行版：

```shell
wsl --update
wsl --set-default-version 2
wsl --list --online
wsl --install -d Ubuntu-24.04
```

完成 Linux 使用者設定後，明確進入新環境；直接執行 `wsl` 仍可能進入原本的預設發行版：

```shell
wsl -d Ubuntu-24.04
```

Docker Desktop 設定：

- 使用 WSL 2 based engine，並對 Ubuntu-24.04 啟用 WSL Integration。
- 在 Resources → Network 啟用 host networking。此配置的 supervisor 使用 host networking 連回 WSL gateway；Docker Desktop 需支援並啟用此功能。
- 確認使用 Linux containers，Docker Engine 符合 checkout 的支援矩陣；本流程以 Engine 28.0 以上為前提。

在 Ubuntu 驗證：

```shell
docker version
docker run --rm hello-world
```

應同時有 Client 和 Server，且 hello-world 成功。只有 Client 版本不能證明 daemon 正常。

若 Desktop 卡在 Stopping／Settings 轉圈，先儲存 WSL 工作，退出 Docker Desktop，再於 PowerShell 執行 `wsl --shutdown` 後重新啟動 Desktop。此命令會停止所有 WSL 程序。先不用 Factory Reset，也不要 unregister Docker 的資料發行版。

## 4. 開發工具與本機設定

在 Ubuntu 安裝 mise，依 repo CONTRIBUTING 的 Bash 啟用方式：

```shell
sudo apt update
sudo apt install -y curl ca-certificates build-essential pkg-config libz3-dev jq
curl https://mise.run | sh
eval "$(~/.local/bin/mise activate bash)"
```

若要每次開 Bash 自動啟用，將以下一行加入 `~/.bashrc`，只加入一次：

```shell
eval "$(~/.local/bin/mise activate bash)"
```

進入 repo 根目錄：

```shell
mise trust
mise install
```

若 uv、buf 因 GitHub 下載超時失敗，可只重試失敗工具；版本應與自己的 checkout 一致，例如：

```shell
MISE_HTTP_DOWNLOAD_TIMEOUT=1h mise install --jobs 1 uv@0.10.12 buf@1.72.0
```

### 選用：暫時停用 sccache

若 sccache 還原產物時出現 `.d` 檔案不存在／設定權限失敗，可先停用這層編譯快取。這是本流程採用的 workaround，並非快取問題的根因修復；沒有該問題時可省略 `RUSTC_WRAPPER` 覆蓋。

建立或合併本機 `mise.local.toml`，不要覆蓋既有區段：

```toml
[env]
RUSTC_WRAPPER = ""

[tasks."lab:build"]
run = "docker build -t openshell-lab:local examples/bring-your-own-container"

[tasks."lab:codex-build"]
run = "docker build -t openshell-codex:local .cache/codex-lab"
```

`lab:build` 是較早的 curl-only 實驗，可選；Codex 路徑只需要 `lab:codex-build`。

確認覆蓋：

```shell
mise exec -- printenv RUSTC_WRAPPER
```

應為空值。Cargo 的正常建置判斷仍有效，不必 `cargo clean`。

### CRLF 換行問題

曾發生 `$'\r': command not found`、`set: pipefail ... invalid option name`，原因是 Bash 腳本使用 CRLF。建議新 checkout 使用 LF；長期開發可放在 WSL Linux 目錄，例如 `~/src/OpenShell`。

只有遇到 CRLF 問題時，才先確認 `git status --short` 與既有修改，再針對實際出錯的腳本轉換。例如：

```shell
git config --local core.autocrlf false
sed -i 's/\r$//' tasks/scripts/gateway-docker.sh
```

不需要為重現本測試而批次改寫整個 repo 的腳本；先處理實際出錯的檔案，並保留原本的使用者修改。

## 5. 啟動 gateway

Terminal A：

```shell
mise run gateway:docker
```

任務會準備本地 supervisor/sandbox 開發映像、建置 gateway、產生開發設定與 metadata，最後以前景程序啟動 gateway。首次可能花較久，後續會使用建置快取。

等待：

```text
Compute driver connected configured_driver=docker
Gateway listener bound address=127.0.0.1:18080
```

Terminal B：

```shell
mise exec -- openshell --gateway docker-dev status
```

此開發 gateway 使用 plaintext 且允許未認證使用者，僅用於本機實驗，不能用這個配置驗證正式環境的身分隔離。不要為解決連線問題直接對外公開 listener。

已遇過的非致命 warning：本地 `openshell/supervisor:dev`、`openshell/sandbox:dev` 從 registry pull 得到 404，但日誌明確表示使用 local image，driver 仍初始化成功。這不需要為不存在的開發 repository 執行 docker login。

若 supervisor 顯示連不回 gateway、啟動配置重試失敗，檢查 Docker Desktop host networking 與 WSL 可達性。CLI 可連到 gateway，不代表容器一定可以連到。

## 6. 建置 Codex image

```shell
mkdir -p .cache/codex-lab
```

建立 `.cache/codex-lab/Dockerfile`：

```dockerfile
FROM node:24-bookworm-slim

RUN apt-get update \
    && apt-get install -y --no-install-recommends \
       bash ca-certificates curl git jq \
    && rm -rf /var/lib/apt/lists/*

RUN npm install -g @openai/codex@0.163.0 \
    && groupadd --gid 1500 app \
    && useradd --uid 1500 --gid app --create-home app \
    && install -d -o app -g app /sandbox

COPY --chmod=755 start-codex.sh /usr/local/bin/start-codex
USER app
WORKDIR /sandbox
CMD ["/bin/bash"]
```

建立 `.cache/codex-lab/start-codex.sh`，使用 LF：

```bash
#!/usr/bin/env bash
set -euo pipefail

: "${CODEX_AUTH_ACCESS_TOKEN:?missing access token}"
: "${CODEX_AUTH_REFRESH_TOKEN:?missing refresh token}"
: "${CODEX_ROUTING_ACCOUNT_ID:?missing routing account id}"

umask 077

node <<'NODE'
const fs = require("fs");
const path = require("path");
const home = process.env.CODEX_HOME ||
  path.join(process.env.HOME, ".codex");
const encode = value =>
  Buffer.from(JSON.stringify(value)).toString("base64url");
const now = Math.floor(Date.now() / 1000);

const idToken = [
  encode({ alg: "none", typ: "JWT" }),
  encode({
    iss: "https://auth.openai.com",
    aud: "codex",
    sub: "openshell-local-lab",
    email: "lab@openshell.local",
    iat: now,
    exp: now + 3600
  }),
  "placeholder"
].join(".");

fs.mkdirSync(home, { recursive: true });
const file = path.join(home, "auth.json");
fs.writeFileSync(file, JSON.stringify({
  auth_mode: "chatgpt",
  OPENAI_API_KEY: null,
  tokens: {
    access_token: process.env.CODEX_AUTH_ACCESS_TOKEN,
    refresh_token: process.env.CODEX_AUTH_REFRESH_TOKEN,
    account_id: process.env.CODEX_ROUTING_ACCOUNT_ID,
    id_token: idToken
  },
  last_refresh: new Date().toISOString()
}, null, 2), { mode: 0o600 });
fs.chmodSync(file, 0o600);
NODE

exec codex -c 'cli_auth_credentials_store="file"' "$@"
```

建置並驗證：

```shell
mise run lab:codex-build
docker run --rm openshell-codex:local \
  bash -lc 'id; command -v codex; readlink -f "$(command -v codex)"; npm root -g; codex --version'
```

預期輸出包含：

```text
uid=1500(app) gid=1500(app) groups=1500(app)
/usr/local/bin/codex
/usr/local/lib/node_modules/@openai/codex/bin/codex.js
/usr/local/lib/node_modules
codex-cli 0.163.0
```

Image 不包含真實登入憑證。上面的身份 JWT 是供 Codex 本機解析的佔位資料，不能取得 OpenAI 授權；實際連線仍依賴 provider 與有效上游認證。這是此驗證版本的相容性 workaround，不是通用或正式的 OAuth 整合方案。

## 7. 匯入 provider profile

```shell
cp providers/codex.yaml .cache/codex-lab/provider.yaml
```

在副本中只修改：

```yaml
id: codex-lab
```

以及 `binaries`：

```yaml
binaries:
  - /usr/local/bin/codex
  - /usr/local/lib/node_modules/@openai/**
```

其餘保留 checkout 範例的 credentials、discovery 與 endpoints。範例不是自動載入的設定，必須匯入；原路徑 `/usr/lib/node_modules/...` 與本 image 不同。

```shell
mise exec -- openshell --gateway docker-dev provider profile lint \
  -f .cache/codex-lab/provider.yaml
mise exec -- openshell --gateway docker-dev provider profile import \
  -f .cache/codex-lab/provider.yaml
```

成功 lint 不代表認證與實際連線成功。若 Connection refused，先恢復 gateway，不要直接修改 YAML。

## 8. WSL 登入與 provider instance

WSL 與 Docker image 是不同環境，需要在 WSL 另外安裝 CLI，以便登入並將認證提供給 gateway：

```shell
mise exec -- npm install -g @openai/codex@0.163.0
mise exec -- codex --version
mise exec -- codex -c 'cli_auth_credentials_store="file"' login --device-auth
```

保持登入命令運行，瀏覽器開啟命令顯示的官方 device URL，登入 ChatGPT 並輸入當次代碼。一般聊天首頁沒有代碼欄位；登入後必要時重新開啟 device URL。依帳號設定，可能需先啟用 device code authentication。

不要把一次性代碼、auth.json 或 token 貼入文件、聊天、issue 或 image。

```shell
mise exec -- codex -c 'cli_auth_credentials_store="file"' login status
test -f "$HOME/.codex/auth.json" && echo "登入檔存在"
```

確認是自己的有效 ChatGPT 登入後，在同一 terminal 載入並建立 provider：

```shell
export CODEX_AUTH_ACCESS_TOKEN="$(jq -r '.tokens.access_token // empty' "$HOME/.codex/auth.json")"
export CODEX_AUTH_REFRESH_TOKEN="$(jq -r '.tokens.refresh_token // empty' "$HOME/.codex/auth.json")"
export CODEX_AUTH_ACCOUNT_ID="$(jq -r '.tokens.account_id // empty' "$HOME/.codex/auth.json")"
export CODEX_AUTH_ID_TOKEN="$(jq -r '.tokens.id_token // empty' "$HOME/.codex/auth.json")"

mise exec -- openshell --gateway docker-dev provider create \
  --name my-codex \
  --type codex-lab \
  --from-existing

unset CODEX_AUTH_ACCESS_TOKEN CODEX_AUTH_REFRESH_TOKEN \
  CODEX_AUTH_ACCOUNT_ID CODEX_AUTH_ID_TOKEN
```

若 provider 已存在，不必重建；本筆記沒有驗證更新既有 provider 的完整流程。

## 9. 啟動可回到 Bash 的 Codex sandbox

從 WSL 的主機登入檔取得真實 account ID，作為路由識別資訊；它不是 access token，但仍是帳號資訊，不需公開：

```shell
export CODEX_ROUTING_ACCOUNT_ID="$(
  jq -er '.tokens.account_id | select(type == "string" and length > 0)' \
    "$HOME/.codex/auth.json"
)"
```

確認前一命令成功且值非空，再建立 sandbox：

```shell
mise exec -- openshell --gateway docker-dev sandbox create \
  --name codex-play \
  --from openshell-codex:local \
  --provider my-codex \
  --env "CODEX_ROUTING_ACCOUNT_ID=$CODEX_ROUTING_ACCOUNT_ID" \
  -- /bin/bash
```

此命令進入 sandbox 的 Bash。名稱已被使用時，選擇新名稱，並將後續命令的 sandbox 名稱一致替換。

在 sandbox 內執行：

```shell
/usr/local/bin/start-codex --sandbox danger-full-access
```

此參數僅在 OpenShell 外層隔離已建立的 sandbox 裡使用，用來停用 Codex 的內層 bubblewrap 隔離；不是對主機直接授予不受限執行。OpenShell policy 仍生效，不需要 privileged container 或更改主機 namespace 設定。

先問 Codex：

```text
請只回覆 OK，不要執行工具。
```

收到模型回覆才確認模型連線成功。`login status` 成功只代表本機認證能被讀取。

退出 Codex 後應回到 Bash；不要退出 Bash，因為它是維持此 sandbox 的主程序。若主程序直接設為 Codex，退出 Codex 會讓 sandbox 進入 Completed，等待不會讓它回到 Ready。

## 10. 重現 agent 的網路拒絕與核准

### 10.1 先開即時日誌

另一個 WSL terminal：

```shell
mise exec -- openshell --gateway docker-dev logs codex-play \
  --tail -n 100 --level info
```

### 10.2 未核准的 curl

在 Codex 輸入：

```text
請執行 curl -i --max-time 15 https://api.github.com/zen。
若失敗，回報原始錯誤，不要嘗試繞過限制。
```

預期 curl 失敗，例如 exit 7、無法連上 port 443。判定必須同時找到對應 OpenShell 拒絕日誌，確認實際執行檔、目的地與原因，例如 `/usr/bin/curl`、`api.github.com:443`、`DENIED`。只有工具錯誤時，尚不能宣告測試通過。

```shell
mise exec -- openshell --gateway docker-dev logs codex-play \
  --since 10m -n 1000 --source sandbox |
  grep -iE 'curl|api\.github\.com|denied'
```

443 是 HTTPS 連線埠，不是 HTTP 狀態碼。curl exit 7 本身只證明連線失敗，不能單獨證明 policy 拒絕。

### 10.3 核准 curl 的唯讀 GitHub API 存取

在 sandbox 外：

```shell
mise exec -- openshell --gateway docker-dev policy update codex-play \
  --rule-name github_api \
  --binary /usr/bin/curl \
  --add-endpoint api.github.com:443:read-only:rest:enforce \
  --wait
```

請 Codex 重試相同 GET。預期收到 `200 OK` 與格言內容。

再請 Codex 執行：

```shell
curl -i --max-time 15 -X POST https://api.github.com/zen
```

此測試不提供認證或寫入資料。預期由 OpenShell 以 HTTP 403 / `policy_denied` 阻擋，而非 GitHub 自己拒絕。確認日誌有對應的 HTTP POST 拒絕；不能只憑遠端回傳的 403 判定。

| 操作 | 通過條件 |
| --- | --- |
| 未授權 curl → api.github.com:443 | 連線失敗，且有對應的 policy 拒絕日誌 |
| 加唯讀規則後 GET | 200 OK |
| 唯讀規則下 POST | OpenShell 回傳拒絕，且有對應的 HTTP POST 拒絕日誌 |

以上只驗證這組請求的預期行為，沒有驗證所有 HTTP method、重新導向或其他協定，也沒有在自己的上游服務獨立記錄請求是否到達。

## 11. 常見問題與診斷

| 症狀 | 確認／處置 |
| --- | --- |
| `jq: command not found` | 補裝 WSL 的 jq；Rust 編譯當時已成功 |
| `curl: command not found` | workload image 缺工具，不是 policy 拒絕；在 image build 安裝 curl |
| `invalid ID token format` | 寫入 auth.json 的 provider 值是 placeholder，無法被 Codex 當 JWT 解析 |
| `selected workspace missing from routing discovery` | 主機與 sandbox account ID 雜湊不同，且確認 sandbox 是替代值；改用獨立傳入的真實 account ID 後啟動成功 |
| `Logged in using ChatGPT` | 只確認本機登入檔可解析，仍要測模型回覆 |
| `bwrap: No permissions to create a new namespace` | Codex 內層 sandbox 無法啟動，curl 尚未執行，不能期待有 OpenShell 網路 DENIED |
| 日誌拒絕 `*.oaiusercontent.com` | 發出者是 Codex，不是目標 curl 實驗；不要混淆，也不要直接放行整個網域 |
| sandbox phase `Completed` | 主程序已退出；使用 Bash 作為主程序並在其中啟動 Codex，方便後續排查 |

Placeholder 檢查應使用 `contains`，不是只用 `startswith`。替代值可能不是 JWT，且 marker 不一定在字串開頭。檢查時僅輸出布林、長度與結構，不要印出 token。

OAuth 限制：上述 ID 佔位與真實 account ID 是 Codex 0.163.0 的實驗 workaround。短時間成功不代表 refresh token placeholder 能完成更新，也未驗證長時間使用、重新登入、token 撤銷或 provider rotation。不要把它當成正式長期認證整合。

## 12. Policy 在哪裡與如何重用

查看管理者的 base policy 與包含 provider 規則的 effective policy：

```shell
mise exec -- openshell --gateway docker-dev policy get codex-play --base
mise exec -- openshell --gateway docker-dev policy get codex-play --full
```

開發 gateway 的資料庫預設是 `.cache/gateway-docker/gateway.db`，policy update 存在 gateway，不會自動寫回本機 YAML。不要直接修改資料庫，也不要分享其副本，因為其中可能包含敏感狀態。

日常可將自己的完整 base policy 保存為例如 `agent-config/codex-policy.yaml`，建立新 sandbox 時加入 `--policy ./agent-config/codex-policy.yaml`。`policy get --base` 可能含 revision 等顯示資訊，保存時只取實際 policy YAML，不要直接把整個畫面當 policy 檔案，也不要將 provider-owned effective rules 複製為自己的規則。

其他來源包含 `OPENSHELL_SANDBOX_POLICY`、image 內 `/etc/openshell/policy.yaml` 與管理者 global policy。Global policy 會替代 sandbox policy，並非簡單追加限制，另見 repo 文件。

網路規則可在執行中更新；檔案系統、Landlock 與程序身份設定在建立／啟動時生效。新 sandbox 不會自動沿用先前在別的 sandbox 加的 GitHub 規則。

## 13. 參考

- [開發環境與 gateway](CONTRIBUTING.md)
- [Docker gateway 任務](tasks/gateway.toml)
- [Docker gateway 啟動腳本](tasks/scripts/gateway-docker.sh)
- [Codex provider 範例](providers/codex.yaml)
- [Provider 範例使用說明](providers/README.md)
- [Agent 範例的認證銜接](examples/agent-driven-policy-management/sandbox-agent.sh)
- [App-server 的不同認證／refresh 範例；不是本實驗的直接替代品](examples/codex-app-server/start-codex-app-server)
- [Sandbox policy 概念](docs/how-it-works/policies/overview.mdx)
- [Policy 管理](docs/how-it-works/policies/manage-policies.mdx)
- [網路規則實驗](docs/tutorials/first-network-policy.mdx)
- [安全回報方式](SECURITY.md)
- [Codex 認證官方文件](https://learn.chatgpt.com/docs/auth)
- [Docker host networking 官方文件](https://docs.docker.com/engine/network/drivers/host/)
