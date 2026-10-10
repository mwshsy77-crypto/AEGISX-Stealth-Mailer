# شرح كامل لمشروع AEGISX

## 1. الخلاصة التنفيذية

**AEGISX** منصة أتمتة بريد إلكتروني جماعي (Bulk Email Automation) مكتوبة بـ **Python 3.12**. تجمع بين:

- إرسال SMTP غير متزامن وعالي التوازي.
- إدارة حملات بريدية من لوحة ويب أو من سطر الأوامر CLI.
- قوالب HTML ومتغيرات شخصية للمستلمين.
- تدوير عناوين المرسل والموضوع والقوالب والروابط.
- توزيع الحملات على عدة خوادم SMTP مع موازنة وفشل احتياطي.
- تحديد معدلات الإرسال، وإعادة المحاولة، وقاطع الدائرة Circuit Breaker.
- جدولة الإرسال لمرة واحدة أو دوريًا أو بفاصل زمني.
- تتبع الفتح والنقر وإلغاء الاشتراك.
- تسجيل الرسائل الناجحة والفاشلة وطابور Dead Letter.
- Webhooks وإحصاءات ولوحة تحكم لحظية عبر Socket.IO.
- دعم SQLite للتجربة وPostgreSQL/Redis للإنتاج.

الإصدار المعلن في `pyproject.toml` هو **2.1.1**، وحالة المشروع في metadata هي **Beta**. الترخيص MIT.

> مهم: هذا المشروع أداة إرسال بريد شرعية يمكن استخدامها للنشرات والتنبيهات والتواصل المصرّح به. يجب استخدام قوائم مستلمين لديهم موافقة، واحترام قوانين مكافحة الرسائل المزعجة وطلبات إلغاء الاشتراك.

---

## 2. الصورة المعمارية

```text
                 ┌─────────────────────────┐
                 │      Web Dashboard      │
                 │ Flask + Socket.IO       │
                 └───────────┬─────────────┘
                             │
                 ┌───────────▼─────────────┐
                 │      REST API / Views   │
                 │ Auth, Campaigns, SMTP   │
                 └───────────┬─────────────┘
                             │
          ┌──────────────────▼──────────────────┐
          │       Shared Domain Services        │
          │ Campaign, Email, SMTP, Tracking,    │
          │ Scheduler, Bounce, Webhook          │
          └──────────┬──────────────────┬───────┘
                     │                  │
        ┌────────────▼──────────┐  ┌────▼─────────────┐
        │ Async SMTP Engine      │  │ SQLAlchemy ORM   │
        │ Pool, rate, retry,     │  │ Repositories +   │
        │ circuit breaker        │  │ Alembic          │
        └────────────┬──────────┘  └────┬─────────────┘
                     │                  │
              ┌──────▼──────┐    ┌─────▼─────────────┐
              │ SMTP servers │    │ SQLite/PostgreSQL │
              └──────────────┘    └───────────────────┘

                 ┌─────────────────────────┐
                 │ CLI: aegisx             │
                 │ YAML-driven campaigns    │
                 └─────────────────────────┘
```

الفكرة الأساسية: **الويب والـCLI يستخدمان طبقة الخدمات والمحرك نفسها**، لكن تشغيل CLI المعتمد على YAML ليس نسخة مطابقة لحالة قاعدة بيانات لوحة الويب؛ توثيق المشروع يذكر أن الـCLI يقرأ YAML ويكتب السجلات، بينما الويب يعتمد على قاعدة البيانات.

---

## 3. هيكل المجلدات

### ملفات الجذر

| الملف/المجلد | الوظيفة |
|---|---|
| `README.md` | نظرة عامة وبداية سريعة |
| `docs/Deployment.md` | دليل التثبيت والتشغيل والنشر الكامل |
| `docs/API.md` | شرح REST API |
| `docs/openapi.yaml` | مواصفة OpenAPI |
| `docs/SECURITY.md` | سياسة الأمان وقائمة تقوية النشر |
| `pyproject.toml` | تعريف الحزمة والاعتماديات والأوامر |
| `requirements.txt` | متطلبات Python التقليدية |
| `.env.example` | قائمة متغيرات البيئة |
| `run.py` | مشغل التطبيق، ينشئ/يستخدم virtualenv ويشغل Gunicorn |
| `Makefile` | أوامر التطوير والاختبار والبناء |
| `Dockerfile` / `docker-compose.yml` | تشغيل الحاويات |
| `config/` | إعدادات YAML وNginx وplaceholders |
| `data/` | بيانات محلية وقاعدة SQLite عند الاستخدام المحلي |
| `logs/` | سجلات التطبيق |
| `tests/` | اختبارات الوحدة والتكامل، 111 ملف اختبار تقريبًا |

### `src/aegisx/`

| الوحدة | الوظيفة |
|---|---|
| `cli/` | أوامر `aegisx` وإنشاء المشاريع وفحص YAML والإرسال |
| `web/` | Application Factory وBlueprints وREST API وSocket.IO |
| `services/` | منطق الأعمال: الحملات والبريد وSMTP والجدولة والتتبع |
| `engine/` | المحرك منخفض المستوى للإرسال غير المتزامن |
| `features/` | القوالب والمتغيرات والتدوير والجغرافيا والتوليد |
| `data/models/` | نماذج SQLAlchemy |
| `data/repositories/` | طبقة الوصول إلى البيانات |
| `security/` | المصادقة والتشفير وHMAC |
| `migrations/` | سلسلة ترحيلات Alembic |
| `worker/` | طابور ARQ الاختياري لفصل تنفيذ الحملات عن الويب |
| `utils/` | التسجيل، القياسات، الملفات، الأقفال والتحقق |

---

## 4. دورة عمل الحملة

1. **إنشاء الحملة** من لوحة الويب أو ملف YAML.
2. **تحميل الإعدادات**: الموضوع، المرسل، القالب، قائمة المستلمين، خوادم SMTP، المعدلات والميزات.
3. **قراءة المستلمين** من CSV/ملف أو قائمة يدوية.
4. **اكتشاف الترميز**: يحاول UTF-8 ثم ترميزات شائعة لتصديرات Excel مثل CP1251 وGBK وShift-JIS.
5. **التحقق من البريد** وإزالة التكرارات وكبح Suppression إن وُجد.
6. **اختيار القالب والموضوع والمرسل** وفق استراتيجية التدوير.
7. **استبدال المتغيرات** مثل `{{first_name}}` و`{{company}}` وبيانات التاريخ والروابط.
8. **إضافة التتبع**: بكسل فتح، تغليف الروابط، ورابط إلغاء الاشتراك الموقّع.
9. **إنشاء رسالة MIME**: نسخة HTML، نسخة نصية احتياطية، headers، ومرفقات.
10. **اختيار خادم SMTP**: خادم مثبت للحملة أو تدوير weighted/round-robin/priority، مع تفضيل الخادم الذي يملك عنوان From.
11. **تطبيق القيود**: معدل لكل خادم، معدل الحملة، معدل عام، والتزامن.
12. **الإرسال عبر connection pool** غير متزامن.
13. **تصنيف الأخطاء** حسب كود SMTP أو نوع الاستثناء:
    - مؤقتة: يعاد إرسالها.
    - Rate limit: تؤجل.
    - Authentication أو mailbox أو 5xx دائم: تفشل غالبًا نهائيًا.
14. **إعادة المحاولة Backoff** للأخطاء المؤقتة.
15. **فتح Circuit Breaker** عند تكرر فشل خادم SMTP، لمنع إغراقه بطلبات جديدة.
16. **تسجيل النتيجة** في `EmailLog` وتحديث عدادات الحملة.
17. **إرسال أحداث Socket.IO/Webhook** لإظهار التقدم أو إخطار الأنظمة الخارجية.
18. **نقل الرسائل التي استنفدت المحاولات إلى Dead Letter Queue** لإعادة المحاولة يدويًا أو التخلص منها.

---

## 5. المكونات الرئيسية بالتفصيل

### 5.1 محرك SMTP

الموجود في `engine/` ويشمل:

- `connection_pool.py`: يحتفظ باتصالات SMTP قابلة لإعادة الاستخدام، ويقيس زمن المصافحة والإرسال.
- `async_sender.py`: يبني الرسالة ويرسلها ويعيد `EmailResult` أو `BulkSendResult`.
- `rate_limiter.py`: Token bucket وحدود بالدقيقة/الساعة.
- `circuit_breaker.py`: حالات closed/open/half-open تقريبًا لمنع استخدام خادم متعطل.
- `retry_queue.py`: جدولة المحاولة التالية مع backoff وحفظ حالة الطابور على القرص.
- `dead_letter_worker.py`: معالجة الرسائل الفاشلة نهائيًا.
- `enhanced_sender.py`: امتداد للإرسال مع سلوكيات إضافية.

التوزيع بين الخوادم يدعم:

- Weighted: الخادم ذو الوزن الأكبر يستقبل حصة أكبر.
- Round-robin: تناوب دوري.
- Priority: تفضيل الأولوية الأعلى.
- From-aware routing: إذا كان عنوان From مرتبطًا بخادم معين، يوجه الإرسال إليه لتقليل رفض SMTP من نوع “From غير مصرح به”.

### 5.2 خدمة البريد والقوالب

`services/email/service.py` تنسق بين القالب، المتغيرات، التتبع، المرفقات، والتشفير/الترميز قبل تسليم الرسالة إلى المحرك.

محرك القالب `features/template_engine.py` يدعم:

- `{{variable}}` للاستبدال.
- `{{if:variable}}...{{else}}...{{endif}}` للمحتوى الشرطي.
- `{{include:path/file.html}}` لتضمين أجزاء القالب ضمن المسارات المسموح بها.
- Spintax بصيغة `{Hello|Hi}`.
- QR Code عند التفعيل.
- أكثر من 50 متغيرًا مدمجًا، من بيانات المستلم والتاريخ والروابط إلى fake placeholders.
- ملفات placeholders إضافية بصيغة YAML/JSON.

### 5.3 التتبع والتحليلات

`services/tracking_service.py` و`web/routes/tracking.py` يدعمان:

- فتح الرسالة عبر صورة 1×1 في `/track/open/<email_id>`.
- النقر عبر `/track/click/<email_id>` مع تمرير الرابط الأصلي.
- إلغاء الاشتراك عبر `/track/unsubscribe/<email_id>/<token>`.
- إحصاءات الحملة: opens/clicks/unsubscribes، الزمن، وبعض بيانات الموقع الجغرافي عند توفرها.

توكن إلغاء الاشتراك موقّع HMAC-SHA256 ويحتوي معرف الرسالة وhash للبريد ووقت الانتهاء. المفتاح هو `UNSUBSCRIBE_SECRET` أو `SECRET_KEY` كبديل.

**ملاحظة تقنية مهمة:** سجل `email_id -> recipient` موجود في الذاكرة كتحسين سريع، مع fallback إلى `EmailLog`. لذلك يجب الحفاظ على سجلات قاعدة البيانات وتهيئة `TRACKING_BASE_URL` الصحيح عند الإنتاج.

### 5.4 الجدولة

`services/scheduler_service.py` يستخدم APScheduler ويدعم:

- تشغيل مرة واحدة.
- Cron متكرر.
- تشغيل بفاصل زمني ثابت.
- Pause / Resume / Cancel / Reschedule.
- `max_runs` لإيقاف الوظيفة بعد عدد معين.

التخزين الافتراضي للمهام هو `MemoryJobStore`، ولذلك يجب الانتباه إلى أن وظائف الجدولة قد لا تبقى بعد إعادة تشغيل العملية ما لم تتم إضافة طبقة تخزين دائمة.

### 5.5 قاعدة البيانات

SQLAlchemy 2.x مع Alembic. النماذج الأساسية:

- `Campaign`: الحملة وحالتها وإحصاءات الإرسال وإعداداتها.
- `Template`: قوالب HTML/text والموضوعات البديلة.
- `RecipientList` و`Recipient`: القوائم والمستلمون والحقول المخصصة وحالة suppression.
- `SMTPServer`: الخادم، TLS، الحدود، الإحصاءات، وكلمة المرور المشفرة.
- `CampaignSMTPConfig`: ربط الحملة بعدة خوادم مع weight/priority.
- `EmailLog`: سجل كل رسالة ونتيجتها وأحداثها.
- `DeadLetter`: الرسائل التي فشلت بعد المحاولات.
- `Attachment`: مكتبة المرفقات.
- `FromEmail` و`SenderName`: هوية المرسل والتدوير.
- `CustomPlaceholder`: متغيرات يضيفها المستخدم.
- `GlobalSetting`: الإعدادات العامة.
- `User`: المستخدمون والمصادقة.

حالات الحملة: `draft`, `scheduled`, `sending`, `paused`, `completed`, `failed`, `cancelled`.

القاعدة المحلية الافتراضية SQLite، أما الإنتاج الموصى به فهو PostgreSQL. `pool_pre_ping` و`pool_recycle` مفعّلان للاتصالات الشبكية.

### 5.6 الويب ولوحة التحكم

Application Factory في `web/app.py` تهيئ:

- Flask وFlask-Login.
- CSRF وCORS وFlask-Limiter.
- Socket.IO.
- Blueprints للواجهة وAPI والتتبع والصحة والأدوات.
- حلقة asyncio خلفية مشتركة.
- سجلات منظمة وPrometheus/Sentry اختياريًا.
- Security headers وقيود cookies.

أهم صفحات الويب:

| الصفحة | المسار |
|---|---|
| Dashboard | `/` |
| الحملات | `/campaigns` |
| SMTP | `/smtp` |
| القوالب | `/templates` |
| المستلمون | `/recipients` |
| الجدولة | `/scheduling` |
| الارتدادات | `/bounces` |
| Dead Letter | `/dead-letter` |
| Webhooks | `/webhooks` |
| السجلات | `/logs` |
| الإعدادات | `/settings` |
| الأدوات | `/tools` |

---

## 6. التشغيل المحلي خطوة بخطوة

### المتطلبات

- Python **3.12.x تحديدًا**؛ `pyproject.toml` يرفض 3.13 وما بعدها.
- RAM: 2GB كحد أدنى، و4GB أفضل للحملات الكبيرة.
- اتصال صادر إلى SMTP على 25/465/587.
- PostgreSQL وRedis اختياريان محليًا، لكنهما موصى بهما للإنتاج.

### التثبيت الموصى به

```bash
cd AEGISX-main
./install.sh
source .venv/bin/activate
```

أو يدويًا:

```bash
python3.12 -m venv .venv
source .venv/bin/activate
pip install -e .
cp .env.example .env
```

### الإعداد الأدنى

أنشئ مفتاحًا عشوائيًا:

```bash
python -c "import secrets; print(secrets.token_urlsafe(48))"
```

ثم ضع في `.env`:

```dotenv
SECRET_KEY=مفتاح_عشوائي_قوي
ADMIN_USERNAME=admin
ADMIN_PASSWORD=كلمة_مرور_قوية
ADMIN_EMAIL=admin@example.com
FLASK_ENV=development
TRACKING_BASE_URL=http://127.0.0.1:5000
```

في أول تشغيل ينشئ التطبيق المدير الأول عند وجود المتغيرات الثلاثة `ADMIN_*`، ويضع `must_change_password=True`.

### الترحيلات والتشغيل

```bash
aegisx db migrate
python run.py
```

ثم افتح:

```text
http://127.0.0.1:5050
```

يوجد اختلاف يجب الانتباه إليه: بعض README/التوثيق يذكر المنفذ 5000، لكن `run.py` يستخدم افتراضيًا `PORT` أو **5050**. يمكن توحيده مثلًا:

```bash
PORT=5000 python run.py
```

الـCLI لا يقرأ `.env` تلقائيًا، لذلك عند استخدامه:

```bash
set -a; source .env; set +a
```

### تشغيل CLI

```bash
aegisx new project
# عدّل config/campaign.yaml و data/recipients.csv
aegisx check config/campaign.yaml
aegisx test config/campaign.yaml
aegisx send config/campaign.yaml --preview
aegisx send config/campaign.yaml
```

أوامر أخرى:

```bash
aegisx show stats
aegisx show logs
aegisx start server
aegisx db current
```

ابدأ دائمًا بـ `--preview` أو `dry_run: true` للتأكد من القالب والمستلمين دون إرسال فعلي.

---

## 7. صيغة حملة YAML

الصيغة الصحيحة الحديثة لخادم SMTP هي:

```yaml
campaign:
  name: "Newsletter"
  description: "Monthly newsletter"

smtp_providers:
  - name: primary
    host: smtp.example.com
    port: 587
    username: "${SMTP_USER}"
    password: "${SMTP_PASS}"
    tls_mode: starttls
    max_per_minute: 30
    max_per_hour: 500
    weight: 2

email:
  subject: "مرحبًا {{first_name}}"
  from_email: sender@example.com
  from_name: "Your Company"
  reply_to: reply@example.com

template:
  html: templates/email.html

recipients:
  source: data/recipients.csv
  email_column: email
  validate: true
  deduplicate: true

sending:
  dry_run: true
  concurrency: 50
  chunk_size: 1000
  pause_between_chunks: 5
  rate_per_minute: 100
  rate_per_hour: 2000

features:
  qr_codes: false
  pdf_attachments: false

placeholders:
  static:
    company_name: "Your Company"

smtp_rotation:
  strategy: weighted
```

### ملاحظة على `config/config.yaml` المرفق

الملف الموجود في الأرشيف يستخدم الحقول القديمة `use_tls: true` و`use_ssl`. بينما README وAPI في الإصدار 2.x يذكران أن الواجهة الحديثة تستخدم حقلًا واحدًا:

```yaml
tls_mode: none       # بدون TLS
tls_mode: starttls   # غالبًا للمنفذ 587
tls_mode: ssl        # غالبًا للمنفذ 465
```

لذلك لا تعتمد على `config/config.yaml` كما هو في الإنتاج قبل تحديثه إلى `tls_mode` والتحقق عبر:

```bash
aegisx check config/config.yaml
```

---

## 8. القالب والمتغيرات

مثال `templates/email.html`:

```html
<h1>مرحبًا {{first_name}}!</h1>
<p>شركتك: {{company}}</p>
<p>{{date_formatted}}</p>

{{if:company}}
<p>هذه رسالة مخصصة لشركتك.</p>
{{else}}
<p>مرحبًا بك.</p>
{{endif}}

<a href="{{link}}">عرض التفاصيل</a>
<a href="{{unsubscribe_link}}">إلغاء الاشتراك</a>
```

مصادر المتغيرات:

1. بيانات صف CSV مثل `first_name`, `last_name`, `company`.
2. متغيرات ثابتة في YAML أو `config/placeholders.yaml`.
3. متغيرات مخصصة من قاعدة البيانات.
4. متغيرات مدمجة للتاريخ والوقت والروابط والتتبع.
5. `{{fake.*}}` عند توفر Faker.
6. `{{location.*}}` عند تثبيت GeoLite2 وضبط `AEGISX_GEOIP_DB`.

---

## 9. REST API

قاعدة API هي `/api`. طرق المصادقة:

- `X-API-Key` عند ضبط `API_KEYS`.
- جلسة المتصفح بعد تسجيل الدخول.

نقاط عامة أو شبه عامة:

- `GET /api/status`
- `GET /live`
- `GET /ready`
- `GET /health/detailed`

أهم مجموعات API:

| المجموعة | أمثلة |
|---|---|
| Campaigns | `GET/POST /api/campaigns`, `POST /api/campaigns/<id>/start`, clone/stats |
| SMTP | `GET/POST /api/smtp`, test، health |
| Templates | list وpreview |
| Recipients | upload، preview، delete |
| Scheduling | إنشاء/إيقاف/استئناف/إلغاء jobs |
| Logs/Stats | نجاح، فشل، إحصاءات |
| Dead Letter | list، retry، requeue، discard، stats |
| Bounces | list وstats |
| Placeholders | قراءة وإدارة custom placeholders |
| Webhooks | تسجيل وحذف ومتابعة الأحداث |

حدود API الافتراضية المذكورة في التوثيق: القراءة 30 طلبًا/دقيقة، الكتابة 10/دقيقة، اختبار SMTP خمسة/دقيقة. في الإنتاج يجب ضبط Redis حتى لا تضيع الحدود عند إعادة التشغيل أو تعدد العمال.

حقل SMTP الحديث في API:

```json
{
  "name": "primary",
  "host": "smtp.example.com",
  "port": 587,
  "username": "user",
  "password": "secret",
  "tls_mode": "starttls"
}
```

كلمات المرور لا تعاد في responses.

---

## 10. Docker والإنتاج

`docker-compose.yml` يبني topology تتكون من:

- `web`: Flask/Gunicorn على 5050 داخليًا.
- `migrations`: حاوية one-shot تنفذ `aegisx db migrate`.
- `db`: PostgreSQL 15.
- `redis`: Redis 7 مع password.
- `proxy`: Caddy اختياري مع TLS عبر profile `proxy`.
- `worker`: ARQ اختياري عبر profile `worker`.

التشغيل العام:

```bash
docker compose up -d
```

مع Caddy:

```bash
docker compose --profile proxy up -d
```

لتفعيل عامل الحملات يجب ضبط:

```dotenv
CAMPAIGN_EXECUTION_MODE=worker
SOCKETIO_MESSAGE_QUEUE=redis://...
CAMPAIGN_QUEUE_REDIS=redis://...
```

### قاعدة التوسع المهمة

`run.py` يفرض Gunicorn بـ worker واحد لأن حلقة asyncio، وجسر Socket.IO، وبعض محددات المعدل/المسابح موجودة داخل العملية. لا ترفع `WEB_CONCURRENCY` إلا بعد توفير:

- Redis لـ `RATE_LIMIT_STORAGE`.
- `SOCKETIO_MESSAGE_QUEUE` للبث بين العمال.
- `CAMPAIGN_EXECUTION_MODE=worker`.

الإنتاج يجب أن يكون خلف reverse proxy وTLS. لا تعرض منفذ التطبيق مباشرة على الإنترنت دون حماية.

---

## 11. الأمان

الإجراءات الموجودة:

- لا توجد بيانات دخول افتراضية `admin/admin`.
- رفض الإقلاع في الإنتاج عند غياب `SECRET_KEY` أو `ADMIN_PASSWORD`.
- رفض SQLite ومخزن rate limit الذاكري في الإنتاج افتراضيًا، مع escape hatch صريح.
- جلسات Flask موقعة، وCookies مع `HttpOnly` و`SameSite=Lax`، و`Secure` في الإنتاج.
- كلمات المرور مخزنة بتجزئة PBKDF2-SHA256 مع 480,000 iteration في `auth.py`.
- بيانات SMTP مشفرة في السكون بمفتاح ومِلْح installation-specific.
- روابط إلغاء الاشتراك موقعة HMAC.
- API keys بمقارنة constant-time.
- CSRF وrate limiting وSecurity headers.
- مسارات include في القوالب تتحقق من بقاء المسار داخل المجلد المسموح.
- Webhooks تحتوي فحص URL وتوقيعًا اختياريًا.

قائمة تقوية عملية:

```dotenv
FLASK_ENV=production
SECRET_KEY=<قيمة عشوائية قوية>
ADMIN_USERNAME=<اسم غير متوقع>
ADMIN_PASSWORD=<كلمة مرور قوية>
ADMIN_EMAIL=<بريد صحيح>
DATABASE_URL=postgresql://...
RATE_LIMIT_STORAGE=redis://...
TRACKING_BASE_URL=https://mail.example.com
UNSUBSCRIBE_SECRET=<مفتاح مستقل مفضل>
API_KEYS=<مفاتيح طويلة عشوائية>
```

لا تضع `.env` أو `data/.encryption.salt` أو كلمات SMTP في Git. احفظ نسخة احتياطية مشفرة لقاعدة البيانات وملف salt معًا؛ فقدان salt قد يمنع فك تشفير بيانات SMTP القديمة.

---

## 12. الاختبارات وجودة الكود

المشروع يحتوي تقريبًا على:

- 134 ملف Python في `src/aegisx`.
- 24 ألف سطر مصدر تقريبًا.
- 111 ملف اختبار.
- 25 ألف سطر اختبار تقريبًا.

الاختبارات تغطي SMTP، الحملات، API، المصادقة، CSRF، التشفير، التتبع، الجدولة، dead letter، Socket.IO، القوالب، المستودعات، الصحة، والتكوين.

أوامر الجودة:

```bash
pip install -e ".[dev]"
make test
make test-fast
make lint
make type-check
make security
make ci
```

فحص البناء:

```bash
make build-check
```

لم يتم في هذه المراجعة تنفيذ إرسال بريد أو تشغيل الإنتاج؛ ذلك يتطلب تثبيت الاعتماديات وإدخال SMTP حقيقي وبيانات بيئة. راجع الاختبارات محليًا بعد إعداد Python 3.12.

---

## 13. نقاط قوة المشروع

1. فصل جيد نسبيًا بين web/services/engine/data.
2. دعم CLI وWeb فوق منطق مشترك.
3. معالجة عملية لأخطاء SMTP مع تصنيف transient/permanent.
4. connection pooling وrate limiting وcircuit breaker مناسبة للإرسال الجماعي.
5. tracking وunsubscribe وwebhooks مدمجة بدل تركها للمستخدم.
6. PostgreSQL/Redis وDocker وAlembic تجعل النشر الإنتاجي ممكنًا.
7. تغطية اختبارية واسعة جدًا على مستوى الملفات.
8. حماية واضحة من الإعدادات الخطرة في الإنتاج.
9. التحقق من ترميز CSV مفيد عند استيراد ملفات Excel.
10. التوثيق التشغيلي مفصل، خصوصًا `Deployment.md`.

---

## 14. الملاحظات والمخاطر التي يجب إصلاحها أو التحقق منها

### أ. تعارض المنفذ
التوثيق و`README.md` يذكران 5000 في أماكن، بينما `run.py` الافتراضي 5050. يجب توحيدها أو توثيق استخدام `PORT` بوضوح.

### ب. إعداد SMTP قديم في `config/config.yaml`
الملف يستخدم `use_tls/use_ssl`، لكن الإصدار الحديث يعتمد `tls_mode`. يجب تحديث المثال حتى لا يفشل المستخدم عند تطبيقه.

### ج. التخزين الذاكري للجدولة والتتبع
`SchedulerService` يستخدم `MemoryJobStore`، و`TrackingService` يحتفظ بجزء من mapping في الذاكرة. هذا مناسب للتطوير، لكن يلزم تخزين دائم مؤكد في بيئة متعددة العمال أو بعد إعادة التشغيل.

### د. فصل حالة CLI عن الويب
توثيق المشروع يصرح بأن CLI ليس مدعومًا بالكامل من قاعدة بيانات الويب. ينبغي تحديد المسار التشغيلي الأساسي للفريق: إما لوحة الويب، أو CLI، أو توحيد الحالتين في إصدار لاحق.

### هـ. عدد العمال
تشغيل أكثر من worker دون Redis/Socket.IO queue/worker execution قد يؤدي إلى حالة غير متناسقة أو فقدان تحديثات التقدم. يجب الالتزام بعامل واحد افتراضيًا.

### و. الخصوصية والامتثال
التتبع يخزن IP وUser-Agent وأحداث الفتح والنقر. يجب إضافة سياسة احتفاظ وحذف واضحة، والامتثال للخصوصية واشتراطات الموافقة في البلد المستهدف.

### ز. قابلية التسليم Deliverability
الميزات مثل `send_as_image` أو التدوير العشوائي لا تضمن وصول الرسائل وقد تضر الوصولية والسمعة. يجب استخدام SPF وDKIM وDMARC، ومراقبة bounce/complaint، وتجنب إرسال جماعي إلى قوائم غير مؤكدة.

### ح. الاعتماديات والتوافق
المشروع يثبت Python على 3.12 تحديدًا ويحتوي على حزم pinned كثيرة. يجب بناء image/venv نظيف وتشغيل `pip-audit` دوريًا قبل النشر.

---

## 15. المسار العملي المقترح لتجربة آمنة

1. تثبيت Python 3.12.
2. إنشاء virtualenv وتثبيت الحزمة.
3. نسخ `.env.example` وتعبئة أسرار محلية غير حقيقية للإنتاج.
4. استخدام SQLite و`FLASK_ENV=development` للتجربة فقط.
5. تحديث `tls_mode` في YAML.
6. إضافة مستلمين تجريبيين يملكون الموافقة.
7. تشغيل `aegisx check`.
8. تشغيل `aegisx send ... --preview` أو `dry_run: true`.
9. تشغيل اختبارات `make test-fast`.
10. اختبار SMTP برسالة واحدة إلى بريد مملوك لك.
11. تجربة tracking وunsubscribe.
12. قبل الإنتاج: PostgreSQL + Redis + TLS + reverse proxy + نسخ احتياطي + SPF/DKIM/DMARC.
13. تنفيذ migration خارج عملية الويب في الإنتاج.
14. البدء بعامل ويب واحد، ثم التوسع فقط بعد تفعيل البنية المشتركة.

---

## 16. الملفات المرجعية الأهم

- `README.md`: بداية سريعة ونظرة على المميزات.
- `docs/Deployment.md`: المرجع التشغيلي الأشمل.
- `docs/API.md`: شرح نقاط API.
- `docs/SECURITY.md`: تقوية الأمان والتبليغ عن الثغرات.
- `src/aegisx/services/campaign_service.py`: تحميل الحملة ودورة التنفيذ.
- `src/aegisx/services/email/service.py`: تجهيز وإرسال البريد.
- `src/aegisx/engine/async_sender.py`: الإرسال الفعلي وتصنيف أخطاء SMTP.
- `src/aegisx/engine/connection_pool.py`: الاتصالات والتوزيع بين الخوادم.
- `src/aegisx/features/template_engine.py`: القالب والمتغيرات والـconditionals.
- `src/aegisx/services/tracking_service.py`: التتبع والروابط.
- `src/aegisx/web/app.py`: تهيئة Flask والإضافات وقواعد الإنتاج.
- `src/aegisx/security/auth.py`: المستخدمون وكلمات المرور وHMAC.
- `docker-compose.yml`: topology الإنتاج بالحاويات.

## الخلاصة

AEGISX عبارة عن **منصة إرسال حملات بريدية متعددة القنوات التشغيلية** وليست مجرد مكتبة SMTP. أقوى أجزاء المشروع هي محرك الإرسال غير المتزامن، إدارة عدة SMTP، التتبع، القوالب، والـAPI. لتشغيله بصورة سليمة يجب أولًا حل نقطتي التوافق الظاهرتين في النسخة المرفقة: **توحيد المنفذ** و**استبدال إعدادات `use_tls/use_ssl` القديمة بـ`tls_mode`**، ثم البدء بـdry-run قبل أي إرسال حقيقي.
