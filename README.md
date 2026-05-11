# Claude for Financial Services — คู่มือภาษาไทย

เอกสารแปลและอธิบายภาษาไทยของ [`anthropics/financial-services`](https://github.com/anthropics/financial-services) — ครอบคลุมทุกโฟลเดอร์ ทุก agent ทุก vertical

**🌐 อ่านออนไลน์**: https://wutgii-worawut.github.io/anthropics-financial-services-thai/

## โครงสร้าง

```
.
├── _config.yml                    # Jekyll + just-the-docs config
├── Gemfile                        # Ruby deps
├── index.md                       # หน้าแรก
├── 01-overview/                   # ภาพรวม
├── 02-folders/                    # โครงสร้างทุกโฟลเดอร์
├── 03-agents/                     # 10 agent plugins
├── 04-verticals/                  # 7 vertical plugins
├── 05-partners/                   # LSEG + S&P Global
├── 06-cookbooks/                  # Managed Agent cookbooks
├── 07-scripts/                    # scripts/ — orchestrate, sync, validate
├── 08-msft365/                    # Microsoft 365 add-in
├── 09-deployment/                 # การ deploy
├── 10-development/                # การ dev — validation, CI
└── .github/workflows/pages.yml    # build + deploy
```

## รันที่เครื่อง (Local Preview)

```bash
bundle install
bundle exec jekyll serve
# เปิด http://localhost:4000/anthropics-financial-services-thai/
```

## License

ต้นทาง: ดู [LICENSE ของ anthropics/financial-services](https://github.com/anthropics/financial-services/blob/main/LICENSE)
เอกสารแปลภาษาไทย: เผยแพร่เพื่อการศึกษา

---

แปล/เรียบเรียงโดย [Apex Oracle](https://github.com/wutgii-worawut/apex-oracle) 🌳📚
