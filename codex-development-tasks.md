# Codex Development Tasks: Automated Web Scraping for US Stock Logos

## 1. Task Brief

พัฒนา automation tool สำหรับดึงโลโก้หุ้น US กลุ่ม Russell 1000 จาก TradingView โดยเริ่มจากการเข้า Stock Screener, เลือก filter เป็น Russell 1000, scroll ตารางเพื่อเก็บ ticker ให้ครบ 1,000 ตัว, เข้า profile ของแต่ละ ticker เพื่อหาโลโก้แบบ SVG, ดาวน์โหลดไฟล์ และจัดเก็บเป็น batch folder ละ 100 รูป

ผลลัพธ์หลักที่ต้องส่งมอบคือ CLI tool ที่รันซ้ำได้, resume ได้, มี report ตรวจสอบผลลัพธ์ และมี test ครอบคลุม logic สำคัญ

## 2. Source Requirement

อ้างอิง requirement จาก:

- `BRD-Automated`
- `automated-us-stock-logo-scraper-spec.md`

Business goal:

- ทีม Design/Dev ต้องได้โลโก้ Russell 1000 จำนวน 10 โฟลเดอร์
- โฟลเดอร์ละ 100 ไฟล์
- ไฟล์ชื่อ `[TICKER].svg`
- ใช้ bulk import เข้า Figma ได้โดยไม่ทำให้เครื่องค้าง

## 3. Recommended Implementation

ให้ implement เป็น Node.js + TypeScript CLI โดยใช้:

- Playwright สำหรับ browser automation
- Native `fetch` หรือ `undici` สำหรับดาวน์โหลด SVG
- Zod สำหรับ validate config
- CSV writer library สำหรับสร้าง `report.csv`
- Test framework ตามมาตรฐาน repo ถ้ามีอยู่แล้ว หาก repo ยังไม่มีให้ใช้ Vitest

หาก codebase ปลายทางมี stack หรือ pattern อยู่แล้ว ให้ยึด pattern ของ repo นั้นก่อน recommendation ด้านบน

## 4. Development Scope

### In Scope

- CLI command สำหรับ run scraping job
- CLI command สำหรับ validate output เดิม
- TradingView screener navigation
- Russell 1000 filter selection ผ่าน UI
- Lazy-loaded table scrolling
- Ticker extraction และ de-duplication
- Logo URL resolution จากหน้า profile
- SVG download และ validation
- Batch folder writer
- JSON/CSV report generation
- Resume mode
- Retry, timeout, rate limit และ controlled concurrency
- Unit tests สำหรับ core logic

### Out of Scope

- Figma import automation
- Replace asset ใน Pocket app
- Production scheduler
- Internal API service
- Bypass captcha, login, paywall หรือ anti-bot protection
- Third-party logo API fallback

## 5. Expected CLI

### Full Run

```bash
stock-logo-scraper run \
  --source tradingview \
  --index "Russell 1000" \
  --target-count 1000 \
  --batch-size 100 \
  --output ./output \
  --format svg \
  --concurrency 3 \
  --max-retries 3 \
  --headless
```

### Resume Failed Items

```bash
stock-logo-scraper run \
  --source tradingview \
  --index "Russell 1000" \
  --output ./output \
  --resume
```

### Validate Existing Output

```bash
stock-logo-scraper validate \
  --output ./output \
  --report ./output/report.json
```

## 6. Output Contract

เมื่อรันสำเร็จ ระบบต้องสร้าง output structure:

```text
output/
  1-100/
  101-200/
  201-300/
  301-400/
  401-500/
  501-600/
  601-700/
  701-800/
  801-900/
  901-1000/
  report.json
  report.csv
  tickers.json
```

ไฟล์โลโก้ต้องบันทึกเป็น:

```text
output/[batch-folder]/[TICKER].svg
```

ตัวอย่าง:

```text
output/1-100/AAPL.svg
output/1-100/MSFT.svg
output/101-200/AMD.svg
```

## 7. Required Report Fields

`report.json` และ `report.csv` ต้องมีข้อมูลต่อ ticker อย่างน้อย:

- `sequence`
- `ticker`
- `exchange`
- `batchFolder`
- `profileUrl`
- `logoUrl`
- `filePath`
- `status`
- `errorCode`
- `errorMessage`
- `retryCount`
- `downloadedAt`

สถานะที่ต้องรองรับ:

- `success`
- `missing_logo`
- `invalid_logo_format`
- `download_failed`
- `retry_exhausted`
- `skipped`

## 8. Suggested Module Breakdown

ให้แยก implementation เป็น module ที่ทดสอบได้ง่าย:

| Module | Responsibility |
|---|---|
| `cli` | parse command และเรียก job |
| `config` | load/default/validate config |
| `job-controller` | orchestrate workflow, retry, resume, summary |
| `screener-navigator` | เปิด TradingView และ apply Russell 1000 filter |
| `ticker-extractor` | scroll table และ extract ticker/exchange |
| `profile-url-builder` | สร้างหรือ resolve profile URL |
| `logo-resolver` | หา SVG logo URL จาก profile page |
| `logo-downloader` | download และ validate SVG |
| `batch-writer` | คำนวณ folder และบันทึกไฟล์ |
| `report-writer` | เขียน JSON/CSV แบบ incremental |
| `validator` | validate output/report หลัง job จบ |
| `logger` | structured logging |

## 9. Implementation Tasks

### Task 1: Project Setup

- ตรวจ package manager และ test framework ของ repo
- เพิ่ม dependency ที่จำเป็นเท่าที่ใช้จริง
- เพิ่ม CLI entrypoint `stock-logo-scraper`
- เพิ่ม TypeScript config หรือใช้ config เดิมของ repo
- เพิ่ม script สำหรับ build, test, lint ถ้ายังไม่มี

Acceptance criteria:

- `stock-logo-scraper --help` แสดง command `run` และ `validate`
- project build ผ่าน
- test command รันได้

### Task 2: Config and CLI Parsing

- implement option parsing สำหรับ `run` และ `validate`
- set default:
  - `source = tradingview`
  - `index = Russell 1000`
  - `targetCount = 1000`
  - `batchSize = 100`
  - `format = svg`
  - `concurrency = 3`
  - `maxRetries = 3`
  - `allowPartial = false`
- validate config ด้วย schema
- expose timeout, rate limit และ selector config

Acceptance criteria:

- invalid config ต้อง fail ด้วย error ชัดเจน
- CLI options override default ได้
- มี unit test สำหรับ default/override/invalid config

### Task 3: Batch and Filename Utilities

- implement batch folder calculation จาก sequence
- implement ticker filename sanitization
- preserve original ticker ใน report
- create output directories ตาม batch size

Acceptance criteria:

- sequence 1-100 map ไป `1-100`
- sequence 101-200 map ไป `101-200`
- special ticker เช่น `BRK.B` ใช้ filename ที่ filesystem รองรับ
- มี unit test ครอบคลุม edge cases

### Task 4: TradingView Screener Automation

- เปิด `https://www.tradingview.com/screener/`
- click filter button
- เลือก index filter เป็น `Russell 1000`
- wait จน table โหลด
- centralize selectors ใน config

Acceptance criteria:

- หาก selector หาย ต้อง fail ด้วย `SELECTOR_NOT_FOUND`
- หาก option Russell 1000 หาย ต้อง fail ด้วย `FILTER_OPTION_NOT_FOUND`
- รองรับ headless/headed mode จาก CLI
- มี log phase ชัดเจน

### Task 5: Lazy Table Ticker Extraction

- อ่าน ticker และ exchange จาก visible rows
- scroll table container เพื่อ trigger lazy loading
- de-duplicate ticker
- หยุดเมื่อครบ `targetCount`
- หยุดเมื่อไม่มี ticker ใหม่เกิน `maxIdleScrolls`
- บันทึก `tickers.json`

Acceptance criteria:

- `tickers.json` มี 1,000 unique tickers เมื่อ scrape ครบ
- duplicate ticker ไม่ถูกนับซ้ำ
- หากได้ไม่ครบและ `allowPartial=false` ต้อง fail ด้วย `INSUFFICIENT_TICKERS`
- หาก `allowPartial=true` ให้ไปขั้น download ต่อพร้อม summary warning

### Task 6: Logo Resolution

- สร้างหรือ resolve profile URL ของแต่ละ ticker
- เปิดหน้า profile
- หา logo element
- resolve full-size logo URL
- validate ว่า URL หรือ response เป็น SVG

Acceptance criteria:

- ticker ที่มีโลโก้ต้องได้ `logoUrl`
- ticker ที่ไม่พบ logo ต้อง mark `missing_logo`
- logo ที่ไม่ใช่ SVG ต้อง mark `invalid_logo_format`
- ticker-level failure ต้องไม่ทำให้ทั้ง job ล้ม

### Task 7: SVG Download

- download SVG ด้วย HTTP client หรือ Playwright request context
- validate HTTP status 2xx
- validate content เป็น SVG parseable
- save เป็น `[TICKER].svg`
- retry transient error ด้วย backoff
- จำกัด concurrency ค่า default 3
- ใส่ random delay/rate limit ระหว่าง request

Acceptance criteria:

- valid SVG ถูก save ใน batch folder ถูกต้อง
- HTTP 429/5xx retry ตาม `maxRetries`
- HTTP 404 mark `download_failed`
- invalid SVG mark `invalid_logo_format`

### Task 8: Report Writer

- เขียน `report.json` แบบ incremental
- เขียน `report.csv`
- summary ต้องรวม total, success, missing, failed, invalid, skipped
- report ต้องเก็บ `startedAt`, `finishedAt`, `jobId`, source, index, targetCount, batchSize

Acceptance criteria:

- process ถูก stop กลางทางแล้ว report ยังอ่าน partial result ได้
- CSV เปิดใน spreadsheet ได้
- JSON parse ได้และตรง schema

### Task 9: Resume Mode

- อ่าน `report.json` เดิม
- skip ticker ที่ `status=success` และไฟล์ยังมีอยู่จริง
- retry ticker ที่ fail/missing/invalid
- ไม่ overwrite success file เว้นแต่มี `--overwrite`

Acceptance criteria:

- `--resume` ไม่ download ไฟล์ success ซ้ำ
- success file ที่หายไปต้องถูก retry ใหม่
- `--overwrite` บังคับ download ใหม่ได้

### Task 10: Validate Command

- validate folder structure
- validate report file
- validate file existence ต่อ success item
- validate SVG parseability
- validate batch folder mapping

Acceptance criteria:

- output ครบต้อง exit code `0`
- output มี missing/invalid ต้อง exit code `1`
- config/report อ่านไม่ได้ต้อง exit code `2`
- validation summary อ่านง่ายใน terminal

### Task 11: Tests

- unit test:
  - config validation
  - batch calculation
  - filename sanitization
  - ticker de-duplication
  - SVG validation
  - report summary
  - resume decision logic
- integration test แบบ mock:
  - target count 10
  - logo success/missing/download fail mixed case
  - output/report ถูกสร้างครบ

Acceptance criteria:

- test suite ผ่านใน local
- core pure functions มี coverage เพียงพอ
- browser-dependent test สามารถ skip ได้ถ้าไม่มี browser runtime ใน CI

## 10. Error Codes

ต้องรองรับ error code ต่อไปนี้:

| Code | Scope | Meaning |
|---|---|---|
| `SOURCE_ACCESS_FAILED` | job | เข้า TradingView ไม่ได้ |
| `SELECTOR_NOT_FOUND` | job | selector สำคัญหาไม่เจอ |
| `FILTER_OPTION_NOT_FOUND` | job | ไม่พบ Russell 1000 filter |
| `FILTER_RESULT_EMPTY` | job | filter แล้วไม่มีข้อมูล |
| `INSUFFICIENT_TICKERS` | job | เก็บ ticker ไม่ครบ target |
| `PROFILE_LOAD_FAILED` | ticker | เปิด profile ไม่สำเร็จ |
| `LOGO_NOT_FOUND` | ticker | ไม่พบ logo |
| `INVALID_LOGO_FORMAT` | ticker | logo ไม่ใช่ SVG |
| `DOWNLOAD_FAILED` | ticker | download ไม่สำเร็จ |
| `FILE_WRITE_FAILED` | ticker | เขียนไฟล์ไม่ได้ |

## 11. Exit Codes

| Code | Meaning |
|---:|---|
| `0` | Completed successfully |
| `1` | Completed with ticker-level failures |
| `2` | Config, argument, or environment error |
| `3` | Source website, selector, filter, or extraction error |
| `4` | Output write or permission error |

## 12. Compliance and Safety Requirements

- ต้องไม่ bypass captcha, login, paywall หรือ access control
- ต้องมี rate limit และ concurrency ต่ำเป็นค่า default
- ต้อง log warning ให้ผู้รันตรวจสอบ Terms of Service ของ TradingView ก่อน production run
- ต้องไม่ hardcode selector กระจายหลายไฟล์
- ต้องไม่ทำ aggressive scraping

## 13. Definition of Done

งานถือว่าเสร็จเมื่อ:

- CLI `run` และ `validate` ใช้งานได้
- สามารถ scrape test run ขนาดเล็ก เช่น `--target-count 10` ได้
- output folder, SVG files, `tickers.json`, `report.json`, `report.csv` ถูกสร้างตาม contract
- resume mode ทำงานถูกต้อง
- validation command ตรวจ output ได้
- unit/integration tests ผ่าน
- README หรือ usage note อธิบายวิธีรัน, resume, validate และข้อจำกัดเรื่อง TradingView ToS

## 14. Ready-to-Use Codex Prompt

ใช้ prompt นี้ใน repo ที่ต้องการให้ Codex พัฒนาจริง:

```text
อ่านไฟล์สเปก `codex-development-tasks.md` และ technical spec ที่เกี่ยวข้อง จากนั้นพัฒนา CLI tool `stock-logo-scraper` สำหรับดึงโลโก้หุ้น US กลุ่ม Russell 1000 จาก TradingView ตาม requirement

ขอบเขตงานรอบนี้:
1. Implement CLI `run` และ `validate`
2. ใช้ Playwright สำหรับ TradingView browser automation
3. ดึง ticker จาก screener ด้วย lazy scroll ให้ครบ target count
4. Resolve และ download SVG logo ราย ticker
5. สร้าง output folder batch ละ 100 ไฟล์
6. สร้าง `tickers.json`, `report.json`, `report.csv`
7. รองรับ retry, resume, timeout, rate limit และ controlled concurrency
8. เพิ่ม unit/integration tests สำหรับ core logic
9. เพิ่ม README usage note

ก่อนแก้โค้ดให้สำรวจ pattern ของ repo ก่อน และใช้ framework/package manager/test framework ที่ repo ใช้อยู่แล้ว หากไม่มี pattern เดิม ให้ใช้ Node.js + TypeScript + Playwright + Vitest

ข้อจำกัด:
- ห้าม bypass captcha, login, paywall หรือ access control
- แยก selector/config ไว้ที่เดียว
- ticker-level failure ต้องไม่ทำให้ทั้ง job ล้ม
- ต้องเขียน report แบบ incremental เพื่อรองรับ resume/debug

เมื่อทำเสร็จให้รัน build/test ที่เกี่ยวข้อง และสรุปไฟล์ที่แก้กับวิธีรันให้ชัดเจน
```

## 15. Open Questions for Product/Team

- ทีมยืนยันสิทธิ์การ scrape TradingView และเงื่อนไขการใช้งานแล้วหรือยัง
- หากบาง ticker ไม่มี SVG ต้อง fallback เป็น PNG หรือ placeholder หรือไม่
- ต้องเก็บ output ไว้ที่ local folder, shared drive, cloud storage หรือ repo ใด
- ต้องการ official Russell membership source เพิ่มเติมเพื่อ cross-check กับ TradingView หรือไม่
- รอบ production ต้องรันแบบ manual หรือ schedule รายเดือน/รายไตรมาส
