# Agent 上傳安全指南

此文件說明如何為 agent 安全地授權並讓 agent 將檔案上傳到 orlysankuai-sudo/Naivi 倉庫。**切勿把私鑰或 token 放到倉庫內容中（包括本檔案）**。

推薦方法（優先順序）
1. Deploy key（單 repo、可限寫入/唯讀） — 推薦用於單一 repo 的 agent。私鑰保存在 agent 或 secret manager 中，公鑰加入本 repo 的 Settings → Deploy keys。
2. Fine-grained PAT（針對 Naivi repo 的內容寫入與 PR 權限）— 適合需要使用 API 的 agent。
3. GitHub App（長期自動化、最細粒度）— 複雜但最安全。

必要資訊（agent 必須有）
- repo: orlysankuai-sudo/Naivi
- git remote (SSH): git@github.com:orlysankuai-sudo/Naivi.git
- git remote (HTTPS): https://github.com/orlysankuai-sudo/Naivi.git
- target_branch: e.g. feature/upload-from-agent
- base_branch: main
- auth_method: deploy_key 或 fine-grained-pat
- 若 deploy_key: 私鑰檔案 (保存在 agent 環境，路徑由 agent 讀取)
- 若 PAT: token 儲存在安全 secret（環境變數或 secret manager）
- commit_message、author name、author email

快速安全步驟（deploy key 範例）
1) 在安全端生成 key（在你的機器或 agent 的安全環境）：
   ssh-keygen -t ed25519 -C "naivi-deploy-key" -f naivi_deploy_key -N ""
   結果：naivi_deploy_key（私鑰，不能公開）、naivi_deploy_key.pub（公鑰，可提供）

2) 把公鑰內容（naivi_deploy_key.pub）加入到倉庫設定：
   Settings → Deploy keys → Add deploy key → 勾選 "Allow write access"（如果 agent 需要 push）

3) 把私鑰放在 agent 的安全儲存（例：Vault、GitHub Actions secret、CI secret），不要提交到 repo。

4) agent 使用私鑰推送（範例流程見 scripts/agent_upload.sh）：
   - 啟動 ssh-agent，載入私鑰
   - git init / add / commit
   - git remote add origin git@github.com:orlysankuai-sudo/Naivi.git
   - git push -u origin <branch>

若要透過 API 自動把公鑰加為 deploy key（需要 admin token）：
- 請在安全環境執行（不要貼 token 在聊天）：
  curl -X POST -H "Authorization: token $ADMIN_TOKEN" \
    -H "Accept: application/vnd.github+json" \
    https://api.github.com/repos/orlysankuai-sudo/Naivi/keys \
    -d '{"title":"naivi-deploy","key":"<PUBLIC_KEY_CONTENT>","read_only":false}'

GitHub Actions 範例
- 另一個安全選項是把 agent 的任務放到 GitHub Actions（若 agent 可以上傳 artifact 或觸發 workflow），使用 Actions 的 secret 或 GITHUB_TOKEN 執行 commit & push。Actions 的 secret 需由你在 repository Settings → Secrets 設定。

風險與緩解
- 絕對不要在 repo 裡放私鑰或 token。即便是 private repo 也不行。
- 使用最小權限：deploy key（單 repo）或 fine-grained PAT（限 Naivi repo，Contents write、Pull requests as needed）。
- 設定過期與輪換策略，並監控異常活動（force push、非預期時間的 push）。

下一步
- 我已在倉庫新增一個 agent 上傳腳本範本（scripts/agent_upload.sh）與一個 workflow 範本（.github/workflows/agent-dispatch-upload.yml）。
- 如果你同意，我可以示範如何在本地生成 deploy key 並使用 API 把公鑰加入倉庫（我會提供命令，你在安全端執行以保證私鑰不外流）。

如需我直接幫你把公鑰加入（用 API），請在安全通道提供一個臨時 admin token；否則請在本地依照上面的步驟生成 key 並把公鑰貼到倉庫設定。