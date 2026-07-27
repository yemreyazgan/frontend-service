# 📊 Frontend Service - Jira to GitHub Issue Migration Project

Bu depo, **frontend-service** modülüne ait kullanıcı arayüzü geliştirmelerinin, profil yönetimi bileşenlerinin, güvenlik modallarının ve bunlara ait alt görevlerin (subtask), efor puanlarının (Story Point) ve sürüm bilgilerinin Jira ortamından GitHub Issues mimarisine kurumsal standartlarda taşınması (migration) amacıyla geliştirilmiştir.

---

## 🎯 Proje Yapısı ve Eşleşmeler

Jira'daki `Jira_Repo2.csv` veri seti üzerinden okunan frontend süreçleri, GitHub mimarisine birebir ve dinamik olarak şu şekilde entegre edilmiştir:

* **Özgün Görev Kodları (Jira Keys):** Jira'daki her bir görev ve alt görev, özgün anahtar kodlarıyla (`MS-13`, `MS-17`, `MS-20` vb.) GitHub Issue başlıklarına birebir taşınmıştır:
  * Ana Task Formatı: `[MS-13] Profil Fotoğrafı Yükleme ve Önizleme Arayüzü (Frontend)`
  * Subtask Formatı: `[MS-14] Avatar Upload & Crop Modal Komponentinin Çizilmesi`
* **Hiyerarşik Sub-Issue Yapısı:** Subtask'lar GitHub'ın **Sub-Issue API** entegrasyonu kullanılarak doğrudan ebeveyn görevlerinin içindeki **Sub-issues (Alt Görevler)** bileşenine dinamik olarak bağlanmıştır.
* **Proje Metadataları (Metadata Table):** Her Issue'nun en üstüne kurumsal formatta Markdown tablosu eklenmiş ve Jira markup'larındaki gereksiz karakterler temizlenmiştir:
  * **Jira Key:** Özgün görev kodu.
  * **Story Point:** Efor puanı.
  * **Fix Version & Affects Version:** Sürüm ve ortam bilgileri.
  * **Build Info:** Derleme/Build detayları.
  * **Due Date:** Bitiş/Teslim tarihi.
* **İş İçerikleri (Description):** Görevlerin teknik detayları ve açıklamaları hatasız bir şekilde biçimlendirilerek taşınmıştır.
* **Sorumlu Atamaları (Assignee):** Taşınan tüm görevler, projenin takibi için ilgili geliştiriciye (`yemreyazgan`) otomatik olarak atanmıştır.

---

## 🏷️ Kurumsal Etiket (Label) Standartları

Sürecin daha şeffaf ve izlenebilir olması için görevler dinamik ve renkli etiketlerle kategorize edilmiştir:

### 1. Öncelik Seviyeleri (Priority)
* `Priority: High` (Kırmızı) -> Kritik ve öncelikli çözülmesi gereken işler.
* `Priority: Medium` (Sarı) -> Standart geliştirme süreçleri.
* `Priority: Low` (Yeşil) -> Düşük öncelikli işler.

### 2. İş Tipleri (Issue Type)
* `Type: Story` (Mavi) -> Kullanıcı hikayeleri ve genel iş gereksinimleri.
* `Type: Task` (Mor) -> Teknik geliştirmeler ve operasyonel işler.
* `Type: Bug` (Koyu Kırmızı) -> Hata düzeltmeleri.
* `Type: Subtask` (Yeşil) -> Alt görev bileşenleri.

### 3. Efor Bağlantıları
* `Points: X` (Gri) -> Görevin efor karmaşıklığını belirten dinamik etiket.

---

## 💻 Geliştirilen Otomasyon Kodu (import_repo2.js)

`Jira_Repo2.csv` dosyasındaki çok satırlı açıklamaları, tırnak işaretlerini ve karmaşık sütun yapılarını `csv-parse` kütüphanesiyle ayrıştırıp `frontend-service` reposuna Sub-Issue hiyerarşisiyle aktaran script aşağıdadır:

```javascript
const fs = require("fs");
const axios = require("axios");
const { parse } = require("csv-parse/sync");

const GITHUB_TOKEN = process.env.GITHUB_TOKEN || "YOUR_GITHUB_TOKEN_HERE"; 
const REPO_OWNER = "yemreyazgan";
const REPO_NAME = "frontend-service"; 
const CSV_FILE_PATH = "Jira_Repo2.csv"; 

const colors = {
    "Priority: High": "d93f0b",
    "Priority: Medium": "fbca04",
    "Priority: Low": "0e8a16",
    "Type: Story": "1d76db",
    "Type: Task": "5319e7",
    "Type: Bug": "d73a4a",
    "Type: Subtask": "0e8a16"
};

async function ensureLabel(labelName, customColor) {
    if (!labelName || labelName.endsWith(':')) return;
    try {
        await axios.post(`[https://api.github.com/repos/$](https://api.github.com/repos/$){REPO_OWNER}/${REPO_NAME}/labels`, {
            name: labelName,
            color: customColor || colors[labelName] || "cccccc"
        }, {
            headers: { 
                "Authorization": `token ${GITHUB_TOKEN}`, 
                "Accept": "application/vnd.github.v3+json",
                "User-Agent": "Migration" 
            }
        });
    } catch (e) {
        // Label zaten varsa pas geç
    }
}

async function startMigration() {
    try {
        console.log(`📄 ${CSV_FILE_PATH} okunuyor ve Sub-Issue hiyerarşisi kuruluyor...`);

        let content = fs.readFileSync(CSV_FILE_PATH, "utf8");
        if (content.startsWith('\uFEFF')) {
            content = content.slice(1);
        }

        const records = parse(content, {
            columns: true,
            skip_empty_lines: true,
            trim: true
        });

        const parentTasks = {};
        const rawSubtasks = [];

        records.forEach(row => {
            const issueType = row["Issue Type"] ? row["Issue Type"].trim() : "";
            const issueKey = row["Issue key"] ? row["Issue key"].trim() : "";
            let summary = row["Summary"] ? row["Summary"].trim() : "";

            if (!summary) return;

            summary = summary.replace(/^Task\s*\d+:\s*/i, "");

            if (issueType === "Subtask") {
                const parentKey = row["Parent key"] ? row["Parent key"].trim() : "";
                rawSubtasks.push({
                    key: issueKey,
                    summary: summary,
                    parentKey: parentKey,
                    priority: row["Priority"] || "Medium"
                });
            } else {
                const rawDesc = row["Description"] || "";

                const affectsMatch = rawDesc.match(/\*?Affects Version:\*?\s*([^\n]+)/i);
                const buildMatch = rawDesc.match(/\*?Build Info:\*?\s*([^\n]+)/i);

                let affectsVersion = affectsMatch ? affectsMatch[1].replace(/[*\{\}]/g, "").trim() : "N/A";
                let buildInfo = buildMatch ? buildMatch[1].replace(/[*\{\}]/g, "").trim() : "N/A";

                let cleanDesc = rawDesc
                    .replace(/\*?\s*\*?Affects Version:\*?[^\n]+\n*/gi, "")
                    .replace(/\*?\s*\*?Build Info:\*?[^\n]+\n*/gi, "")
                    .replace(/^---/gm, "")
                    .trim();

                const storyPoints = row["Custom field (Story point estimate)"] || "N/A";
                const fixVersion = row["Fix versions"] || "N/A";
                const priority = row["Priority"] || "Medium";
                
                let rawDueDate = row["Due date"] || "N/A";
                let dueDate = (rawDueDate && rawDueDate !== "N/A") ? rawDueDate.split(" ")[0] : "N/A";

                parentTasks[issueKey] = {
                    key: issueKey,
                    type: issueType || "Task",
                    summary: summary,
                    description: cleanDesc,
                    storyPoints: storyPoints !== "" ? storyPoints : "N/A",
                    fixVersion: fixVersion !== "" ? fixVersion : "N/A",
                    affectsVersion: affectsVersion,
                    buildInfo: buildInfo,
                    dueDate: dueDate,
                    priority: priority,
                    githubIssueNumber: null,
                    githubNodeId: null,
                    subtaskIssues: []
                };
            }
        });

        console.log(`📌 Ana Görev Sayısı: ${Object.keys(parentTasks).length}`);
        console.log(`📌 Bağlanacak Subtask Sayısı: ${rawSubtasks.length}\n`);

        console.log(`🚀 1. Adım: Ana Görevler [${REPO_NAME}] reposuna yükleniyor...\n`);

        for (const key in parentTasks) {
            const t = parentTasks[key];

            const priorityLabel = `Priority: ${t.priority}`;
            const typeLabel = `Type: ${t.type}`;
            const pointLabel = `Points: ${t.storyPoints}`;

            await ensureLabel(priorityLabel);