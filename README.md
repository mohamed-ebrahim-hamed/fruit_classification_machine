# fruit classification machine — آلة تصنيف الفاكهة

> **بالعربي:** هذا المشروع عبارة عن آلة ذكية تصنّف الفاكهة بناءً على جودتها (سليمة / تالفة) باستخدام كاميرا ويب ونموذج ذكاء اصطناعي (TensorFlow)، ويتحكم فيها Arduino يحرّك حزام ناقل وذراع سيرفو لتوجيه الفاكهة.

This system classifies fruits based on their quality (normal / abnormal) using a webcam and a trained AI model, while an Arduino-controlled conveyor belt + servo arm physically sorts them.

<img src="media/view1.jpg" width="800">

---

## 📁 هيكل المشروع — Project Structure

```
fruit_classification_machine/
│
├── arduino/
│   ├── HW_controller/
│   │   └── HW_controller.ino      ← كود التحكم الرئيسي في الأردوينو
│   └── hardware_test/
│       └── hardware_test.ino      ← كود اختبار الهاردوير
│
├── programs/
│   ├── main.py                    ← برنامج Python الرئيسي (واجهة المستخدم)
│   ├── fruit_detection&classification.py  ← سكريبت اختبار الكشف بدون GUI
│   ├── models/
│   │   └── lemon_quality/         ← نموذج TensorFlow الجاهز للليمون
│   └── dataset&buildingModel/
│       ├── build_model.ipynb      ← Jupyter Notebook لبناء النموذج
│       ├── links.txt              ← روابط الداتاسيت
│       └── data/
│           ├── normal/            ← صور فاكهة سليمة
│           ├── abnormal/          ← صور فاكهة تالفة
│           └── other/             ← صور خلفية فارغة
│
└── media/
    ├── view1.jpg … view7.jpg      ← صور المشروع الحقيقي
    ├── circuit.jpg                ← مخطط الدائرة الكهربائية
    ├── GUI.jpg                    ← صورة واجهة البرنامج
    └── components.csv             ← قائمة المكونات مع الأسعار
```

---

## 🔧 الهاردوير — Hardware

<img src="media/circuit.jpg" width="800"/>

يمكنك الحصول على قائمة المكونات مع الأسعار والروابط من [هنا](media/components.csv).  
You can find the full component list with prices and links [here](media/components.csv).

| المكوّن | الوصف |
|---------|-------|
| **Arduino UNO** | وحدة التحكم المركزية — تتلقى الأوامر من الكمبيوتر عبر Serial وتتحكم في المحرك والسيرفو |
| **Webcam** | كاميرا لالتقاط صور الفاكهة على الحزام الناقل |
| **GW370 DC Geared Motor + Encoder** | محرك الحزام الناقل — الإنكودر يقيس السرعة الفعلية |
| **MG90S Micro Servo Motor** | ذراع التصنيف — يتحرك لليمين للفاكهة السليمة أو لليسار للتالفة |
| **L298N Motor Driver** | درايفر يتحكم في اتجاه وسرعة محرك الحزام |

### توصيل الأطراف — Pin Connections

| الجهاز | الطرف في Arduino |
|--------|----------------|
| Motor PWM | Pin 10 |
| Motor IN-A | Pin 8 |
| Motor IN-B | Pin 9 |
| Encoder Channel A (interrupt) | Pin 3 |
| Encoder Channel B | Pin 5 |
| Servo Signal | Pin 6 |

---

## 💻 السوفتوير — Software

<img src="media/GUI.jpg" width="800"/>

### 1. `arduino/HW_controller/HW_controller.ino` — كود الأردوينو الرئيسي

هذا الملف هو قلب التحكم في الهاردوير. مهامه:

- **استقبال الأوامر** من Python عبر Serial بصيغة:  
  `target_speed , target_servo_pos , Kp , Ki , Kd`
- **التحكم في سرعة الحزام** باستخدام **PID Controller** لضمان السرعة المطلوبة بدقة.
- **التحكم في السيرفو** بكتابة إشارة PWM يدوية (بدلاً من مكتبة Servo القياسية) حتى يتحرر Pin 6 لاستخدامه كـ PWM.
- **إرسال السرعة الحالية** إلى Python عبر Serial.
- **إيقاف الحزام تلقائياً** إذا انقطع التواصل مع الكمبيوتر لأكثر من 10 حلقات (safety stop).

**دالة PID:**
```cpp
error = target_encoder_pulses - current_encoder_pulses;
cumError += error * elapsedTime;          // integral
rateError = (error - lastError)/elapsedTime; // derivative
output = Kp*error + Ki*cumError + Kd*rateError;
```

**دالة السيرفو اليدوية:**
```cpp
// تحويل الزاوية إلى نبضة PWM بالميكرو ثانية
int val = (degree * 10.25) + 500;
```

---

### 2. `arduino/hardware_test/hardware_test.ino` — كود اختبار الهاردوير

سكريبت بسيط للتأكد من صحة التوصيلات قبل تشغيل النظام الكامل:
- يحرّك الحزام للأمام.
- يحرّك السيرفو ذهاباً وإياباً من الحد الأدنى (40°) إلى الأقصى (150°).
- يطبع قراءة الإنكودر (بالنبضات وبالسنتيمتر) على Serial Monitor.

---

### 3. `programs/main.py` — البرنامج الرئيسي (Python + GUI)

هذا هو البرنامج الذي يعمل على الكمبيوتر. يحتوي على كلاس `App` مبني على **Tkinter**.

#### واجهة المستخدم تحتوي على:
| الحقل | الوصف |
|-------|-------|
| Arduino Port | منفذ Serial للأردوينو (مثل COM3 على Windows أو /dev/ttyUSB0 على Linux) |
| Cam Source | رقم الكاميرا (0, 1, 2 ...) |
| Model | اسم مجلد النموذج داخل `programs/models/` |
| Speed (cm/s) | السرعة المطلوبة للحزام بالسنتيمتر/ثانية |
| Kp, Ki, Kd | معاملات PID Controller |
| White Sensitivity | حساسية كشف الفاكهة بالألوان (كلما زاد كلما تم الكشف عن ألوان أفتح) |
| Score Threshold | عتبة التصنيف (إذا كانت النتيجة أعلى منها → سليمة) |

#### طريقة عمل الكلاس:

```
start()  →  يفتح Serial + يحمّل النموذج + يبدأ الكاميرا → يبدأ update()
update() →  يقرأ إطار من الكاميرا → classify() → HW_update() → يعرض الإطار
classify() → يكشف الفاكهة بالـ HSV mask → يصنّفها بالنموذج → يرسم مستطيل
HW_update() → يرسل الأوامر للأردوينو → يقرأ السرعة الحالية
```

#### خوارزمية الكشف والتصنيف (`classify`):
1. تحويل الإطار من BGR إلى **HSV**.
2. إنشاء **قناع (Mask)** لعزل الأجسام الملوّنة (غير البيضاء) بناءً على White Sensitivity.
3. إيجاد الحدود (Contours) وتجاهل الأجسام الصغيرة (< 2000 بكسل).
4. لكل جسم: قصّ صورته، تغيير حجمها لـ **180×180**، إرسالها للنموذج.
5. إذا كان score > threshold → سليمة (مستطيل أخضر) وإلا → تالفة (مستطيل أحمر).

---

### 4. `programs/fruit_detection&classification.py` — سكريبت الاختبار

نسخة مبسّطة بدون GUI أو Arduino، تعرض نافذة **OpenCV** مباشرةً مع **Trackbars** للتحكم في قيم HSV يدوياً. مفيد لضبط معاملات الكشف.

---

### 5. `programs/dataset&buildingModel/build_model.ipynb` — بناء النموذج

Jupyter Notebook لتدريب نموذج CNN على صور الفاكهة. الداتاسيت المستخدم:
- **[Lemon Quality Dataset](https://www.kaggle.com/datasets/yusufemir/lemon-quality-dataset)** على Kaggle.
- مرجع: **[Lemon - Good or Bad? (99% Accuracy)](https://www.kaggle.com/code/mokshitsurana/lemon-good-or-bad-99-accuracy)**.

البيانات مقسّمة إلى:
- `data/normal/` — صور ليمون جيد الجودة.
- `data/abnormal/` — صور ليمون رديء الجودة.
- `data/other/` — صور خلفية فارغة (empty background).

---

### 6. `programs/models/lemon_quality/` — النموذج الجاهز

نموذج **TensorFlow/Keras** مدرّب مسبقاً لتصنيف الليمون:
- مدخل: صورة 180×180 RGB.
- مخرج: رقم بين 0 و1 (كلما اقترب من 1 → جودة عالية / سليم).
- محفوظ بصيغة **SavedModel** (`saved_model.pb` + `variables/`).

---

## 🔄 كيف يعمل النظام كاملاً — Full System Flow

```
[الكاميرا] ──→ [Python: التقاط الإطار]
                        │
                        ▼
              [كشف الفاكهة بـ HSV Mask]
                        │
                        ▼
              [تصنيف بنموذج TensorFlow]
                        │
              ┌─────────┴──────────┐
           سليمة                 تالفة
              │                    │
              ▼                    ▼
   [إرسال servo_pos=42]  [إرسال servo_pos=148]
              │
              ▼
    [Arduino: PID → سرعة الحزام]
    [Arduino: السيرفو يوجه الفاكهة]
              │
              ▼
    [إرسال السرعة الحالية → Python GUI]
```

---

## 🚀 طريقة التشغيل — How to Use

### 1. رفع كود الأردوينو
افتح `arduino/HW_controller/HW_controller.ino` في **Arduino IDE** ورفعه على الأردوينو UNO.

### 2. تثبيت Python وبيئة العمل
```bash
pip install opencv-python pillow pyserial tensorflow
```

### 3. تشغيل البرنامج
```bash
cd programs
python main.py
```

### 4. إعداد الواجهة
- اكتب منفذ الأردوينو (مثال: `COM3` على Windows أو `/dev/ttyUSB0` على Linux).
- اختر رقم الكاميرا.
- اختر النموذج (`lemon_quality`).
- اضبط السرعة ومعاملات PID وعتبة التصنيف.
- اضغط **Start**.

### 5. بناء نموذج جديد (اختياري)
افتح `programs/dataset&buildingModel/build_model.ipynb` في Jupyter وأضف صورك في مجلدات `data/normal` و`data/abnormal`.

---

## 📸 صور المشروع — Project Photos

| | | |
|--|--|--|
| <img src="media/view1.jpg" width="250"> | <img src="media/view2.jpg" width="250"> | <img src="media/view3.jpg" width="250"> |
| <img src="media/view4.jpg" width="250"> | <img src="media/view5.jpg" width="250"> | <img src="media/view6.jpg" width="250"> |

---

## 🛠️ المكونات والتكاليف — Components & Cost

| المكوّن | الكمية | السعر (جنيه) |
|---------|--------|-------------|
| Arduino UNO SMD | 1 | 325 |
| Webcam | 1 | 100 |
| MG90S Micro Servo Motor | 1 | 145 |
| GW370 DC 12V Geared Motor + Encoder | 1 | 600 |
| L298N Motor Driver | 1 | 65 |
| KP000 Pillow Block Bearing | 4 | 180 |
| Motor Holder Bracket | 1 | 30 |
| AC Adapter 12V/2A | 1 | 50 |
| خشب + تشغيل | — | 1600 |
| **الإجمالي مع الشحن** | | **~2440** |

---

*BY: Abdelraouf Hawash — abdelraouf.hawash@gmail.com*
