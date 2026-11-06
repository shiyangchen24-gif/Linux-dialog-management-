#  Linux `dialog` 建立藥品管理介面

- 環境：Ubuntu 24.04、Bash、dialog 1.3-20240101、curl、jq（AI 助理功能使用）

- Github：{%preview https://github.com/shiyangchen24-gif/Linux-dialog-management- %}
---

## 一、專案說明

用 Bash 和 `dialog` 套件，在終端機中寫出一個有圖形化選單的**藥品管理系統**。系統要能新增、列出、查詢、修改庫存和刪除藥品，也要能提醒庫存不足和快要過期的藥品。我加入了 **AI 藥品助理**：透過AI分析庫存並回答問題。

最後完成的檔案：

| 檔案 | 內容 |
|------|------|
| `drug_manager.sh` | 主程式 |
| `drugs.sample.db` | 建立的 5 筆範例藥品資料 |
| `images/` | 執行過程的截圖 |


---

## 二、建立過程

### 步驟 1：安裝 dialog

- 先安裝 `dialog`，：

```bash
sudo apt update
sudo apt install -y dialog
dialog --version          # 確認安裝成功，顯示 Version: 1.3-20240101
```

- 介面要顯示為中文，確認終端機使用 UTF-8 語系：

```bash
export LANG=C.UTF-8
```

---

### 步驟 2：dialog 的基本用法

- 我先查了 `man dialog`，整理出所有元件的共同格式：

```bash
dialog [共用選項] --元件類型 "提示文字" 高度 寬度 [其他參數]
```
---

#### 2-1 把使用者輸入存進變數

```bash
name=$(dialog --inputbox "名稱" 8 40 3>&1 1>&2 2>&3)
```

- `3>&1`：先用 fd 3 記住原本的 stdout
- `1>&2`：stdout 改送到 stderr，畫面才會出現在終端機
- `2>&3`：stderr（使用者輸入）送到 fd 3，被 `$( )` 抓到

---

- 按鈕的結果是用結束代碼 `$?` 判斷：

| `$?` | 代表 |
|------|------|
| 0 | 按 OK / Yes |
| 1 | 按 Cancel / No |
| 255 | 按 ESC |


---


### 步驟 3：設計資料格式

- 將藥品資料存在純文字檔 `drugs.db`，一行一筆，用 `|` 分隔
- 日期用 `YYYY-MM-DD` 格式，之後判斷是否過期時，可以直接用字串比大小。


```
編號|藥品名稱|類別|數量|單位|有效期限|存放位置
D001|普拿疼|成藥|120|盒|2027-08-31|A-01
D002|阿莫西林|處方藥|15|盒|2027-03-15|B-02
D003|胃藥錠|指示藥|60|瓶|2026-12-20|A-03
D004|嗎啡注射液|管制藥品|8|支|2027-01-10|C-01
D005|維他命C|保健食品|200|罐|2026-10-01|D-05
```
---

### 步驟 4：程式的設定區和共用函式
- `10#` 避免 `008` 被 Bash 當成八進位而出錯。

```bash
#!/usr/bin/env bash
export LANG=${LANG:-C.UTF-8}

APP_TITLE="藥品管理系統"
DB_FILE="${DB_FILE:-$(dirname "$0")/drugs.db}"
ADMIN_PASS="${ADMIN_PASS:-admin123}"
LOW_STOCK=20          # 低於 20 算庫存不足
EXPIRE_DAYS=90        # 90 天內到期要提醒
TMP=$(mktemp)
trap 'rm -f "$TMP"; clear' EXIT   # 程式結束時刪除暫存檔、清除畫面

touch "$DB_FILE"

next_id() {   # 自動產生下一個編號 D001、D002...
    local last
    last=$(cut -d'|' -f1 "$DB_FILE" | sed 's/^D//' | sort -n | tail -1)
    printf "D%03d" $(( 10#${last:-0} + 1 ))
}

msg() {       # 共用的訊息框
    dialog --backtitle "$APP_TITLE" --title "$1" --msgbox "$2" 14 50
}
```

---

### 步驟 5：登入畫面（`--passwordbox`）
- 新增 `--insecure` ，輸入密碼時會顯示 `*`，使用者才知道自己打了幾個字

```bash
login() {
    local pass
    pass=$(dialog --backtitle "$APP_TITLE" --title "管理員登入" \
        --insecure --passwordbox "請輸入管理員密碼：" 8 40 3>&1 1>&2 2>&3) || exit 0
    if [ "$pass" != "$ADMIN_PASS" ]; then
        msg "登入失敗" "密碼錯誤，程式結束。"
        exit 1
    fi
}
```

---

### 步驟 6：載入進度條（`--gauge`）
- `--gauge` 會讀stdin（0～100）來更新進度，所以用 `for` 迴圈，再傳進去

```bash
loading() {
    for i in 0 20 40 60 80 100; do
        echo "$i"; sleep 0.3
    done | dialog --backtitle "$APP_TITLE" --title "系統啟動" \
        --gauge "正在載入藥品資料庫..." 8 50 0
}
```

---

### 步驟 7：主選單（`--menu`）

- 主選單放在 `while true` 迴圈，每個功能做完都會回到主選單

```bash
main_menu() {
    local choice
    while true; do
        choice=$(dialog --backtitle "$APP_TITLE" --title "主選單" \
            --cancel-label "離開" \
            --menu "請選擇功能（↑↓ 移動，Enter 確認）：" 17 50 8 \
            1 "新增藥品"  2 "藥品清單"  3 "查詢藥品" \
            4 "修改庫存（進貨/領用）"  5 "刪除藥品" \
            6 "庫存與效期警示"  7 "AI 藥品助理"  0 "離開系統" \
            3>&1 1>&2 2>&3) || choice=0
        case "$choice" in
            1) add_drug ;;     2) list_drugs ;;   3) search_drug ;;
            4) update_stock ;; 5) delete_drug ;;  6) alert_report ;;
            7) ai_assistant ;;
            0) dialog --backtitle "$APP_TITLE" --title "離開" \
                   --yesno "確定要離開藥品管理系統嗎？" 7 40 && break ;;
        esac
    done
}
```

---

### 步驟 8：新增藥品（`--form` + `--radiolist` + `--calendar`）

- (1) `--form` 填基本資料：每個欄位要 8 個參數（標籤、標籤位置、預設值、輸入框位置、輸入框寬度、最大長度）。送出後每欄各輸出一行，所以用 `read` 

```bash
form=$(dialog --title "新增藥品" \
    --form "請填寫藥品資料（↑↓ 切換欄位）：" 14 60 4 \
    "藥品名稱：" 1 1 ""   1 14 30 40 \
    "數    量：" 2 1 ""   2 14 10 10 \
    "單    位：" 3 1 "盒" 3 14 10 10 \
    "存放位置：" 4 1 ""   4 14 20 30 \
    3>&1 1>&2 2>&3) || return
{ read -r name; read -r qty; read -r unit; read -r loc; } <<< "$form"

# 檢查：名稱不能空白，數量要是數字
if [ -z "$name" ] || ! [[ "$qty" =~ ^[0-9]+$ ]]; then
    msg "輸入錯誤" "藥品名稱不可空白，數量必須為正整數。"; return
fi
```

---
- (2) `--radiolist` 選類別（只能單選，用空白鍵勾選）

```bash
cat=$(dialog --title "藥品類別" \
    --radiolist "請選擇「$name」的類別（空白鍵選取）：" 14 50 5 \
    "處方藥" "需醫師處方" on   "指示藥" "需藥師指示" off \
    "成藥" "可自行購買" off    "管制藥品" "需特別管理" off \
    "保健食品" "非藥品類" off \
    3>&1 1>&2 2>&3) || return
```

---
(3) `--calendar` 選有效期限：預設回傳格式是 `日/月/年`，加上 `--date-format` 改成 `YYYY-MM-DD`，才能和資料檔格式一致

```bash
exp=$(dialog --title "有效期限" --date-format "%Y-%m-%d" \
    --calendar "請選擇「$name」的有效期限：" 0 0 3>&1 1>&2 2>&3) || return

id=$(next_id)
echo "$id|$name|$cat|$qty|$unit|$exp|$loc" >> "$DB_FILE"
```

---
### 步驟 9：藥品清單（`--textbox`）

- `--textbox` 用來顯示檔案，所以先把表格寫進暫存檔 `$TMP`，再顯示出來

```bash
pad() {   # pad "文字" 寬度
    local w
    w=$(printf '%s' "$1" | wc -L)
    printf '%s%*s' "$1" $(( $2 > w ? $2 - w : 0 )) ""
}

list_drugs() {
    {
        echo "$(pad 編號 6)$(pad 藥品名稱 16)$(pad 類別 10)$(pad 數量 6)$(pad 單位 6)$(pad 有效期限 12)位置"
        printf -- '-%.0s' {1..64}; echo
        while IFS='|' read -r id name cat qty unit exp loc; do
            [ -n "$id" ] && echo "$(pad "$id" 6)$(pad "$name" 16)$(pad "$cat" 10)$(pad "$qty" 6)$(pad "$unit" 6)$(pad "$exp" 12)$loc"
        done < "$DB_FILE"
        echo; echo "共 $(grep -c . "$DB_FILE") 筆藥品資料"
    } > "$TMP"
    dialog --title "藥品清單" --exit-label "返回" --textbox "$TMP" 20 76
}
```


---
### 步驟 10：查詢藥品（`--inputbox`）

- 使用 `grep -i` 找關鍵字，再用 `awk` 排版

```bash
key=$(dialog --title "查詢藥品" \
    --inputbox "請輸入藥品名稱、編號或類別關鍵字：" 8 50 3>&1 1>&2 2>&3) || return
result=$(grep -i -- "$key" "$DB_FILE" | \
    awk -F'|' '{printf "[%s] %s\n   類別:%s  數量:%s %s\n   期限:%s  位置:%s\n\n",$1,$2,$3,$4,$5,$6,$7}')
[ -z "$result" ] && result="查無符合「$key」的藥品。"
dialog --title "查詢結果：$key" --msgbox "$result" 18 60
```

---
### 步驟 11：修改庫存（動態產生選單）

- 讓使用者從**現有的藥品**中選一個，所以選單內容不寫死。先使用一個函式輸出「編號、說明」交錯的清單，再用 `mapfile` 讀成陣列，丟給 `--menu`

```bash
drug_items() {
    while IFS='|' read -r id name cat qty unit exp loc; do
        [ -n "$id" ] && printf '%s\n%s\n' "$id" "$name (庫存 $qty $unit)"
    done < "$DB_FILE"
}

mapfile -t items < <(drug_items)
id=$(dialog --title "修改庫存" --menu "請選擇要異動的藥品：" 16 60 8 \
    "${items[@]}" 3>&1 1>&2 2>&3) || return
```
---

- 選完藥品、異動類型（進貨 IN／領用 OUT）、數量後，使用 `awk` 更改第 4 欄的數量。領用時，如果數量不夠，就跳提示，不扣庫存

```bash
if [ "$action" = "IN" ]; then
    newqty=$(( qty + amount ))
else
    newqty=$(( qty - amount ))
    [ "$newqty" -lt 0 ] && { msg "庫存不足" "目前庫存只有 $qty，無法領用 $amount。"; return; }
fi
awk -F'|' -v OFS='|' -v id="$id" -v q="$newqty" '$1==id{$4=q}1' \
    "$DB_FILE" > "$TMP" && cat "$TMP" > "$DB_FILE"
```
---

### 步驟 12：庫存與效期警示

- 使用 `date` 算出今天和 90 天後的日期，再逐筆檢查藥品
- 因為日期格式是 `YYYY-MM-DD`，所以直接用 `[[ "$exp" < "$today" ]]` 比較字串即可，不用轉換成數字

```bash
today=$(date +%Y-%m-%d)
limit=$(date -d "+$EXPIRE_DAYS days" +%Y-%m-%d)
# 數量 < 20          → 庫存不足
# 期限 < today       → 已過期
# today ≤ 期限 ≤ limit → 即將到期
```

--- 

### 步驟 13：刪除藥品（`--yesno`）

- 刪除比較危險，所以加上 `--defaultno`，讓游標預設停在「No」，避免使用者不小心按 Enter 刪除

```bash
if dialog --title "確認刪除" --defaultno \
    --yesno "確定要刪除「$id $name」嗎？\n此動作無法復原！" 8 45; then
    grep -v "^$id|" "$DB_FILE" > "$TMP"; cat "$TMP" > "$DB_FILE"
    msg "刪除完成" "已刪除：$id $name"
fi
```

---
### 步驟 14：AI 藥品助理（呼叫 Claude API）

- 主程式完成後，再加入 AI 助理功能。概念是把**問題和目前的藥品資料**一起送給 AI，再把 AI 的回答用 dialog 顯示出來

```
dialog 選單／輸入框 → Bash 變數 → jq 組 JSON → curl 呼叫 Claude API → jq 取出回答 → dialog --textbox 顯示
```

#### **(1) 安裝套件、設定金鑰**

```bash
sudo apt install -y curl jq
export ANTHROPIC_API_KEY="你的金鑰"     # 金鑰不寫進程式，也不上傳 GitHub
```

- 在設定區多加一個變數，方便以後換模型：

```bash
AI_MODEL="${AI_MODEL:-claude-opus-5-5}"
```

---
#### **(2) 寫 `ai_ask` 函式：組 JSON、呼叫 API、取出回答**

```bash
ai_ask() {   # ai_ask "問題" → 輸出 AI 的回答文字
    local body resp
    body=$(jq -n --arg model "$AI_MODEL" --arg q "$1" \
        --rawfile db "$DB_FILE" --arg today "$(date +%F)" '{
        model: $model,
        max_tokens: 4000,
        output_config: {effort: "low"},
        fallbacks: "default",
        system: "你是藥局庫存管理助理。只根據提供的藥品資料回答，使用繁體中文，回答精簡、條列清楚，不使用 Markdown 語法。涉及用藥或醫療判斷時，提醒以醫師或藥師判斷為準。",
        messages: [{role: "user",
            content: ("今天日期：" + $today
                + "\n藥品資料（編號|名稱|類別|數量|單位|有效期限|位置）：\n" + $db
                + "\n問題：" + $q)}]
    }')

    resp=$(curl -s --max-time 120 https://api.anthropic.com/v1/messages \
        -H "Content-Type: application/json" \
        -H "x-api-key: $ANTHROPIC_API_KEY" \
        -H "anthropic-version: 2026-10-21" \
        -H "anthropic-beta: server-side-fallback-2026-07-01" \
        -d "$body") || { echo "連線失敗，請檢查網路。"; return 1; }

    if [ "$(jq -r '.stop_reason // empty' <<< "$resp" 2>/dev/null)" = "refusal" ]; then
        echo "AI 無法回答這個問題，請換個方式提問。"
        return 1
    fi
    # 回應的 content 可能含 thinking 區塊，只取 type 為 text 的部分
    jq -r '.content[]? | select(.type == "text") | .text' <<< "$resp" 2>/dev/null | grep . \
        || { echo "呼叫失敗：$(jq -r '.error.message // "無法解析回應"' <<< "$resp" 2>/dev/null)"; return 1; }
}
```


#### **(3) 寫 `ai_assistant` 介面函式**
- 先提供 3 個常用問題，不用每次打字；也保留自行輸入的選項。
- 呼叫 API 需要幾秒，所以先用 `--infobox` 顯示「AI 分析中」。和 `--msgbox` 的差別是不會等使用者按鍵。
- **AI 只負責回答，不會修改到庫存資料 `drugs.db`**。藥品資料關係到用藥安全，進貨、領用、刪除還是要由人操作並確認。

```bash
ai_assistant() {
    local question
    for cmd in curl jq; do
        command -v "$cmd" >/dev/null || { msg "缺少套件" "AI 助理需要 $cmd，請先安裝：\n\nsudo apt install -y curl jq"; return; }
    done
    if [ -z "$ANTHROPIC_API_KEY" ]; then
        msg "尚未設定 API 金鑰" "請先在終端機設定 Claude API 金鑰再啟動程式：\n\nexport ANTHROPIC_API_KEY=\"你的金鑰\"\n./drug_manager.sh"
        return
    fi

    question=$(dialog --backtitle "$APP_TITLE" --title "AI 藥品助理" \
        --menu "請選擇常用問題，或選「自行輸入」：" 14 64 5 \
        1 "分析目前庫存，列出需要優先補貨的藥品" \
        2 "列出已過期和即將到期的藥品，並建議處理方式" \
        3 "管制藥品的庫存管理要注意什麼" \
        4 "自行輸入問題" \
        3>&1 1>&2 2>&3) || return
    case "$question" in
        1) question="分析目前庫存，列出需要優先補貨的藥品並說明原因" ;;
        2) question="列出已過期和即將到期的藥品，並建議處理方式" ;;
        3) question="根據目前資料，管制藥品的庫存管理要注意什麼" ;;
        4) question=$(dialog --backtitle "$APP_TITLE" --title "AI 藥品助理" \
               --inputbox "請輸入問題：" 8 60 3>&1 1>&2 2>&3) || return
           [ -z "$question" ] && return ;;
    esac

    # --infobox 顯示後不等按鍵，直接往下執行，適合當「處理中」提示
    dialog --backtitle "$APP_TITLE" --title "AI 藥品助理" \
        --infobox "AI 分析中，請稍候...\n\n問題：$question" 7 64
    ai_ask "$question" > "$TMP"
    dialog --backtitle "$APP_TITLE" --title "AI 回覆" \
        --exit-label "返回" --textbox "$TMP" 22 80
}
```

---

### 步驟 15：程式進入點

- 檔案最後依序呼叫三個函式：

```bash
login
loading
main_menu
```

- 完整程式碼於檔案： [`drug_manager.sh`](drug_manager.sh)。

---

## 三、執行過程與截圖

### 準備與啟動

```bash
chmod +x drug_manager.sh          # 給執行權限
cp drugs.sample.db drugs.db       # 載入範例資料
./drug_manager.sh
```

### 1. 輸入管理員密碼 `admin123`

![image](https://hackmd.io/_uploads/Bk15h_sTGl.png)


### 2. 登入後顯示載入進度條

![image](https://hackmd.io/_uploads/H1cc3_sTMl.png)


### 3. 進入主選單

![image](https://hackmd.io/_uploads/HkEjn_i6Ge.png)


### 4. 新增藥品：填入「布洛芬止痛錠」、數量 50、位置 B-03

![image](https://hackmd.io/_uploads/Sk12n_jaMl.png)


### 5. 用空白鍵把類別選為「指示藥」

![image](https://hackmd.io/_uploads/S1Fp3diaze.png)


### 6. 按 PgDn 翻到 2027 年 7 月，選擇有效期限

![image](https://hackmd.io/_uploads/BJzR2_spMg.png)


### 7. 新增成功，系統自動編號為 D006

![image](https://hackmd.io/_uploads/SklkTOipGe.png)

### 8. 查看藥品清單，D006 已經加入


![image](https://hackmd.io/_uploads/r1QZpus6ze.png)


### 9. 查詢：輸入關鍵字「處方」

![image](https://hackmd.io/_uploads/r1SMausTGe.png)


### 10. 查到 D002 阿莫西林

![image](https://hackmd.io/_uploads/S1mX6doTGg.png)


### 11. 修改庫存：選擇 D002 阿莫西林

![image](https://hackmd.io/_uploads/HyC7Tuoazl.png)


### 12. 選擇「OUT 領用」

![image](https://hackmd.io/_uploads/SJ5VTdoTze.png)


### 13. 輸入數量 5

![image](https://hackmd.io/_uploads/BJzSa_ipzg.png)


### 14. 庫存從 15 變成 10

![image](https://hackmd.io/_uploads/r1JIT_ipMx.png)


### 15. 查看警示：剛領用後只剩 10 盒的阿莫西林出現在「庫存不足」

- 畫面中也列出已過期的維他命C、90天內到期的胃藥錠和嗎啡注射液

![image](https://hackmd.io/_uploads/BJKIpOoaMx.png)


### 16. 刪除藥品：選擇剛剛新增的 D006

![image](https://hackmd.io/_uploads/rJ9u6uipfx.png)


### 17. 確認刪除（預設在 No，我按 ← 移到 Yes）

![image](https://hackmd.io/_uploads/SJHY6Ooafx.png)


### 18. 刪除完成

![image](https://hackmd.io/_uploads/S1S5TuiTfl.png)


### 19. 選擇 0 離開系統，按 Yes 結束程式

![image](https://hackmd.io/_uploads/HkEsTusTGx.png)


### 20～23. AI 藥品助理

**沒有設定 API 金鑰時**，選 `7` 會顯示設定方式，不會出錯：

![image](https://hackmd.io/_uploads/BJFa6dspzx.png)

---

**設定金鑰後**，重新啟動程式，選 `7` 進入 AI 助理的問題選單：

![image](https://hackmd.io/_uploads/r14yCOoaGg.png)

---

選「4 自行輸入問題」，輸入「哪些藥品需要優先補貨？」：

![image](https://hackmd.io/_uploads/ry1bA_ipzl.png)

---

送出後先顯示「AI 分析中」，接著顯示回覆

![image](https://hackmd.io/_uploads/Byy3AOjazx.png)

![image](https://hackmd.io/_uploads/S153COjpGe.png)



---

## 五、心得

1. 原本以為終端機只能做文字指令，用了 `dialog` 才發現 Shell Script 也能做出有選單、表單、日曆的介面，而且只需要一個套件
2. 學到最重要的觀念是 **stdout / stderr 和檔案描述子**。`3>&1 1>&2 2>&3` 要把每個 fd 的指向畫出來才比較好理解。
3. 處理中文時要注意「字元數」和「顯示寬度」不一樣，這是我之前沒想過的。
4. 加入 AI 助理後發現，Shell Script 不只能做介面，也能透過 `curl` + `jq` 串接網路 API。把 AI 定位成提供建議的工具，資料修改還是要經由專業人員確認，這在藥品管理這種需要安全的情境很重要。
6. 每個 dialog 元件的參數順序都不太一樣（特別是 `--form` 每欄要 8 個參數），寫之前查 `man dialog` 真的有幫助
7. 【使用ＡＩ審核我的設計方式】我之後可以再改進的地方：
   - 用 `--checklist` 一次刪除多筆
   - 把每次進貨／領用寫進紀錄檔，再用 `--tailbox` 查看歷史
   - 改用 `sqlite3` 存資料，支援更複雜的查詢

---

## 附錄：本次用到的 dialog 元件整理

| 元件 | 語法 | 回傳 | 我用在哪裡 |
|------|------|------|-----------|
| `--passwordbox` | `--passwordbox 文字 高 寬` | 輸入的字串 | 登入 |
| `--gauge` | `指令 \| dialog --gauge 文字 高 寬 %` | 無 | 載入進度 |
| `--menu` | `--menu 文字 高 寬 列數 tag 說明 ...` | 選中的 tag | 主選單、選藥品 |
| `--form` | `--form 文字 高 寬 表單高 [每欄 8 個參數]` | 每欄一行 | 新增藥品 |
| `--radiolist` | `--radiolist 文字 高 寬 列數 tag 說明 on/off ...` | 選中的 tag | 藥品類別 |
| `--calendar` | `--calendar 文字 高 寬` | 日期 | 有效期限 |
| `--textbox` | `--textbox 檔案 高 寬` | 無 | 清單、警示 |
| `--inputbox` | `--inputbox 文字 高 寬` | 輸入的字串 | 查詢、數量 |
| `--msgbox` | `--msgbox 文字 高 寬` | 無 | 各種提示訊息 |
| `--yesno` | `--yesno 文字 高 寬` | 結束代碼 0／1 | 刪除、離開確認 |
| `--infobox` | `--infobox 文字 高 寬` | 無（不等按鍵） | AI 分析中提示 |
