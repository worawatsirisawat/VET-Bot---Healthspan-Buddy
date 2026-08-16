# Health Companion — Desktop Overlay Companion

Prototype สำหรับ **HIShack 2026** (โจทย์ Longevity Innovation — Early Detection & Prevention)

Desktop companion application ที่แสดงตัวละครมาสคอต (TV-head character) ลอยอยู่บนหน้าจอ
ตลอดเวลา ทำงานเบื้องหลังเพื่อสังเกตพฤติกรรมการใช้คอมพิวเตอร์ผ่าน keyboard/mouse activity
แล้วให้คำแนะนำเชิงป้องกัน Office Syndrome และพฤติกรรมเสี่ยงสุขภาพที่เกี่ยวข้อง พร้อมคะแนน
สุขภาพแบบ real-time และคำแนะนำเชิงลึกที่อ้างอิงงานวิจัยจริง

---

## หลักการออกแบบ (Design Principles)

หลักการเหล่านี้สรุปจาก docstring/comment ที่อยู่ในโค้ดจริงของโปรเจกต์:

- **Privacy-first — ไม่เก็บเนื้อหาที่พิมพ์** — `core/activity_monitor.py` จับเฉพาะ "มี
  keyboard/mouse event เกิดขึ้นเมื่อไหร่" (timestamp) เท่านั้น ไม่มีการ log ตัวอักษร/ปุ่มที่กด
  หรือตำแหน่งเมาส์จริง (ดู docstring ของ `ActivityMonitor`)
- **ไม่ใช้กล้องหรือเซนเซอร์เพิ่มเติมใดๆ** — ระบบทั้งหมดอิงจาก input device (keyboard/mouse)
  ที่ผู้ใช้มีอยู่แล้วเท่านั้น
- **Proxy-based ไม่ใช่การตรวจจับจริง** — `core/behavior_engine.py` ใช้ "สัญญาณพฤติกรรมทาง
  อ้อม" (behavioral proxy) จาก keyboard/mouse activity เพื่อประเมินพฤติกรรม ไม่มีการตรวจจับ
  ท่าทางร่างกายจริง ทุกคำแนะนำจึงเป็น **"การให้คำแนะนำ" ไม่ใช่ "การวินิจฉัย/ยืนยันข้อเท็จจริง
  ทางกาย"** (ระบุไว้ตรงๆ ใน docstring ของโมดูล และย้ำอีกครั้งในข้อความ `POSTURE_BREAK`)
- **Local-only persistence** — `core/stats_store.py` เก็บสถิติผ่าน SQLite บนเครื่องผู้ใช้
  เท่านั้น "ไม่ส่งขึ้น cloud" (ระบุไว้ใน docstring ของโมดูล) และเก็บเฉพาะข้อมูลสรุปเชิง
  พฤติกรรม (ชั่วโมงทำงาน, การพัก) ไม่เก็บเนื้อหาการพิมพ์ใดๆ
- **Config-driven ไม่ hardcode ตายตัว** — threshold ทุกตัวใน `BehaviorConfig` ปรับได้ผ่าน
  GUI (`core/settings_dialog.py`) เพื่อรองรับ personalization
- **ห้ามแต่งข้อมูลงานวิจัยขึ้นมาเอง** — `core/health_tips.py` มีกติกาเขียนไว้ตรงๆ ในหัวไฟล์
  ว่าคำแนะนำที่จะแสดงจริง (`verified=True`) ต้องมีแหล่งอ้างอิงที่ตรวจสอบได้เท่านั้น หมวดที่ยัง
  หาแหล่งอ้างอิงไม่ได้ต้องเป็น placeholder (`verified=False`) และมีกลไก `get_verified_tips()`
  กรองไม่ให้ placeholder หลุดไปแสดงผลจริง

---

## สถาปัตยกรรมระบบ (Architecture)

การ wiring ทั้งหมดอยู่ใน `main.py` (`class HealthCompanionApp`) สรุปเป็นไดอะแกรมได้ดังนี้:

```
┌─────────────────────────┐
│ ActivityMonitor          │  Loop 1 — background thread (pynput)
│ core/activity_monitor.py │  จับ timestamp ล่าสุดที่มี keyboard/mouse event
└────────────┬─────────────┘
             │ seconds_since_last_activity()
             ▼
┌──────────────────────────────────────────────┐        ┌──────────────────────┐
│ BehaviorEngine                                 │──────▶│ StatsStore            │
│ core/behavior_engine.py                        │ save/  │ core/stats_store.py   │
│  Loop 2/3 — evaluate() เรียกทุก POLL_INTERVAL   │ load   │ (SQLite, local-only)  │
│  ทุก 30 วิ จาก QTimer ใน main.py                │◀──────│  - daily_stats table  │
│                                                  │        │  - user_settings table│
│  evaluate()          -> AdviceType              │        └──────────────────────┘
│  get_current_emotion() -> "normal"/"angry"      │
│  get_health_score()  -> int 0-100               │
└────────────┬─────────────────────┬──────────────┘
             │                     │
   show_advice(advice)   set_emotion()/set_health_score()
             │                     │
             ▼                     ▼
┌────────────────────────────────────────────────────────┐
│ CharacterOverlay (Loop 4 — PyQt6 always-on-top widget)   │
│ core/character_overlay.py                                │
│  - วาด sprite จาก core/emotion_sprites.py (normal/angry)  │
│  - badge คะแนนสุขภาพ (สี traffic-light) มุมขวาบน           │
│  - SpeechBubble แสดงข้อความ (advice / chatter / health tip)│
│  - แยกคลิก (react_to_click) กับลากย้ายตำแหน่ง               │
│  - คลิกขวา -> เมนู "สถิติวันนี้" / "ตั้งค่า" (signal ไปที่ main)│
└──────────────┬───────────────────────────┬───────────────┘
               │ request_stats             │ request_settings
               ▼                           ▼
   _show_stats_dialog() [main.py]   _show_settings() [main.py]
   อ่านจาก get_today_summary() +          เปิด SettingsDialog
   store.load_recent(7)                   core/settings_dialog.py
                                           -> behavior_engine.update_config()
                                           -> store.save_setting() (persist)

┌───────────────────────────┐   ┌───────────────────────────────┐
│ ChatterEngine (idle)       │   │ HealthTipEngine                │
│ core/chatter_engine.py     │   │ core/health_tips.py             │
│ ทุก 10 นาที (QTimer)        │   │ ทุก 45 นาที (QTimer) — สุ่มจาก   │
│ CHATTER_LINES /             │   │ get_verified_tips() เท่านั้น     │
│ CLICK_REACTION_LINES        │   │ (หมุนเวียนไม่ให้หมวดซ้ำติดกัน)     │
└──────────────┬──────────────┘   └───────────────┬────────────────┘
               └──────────────┬───────────────────┘
                               ▼
                  CharacterOverlay.show_chatter(text, mood)
                  (ใช้ SpeechBubble เดิม)

┌───────────────────────────┐
│ TrayApp (Loop 4 ส่วนควบคุม) │  QSystemTrayIcon — รันบน event loop เดียวกับ GUI
│ core/tray_app.py           │  เมนู: แสดง/ซ่อนตัวละคร, สถิติวันนี้, ออกจากโปรแกรม
└─────────────────────────────┘
```

---

## รายละเอียดแต่ละโมดูล

| ไฟล์ | หน้าที่หลัก | Class / Function สำคัญ |
|---|---|---|
| `main.py` | ประกอบทุกโมดูลเข้าด้วยกัน, คุม QTimer 3 ตัว (eval / chatter / health tip) | `HealthCompanionApp.__init__/start/_evaluation_tick/_chatter_tick/_health_tip_tick/_show_stats_dialog/_show_settings/_quit` |
| `core/activity_monitor.py` | Loop 1 — จับ keyboard/mouse event เป็น timestamp ผ่าน background thread (pynput) | `ActivityMonitor.start/stop/seconds_since_last_activity/last_active_datetime` |
| `core/behavior_engine.py` | Loop 2/3 — ตัดสินใจว่าจะเตือนอะไร + คำนวณอารมณ์/คะแนนสุขภาพ, เก็บ threshold ทั้งหมด | `BehaviorConfig` (dataclass ค่า threshold), `DailyStats` (สถิติวันนี้), `AdviceType` (enum: NONE/POSTURE_BREAK/MEAL_REMINDER/OVERWORK_WARNING), `BehaviorEngine.evaluate/get_current_emotion/get_health_score/update_config/mark_meal_break_taken/get_today_summary`, `EMOTION_NORMAL`/`EMOTION_ANGRY`, `ADVICE_MESSAGES`, `config_to_settings/config_from_settings` (แปลง config ↔ dict สำหรับเก็บ SQLite) |
| `core/character_overlay.py` | Loop 4 — วาดตัวละคร (sprite + badge คะแนน), จัดการ mouse interaction, speech bubble | `SpeechBubble` (กล่องข้อความลอย), `CharacterOverlay.paintEvent/mousePressEvent/mouseMoveEvent/mouseReleaseEvent/react_to_click/show_advice/show_chatter/set_emotion/set_health_score` |
| `core/emotion_sprites.py` | โหลดภาพ `Emotion/Good_Normal.png` / `Emotion/Angry.png`, ลบพื้นหลังขาวด้วย flood-fill, cache เป็น QPixmap | `emotion_pixmap(emotion, box)`, `available_emotions()` |
| `core/chatter_engine.py` | สุ่มประโยคทักทาย/ปฏิกิริยาคลิก แบบไม่ซ้ำประโยคก่อนหน้าทันที | `ChatterLine` (dataclass: text, mood), `CHATTER_LINES` (ทักทายทุก 10 นาที), `CLICK_REACTION_LINES` (ปฏิกิริยาตอนคลิก), `ChatterEngine.next_line` |
| `core/health_tips.py` | คลังคำแนะนำสุขภาพอ้างอิงงานวิจัยจริง 4 หมวด, กรอง placeholder ที่ยังไม่ verified | `HealthTip` (dataclass: category, text, source, verified), `HEALTH_TIPS`, `CATEGORIES`/`CATEGORY_LABELS`, `get_verified_tips(category=None)`, `HealthTipEngine.next_tip` |
| `core/stats_store.py` | Persistence — SQLite (`data/health_stats.db`) เก็บสถิติรายวันและ user settings | `StatsStore.save_daily/load_daily/load_recent`, `save_setting/load_setting/load_all_settings` |
| `core/settings_dialog.py` | GUI (QDialog) ปรับ threshold หลัก 3 ค่า พร้อม validate | `SettingsDialog.__init__/_on_save/get_config` |
| `core/tray_app.py` | System tray icon (QSystemTrayIcon) ควบคุมแอปจากนอกตัวละคร | `TrayApp.start/stop/_build_menu/_on_activated` |
| `tests/test_behavior.py` | Unit test สำหรับ logic ล้วน (ไม่ต้องเปิด GUI) | ดูหัวข้อ "การรัน Test Suite" ด้านล่าง |

---

## การติดตั้งแบบละเอียด

### 1) ข้อกำหนดเวอร์ชัน Python

`requirements.txt` ปัจจุบันมีแค่:

```
PyQt6>=6.5.0
pynput>=1.8.0
```

ไม่ได้ pin เวอร์ชัน Python ตรงๆ แต่จากการติดตั้งจริงบนโปรเจกต์นี้ **Python 3.12.10
ติดตั้งได้ราบรื่นที่สุด** — `pip install -r requirements.txt` ได้ prebuilt wheel ครบ
(`PyQt6-6.11.0`, `pynput-1.8.2`) ไม่ต้อง build จาก source เลย แนะนำให้ใช้ **Python
3.12 หรือ 3.13**

> **Known issue ที่เคยเจอจริงในโปรเจกต์นี้:** ช่วงแรกโปรเจกต์เคย depend on `Pillow` +
> `pystray` (สำหรับวาดไอคอน system tray) ซึ่งบน **Python 3.14** ยังไม่มี prebuilt wheel
> ของ Pillow ทำให้ pip พยายาม build จาก source แล้วล้มที่ `zlib` บน Windows
> วิธีแก้ที่ใช้จริง: เขียน `core/tray_app.py` ใหม่ให้ใช้ `QSystemTrayIcon` ของ PyQt6 แทน
> (วาดไอคอนด้วย `QPainter` เอง ไม่ต้องพึ่งไฟล์ภาพภายนอก) ตอนนี้ `requirements.txt` จึงไม่มี
> `Pillow`/`pystray` แล้ว ปัญหานี้จึงไม่เกิดซ้ำ แต่ถ้าเครื่องคุณมีแค่ Python เวอร์ชันใหม่มากๆ
> (3.14+) และ `pip install` ล้มเหลว ให้ลองสลับไปใช้ Python 3.12/3.13 ก่อน

ตรวจสอบเวอร์ชัน Python ที่มีในเครื่อง:

```bash
# Windows
py -0p

# macOS / Linux
python3 --version
```

### 2) สร้าง Virtual Environment

**Windows (PowerShell):**

```powershell
py -3.12 -m venv venv
.\venv\Scripts\Activate.ps1
```

**macOS / Linux:**

```bash
python3.12 -m venv venv
source venv/bin/activate
```

### 3) ติดตั้ง Dependencies

```bash
pip install --upgrade pip
pip install -r requirements.txt
```

### 4) รันแอป

```bash
python main.py
```

**หมายเหตุ:** แอปนี้เป็น native desktop application (PyQt6 always-on-top overlay + system
tray) ต้องรันบนเครื่องที่มี display จริง (Windows/macOS/Linux desktop environment)
**ไม่สามารถรันใน headless server/container ได้**

---

## การรัน Test Suite

`core/behavior_engine.py`, `core/chatter_engine.py`, `core/health_tips.py` และ
`core/stats_store.py` ออกแบบให้ทดสอบแยกจาก GUI ได้ทั้งหมด (ไม่ต้องเปิดหน้าจอ):

```bash
python tests/test_behavior.py
```

ทดสอบทั้งหมด **35 เคส แบ่งเป็น 12 กลุ่ม**:

| กลุ่ม | ครอบคลุม |
|---|---|
| 1. Advice trigger logic | fresh-start ไม่ trigger, trigger เมื่อเกิน threshold, throttle กันเตือนซ้ำ, idle ยาวรีเซ็ต |
| 2. การนับเวลาแบบ elapsed จริง | สะสมชั่วโมงทำงานถูกต้องจาก `elapsed_since_last_eval` |
| 3. Overwork flag | เตือนเมื่อชั่วโมงทำงานสะสมเกิน `daily_work_hour_limit` |
| 4. Persistence ข้าม session | สถิติ SQLite คงอยู่หลังสร้าง `BehaviorEngine` ใหม่ |
| 5. Chatter engine | สุ่มไม่ซ้ำติดกัน, mood ครบ, คลังประโยคเดียวไม่ crash |
| 6. Click reaction lines | ไม่ซ้ำกับคลัง chatter เดิม, สุ่มไม่ซ้ำติดกัน |
| 7. `get_current_emotion` | angry เมื่อพ้นช่วงพักกินข้าวแล้วยังไม่ได้พัก, กลับ normal ทันทีที่พัก |
| 8. `update_config` | เปลี่ยน threshold runtime มีผลทันที โดยไม่รีเซ็ตสถิติสะสม |
| 9. Settings persistence | ค่า Settings (SQLite `user_settings`) คงอยู่ข้าม session |
| 10. `get_health_score` edge cases | สมบูรณ์ = 100, แย่ที่สุด = ใกล้ 0 |
| 11. Default `continuous_session_limit_min` | ต้องเป็น 30 (ไม่ใช่ 60 เดิม) |
| 12. Health tips | placeholder ที่ยังไม่ verified ถูกกรองออกเสมอ, มี verified tip อย่างน้อย 1 หมวด |

---

## ตารางสถานะฟีเจอร์

สำรวจจากโค้ดจริง ณ ปัจจุบัน (grep หา `TODO`/`placeholder` ทั่วโปรเจกต์):

| ฟีเจอร์ | สถานะ | หมายเหตุ |
|---|---|---|
| ตรวจจับ continuous session → เตือนเปลี่ยนอิริยาบถ | ✅ ทำงานสมบูรณ์ | proxy-based, default 30 นาที อ้างอิงงานวิจัย active microbreaks |
| เตือนพักกินข้าวตามช่วงเวลาที่ตั้งไว้ | ✅ ทำงานสมบูรณ์ | เทียบกับ meal window ใน `BehaviorConfig` |
| นับชั่วโมงทำงานสะสม/วัน + เตือน overwork | ✅ ทำงานสมบูรณ์ | |
| Health Score แบบ real-time (0-100) | ✅ ทำงานสมบูรณ์ | `get_health_score()`, แสดงเป็น badge สีมุมขวาบนตัวละคร |
| Emotion sprite (normal/angry ตามพลาดมื้อข้าว) | ✅ ทำงานสมบูรณ์ | ใช้ภาพจริงจาก `Emotion/` ผ่าน `core/emotion_sprites.py` |
| Idle chatter (ทักทายทุก 10 นาที) | ✅ ทำงานสมบูรณ์ | |
| ปฏิกิริยาตอนคลิกตัวละคร (แยกจากลาก) | ✅ ทำงานสมบูรณ์ | วัดระยะ press→release ≤ 5px |
| คำแนะนำสุขภาพอ้างอิงงานวิจัย (ทุก 45 นาที) | 🟡 ทำงานบางส่วน | มี verified tip ครบเฉพาะหมวด **การออกกำลังกาย** เท่านั้น อีก 3 หมวด (ท่านั่ง/ระดับสายตา, พฤติกรรมใช้ PC, การรับประทานอาหาร) เป็น `TODO` placeholder ใน `core/health_tips.py` — ถูกกรองไม่ให้แสดงผลจริงโดยอัตโนมัติ |
| Settings UI (ปรับ threshold ผ่าน GUI) | ✅ ทำงานสมบูรณ์ | `core/settings_dialog.py`, validate + persist ผ่าน SQLite |
| System tray control | ✅ ทำงานสมบูรณ์ | `QSystemTrayIcon`, ไม่ใช้ thread แยกอีกต่อไป |
| Data persistence (SQLite) ข้าม session | ✅ ทำงานสมบูรณ์ | ทั้งสถิติรายวันและ user settings |
| สรุปสถิติย้อนหลัง 7 วัน | ✅ ทำงานสมบูรณ์ | |

---

## Known Limitations

จากสิ่งที่ระบุไว้ตรงๆ ในโค้ด:

- **False positive จาก proxy-based detection** — ระบบใช้สัญญาณพฤติกรรมทางอ้อม
  (keyboard/mouse activity) เท่านั้น หากผู้ใช้อ่านเอกสารนิ่งๆ โดยไม่แตะ keyboard/mouse
  ระบบจะตีความว่า "ได้พักแล้ว" (idle reset) ทั้งที่อาจยังนั่งท่าเดิม — เป็นข้อจำกัดโดยธรรมชาติ
  ของการไม่ใช้กล้อง/เซนเซอร์เพิ่ม (ตามที่ระบุไว้ใน docstring ของ `core/behavior_engine.py`)
- **คำแนะนำเป็นการ "ให้คำแนะนำ" ไม่ใช่ "การวินิจฉัย"** — ระบบไม่ยืนยันข้อเท็จจริงทางกายภาพ
  ใดๆ (เช่นไม่รู้ว่าท่านั่งจริงเป็นอย่างไร) ข้อความ `POSTURE_BREAK` ระบุไว้ตรงๆ ว่า "ประเมินจาก
  ระยะเวลาที่ใช้งานต่อเนื่อง ไม่ใช่การตรวจจับท่านั่งจริง"
- **Health tips ยังไม่ครบทุกหมวด** — มีเพียงหมวดการออกกำลังกาย (`CATEGORY_EXERCISE`) ที่มี
  แหล่งอ้างอิงงานวิจัยจริง (`verified=True`) อีก 3 หมวดเป็น placeholder ที่ยังหางานวิจัยมา
  รองรับไม่ได้ (ดู `core/health_tips.py`)

---

## Roadmap (Future Plan)

เฉพาะสิ่งที่มี comment ระบุไว้จริงในโค้ดว่าเป็นแผนอนาคต:

1. **หางานวิจัยที่ verified มาใส่ 3 หมวดที่เหลือใน `core/health_tips.py`** — ท่านั่งและ
   ระดับสายตา (`CATEGORY_POSTURE`), พฤติกรรมการใช้งาน PC (`CATEGORY_PC_USAGE`), การ
   รับประทานอาหาร (`CATEGORY_MEAL`) — ตอนนี้เป็น `TODO` placeholder ที่ระบุไว้ตรงๆ ว่า
   "ต้องหางานวิจัยมาใส่แหล่งอ้างอิงก่อนใช้จริง — ยังไม่ verified"
2. **Personalization ตาม pattern รายบุคคล** — `BehaviorConfig` ออกแบบให้ทุก threshold
   เป็นค่าเริ่มต้นที่ปรับได้ (config-driven) "เพื่อรองรับ personalization ในอนาคต" (ระบุไว้ใน
   docstring ของ `core/behavior_engine.py`) — ปัจจุบันปรับได้ผ่าน Settings UI แล้ว แต่ยังเป็น
   การปรับด้วยตนเอง ไม่ใช่การเรียนรู้ pattern พฤติกรรมของผู้ใช้แต่ละคนโดยอัตโนมัติ
