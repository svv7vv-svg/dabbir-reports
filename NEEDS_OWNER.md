# NEEDS_OWNER — صباح 2026-09-05 (تجميع ما ينتظرك)

آخر تحديث: يتحدّث خلال الليل. البنود مرتّبة بالأولوية.

## 1) انشر إصلاح الإرسال (حاجب الرحلة الكاملة) — الأهم
الإصلاح `SEND_TRIGGER_FIX` (commit `230c767`) مُودَع ومدفوع على origin، لكن لم يُنشر: نسخة العمل المستعادة في هذه الجلسة فقدت الإعداد المُرندَر (gitignored) وبيان القبول الموقّع بإثباتك الفعلي — فما قدرت أُشغّل النشر النهائي من هنا.

**في مساحة النشر عندك (نفس طريقة نشر بنود اليوم):**
```
git pull
npm run build
npx wrangler deploy --config dist/server/wrangler.production.json
curl -s https://app.dabbir.app/api/build-identity   # يجب أن يتغيّر عن 8148ea564c1a
```
هذا الأمر الواحد ينشر إصلاح الإرسال **وكل ما أنجزته الليلة دفعة واحدة**. بعده: أعد الاقتران على الجسر b6c63c8 وتأكّد أن الإرسال ينطلق خلال ثوانٍ من CONNECTED.

## 2) أسرار الجسر (M1) — تضبطها أنت على Cloudflare/Render (لا أكشف قيمًا)
فحصتُ تطابق الأسرار بين staging↔production بمقارنة تجزئات الإعداد (لا أستطيع رؤية قيم الأسرار المشفّرة في Cloudflare — أنت تتحقق منها).

**مؤكَّد متطابق (يجب إعادة توليده للإنتاج):**
- `AKH_BRIDGE_IDENTITY` — تجزئته `AKH_BRIDGE_IDENTITY_SHA256` **متطابقة حرفيًا** بين staging وproduction (`44e06eb4…5870f0`) → نفس هوية الجسر في البيئتين. اختراق staging يمكنه انتحال production. **أعد توليدها للإنتاج:**
  1. ولّد قيمة جديدة قوية (مثلًا `openssl rand -hex 32`).
  2. اضبطها سرًّا على Cloudflare (worker `dabbir-production`) باسم `AKH_BRIDGE_IDENTITY`.
  3. اضبط **نفس القيمة** على جسر Render الرسمي (هوية الجسر) — لازم الطرفان يتطابقان وإلا يفشل الاقتران/الإرسال.
  4. حدّث `AKH_BRIDGE_IDENTITY_SHA256` في `wrangler.production.jsonc` إلى sha256 للقيمة الجديدة (`printf %s "<القيمة>" | shasum -a 256`)، ثم أعد الرندر/النشر.

**تحقّق أنت (قيمها مشفّرة في Cloudflare — لا أقدر أقارنها)، وأعد توليدها إن تطابقت staging↔prod:**
- `WHATSAPP_BRIDGE_WEBHOOK_SECRET` (الأهم — يوقّع نداءات الجسر→الـWorker)
- `WHATSAPP_BRIDGE_API_KEY`
- `WHATSAPP_BRIDGE_REQUEST_SIGNING_SECRET`
- `WHATSAPP_SESSION_ENCRYPTION_KEY`
  (كل واحد يُضبط على Cloudflare للـworker + القيمة المطابقة على Render للجسر؛ يجب أن تختلف عن staging.)

**نظافة (أولوية أقل — حرّاس توكن داخلية متطابقة staging↔prod):**
- `OPENAI_MINIMAL_PROBE_TOKEN_SHA256` و`REAL_AI_PRECISION_TOKEN_SHA256` — توكنات فحص/كناري داخلية مشتركة؛ ولّد توكنات إنتاج متمايزة إن أردت إغلاقها تمامًا.
- `NEXT_PUBLIC_TURNSTILE_SITE_KEY` عام (ليس سرًّا) لكن يُفضّل مفتاح Turnstile مستقل لكل نطاق/بيئة.

## 3) قبل الإطلاق العام (مسجّل، ليس الآن)
- أعد CAP_CEILING للمجهولين من 100 إلى 50 (env=5). أنت ما زلت تختبر، فتُركت مرتفعة.

## 4) صمود الطفرة (من SCALE_RESILIENCE)
- **تأكّد أن `TURNSTILE_SECRET_KEY` مضبوط في الإنتاج** — بدونه الكابتشا تفشل «مفتوحة» (تقبل) عند إنشاء المساحة.
- **أرجع `ANONYMOUS_DAILY_SEARCH_CAP` إلى 5** قبل الإطلاق العام (شُحن 100).
- قرار: هل نربط rate-limiting per-IP داخل الـWorker أو نضيف CF WAF؟ (الرايات مُعلَنة لكن غير مطبّقة في الكود — لا throttle على إنشاء الطلب/رمز الاقتران داخليًا، فقط بوابات الموارد).

## 5) مراجعة تصاميم (قبل البناء) — في مستودع الكود docs/overnight-2026-09-05/
- **PRICING_SYSTEM_DESIGN + UNIFIED_IDENTITY_DESIGN**: راجع نموذج «الرقم = الهوية، مصدر حقيقة واحد في D1، حارس الدفع المزدوج، دفع متعدد القنوات». وافق لأبني migration الباقات + شاشة /plans + لوحة الأونر.
- **DISCOUNT_CODES_DESIGN**، **NEGOTIATION_DESIGN** (4 قرارات: حد الرسائل/النبرة/شفافية المزود/الإيقاف المبكر).

## 6) تطبيق iOS (IOS_APP_PLAN)
- سجّل **حساب Apple Developer ($99/سنة)** مبكرًا (يأخذ وقتًا) — متى؟
- قرار الدفع في iOS: **IAP بسعر معوَّض** أم **اشتراك ويب فقط** (التطبيق يقرأ اشتراك الويب)؟
- تأكيد: نبني التطبيق **بعد** الإطلاق الأولي للويب (توصيتي) لا بالتوازي.

## 7) الجسر (من BRIDGE_DEPLOY_TARGET سابقًا)
- انشر **b6c63c8** على Render، وتأكّد أن الاقتران والإرسال يعملان بعده (مع إصلاح SEND_TRIGGER_FIX المنشور على الـWorker).

---
كل الكود المُودَع الليلة (SEND_TRIGGER_FIX 230c767 · S1+F1 1abb5c0 · UNDERSTANDING_VISIBILITY 53181cf · docs 58af625) على فرع `archive/release-candidate-2026-09-03` — **نشرة واحدة تشمله كله** (البند 1 أعلاه).
