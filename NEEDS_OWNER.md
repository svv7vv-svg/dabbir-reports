# NEEDS_OWNER — آخر تحديث 2026-09-05T20:05Z

بنود تنتظرك، مرتّبة بالأولوية. لا تُحذف حتى تؤكّد إنجازها.

## 1) انشر الدفعة — كل ما أُنجز منشور على GitHub وينتظر أمر نشر واحد
الإنتاج ما زال على نشرتك ~09:47 (e35d8d0f). كل ما تحته منشور على الفرع وينتظر:
الطلب العالق (`3fba29a`) · تسريع الإرسال (`32d685e`) · نصوص الربط (`8887831`) · سعر /plans (`c70c56f`) · الخريطة (`531fe1b`) · لوحة الأونر كل الصفحات (`211088e`) · design-system (`79e8518`) · أرقام إنجليزية (`b93102a`) · SCALE_25 سقف 25 + Serper 5000 (`83d6639`،`4bedb00`).
**مهم:** طبّق migration 0093 (البند 2) **قبل** النشر أو بعده — كلاهما آمن، لكن السقف لا يصير 25 إلا بعد تطبيق الـmigration.

**أمر النشر الواحد (من طرفيتك):**
```bash
cd /private/tmp/claude-501/-Volumes-Claude/9dbc5013-8325-4426-b8ca-12796f23e03e/scratchpad/dabbir-src-restored && git pull && npm run build && node scripts/launch/compose-production-config.mjs dist/server/wrangler.production.json && npx wrangler deploy --config dist/server/wrangler.production.json
```
بعده تحقّق: `curl -s https://app.dabbir.app/api/build-identity` (يجب أن يتغيّر عن e35d8d0f)، ثم افتح /go (الخريطة + المؤشر) و/owner (اللوحة الجديدة).

## 2) SCALE_25 — سقف الجلسات 25: **نُفِّذ** (يحتاج تطبيق migration + نشر)
رفعت البوابة الفعلية لرحلة /go إلى 25 (`83d6639`،`4bedb00`، مؤكّد محليًا: 25 يدخلون، الـ26 ينتظر). اكتشاف مهم: **الأثر الموقّع `FIVE_REAL_CONCURRENT_SESSION_PROOF` مجرد رمز تأجيل لا يُقرأ وقت التشغيل، ولا يحمل رقم السعة — فلا تحتاج توقيع أي شيء.** العقد الموقّع/جدول التفاوض تركتهما عند 5 لأنهما نظام مؤجّل منفصل ولا يبوّبان /go.

**اعمل خطوتين بالترتيب (من طرفيتك):**
1) طبّق migration 0093 على D1 الإنتاج (يعيد بناء جدول القبول بـCHECK 1..25، يحفظ الصفوف الحيّة):
```bash
cd /private/tmp/claude-501/-Volumes-Claude/9dbc5013-8325-4426-b8ca-12796f23e03e/scratchpad/dabbir-src-restored && git pull && npx wrangler d1 migrations apply dabbir-official-controlled-release --remote --config wrangler.production.jsonc
```
2) ثم انشر (أمر النشر في البند 1 أعلاه). لو نشرت قبل تطبيق 0093 فلا ضرر — يبقى السقف الفعلي 5 حتى يُطبَّق الـmigration ثم يرتفع 25.

**التدرّج المراقب (اختياري، بالتزامن):** لتثبيت 15 أولًا: اضبط على Cloudflare متغيّر `REAL_SESSION_ACTIVE_LIMIT=15` **مع** `BRIDGE_MAX_SESSIONS=15` على Render، راقب Render Metrics (RAM/CPU)، ثم ارفع الاثنين معًا إلى 25 (أو احذف REAL_SESSION_ACTIVE_LIMIT ليعود للسقف 25). لا تخفض الجسر دون الـworker وحده.

**Serper:** الافتراضي الآن 5000/يوم؛ اضبط `SERPER_DAILY_CALL_BUDGET` على Cloudflare ليطابق باقة Serper عندك (أعطني الباقة لو تبي رقمًا دقيقًا).

**heavy ops (البحث/الإرسال المتزامن) تُركت عند 10 بوعي:** رفعها 10→25 يحتاج إعادة بناء جدول heavy مشتبكًا بـ~15 trigger على مسار الإرسال — خطِر قبل اختبار الإعلانات. heavy=10 وظيفي (ينتظر لحظات تحت الذروة، لا انهيار). أنفّذها كمتابعة مُختبَرة بعناية إن ظهر عنق بحث في القياس.

## 3) أسرار الجسر (M1) — تضبطها أنت على Cloudflare/Render (لا أكشف قيمًا)
- `AKH_BRIDGE_IDENTITY`: تجزئته **متطابقة حرفيًا** staging↔prod (`44e06eb4…5870f0`) → نفس هوية الجسر في البيئتين. أعد توليدها للإنتاج: (1) `openssl rand -hex 32` (2) اضبطها على worker `dabbir-production` (3) نفس القيمة على جسر Render (4) حدّث `AKH_BRIDGE_IDENTITY_SHA256` في wrangler.production.jsonc ثم أعد الرندر/النشر.
- تحقّق (قيمها مشفّرة، لا أقارنها) وأعد توليدها إن تطابقت: `WHATSAPP_BRIDGE_WEBHOOK_SECRET` (الأهم) · `WHATSAPP_BRIDGE_API_KEY` · `WHATSAPP_BRIDGE_REQUEST_SIGNING_SECRET` · `WHATSAPP_SESSION_ENCRYPTION_KEY` (كلٌّ على Cloudflare للـworker + مطابقته على Render، مختلفة عن staging).

## 4) قبل الإطلاق العام (مسجّل، ليس الآن)
- **سقف المجهولين 100→5**: أرجع `ANONYMOUS_DAILY_SEARCH_CAP` (وCAP_CEILING) إلى قيمة الإطلاق قبل فتح العامة (شُحن 100 للاختبار).
- **تأكّد أن `TURNSTILE_SECRET_KEY` مضبوط في الإنتاج** — بدونه الكابتشا تفشل «مفتوحة» (تقبل الكل) عند إنشاء المساحة.
- قرار: rate-limiting per-IP داخل الـWorker أو CF WAF قبل الطفرة (الرايات معلنة غير مطبّقة).

## 5) لوحة الاقتصاد — الأرصدة المالية الحيّة تحتاج مفاتيح
صفحة «الاقتصاد» الجديدة تعرض الاستهلاك المقيس داخليًا (توكنات الذكاء، رسائل المزوّدين، الأرصدة والباقات). لعرض **التكلفة بالريال حيًّا** أضِف مفاتيح قراءة الفوترة: OpenAI (usage API) · Serper (لوحة الحساب) · Cloudflare (Workers analytics). أعطِني إياها (أو اضبطها كأسرار) لأصلها باللوحة.

## 6) تأكيدات سريعة
- **migration 0091**: طُبِّق فعلًا على D1 الإنتاج في جلسة سابقة (جداول الباقات/الاشتراكات/الخصم، بذور الباقات مؤكّدة). للتأكيد فقط.
- **الجسر b6c63c8** على Render: تأكّد أنه منشور وأن الاقتران/الإرسال يعملان بعده.

## 7) مؤجّل بوعي (لا يُلمس الآن)
- تفعيل التفاوض (مبني، الراية مطفأة — بعد الإطلاق التجريبي) · الدفع الفعلي (بوابة) · تطبيق iOS (حساب Apple Developer $99/سنة + قرار IAP مقابل ويب) · حذف المؤلّف الميت (2292 سطر، مربوط بـ~14 اختبارًا).
