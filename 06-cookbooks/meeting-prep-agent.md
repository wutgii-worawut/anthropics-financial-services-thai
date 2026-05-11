---
title: Meeting Prep Agent
parent: 06 Managed Agent Cookbooks
nav_order: 6
---

# Meeting Prep Agent Cookbook

## โครงสร้าง Agent

**Meeting Prep Agent** เป็น managed agent ที่เตรียมเอกสารประชุม แนวทางการสนทนา และข้อมูลพื้นหลังสำหรับการประชุมกับลูกค้า หุ้นส่วน หรือนักลงทุน

```yaml
model: claude-opus-4-7
tools:
  - agent_toolset_20260401 (read, grep, glob เท่านั้น)
  - mcp_toolset: crm (CRM Data)
  - mcp_toolset: capiq (Capital IQ)
```

## Subagents

Meeting Prep Agent ประกอบด้วย 3 subagents:

| Subagent | บทบาท | หน้าที่ |
|----------|------|--------|
| `profiler` | Contact profiling | ดึงประวัติลูกค้า, meeting history, interests จาก CRM |
| `news-reader` | News & context | ดึง recent news, press releases, key developments |
| `pack-writer` | Document generation | เขียน meeting brief, talking points, background materials |

## Meeting Prep Workflow

Meeting Prep Agent สร้าง complete meeting package:

1. **Contact Intelligence** — ดึง company profile, previous interactions, preferences
2. **Recent Developments** — รวบรวม news, announcements, market events
3. **Customized Brief** — เขียน talking points customized สำหรับบุคคลและวันที่ประชุม
4. **Package** — สร้าง Word document + reference sheets

## การ Deploy

### 1. เตรียมสภาพแวดล้อม

```bash
export CRM_MCP_URL="https://your-crm-mcp-server"
export CAPIQ_MCP_URL="https://your-capiq-mcp-server"
```

### 2. Deploy Agent

```bash
claude-code deploy ./managed-agent-cookbooks/meeting-prep-agent/agent.yaml
```

### 3. เรียกใช้ via API

```bash
curl -X POST https://api.claude.ai/v1/agents/meeting-prep-agent/run \
  -H "Authorization: Bearer YOUR_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "contact_id": "crm_123456",
    "meeting_date": "2026-05-20",
    "meeting_type": "investor_update",
    "attendees": ["John Smith", "Jane Doe"],
    "duration_minutes": 60,
    "key_topics": ["Q2 guidance", "new product launch"]
  }'
```

## โครงสร้าง Output

```json
{
  "status": "completed",
  "contact": {
    "name": "Acme Capital Partners",
    "industry": "Asset Management",
    "aum": "45B",
    "our_relationship_duration": "5 years"
  },
  "meeting": {
    "date": "2026-05-20",
    "type": "investor_update",
    "attendees": ["John Smith - PM", "Jane Doe - Analyst"],
    "duration": 60
  },
  "relationship_context": {
    "last_meeting": "2026-03-15",
    "meeting_frequency": "Quarterly",
    "previous_key_discussions": ["ESG integration", "China exposure reduction"],
    "relationship_status": "Positive - recent large deployment"
  },
  "news_and_context": {
    "recent_news": [
      {
        "headline": "Acme Capital Launches Sustainable Tech Fund",
        "date": "2026-05-08",
        "relevance": "HIGH - aligns with our ESG focus"
      }
    ],
    "market_context": "Tech sector has outperformed by 8% YTD",
    "competitive_landscape": "2 new competitors in space"
  },
  "talking_points": {
    "company_updates": [
      "Q1 performance vs benchmark",
      "New product launches",
      "Team expansions"
    ],
    "investor_specific": [
      "Their tech exposure performed well in our portfolio",
      "Opportunity to increase allocation in new fund"
    ]
  },
  "potential_questions": [
    "How are we performing vs XYZ benchmark?",
    "What's our China allocation?",
    "Are we considering emerging markets?"
  ],
  "output_files": {
    "meeting_brief": "Acme_Capital_Meeting_Brief_2026_05_20.docx",
    "background_sheet": "Acme_Capital_Background.docx",
    "recent_news_summary": "News_Summary.xlsx"
  }
}
```

## ข้อมูล MCP Servers ที่ต้องการ

| Server | ตัวแปร Environment | ข้อมูลที่ให้บริการ |
|--------|-------------------|------------------|
| **CRM** | `CRM_MCP_URL` | Contact data, meeting history, relationship notes, preferences |
| **Capital IQ** | `CAPIQ_MCP_URL` | Company/fund data, performance benchmarks, portfolio composition |

## Meeting Brief Structure

Output document มี:

1. **Executive Summary** — Key points สำหรับการเปิดการประชุม
2. **Relationship Context** — Meeting history, key discussions
3. **Recent Developments** — News, announcements ที่เกี่ยวข้อง
4. **Talking Points** — By topic with supporting data
5. **Q&A Prep** — คำถามที่คาดว่าจะถูกถาม + คำตอบแนะนำ
6. **Follow-up Actions** — Items to track ระหว่าง/หลังการประชุม

## ความแตกต่างจาก Interactive Plugin

**Meeting Prep Plugin** (Interactive):
- ผู้ใช้ป้อน contact name หรือ ID
- Plugin ถามคำถามเพิ่มเติมเกี่ยวกับวัตถุประสงค์ของการประชุม
- Generates brief step-by-step

**Meeting Prep Cookbook** (Managed):
- API ให้ contact + meeting details
- Create brief อัตโนมัติ
- ส่งออก ready-to-read Word document

---

**หมายเหตุ**: Meeting Prep Agent ประหยัดเวลา 30-45 นาที ในการเตรียมการประชุมเมื่อเปรียบเทียบกับการค้นหาข้อมูลด้วยตนเอง
