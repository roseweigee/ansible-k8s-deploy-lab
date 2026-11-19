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
pip install ansible-core kubernetes ansible-lint
ansible-galaxy collection install -r requirements.yml
ansible --version
```

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

- 現象：「Apply Deployment and wait」那個 task 等 60 秒後失敗。
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

## 8. 清掉

```bash
ansible-playbook playbooks/teardown.yml
ansible-playbook -i inventories/prod/hosts.yml playbooks/teardown.yml --ask-vault-pass
kind delete cluster --name ansible-lab      # 不玩了再刪
```

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
