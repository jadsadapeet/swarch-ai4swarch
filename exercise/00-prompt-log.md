**ภาษา:** ภาษาไทย | [English](00-prompt-log.en.md)

**ชื่อ / รหัสนิสิต:** เจษฎา วินัยโกศล / 6772012221

**Agent:** GitHub Copilot (Agent mode)

**โมเดล:** gpt-6-luna

**แผน / รุ่น:** Copilot Free



| # | งาน / run | agent + โมเดล | ข้อความที่ส่ง (ย่อ) | ไฟล์ที่ agent อ่าน | อยู่ในขอบเขตหลักฐานที่อนุญาตหรือไม่ | มีประโยชน์ (1–5) | หมายเหตุ |
|---|---|---|---|---|---|---|---|
| 1 | 1-A | Copilot / gpt-6-luna | `prompts/1-recover-architecture.md` run A | prompts/1-recover-architecture.md, protos/demo.proto | อยู่ | 4 | |
| 2 | 1-B | Copilot / gpt-6-luna | `prompts/1-recover-architecture.md` run B | prompts/1-recover-architecture.md, protos/demo.proto, kubernetes-manifests/ (ทุกไฟล์), **exercise/00-prompt-log.md** | ไม่อยู่ (บางส่วน) | 4 | agent อ่าน prompt log ซึ่งอยู่นอกหลักฐาน (agent ระบุเองในไฟล์ 1B) |
| 3 | 2-a | Copilot / gpt-6-luna | `prompts/2-write-adr.md` run a | prompts/2-write-adr.md, docs/adr/template.md | อยู่ | 4 | ลืมใส่ R-12 ตอนเริ่มต้น |
| 4 | 2-b | Copilot / gpt-6-luna | `prompts/2-write-adr.md` run b | prompts/2-write-adr.md, docs/scenario.md, docs/adr/0001-…, checkoutservice/main.go, **docs/scenario.th.md** | ไม่อยู่ (บางส่วน) | 4 | scenario.th.md ไม่อยู่ใน context ที่อนุญาต |
| 5 | 3 | Copilot / gpt-6-luna | `prompts/3-review-adr.md` | docs/adr/0002-async-order-processing-with-rabbitmq.md, docs/scenario.md | อยู่ | 5 | |
| 6 | ตรวจงาน + ส่วน B | Claude Code / Opus 5.5 | ขอให้อ่าน repo, ตรวจใบงาน 00–05, จำลองการประชุมคณะกรรมการ | ทั้ง repo ยกเว้น docs/review-board/ (แต่เห็นการ์ด P และ S จากข้อความที่เลือกใน editor) | ไม่ใช่งานตาม prompt | – | Claude ไม่ได้แก้ใบงาน ให้แค่ข้อเสนอแนะ |

