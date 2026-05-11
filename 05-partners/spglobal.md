---
title: S&P Global
parent: 05 Partner Plugins
nav_order: 2
---

# S&P Global Plugin

## S&P Global คือใคร

**S&P Global** (ผ่าน Kensho Technologies) เป็นผู้ให้บริการข้อมูลการเงิน การให้คะแนน และข้อมูลความเสี่ยงชั้นนำของโลก บริษัทนี้รู้จักกันจากผลิตภัณฑ์หลัก:

- **Capital IQ** — ฐานข้อมูลบริษัท, ข้อมูลการทำธุรกรรม, ประมาณการนักวิเคราะห์
- **S&P Global Ratings** — คะแนนสินเชื่อ, ความเสี่ยง, มุมมองของตลาด
- **Market Intelligence** — ข้อมูลการทำธุรกรรม, ข้อมูลแข่งขัน, แนวโน้มตลาด

S&P Global Plugin (ที่สร้างโดย Kensho) นำเสนอ skills ที่ออกแบบมาสำหรับ AI agents ซึ่งให้ข้อมูล real-time และเครื่องมือสร้าง documents

## ผลิตภัณฑ์และความสามารถ

Plugin นี้มี 3 skills หลัก:

### 1. Tearsheet (Company Snapshot)

สร้างเอกสาร Word ขนาด 1-2 หน้า สำหรับบริษัทเฉพาะ โดยใช้ข้อมูล live จาก Capital IQ

**ประเภท:**
- **Equity Research** — ภาพรวมวิทยานิพนธ์การลงทุน สำหรับนักวิเคราะห์ (buy-side/sell-side)
- **Investment Banking / M&A** — โปรไฟล์บริษัทในบริบทธุรกรรม
- **Corporate Development** — โปรไฟล์เป้าหมายการ M&A สำหรับทีมกลยุทธ์ภายใน
- **Sales / Business Development** — เตรียมประชุมสำหรับทีมขาย

**ตัวอย่าง:** `"Generate a business development tearsheet for Palantir"`

### 2. Industry Transaction Summaries

สรุปกิจกรรม M&A และการทำธุรกรรมล่าสุดในภาคส่วนหรือสำหรับบริษัทเฉพาะ โดยใช้ Capital IQ

**ใช้สำหรับ:**
- การแมปตลาด (Market mapping)
- การเตรียมข้อมูลการสนับสนุน (Pitch preparation)
- สืบสวนข้อมูลประสบการณ์ (Competitive intelligence)

**ตัวอย่าง:** `"Summarize recent transactions in the data infrastructure space"`

### 3. Earnings Preview

สร้างภาพรวมการรายงานผลลัพธ์ที่กำลังจะเกิดขึ้น รวมถึง:
- ประมาณการ consensus (EPS, revenue, EBITDA)
- guidance ล่าสุด
- ความรู้สึกของนักวิเคราะห์
- ประเด็นสำคัญที่ต้องจับตา

ทั้งหมดมาจาก Capital IQ

**ตัวอย่าง:** `"Give me an earnings preview for Salesforce"`

## Data Sources และ MCP Integration

Plugin นี้ใช้ **S&P Global LLM-Ready API** (ผ่าน Kensho) ซึ่งให้บริการผ่าน MCP server

### ประเภทข้อมูลที่ใช้ได้

| ประเภท | รายละเอียด |
|--------|----------|
| **Company Data** | โปรไฟล์บริษัท, ข้อมูลการเงิน, ประมาณการนักวิเคราะห์ |
| **Transaction Data** | M&A, venture financing, strategic investments |
| **Earnings Data** | ประมาณการ, guidance, ผลลัพธ์ในอดีต |
| **Market Intelligence** | ข่าวสาร, แนวโน้ม, ข้อมูลแข่งขัน |
| **Sector Coverage** | ข้อมูลภาคส่วน, ความเชื่อมโยง, comparable companies |

## การใช้งาน

### ติดตั้งใน Claude Desktop

1. ไปที่ **Cowork** tab
2. คลิก **Customize with Plugins**
3. ค้นหา **S&P Global** ในรายการ Personal
4. ยืนยันการเข้าถึงด้วย S&P Global credentials เมื่อได้รับพร้อมท์

Skills จะเปิดใช้งานโดยอัตโนมัติตามบริบท คุณสามารถเรียกใช้โดยพิมพ์ `/` เพื่อดูคำสั่ง

### Customization

คุณสามารถปรับแต่ง plugin สำหรับ template, terminology, หรือ workflow ของบริษัทได้ โดยการคลิก **Customize** ในขณะดู plugin

## ข้อกำหนด

- สิทธิ์การเข้าถึง [S&P Global LLM-Ready API](https://www.marketplace.spglobal.com/en/solutions/kensho-llm-ready-api) หรือ [Capital IQ Pro](https://www.spglobal.com/market-intelligence/en/solutions/products/sp-capital-iq-pro)
- Claude Pro, Max, Team, หรือ Enterprise plan (หากใช้ใน Cowork)
- Claude Desktop app สำหรับ macOS หรือ Windows

## ความแตกต่างจาก Vertical Plugins

ต่างจาก vertical plugins (equity-research, investment-banking เป็นต้น) ที่มุ่งเน้นไปที่ workflow ด้านหนึ่ง S&P Global Plugin เน้นการให้ข้อมูล real-time จากแหล่งข้อมูล Capital IQ ที่มีอำนาจ

---

**เว็บไซต์**: [S&P Global LLM-Ready API](https://www.marketplace.spglobal.com/en/solutions/kensho-llm-ready-api)  
**การสนับสนุน**: contact [commercial@kensho.com](mailto:commercial@kensho.com)
