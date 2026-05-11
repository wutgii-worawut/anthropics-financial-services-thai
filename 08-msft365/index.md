---
title: 08 Microsoft 365 Add-in
nav_order: 9
has_children: true
permalink: /08-msft365/
---

# Microsoft 365 Add-in

`claude-for-msft-365-install/` — plugin สำหรับติดตั้ง Claude ใน **Excel / PowerPoint / Word** ระดับองค์กร (admin-provisioned)

## ทำไมต้องมี?

นักวิเคราะห์การเงิน **ใช้ Excel เป็นหลัก** — ถ้า agent ต้องสร้าง/แก้ไฟล์ .xlsx, .pptx ต้องเข้าถึงได้ผ่าน Microsoft Graph API

## โครงสร้าง

```
claude-for-msft-365-install/
├── .claude-plugin/
│   └── plugin.json
├── commands/           # slash commands สำหรับ admin provision
├── examples/           # ตัวอย่างไฟล์ Excel/PowerPoint
├── scripts/            # script สำหรับ tenant setup
└── README.md
```

อ่านรายละเอียดที่ [Microsoft 365 Add-in](./msft365.md)
