# Fruit Classification Machine — ماكينة تصنيف الفاكهة

> **ملخص / Summary**
> نظام آلي يصنّف الفاكهة (جودة / نوع) باستخدام كاميرا ونموذج ذكاء اصطناعي، ويتحكم في حزام ناقل وذراع توجيه ميكانيكية عبر Arduino.
>
> An automated system that classifies fruit (quality / type) using a webcam and an AI model, then controls a conveyor belt and a diverting servo arm via Arduino.

<img src="media/view1.jpg" width="800">

---

## فهرس المحتويات / Table of Contents

1. [نظرة عامة / Project Overview](#نظرة-عامة--project-overview)
2. [بنية النظام / System Architecture](#بنية-النظام--system-architecture)
3. [المكونات الصلبة / Hardware](#المكونات-الصلبة--hardware)
4. [كود Arduino](#كود-arduino)
   - [HW_controller](#hw_controller)
   - [hardware_test](#hardware_test)
5. [برامج الكمبيوتر / PC Software](#برامج-الكمبيوتر--pc-software)
   - [main.py](#mainpy)
   - [fruit_detection&classification.py](#fruit_detectionclassificationpy)
   - [build_model.ipynb](#build_modelipynb)
6. [نموذج الذكاء الاصطناعي / AI Model](#نموذج-الذكاء-الاصطناعي--ai-model)
7. [تثبيت وتشغيل / Installation & Usage](#تثبيت-وتشغيل--installation--usage)

---

## نظرة عامة / Project Overview

### العربية
الماكينة دي بتاخد الفاكهة على حزام ناقل (conveyor belt)، الكاميرا بتصور الفاكهة، والبرنامج بيحلل الصورة باستخدام نموذج Deep Learning ويقرر:
- لو الفاكهة **طبيعية (normal)** → الحزام يكمل وتعدي للجهة الصح.
- لو الفاكهة **غير طبيعية (abnormal)** → ذراع السيرفو يتحرك ويحولها للجهة التانية.

### English
The machine places fruit on a conveyor belt. A webcam captures each fruit; the PC software analyses the image with a Deep Learning model and decides:
- **Normal fruit** → belt continues, servo arm stays at 42°.
- **Abnormal fruit** → servo arm moves to 148° to divert the fruit.

---

## بنية النظام / System Architecture

```
┌─────────────────────────────────────────────┐
│                  PC (Python)                │
│  ┌──────────┐   ┌───────────┐  ┌─────────┐ │
│  │ Tkinter  │   │ OpenCV    │  │TensorFlow│ │
│  │   GUI    │◄──│ Camera    │──►│  Model  │ │
│  └────┬─────┘   └───────────┘  └─────────┘ │
│       │  Serial (115200 baud)               │
└───────┼─────────────────────────────────────┘
        │
        ▼
┌───────────────────────────────────────┐
│            Arduino UNO                │
│  ┌──────────────┐  ┌───────────────┐  │
│  │  PID Motor   │  │  Servo Arm    │  │
│  │  Controller  │  │  Controller   │  │
│  └──────┬───────┘  └──────┬────────┘  │
│         │                 │           │
│  ┌──────▼───────┐  ┌──────▼────────┐  │
│  │  L298N +     │  │  MG90S Servo  │  │
│  │ Geared Motor │  │  (pin 6)      │  │
│  │ + Encoder    │  └───────────────┘  │
│  └──────────────┘                     │
└───────────────────────────────────────┘
```

**تدفق البيانات / Data flow:**
1. الكاميرا تصور الفاكهة على الحزام.
2. `main.py` يكشف الفاكهة عن طريق HSV color masking.
3. نموذج TensorFlow يعطي score بين 0 و 1.
4. البرنامج يرسل للـ Arduino عبر Serial: `السرعة, موضع السيرفو, Kp, Ki, Kd`.
5. الـ Arduino يتحكم في موتور الحزام بـ PID وفي السيرفو.
6. الـ Arduino يرد بالسرعة الحالية.

---

## المكونات الصلبة / Hardware

<img src="media/circuit.jpg" width="800"/>

| المكون | العدد | السعر (ج.م.) | الرابط |
|---|---|---|---|
| Arduino UNO SMD | 1 | 325 | [رابط](https://www.ampere-electronics.com/product/arduino-uno-smd-with-usb-cable/) |
| WebCam | 1 | 100 | — |
| MG90S Micro Servo (2.2 kg.cm) | 1 | 145 | [رابط](https://www.ampere-electronics.com/product/mg90s-micro-servo-motor/) |
| GW370 DC 12V 80rpm Encoder Motor | 1 | 600 | [رابط](https://www.ampere-electronics.com/product/25ga370-dc-gear-motor-encoder-200rpm/) |
| L298N Motor Driver Module | 1 | 65 | [رابط](https://www.ampere-electronics.com/product/l298n-motor-driver-module/) |
| Pillow Block Bearing (x4) | 4 | 180 | [رابط](https://www.ampere-electronics.com/product/kp000-pillow-block-bearing/) |
| Motor Bracket | 1 | 30 | [رابط](https://www.ampere-electronics.com/product/25ga370-geared-motor-holder/) |
| AC Adapter 12V/2A | 1 | 50 | — |
| Switch ON-OFF | 1 | 3 | [رابط](https://www.ampere-electronics.com/product/boat-rocker-switch-on-off-black-6a250vac/) |
| Female DC Power Plug | 1 | 10 | [رابط](https://www.ampere-electronics.com/product/female-dc-power-plug-to-2-pin-screw-terminal/) |
| Printer Cable 2m | 1 | 25 | — |
| Male-to-Female Jumper Wires | 1 | 7.5 | [رابط](https://www.ampere-electronics.com/product/male-to-female-20cm-40-pin-jumper-wire-set-3/) |
| خشب + حزام (materials) | — | 1600 | — |
| **الإجمالي (15 مكوّن)** | **15** | **~2440 ج.م.** | |

> قائمة المكونات الكاملة: [components.csv](media/components.csv)

**توصيل أطراف الـ Arduino / Arduino pin mapping:**

| الطرف (Pin) | الوظيفة |
|---|---|
| 10 (PWM) | سرعة الموتور (L298N ENA) |
| 8 | اتجاه الموتور (L298N IN1) |
| 9 | اتجاه الموتور (L298N IN2) |
| 3 (interrupt) | Encoder Channel A |
| 5 | Encoder Channel B |
| 6 (digital — manual PWM for servo, avoids using a timer-PWM pin) | السيرفو MG90S |

---

## كود Arduino

### HW_controller

**الملف:** [`arduino/HW_controller/HW_controller.ino`](arduino/HW_controller/HW_controller.ino)

هذا الكود هو المتحكم الرئيسي الذي يعمل على الـ Arduino أثناء تشغيل الماكينة.

#### ما الذي يفعله؟
1. **يستقبل أوامر من الـ PC** عبر Serial بصيغة:
   ```
   target_speed,target_servo_pos,Kp,Ki,Kd
   ```
   مثال: `6.0,42,0.3,0.001,0.002`

2. **يتحكم في سرعة الحزام** باستخدام **PID controller**:
   - يقيس السرعة الفعلية من encoder بالـ interrupt.
   - يحسب الخطأ بين السرعة المطلوبة والفعلية.
   - يصدر أمر PWM للموتور عبر L298N.

3. **يتحكم في موضع السيرفو** (يدوياً بدون مكتبة Servo):
   - الفاكهة الطبيعية: `42°` (يمر مباشرة).
   - الفاكهة الغير طبيعية: `148°` (يُحول للجهة الأخرى).

4. **يرسل السرعة الحالية** للـ PC بعد كل أمر.

5. **نظام الأمان:** لو الـ Arduino توقف عن استقبال أوامر لأكثر من 10 دورات (10 × 300ms = 3 ثواني) → يوقف الموتور تلقائياً.

#### متغيرات مهمة:
| المتغير | القيمة | المعنى |
|---|---|---|
| `encoder_Puls_per_cm` | 62.5 | نبضات الـ encoder لكل سنتيمتر |
| `loop_time` | 300 ms | زمن دورة التحكم |
| `noCommLoopMax` | 10 | حد الدورات بدون تواصل |
| `max_servo_pos` | 150 | أقصى زاوية للسيرفو |
| `min_servo_pos` | 40 | أدنى زاوية للسيرفو |

#### دالة `computePID()`:
```
error = target_pulses - actual_pulses
integral += error × Δt
derivative = (error - prev_error) / Δt
PWM = Kp×error + Ki×integral + Kd×derivative
```

#### دالة `servo_write(degree, pin)`:
تتحكم في السيرفو يدوياً عن طريق توليد نبضات PWM:
```
pulse_width = (degree × 10.25) + 500  microseconds
```
تكرر 5 مرات كل دورة لتثبيت الموضع.

---

### hardware_test

**الملف:** [`arduino/hardware_test/hardware_test.ino`](arduino/hardware_test/hardware_test.ino)

كود للاختبار فقط (ليس للتشغيل الحقيقي). يحرك السيرفو من الحد الأدنى للأقصى والعكس بينما يطبع قراءة الـ encoder على الـ Serial Monitor. يُستخدم للتأكد من أن الموتور والسيرفو والـ encoder يعملون بشكل صحيح.

---

## برامج الكمبيوتر / PC Software

### main.py

**الملف:** [`programs/main.py`](programs/main.py)

البرنامج الرئيسي الذي يشغّل الواجهة الرسومية ويربط كل شيء معاً.

<img src="media/GUI.jpg" width="800"/>

#### الكلاس `App` — شرح كل دالة:

**`__init__(window, window_title)`** — تهيئة الواجهة:
بتبني الواجهة الرسومية بـ Tkinter وتضع فيها:
- حقل Arduino port (افتراضي: `COM1`)
- قائمة اختيار كاميرا (0–4)
- قائمة اختيار النموذج (مقروءة تلقائياً من مجلد `models/`)
- حقول السرعة (cm/s) ومعاملات PID (Kp, Ki, Kd)
- حقل حساسية اللون الأبيض (white sensitivity)
- حقل score threshold للتصنيف
- زر Start/Stop
- Canvas بحجم 640×480 لعرض صورة الكاميرا

**`start()`** — تشغيل أو إيقاف:
- **عند الضغط Start:**
  1. يفتح Serial port للـ Arduino.
  2. يحمّل نموذج TensorFlow المختار.
  3. يفتح الكاميرا.
  4. يبدأ حلقة `update()`.
- **عند الضغط Stop:** يغلق كل الاتصالات.

**`update()`** — الحلقة الرئيسية (تُستدعى كل 30ms):
1. يقرأ frame من الكاميرا.
2. يستدعي `classify(frame)` للكشف والتصنيف.
3. يستدعي `HW_update()` للتواصل مع Arduino.
4. يعرض الصورة في الـ Canvas.

**`classify(frame)`** — الكشف والتصنيف:
1. يحول الصورة لـ HSV.
2. يطبق mask لعزل الألوان غير البيضاء (يعزل الفاكهة عن الخلفية البيضاء).
3. يجد الـ contours.
4. لكل contour مساحته > 2000 بكسل:
   - يقتطع منطقة الفاكهة ويعيد حجمها لـ 180×180.
   - يمررها للنموذج ويحصل على score.
   - Score > threshold → **طبيعية** (مستطيل أخضر).
   - Score ≤ threshold → **غير طبيعية** (مستطيل أحمر).

**`HW_update()`** — التواصل مع Arduino:
- يرسل: `target_speed,target_servo_pos,Kp,Ki,Kd`
  - `target_servo_pos = 42` للفاكهة الطبيعية
  - `target_servo_pos = 148` للفاكهة الغير طبيعية
- يستقبل: السرعة الحالية ويعرضها في الواجهة.

---

### fruit_detection&classification.py

**الملف:** [`programs/fruit_detection&classification.py`](programs/fruit_detection&classification.py)

نسخة مبسطة بدون Arduino وبدون GUI، تفتح نافذة OpenCV مباشرة مع **trackbars** لضبط حدود HSV يدوياً في الوقت الفعلي. مفيدة لاختبار وضبط قيم HSV للكشف عن الفاكهة قبل تشغيل النظام الكامل.

---

### build_model.ipynb

**الملف:** [`programs/dataset&buildingModel/build_model.ipynb`](programs/dataset&buildingModel/build_model.ipynb)

Jupyter Notebook لبناء وتدريب نموذج Deep Learning جديد.

#### الخطوات:
1. **تجهيز البيانات:** ضع الصور في المجلدات:
   - `data/normal/` — صور الفاكهة الجيدة
   - `data/abnormal/` — صور الفاكهة الرديئة
   - `data/other/` — خلفيات فارغة (اختياري)
2. **تحميل البيانات** بـ `tf.keras.utils.image_dataset_from_directory` بحجم 180×180.
3. **بناء النموذج** — CNN متتالية:
   ```
   Input (180×180×3)
   → Rescaling (/255)
   → Conv2D(16) + MaxPool
   → Conv2D(32) + MaxPool
   → Conv2D(64) + MaxPool
   → Conv2D(128) + MaxPool
   → Conv2D(256) + MaxPool
   → Dropout(0.2)
   → Flatten
   → Dense(128, relu)
   → Dense(1)           ← score بين 0 و 1
   ```
4. **التدريب** بـ Adam optimizer ودالة خسارة MSE لمدة 10 epochs.
5. **حفظ النموذج:** `model.save('../models/your_model_name')`
6. **اختبار النموذج** على صور فردية أو مع الكاميرا.

> **ملاحظة:** بيانات الليمون متاحة على Kaggle:
> - [Lemon Quality Dataset](https://www.kaggle.com/datasets/yusufemir/lemon-quality-dataset)
> - [Lemon — Good or Bad? (99% Accuracy)](https://www.kaggle.com/code/mokshitsurana/lemon-good-or-bad-99-accuracy)

---

## نموذج الذكاء الاصطناعي / AI Model

**الموقع:** `programs/models/lemon_quality/`

النموذج المحفوظ هو نموذج TensorFlow SavedModel لتصنيف جودة الليمون.

| الملف | الوظيفة |
|---|---|
| `saved_model.pb` | بنية النموذج والأوزان |
| `keras_metadata.pb` | بيانات Keras الوصفية |
| `fingerprint.pb` | بصمة النموذج للتحقق |
| `variables/` | أوزان النموذج المدرب |

**كيف يعمل التصنيف:**
- النموذج يُخرج قيمة واحدة (score) بين 0 و 1.
- **score عالي (> threshold)** = فاكهة طبيعية.
- **score منخفض (≤ threshold)** = فاكهة غير طبيعية.
- القيمة الافتراضية لـ threshold = **0.2**.

---

## تثبيت وتشغيل / Installation & Usage

### 1. رفع كود Arduino
```
افتح arduino/HW_controller/HW_controller.ino في Arduino IDE
اختر Board: Arduino UNO
اختر الـ COM port الصحيح
ارفع الكود على الـ Arduino
```

### 2. تثبيت Python
```bash
pip install opencv-python
pip install tk
pip install Pillow
pip install pyserial
pip install tensorflow
```

### 3. تشغيل البرنامج الرئيسي
```bash
cd programs
python main.py
```

### 4. إعدادات الواجهة
| الحقل | الوصف | القيمة الافتراضية |
|---|---|---|
| Arduino port | رقم منفذ الـ Arduino | `COM1` |
| Cam source | رقم الكاميرا | `1` |
| Model | اسم النموذج | `lemon_quality` |
| Speed (cm/s) | سرعة الحزام | `6.0` |
| Kp / Ki / Kd | معاملات PID | `0.3 / 0.001 / 0.002` |
| White sensitivity | حساسية كشف الخلفية البيضاء | `100` |
| Score threshold | حد التصنيف | `0.2` |

### 5. بناء نموذج جديد (اختياري)
```
1. ضع صورك في programs/dataset&buildingModel/data/normal/ و data/abnormal/
2. افتح programs/dataset&buildingModel/build_model.ipynb في Jupyter
3. نفّذ كل الخلايا
4. احفظ النموذج في programs/models/your_model_name/
5. اختر النموذج الجديد من الواجهة
```

### 6. اختبار بدون Arduino (للمطورين)
```bash
cd programs
python "fruit_detection&classification.py"
```
هذا يفتح نافذة OpenCV مع trackbars لضبط HSV يدوياً بدون الحاجة لـ Arduino.

---

## صور المشروع / Project Photos

<img src="media/view1.jpg" width="400"> <img src="media/view2.jpg" width="400">
<img src="media/view3.jpg" width="400"> <img src="media/view4.jpg" width="400">
<img src="media/view5.jpg" width="400"> <img src="media/view6.jpg" width="400">
<img src="media/view7.jpg" width="400">

---

*By: Abdelraouf Hawash — abdelraouf.hawash@gmail.com — April 2023*
