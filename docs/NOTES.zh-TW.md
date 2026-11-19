# 實作筆記：用 Ansible 把應用程式部署到 Kubernetes

目標：自己寫 role、inventory、group_vars，用 Ansible 把一個小網頁部署到 kind，分 dev 和 prod 兩個環境；再故意弄壞幾次，練習看錯誤訊息找原因。

每一步做完請截圖，存到 `docs/screenshots/`，檔名照下面的編號。

---

## 0. 事前準備（Mac）

```bash
# Docker Desktop 要先開著
kind get clusters
```

- 如果有上次 KServe lab 的 cluster 也可以用，但它比較吃資源，建議另開一個乾淨的：

```bash
kind create cluster --name ansible-lab
kubectl config current-context        # 應該是 kind-ansible-lab
```

## 1. 安裝 Ansible

```bash
cd ~/ansible-k8s-deploy-lab
python3 -m venv .venv
source .venv/bin/activate
pip install ansible-core kubernetes hvac ansible-lint
ansible-galaxy collection install -r requirements.yml
ansible --version
```

- `hvac` 是 HashiCorp Vault 的 Python client，第 9 步會用到。
- 第 9 步還需要 `helm`：`helm version` 看有沒有，沒有就 `brew install helm`。

為什麼用 venv：Ansible 的 k8s module 要用到 Python 的 `kubernetes` 套件，裝在 venv 裡不會弄亂 Mac 系統的 Python。inventory 裡的 `ansible_python_interpreter: "{{ ansible_playbook_python }}"` 就是讓 Ansible 用同一個 venv 的 Python。

## 2. 先離線檢查（不需要 cluster）

```bash
ansible-playbook tests/render.yml
ansible-lint
```

- `render.yml` 會把每個 template 產出到 `out/dev/`，確認都是合法 YAML、replicas 正確。
- `ansible-lint` 用 production 等級檢查，應該是 `Passed`。

📸 `01-render-and-lint.png`

## 3. 部署 dev

```bash
ansible-playbook playbooks/deploy.yml
```

順序：檢查變數 → Namespace → ConfigMap → Secret → Deployment（等到 Available 才繼續）→ Service。

看結果：

```bash
kubectl get all -n hello-dev
ansible-playbook playbooks/verify.yml
```

`verify.yml` 會檢查 ready replicas，再進 pod 裡抓網頁，確認內容是 `env=dev`。

📸 `02-deploy-dev.png`、`03-verify-dev.png`

## 4. 冪等性：再跑一次

```bash
ansible-playbook playbooks/deploy.yml
```

預期 PLAY RECAP 是 `changed=0`：狀態已經對了，就不再改。這是 Ansible 跟「每次都重跑 shell script」最大的差別，面試很常問。

如果不是 0，看是哪個 task 顯示 changed，用 `--diff` 再跑一次看差在哪，這本身就是一題好的排查練習，記下來。

📸 `04-idempotent.png`

## 5. 改設定前先預覽：`--check --diff`

把 `inventories/dev/group_vars/all/main.yml` 的 `app_message` 改成別的字，然後：

```bash
ansible-playbook playbooks/deploy.yml --check --diff     # 只預覽，不會真的改
ansible-playbook playbooks/deploy.yml                    # 真的套用
kubectl get pods -n hello-dev -w                         # 會看到 pod 換新
```

為什麼 pod 會換：Deployment 的 pod template 有一個 `checksum/html` annotation，是網頁內容的 hash。內容變了 → hash 變了 → Kubernetes 做 rolling update。少了這個，ConfigMap 改了 pod 也不會重啟。

📸 `05-check-diff.png`、`06-rollout.png`

## 6. prod：用 ansible-vault 保護密鑰

```bash
cp inventories/prod/vault.example.yml inventories/prod/group_vars/all/vault.yml
# 把 change-me 改成你自己的值
ansible-vault encrypt inventories/prod/group_vars/all/vault.yml
cat inventories/prod/group_vars/all/vault.yml           # 開頭是 $ANSIBLE_VAULT;1.1;AES256

ansible-playbook -i inventories/prod/hosts.yml playbooks/deploy.yml --ask-vault-pass
ansible-playbook -i inventories/prod/hosts.yml playbooks/verify.yml --ask-vault-pass
```

- prod 是 2 個 replicas，namespace `hello-prod`。
- `vault.yml` 在 `.gitignore` 裡，不會上 GitHub；repo 只放 `vault.example.yml`。
- Secret 的 task 有 `no_log: true`，就算加 `-vvv` 也不會把 token 印在畫面上。

📸 `07-vault-encrypted.png`、`08-deploy-prod.png`

## 7. 故障演練（重點）

每一題：先跑、看錯誤、寫下你判斷的原因，再修好。

### 7-1 image 打錯

```bash
ansible-playbook playbooks/deploy.yml -e app_image=nginxinc/nginx-unprivileged:does-not-exist -e app_rollout_timeout=60
```

- 現象：「Apply Deployment and wait until the rollout completes」那個 task 等 60 秒後失敗。
- 第一版的坑（GitHub Actions 抓到的）：原本寫 `wait_condition: type: Available`，結果 image 打錯 playbook 竟然顯示成功。因為舊 Pod 還在跑，Deployment 一直是 Available，條件馬上成立，沒等新版本。改成只寫 `wait: true`，module 會等 rollout 本身完成（新的 generation 被看到、每個 replica 都更新且 available），壞掉的 rollout 才會被擋下來。面試可以講：「部署成功」要看 rollout，不是看服務還活著。
- 排查：

```bash
kubectl get pods -n hello-dev
kubectl describe pod -n hello-dev <新的 pod 名稱> | tail -20
```

  會看到 `ErrImagePull` / `ImagePullBackOff`。舊的 pod 還在跑，因為新 pod 沒 ready，rolling update 不會把舊的砍掉，服務沒中斷。
- 修復：不帶 `-e app_image` 再跑一次 `deploy.yml`。

📸 `09-drill-bad-image.png`

### 7-2 變數型別錯

```bash
ansible-playbook playbooks/deploy.yml -e app_replicas=three
```

- 現象：第一個 task「Validate input variables」就失敗，還沒碰到 cluster。
- 重點：把錯誤擋在最前面，訊息直接說哪個變數錯。

📸 `10-drill-bad-var.png`

### 7-3 忘了給 vault 密碼

```bash
ansible-playbook -i inventories/prod/hosts.yml playbooks/deploy.yml
```

- 現象：`Attempting to decrypt but no vault secrets found`。
- 原因：prod 的 group_vars 有加密檔，沒給密碼就讀不到。加 `--ask-vault-pass`。

📸 `11-drill-vault.png`

### 7-4 用 `-vvv` 看細節

```bash
ansible-playbook playbooks/deploy.yml -vvv 2>&1 | less
```

- 可以看到每個 task 用的 Python、送給 module 的參數、API 回來的結果。
- Secret 那個 task 只會顯示 `censored`（因為 `no_log`）。

## 8. 清掉 prod

接下來第 9 步會沿用 dev（`hello-dev`）和這個 cluster，所以這裡只清 prod：

```bash
ansible-playbook -i inventories/prod/hosts.yml playbooks/teardown.yml --ask-vault-pass
```

---

## 9. 進階：改用 HashiCorp Vault 管密鑰

前面 prod 用的是 **ansible-vault**：密鑰加密放在 git 裡，部署時解開，再建成 K8s Secret。
這一步改用 **HashiCorp Vault**：密鑰集中放在 Vault 服務裡，有權限控管和稽核紀錄。分兩種用法，兩種都做。

| 模式（`app_secret_mode`） | 密鑰從哪裡來 | K8s 裡有沒有 Secret |
| --- | --- | --- |
| `ansible`（預設） | Ansible 變數（prod 用 ansible-vault 加密） | 有 |
| `hashi_lookup` | 部署時 Ansible 去 HashiCorp Vault 讀 | 有 |
| `hashi_injector` | Pod 啟動時自己用 ServiceAccount 登入 Vault 拿 | **沒有** |

### 9-1 安裝 Vault（dev 模式）

```bash
ansible-playbook playbooks/vault-install.yml
kubectl get pods -n vault          # vault-0 和 vault-agent-injector-xxx 都要 Running
```

- dev 模式：資料放記憶體、自動解封、root token 是 `root`。只適合 lab，Vault 重啟資料就沒了。
- 用的是 `kubernetes.core.helm` module，所以連 Helm 安裝也是 Ansible 管的。

📸 `12-vault-install.png`

### 9-2 設定 Vault

開**另一個終端機**，讓它一直開著：

```bash
kubectl port-forward -n vault svc/vault 8200:8200
```

回到原本的終端機：

```bash
export VAULT_TOKEN=root
ansible-playbook playbooks/vault-configure.yml       # 會要你輸入要存的 token，自己編一個
```

> **Mac 才有的坑：`A worker was found in a dead state`**
> 9-3 的 `hashi_lookup` 在 Mac 上可能直接出現這個錯誤，worker 程序被系統砍掉。原因是 Ansible 會 fork worker 執行 task，而 `hvac`/`requests` 在 Mac 上會讀系統 proxy 設定，呼叫到 Objective-C；macOS 的 fork 安全機制看到 fork 後的子程序用 Objective-C，就直接終止它。Linux（包括 GitHub Actions）不會發生。
> 解法：在同一個終端機先設定
> ```bash
> export OBJC_DISABLE_INITIALIZE_FORK_SAFETY=YES
> export no_proxy='*'
> ```
> 注意：deploy 中斷後如果接著跑 verify，看到的會是**上一次**的部署結果，不代表這次成功。要看 deploy 自己的 PLAY RECAP。

它做了四件事：
1. 開啟 **Kubernetes auth**：讓 Pod 可以用自己的 ServiceAccount token 登入 Vault。
2. 建 policy `hello-dev-read`：**只能讀** `secret/hello/dev` 這一個路徑。
3. 建 role `hello-dev`：只有 namespace `hello-dev` 裡、名叫 `hello` 的 ServiceAccount 才能用這個 policy。
4. 把 token 存進 KV v2：`secret/hello/dev`。

注意：`vault_write` 這類 module 每次都會回報 changed（它不會先比對），所以這個 playbook 第二次跑不會是 `changed=0`。只有 `vault_kv2_write` 會先比對內容。面試被問冪等性時，這是一個誠實的例外。

📸 `13-vault-configure.png`

### 9-3 模式一：`hashi_lookup`（Ansible 讀 Vault）

```bash
ansible-playbook playbooks/deploy.yml -e app_secret_mode=hashi_lookup
ansible-playbook playbooks/verify.yml
```

- 密鑰不再放在 git 的任何地方，部署時才從 Vault 讀。
- 但 K8s 裡還是會有 Secret，用 `kubectl get secret hello-token -n hello-dev` 看得到。

📸 `14-hashi-lookup.png`

### 9-4 模式二：`hashi_injector`（Pod 自己去 Vault 拿）

```bash
ansible-playbook playbooks/deploy.yml -e app_secret_mode=hashi_injector
ansible-playbook playbooks/verify.yml -e app_secret_mode=hashi_injector
kubectl get pod -n hello-dev -o jsonpath='{.items[0].spec.initContainers[*].name}'; echo
kubectl get secret -n hello-dev                     # hello-token 已經不見了
```

流程：
1. Deployment 的 pod template 多了 `vault.hashicorp.com/*` annotation。
2. Injector（一個 mutating webhook）看到後，幫 Pod 加一個 `vault-agent-init` init container。
3. init container 用 Pod 的 ServiceAccount token 登入 Vault（role `hello-dev`），把密鑰寫到 `/vault/secrets/api-token`。
4. 主程式啟動時直接讀檔案，**K8s 的 etcd 裡完全沒有這個密鑰**。

`verify.yml` 會檢查：有 `vault-agent-init`、檔案存在且不是空的（不會印出內容）、K8s Secret 已經刪掉。

有設 `agent-pre-populate-only: "true"`：只在啟動時拿一次，不留 sidecar，比較省資源；代價是 Vault 裡的值改了，Pod 要重啟才會拿到新的。

📸 `15-hashi-injector.png`

### 9-5 Vault 故障演練

**A. 忘了 port-forward**

把 port-forward 那個終端機關掉，再跑：

```bash
ansible-playbook playbooks/deploy.yml -e app_secret_mode=hashi_lookup
```

- 現象：task 失敗，結果顯示 `censored`，因為那個 task 有 `no_log`。
- 但上面會先印出真正原因：`Connection refused ... port=8200`。
- 重點：`no_log` 保護了密鑰，也會把錯誤藏起來。排查時要往上看 lookup 的錯誤訊息，不要只看 `censored`。

**B. ServiceAccount 對不上**

```bash
ansible-playbook playbooks/deploy.yml -e app_secret_mode=hashi_injector -e app_service_account=wrong-sa -e app_rollout_timeout=60
```

- 現象：等 60 秒失敗。新 Pod 卡在 `Init:0/1`。
- 排查：

```bash
kubectl get pods -n hello-dev
kubectl logs -n hello-dev <卡住的 pod> -c vault-agent-init
```

  log 會看到 `permission denied` 或 `service account name not authorized`：Vault 的 role 只允許 `hello` 這個 ServiceAccount。
- 修復：不帶 `-e app_service_account` 再部署一次。

📸 `16-drill-vault-sa.png`

### 9-6 全部清掉

```bash
ansible-playbook playbooks/teardown.yml               # 刪 hello-dev
helm uninstall vault -n vault
kubectl delete namespace vault
kind delete cluster --name ansible-lab                # 不玩了再刪
```

#---

## 實測紀錄（2026-11-19，MacBook Pro + kind v1.37.0）

| 步驟 | 結果 |
| --- | --- |
| render + lint | `dev: manifests render, replicas=1`；ansible-lint production `Passed` |
| 部署 dev | `changed=6 failed=0`；verify `1/1 replicas ready`、`Page OK: env=dev` |
| 冪等性 | 第二次 `changed=0` |
| 7-1 image 打錯 | 新 Pod `ErrImagePull` → `ImagePullBackOff`；Events：`...:does-not-exist: not found`；舊 Pod 持續 `Running` |
| 7-2 變數錯 | 停在 Validate input variables；`assertion: app_replicas \| string is match('^[0-9]+$')` |
| 7-3 沒給 vault 密碼 | `Attempting to decrypt but no vault secrets found`，第一個 task 前就停 |
| prod（ansible-vault） | `2/2 replicas ready in hello-prod` |
| Vault 安裝 | `vault-0`、`vault-agent-injector` 都 `1/1 Running` |
| hashi_lookup | Secret `changed`，Deployment `ok`（Pod 沒重啟，見下方說明） |
| hashi_injector | `vault-agent-init ran`；`Token delivered by Vault ... no K8s Secret`；`get secret` → `No resources found` |
| 9-5 錯的 ServiceAccount | Pod 卡 `Init:0/1`；vault-agent-init log：`Code: 403 ... service account name not authorized` |

### 實際踩到的坑

1. **參數多了一個字元**：`-e app_rollout_timeout=60~`（終端機多送了 `~`）→ `argument 'wait_timeout' ... "60~" cannot be converted to an int`。module 先檢查參數型別，錯了就不會送到 cluster，所以 `get pods` 只看到舊 Pod。
2. **Mac 上 worker 死掉**：`A worker was found in a dead state`（見 9-2 的說明），設 `OBJC_DISABLE_INITIALIZE_FORK_SAFETY=YES`、`no_proxy='*'` 解決。
3. **hashi_lookup 換了 Secret，Pod 沒重啟**：Secret 是 `changed`，Deployment 是 `ok`。環境變數只在容器啟動時讀一次，所以 Pod 拿到的還是舊值。要嘛把 Secret 的 hash 也放進 pod template annotation 觸發 rollout，要嘛用 injector。
4. **Available 不等於部署成功**：9-5 失敗時，Deployment status 同時是 `Available: True` 和 `unavailableReplicas: 1`。只等 Available 條件會把壞掉的 rollout 判成成功，所以 role 用 `wait: true` 等 rollout 本身完成。

---

## 面試可以講的重點（HashiCorp Vault）

1. **ansible-vault 和 HashiCorp Vault 差在哪**：ansible-vault 是「加密檔案」，密碼要另外保管，誰有密碼誰就能看全部；HashiCorp Vault 是「密鑰服務」，每個應用只拿得到自己那條路徑，誰在什麼時候讀了什麼都有紀錄。
2. **Kubernetes auth**：Pod 不需要事先知道任何密碼，用 ServiceAccount 證明自己是誰，Vault 再依 role 決定給什麼。
3. **Injector 的好處**：密鑰不進 etcd、不進 git、不進 Ansible 的變數。銀行稽核最在意這點。
4. **最小權限**：policy 只給 read、只給一個路徑；role 綁死 namespace 和 ServiceAccount。
5. **誠實說明**：這是 dev 模式的 lab，正式環境還要處理 HA、unseal（例如用 KMS auto-unseal）、稽核 log 和備份，這些沒有做。

---

## 面試可以講的重點

1. **為什麼用 role + 兩個 inventory**：同一份部署邏輯，環境差異只放在 group_vars，加一個 UAT 只要多一個資料夾。
2. **冪等性**：第二次跑 `changed=0`；用 `--check --diff` 先預覽再套用，銀行的變更管理很需要這個。
3. **密鑰**：ansible-vault 加密、加密檔不進 git、`no_log` 防止印到 log。
4. **ConfigMap 改了 pod 不會重啟**：用 checksum annotation 解決。
5. **排查流程**：Ansible 的錯誤只告訴你哪個 task 失敗，原因要回到 Kubernetes 看（`get pods`、`describe`、events）；這和在 Kyndryl 看 playbook 錯誤找原因是同一套方法，現在加上自己會寫。

## 跟 OpenShift 的關係

- 同一個 role 換成 OpenShift 也能跑：`kubernetes.core.k8s` 打的是 API，`oc login` 之後 kubeconfig 就指向 OpenShift。
- OpenShift 的 restricted SCC 不准用 root 跑，所以 deployment 已經設 `runAsNonRoot`、drop 所有 capabilities，image 用 nginx-unprivileged（port 8080）。
