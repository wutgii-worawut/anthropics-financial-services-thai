---
title: LSEG
parent: 05 Partner Plugins
nav_order: 1
---

# LSEG Financial Analytics Plugin

## LSEG คือใคร

**LSEG** (London Stock Exchange Group) หรือที่รู้จักกันในชื่อ **Refinitiv** เป็นผู้ให้บริการข้อมูลทางการเงิน ความเชี่ยวชาญ และเครื่องมือวิเคราะห์ที่อันดับต้น ๆ ของโลก บริษัทนี้ให้บริการแก่สถาบันการเงิน บริษัทลงทุน และสถาบันอื่น ๆ ที่ต้องการข้อมูล real-time เกี่ยวกับตลาดการเงิน

LSEG Financial Analytics Plugin นำเสนอการเข้าถึงข้อมูลและเครื่องมือของ LSEG ผ่าน AI agent ทำให้ผู้ใช้สามารถวิเคราะห์หลักทรัพย์ ประเมินค่า และพัฒนากลยุทธ์ได้อย่างมีประสิทธิภาพ

## ประเภทข้อมูลที่ LSEG Plugin ให้บริการ

Plugin นี้ให้การเข้าถึงข้อมูลหลากหลาย:

| ประเภท | รายละเอียด |
|--------|----------|
| **ข้อมูลพันธบัตร** | ราคา อัตราผลตอบแทน ระยะเวลา และระบบการจ่ายเงิน |
| **ข้อมูลอัตราแลกเปลี่ยน** | ราคา spot, forward, และการวิเคราะห์ carry trade |
| **เส้นอัตราผลตอบแทน** | เส้นเลื่อนราคาหรือเส้นสวาป การเปรียบเทียบกับเส้นรัฐบาล |
| **ข้อมูลสวาป** | ราคาการแลกเปลี่ยนอัตราดอกเบี้ย DV01 และความเสี่ยง |
| **ข้อมูลออปชั่น** | ราคา Greeks (delta, gamma, vega, theta) วิเคราะห์ความผันผวน |
| **ข้อมูลความผันผวน** | Surface ความผันผวนของหลักทรัพย์ต่าง ๆ แบบ SABR |
| **ข้อมูลประมาณการและพื้นฐาน** | ประมาณการของนักวิเคราะห์, ข้อมูลทางการเงินของบริษัท, ราคาหุ้นในอดีต |
| **ข้อมูลเศรษฐศาสตร์มหภาค** | ตัวชี้วัดเศรษฐกิจ, เส้นเลื่อนผลตอบแทน, อัตราจริง |
| **ข้อมูลพันธบัตรอ้างอิง** | การอ้างอิง, นำเงิน, สถานการณ์สมมติ, ความเสี่ยง (YieldBook) |

## Commands ของ LSEG Plugin

LSEG Plugin มี 8 commands หลักที่ออกแบบมาสำหรับงานวิเคราะห์ทั่วไป:

```
/analyze-bond-rv          # วิเคราะห์มูลค่าสัมพัทธ์พันธบัตร (relative value)
/analyze-fx-carry         # ประเมินโอกาสการทำ FX carry trade
/research-equity          # สร้างภาพรวมการวิจัยหุ้นอย่างรวดเร็ว
/analyze-swap-curve       # วิเคราะห์เส้นอัตราการแลกเปลี่ยน
/analyze-option-vol       # วิเคราะห์ความผันผวนของออปชั่น
/review-fi-portfolio      # ทบทวนพอร์ตโฟลิโอพันธบัตร
/macro-rates              # สร้างแดชบอร์ด macroeconomics และอัตรา
/analyze-bond-basis       # วิเคราะห์ฐาน (basis) พันธบัตรฟิวเจอร์ส
```

แต่ละ command ประกอบด้วยการผสมผสาน 4-5 tool calls เพื่อให้การวิเคราะห์ที่สมบูรณ์แบบ แทนที่จะเรียกใช้เครื่องมือทีละชิ้น

## Skills และความรู้เชิงลึก

ทั้ง 8 commands ได้รับการสนับสนุนด้วย skills ที่ประกอบด้วยความเชี่ยวชาญเชิงลึก:

| Skill | ความรู้เชิงลึก |
|-------|------------|
| `bond-relative-value` | กรอบการวิเคราะห์ spread, G-spread/Z-spread/OAS, การวิเคราะห์ rich-cheap |
| `fx-carry-trade` | กลศาสตร์ carry, อัตราส่วน carry-to-vol, พลวัตของ carry ใน G10 และ EM |
| `equity-research` | การตีความประมาณการ IBES, การวิเคราะห์พื้นฐาน, ตัวชี้วัดการประเมินค่า |
| `swap-curve-strategy` | การสร้างเส้นสวาป, กลยุทธ์การซื้อขาย, การวิเคราะห์อัตราจริง |
| `option-vol-analysis` | การตีความ vol surface, โมเดล SABR, Greeks, ความผันผวนโดยนัยเทียบกับความผันผวนจริง |
| `fixed-income-portfolio` | การวิเคราะห์พอร์ตโฟลิโอ, key rate duration, การวิเคราะห์กระแสเงิน, การทดสอบสถานการณ์สมมติ |
| `macro-rates-monitor` | ตัวชี้วัดเศรษฐศาสตร์มหภาค, รูปแบบของเส้นเลื่อนผลตอบแทน, อัตราจริง, เงื่อนไขทางการเงิน |
| `bond-futures-basis` | กลศาสตร์ CTD (Cheapest-to-Deliver), การคำนวณ basis, implied repo rate, ตัวเลือกการส่งมอบ |

## การเชื่อมต่อ MCP (MCP Connectors)

LSEG Plugin ใช้ **LFA MCP Server** (LSEG Financial Analytics) เพื่อให้การเข้าถึงข้อมูล ไม่จำเป็นต้องมี connector เพิ่มเติม

### หมวดหมู่เครื่องมือ (Tool Categories)

| หมวดหมู่ | เครื่องมือหลัก | รายละเอียด |
|---------|------------|----------|
| **Bond Pricing** | `bond_price`, `bond_future_price` | ประเมินค่าพันธบัตร, key rate duration, DV01 |
| **FX Pricing** | `fx_spot_price`, `fx_forward_price` | อัตราแลกเปลี่ยน spot และ forward |
| **Interest Rate Curves** | `interest_rate_curve`, `inflation_curve` | เส้นเลื่อนของรัฐบาล, breakeven inflation |
| **Credit Curves** | `credit_curve` | เส้น spread ด้านเครดิต |
| **FX Curves** | `fx_forward_curve` | เส้น forward points |
| **Options** | `option_value`, `option_template_list` | ประเมินค่า, Greeks |
| **Swaps** | `ir_swap` | ประเมินค่า interest rate swaps |
| **Volatility Surfaces** | `fx_vol_surface`, `equity_vol_surface` | ผิวความผันผวน SABR |
| **Quantitative Analytics** | `qa_ibes_consensus`, `qa_company_fundamentals`, `qa_macroeconomic` | ประมาณการ, พื้นฐาน, ข้อมูลมหภาค |
| **Time Series** | `tscc_historical_pricing_summaries` | ข้อมูลราคาในอดีต |
| **Fixed Income Analytics** | `yieldbook_*`, `fixed_income_risk_analytics` | ข้อมูลอ้างอิงพันธบัตร, OAS, duration |

## ติดตั้ง

```bash
claude plugins add lseg
```

## ข้อกำหนด

- การเข้าถึง LFA MCP Server ที่ได้รับการยืนยันด้วย credentials
- สิทธิการใช้ข้อมูล LSEG สำหรับผลิตภัณฑ์ที่เกี่ยวข้อง

---

**สำหรับข้อมูลเพิ่มเติม** ให้ดู [CONNECTORS.md](../origin/plugins/partner-built/lseg/CONNECTORS.md) สำหรับการอ้างอิง tool ที่สมบูรณ์
