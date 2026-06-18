# 個人部落格建置規劃文件

> 目標：建立具備發布文章、留言功能的個人部落格，以 Cloudflare 生態系為核心，兼顧低成本與基本資安防護。

---

## 已確認決策

| 項目         | 決定                                        |
| ------------ | ------------------------------------------- |
| **網域名稱** | `watakushi-cloud.com`（已購買）             |
| **文章管理** | 原生 Markdown 檔案（存放於 Git repo）       |
| **留言系統** | Giscus（需登入 GitHub，使用者已接受此門檻） |

> 以下內容已依照上述決策調整，原本的「方案比較」段落保留供日後參考，但實作將直接採用已確認的選項。

---

## 一、整體架構概覽

```
使用者瀏覽器
     │
     ▼
Cloudflare CDN（快取、DDoS防護、WAF）
     │
     ▼
Cloudflare Pages（前端靜態網站，網域 Watakushi-Cloud.com）
     │
     ├─── 讀取文章 ──▶ Tina CMS（視覺化編輯）──▶ Git 儲存（Markdown/MDX）
     │
     └─── 留言區塊 ──▶ Giscus（嵌入式 Widget）──▶ GitHub Discussions（留言實際存放處）
```

> 採用 Giscus 後，留言資料直接存放在 GitHub Discussions，**不需要**自建 Cloudflare Workers + D1 留言系統，架構比自建方案更單純。

---

## 二、技術選型

### 2-1. 前端框架（三選一）

| 框架        | 適合情況               | 優點                        | 缺點       |
| ----------- | ---------------------- | --------------------------- | ---------- |
| **Astro**   | 推薦首選，內容導向網站 | 極快、支援 Markdown、彈性高 | 生態較新   |
| **Next.js** | 熟悉 React 的開發者    | 生態成熟、功能全面          | 比較複雜   |
| **Hugo**    | 純靜態、不需要 JS 邏輯 | 速度最快、最簡單            | 客製化有限 |

**建議選 Astro**：

- 原生支援 Markdown/MDX 寫文章
- 可以混用靜態頁面 + 動態 API（透過 Cloudflare Adapter）
- 非常適合部落格這種內容型網站

### 2-2. 文章管理方式（二選一）

**方案 A：Markdown 檔案存在 Git（推薦初期）**

- 文章直接寫成 `.md` 檔，放在專案資料夾裡
- 透過 GitHub 管理版本
- 優點：免費、簡單、版本控制清楚
- 缺點：需要用程式碼編輯器寫文章，不夠直覺

**方案 B：Headless CMS（後期可升級）**

- 用 Web UI 寫文章，不需要開程式碼
- 推薦選項：
  - **Sanity**（免費方案夠用）
  - **Contentful**（免費方案有限制）
  - **Tina CMS**（可整合 Git，很適合 Astro）
- 優點：有視覺化編輯界面，適合長期維護
- 缺點：多一個外部服務要管理

### 2-3. 留言系統（三選一）

| 方案             | 成本           | 難度 | 說明                                                     |
| ---------------- | -------------- | ---- | -------------------------------------------------------- |
| **Giscus**       | 免費           | 低   | 用 GitHub Discussions 當留言後端，需要讀者有 GitHub 帳號 |
| **自建（推薦）** | 免費（D1）     | 中   | 用 Cloudflare Workers + D1 自建，最彈性                  |
| **Disqus**       | 免費（含廣告） | 低   | 嵌入即用，但有廣告、隱私問題                             |

**建議選 Giscus（初期）或自建（進階）**：

- Giscus 最快速，適合初期啟動
- 自建可以完全控制留言資料，沒有第三方依賴

### 2-4. 圖片與媒體儲存

- **Cloudflare R2**：免費額度 10GB 儲存，每月 100 萬次讀取
- 文章內的圖片、封面圖都存在 R2
- 透過公開 URL 直接嵌入 Markdown

---

## 三、Cloudflare 服務清單與費用

### 使用到的服務

| 服務                     | 用途                 | 費用                               |
| ------------------------ | -------------------- | ---------------------------------- |
| **Cloudflare Registrar** | 購買網域             | .com 約 $9.77/年（成本價，無溢價） |
| **Cloudflare Pages**     | 託管靜態網站         | 免費（500 次建置/月）              |
| **Cloudflare Workers**   | 留言 API、後端邏輯   | 免費（10 萬次請求/天）             |
| **Cloudflare D1**        | 留言資料庫（SQLite） | 免費（5GB、每天 500 萬次讀取）     |
| **Cloudflare R2**        | 圖片、媒體儲存       | 免費（10GB、每月 100 萬次讀取）    |
| **Cloudflare CDN**       | 全球加速、快取       | 免費（含 DDoS 防護）               |
| **Cloudflare SSL/TLS**   | HTTPS 憑證           | 免費（自動發放、自動更新）         |
| **Cloudflare Turnstile** | 留言驗證（防機器人） | 免費                               |
| **Cloudflare WAF**       | Web 應用防火牆       | 免費（基本規則）                   |

### 預估費用

```
網域 .com：約 NT$300/年（~$10 USD）
所有其他服務：免費

每年總費用：約 NT$300
```

> 注意：若流量大幅超出免費額度才需要付費升級，個人部落格幾乎不會觸及上限。

---

## 四、資安規劃

### 4-1. Cloudflare 提供的防護（設定後自動生效）

- **DDoS 防護**：L3/L4/L7 攻擊自動緩解
- **WAF（Web 應用防火牆）**：阻擋常見攻擊（SQLi、XSS 等）
- **SSL/TLS 加密**：所有流量強制 HTTPS
- **Bot Management**：基本機器人過濾（免費方案）

### 4-2. 留言系統資安

- 使用 **Cloudflare Turnstile** 做留言驗證，防止垃圾留言（比 reCAPTCHA 更友善）
- 留言內容做 HTML 轉義（Sanitize），防止 XSS 注入
- 對留言 API 設定 Rate Limiting（每個 IP 每分鐘最多 N 次請求）

### 4-3. 內容管理資安

- 後台寫文章不需要對外公開（直接用 GitHub 或 CMS）
- 如果自建後台，需要加上身份驗證（Cloudflare Access 可免費做到）
- 定期備份 D1 資料庫（Cloudflare 有提供匯出工具）

### 4-4. 你需要手動注意的事項

- [ ] 網域 WHOIS 隱私保護（Cloudflare 預設免費提供）
- [ ] 開啟 DNSSEC（防 DNS 劫持，在 Cloudflare DNS 設定中啟用）
- [ ] 設定 SPF / DKIM（如果未來要用自訂網域寄信）
- [ ] 定期查看 Cloudflare Security Analytics，了解是否有異常流量

---

## 五、開發流程規劃

### 第一階段：基礎建置（1-2 週）

1. **申請網域並完成初始設定**（`watakushi-cloud.com` 已購買 ✅）

   - ~~到 Cloudflare Registrar 搜尋並購買網域~~（已完成）
   - 完成第九節的初始設定清單（DNS、SSL、DNSSEC、自動續約等）
   - 設定 DNS CNAME 指向 Cloudflare Pages（連接 Pages 時自動產生）

2. **建立 GitHub 儲存庫**

   - 新建一個 Git repo（例如 `my-blog`）
   - 用 Astro 初始化專案

3. **設定 Cloudflare Pages**

   - 連接 GitHub repo
   - 設定自動部署（每次 push 到 main 自動更新網站）

4. **完成基本頁面**
   - 首頁（文章列表）
   - 文章頁面
   - 關於我頁面
   - 基本 SEO（標題、描述、Open Graph）

### 第二階段：留言功能（1 週）

5. **整合留言系統**
   - 快速方案：嵌入 Giscus
   - 自建方案：
     - 建立 Cloudflare D1 資料庫
     - 撰寫 Cloudflare Workers API（新增留言、讀取留言）
     - 前端串接 API + Turnstile 驗證

### 第三階段：優化（持續）

6. **SEO 優化**

   - Sitemap 自動生成
   - RSS Feed
   - 結構化資料（Schema.org）

7. **效能優化**

   - 圖片使用 WebP 格式
   - Cloudflare Cache 規則調整
   - Core Web Vitals 監控

8. **CMS 升級（可選）**
   - 若未來覺得純 Markdown 不夠方便，可考慮加入 Tina CMS 等視覺化編輯器

---

## 六、你需要做的決策

在開始實作之前，以下問題需要先確認：

### 必須決定的

- [ ] **網域名稱**：想用什麼域名？（例如 `yourname.com`、`yourblog.tw`）
- [ ] **文章語言**：以繁體中文為主？還是中英混合？
- [ ] **文章管理**：接受用 Markdown + 程式碼編輯器寫文章？還是需要視覺化編輯界面？

### 可以晚一點決定的

- [ ] **留言方式**：Giscus（讀者需要 GitHub 帳號）或自建（所有人都能留言）
- [ ] **文章分類**：要做標籤（Tags）或分類（Categories）嗎？
- [ ] **搜尋功能**：文章多了之後是否需要站內搜尋（可用 Pagefind 免費實作）

---

## 七、推薦的技術組合（總結）

```
前端框架：  Astro
部署平台：  Cloudflare Pages
文章管理：  Markdown 檔案（原生，存於 Git repo）
ㄅ留言系統：  Giscus（初期），自建 Workers + D1（進階）
媒體儲存：  Cloudflare R2
驗證防護：  Cloudflare Turnstile + WAF
網域：      Cloudflare Registrar
每年費用：  約 NT$300（只有網域費）
```

---

## 八、參考資源

- [Astro 官方文件](https://docs.astro.build)
- [Cloudflare Pages 部署 Astro 教學](https://developers.cloudflare.com/pages/framework-guides/deploy-an-astro-site/)
- [Cloudflare D1 文件](https://developers.cloudflare.com/d1/)
- [Cloudflare Workers 文件](https://developers.cloudflare.com/workers/)
- [Giscus 留言系統](https://giscus.app)
- [Cloudflare Turnstile 文件](https://developers.cloudflare.com/turnstile/)

---

## 九、網域購買後初始設定清單

> 網域 `watakushi-cloud.com` 已購買完成，以下是部署 Cloudflare Pages 部落格之前必須完成的設定步驟，**建議照順序執行**。

### 9-1. DNS 記錄設定（DNS > Records）

Cloudflare Pages 連接網域後會自動產生 CNAME 記錄，但需確認以下設定正確：

| 類型    | 名稱          | 目標值                     | 雲朵    | 說明               |
| ------- | ------------- | -------------------------- | ------- | ------------------ |
| `CNAME` | `@`（根網域） | `<your-project>.pages.dev` | 橘色 ✅ | 主網域指向 Pages   |
| `CNAME` | `www`         | `watakushi-cloud.com`      | 橘色 ✅ | www 子網域重新導向 |

> Cloudflare Pages 在「自訂網域」設定後通常會自動建立這些記錄，確認一下即可。

### 9-2. 代理模式（橘色 vs 灰色雲朵）

每筆 DNS 記錄旁邊的雲朵圖示控制流量是否經過 Cloudflare：

- **橘色雲朵（Proxied）**：流量走 Cloudflare，享有 CDN、DDoS 防護、WAF，**且隱藏伺服器真實 IP**
- **灰色雲朵（DNS Only）**：只做 DNS 解析，直接連到 Pages，不享有任何防護

**部落格設定原則**：

- `@` 和 `www` 的 CNAME → **橘色**（享受全部防護）
- MX 記錄（Email）→ **灰色**（Email 不能走 Proxy）

### 9-3. SSL/TLS 設定（SSL/TLS > Overview）

選擇加密模式，**必須設定正確，否則會出現憑證錯誤或無限重新導向**：

| 模式              | 說明                                    | 適用情況    |
| ----------------- | --------------------------------------- | ----------- |
| Flexible          | 用戶到 CF 是 HTTPS，CF 到 Pages 是 HTTP | 不推薦      |
| Full              | 全程 HTTPS，不驗證憑證                  | 次選        |
| **Full (Strict)** | 全程 HTTPS，驗證憑證有效性              | **✅ 推薦** |

Cloudflare Pages 預設提供有效 SSL 憑證，因此可以直接選 **Full (Strict)**。

**另外需開啟的選項**（在 SSL/TLS > Edge Certificates）：

- [ ] **Always Use HTTPS**：自動將所有 HTTP 請求轉向 HTTPS
- [ ] **Automatic HTTPS Rewrites**：修正頁面內殘留的 HTTP 連結（防止 Mixed Content 警告）
- [ ] **HSTS**（選用）：開啟前確保 HTTPS 完全正常，一旦開啟難以回頭

### 9-4. 重新導向設定（Rules > Redirect Rules）

建議設定 `www` → 根網域（或反向），讓兩個版本都能正常訪問並統一：

**範例：強制將 `www.watakushi-cloud.com` 導向 `watakushi-cloud.com`**

在 **Rules > Redirect Rules** 新增規則：

- 條件：`Hostname equals www.watakushi-cloud.com`
- 動作：Dynamic Redirect → `concat("https://watakushi-cloud.com", http.request.uri.path)`
- 狀態碼：`301`（永久轉向）

### 9-5. DNSSEC（DNS > Settings）

開啟 DNSSEC 可防止 DNS 快取污染攻擊（有人偽造 DNS 回應把訪客導向假網站）。

- 在 **DNS > Settings** 頁面點擊「Enable DNSSEC」
- 因為 Registrar 也是 Cloudflare，**DS 記錄會自動設定**，不需要手動操作
- 開啟後等待幾分鐘生效即可

- [ ] 確認 DNSSEC 狀態顯示為 **Active**

### 9-6. 安全性強化（Security）

| 設定位置                | 建議操作                                              |
| ----------------------- | ----------------------------------------------------- |
| **Security > Bots**     | 開啟 **Bot Fight Mode**（免費，阻擋惡意爬蟲）         |
| **Security > Settings** | Security Level 設為 **Medium**（預設值，適合部落格）  |
| **WAF > Rate Limiting** | 可設定留言 API 端點的速率限制（付費功能，初期可略過） |

### 9-7. 自動續約確認（Domain Registration）

- [ ] 進入 **Domain Registration** 確認 `watakushi-cloud.com` 的 **Auto-renew 已開啟**
- [ ] 確認付款方式有效（避免到期時扣款失敗導致網域釋放）
- [ ] 確認 WHOIS 隱私保護已開啟（Cloudflare 預設免費提供，隱藏個人聯絡資訊）

### 9-8. Email 設定（若未來需要用自訂網域寄信）

若要使用 `@watakushi-cloud.com` 的 Email（如 Google Workspace 或 Zoho），需新增以下 DNS 記錄：

```
MX   @   指向 Email 服務商（灰色雲朵）
TXT  @   v=spf1 include:_spf.google.com ~all       （SPF，防止偽造寄件人）
TXT  @   v=DMARC1; p=quarantine; rua=mailto:...    （DMARC）
CNAME    郵件服務商提供的 DKIM 記錄（灰色雲朵）
```

> **初期若不需要自訂網域 Email 可跳過**，待有需求再設定。

### 9-9. 設定完成後的驗證清單

全部設定完成後，用以下方式確認正常：

- [ ] 瀏覽 `https://watakushi-cloud.com` → 應顯示 HTTPS 鎖頭
- [ ] 瀏覽 `http://watakushi-cloud.com` → 應自動導向 HTTPS
- [ ] 瀏覽 `https://www.watakushi-cloud.com` → 應導向根網域
- [ ] Cloudflare Dashboard 的 **SSL/TLS** 頁面顯示綠色 Active
- [ ] **DNS > Settings** 的 DNSSEC 顯示 Active

---

## 十、參考資源（補充）

- [Cloudflare Pages 自訂網域設定](https://developers.cloudflare.com/pages/configuration/custom-domains/)
- [Cloudflare SSL/TLS 加密模式說明](https://developers.cloudflare.com/ssl/origin-configuration/ssl-modes/)
- [Cloudflare DNSSEC 設定指南](https://developers.cloudflare.com/dns/dnssec/)
- [Cloudflare Redirect Rules](https://developers.cloudflare.com/rules/url-forwarding/)
