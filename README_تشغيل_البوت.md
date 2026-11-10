# تشغيل FB Comment to Order Bot — دليل الإعداد والحل

## 1) التعديلات اللي اتعملت في الورك فلو

| المشكلة | قبل | بعد |
|---|---|---|
| Verify Token | `my_verify_token_123` | `my_verify_token_122` (مطابق للتوكن اللي في فيسبوك) |
| رد الـ challenge | `$('FB Verify (GET)').item.json.query['hub.challenge']` | `$json.query['hub.challenge'] \|\| $json.query.hub_challenge` |
| Save Lead | مفيش أعمدة متظبطة + مفيش matching | كل الأعمدة متظبطة + matching على `user_id` |
| Save Order | مفيش أعمدة متظبطة | كل الأعمدة متظبطة |
| حالة الورك فلو | غير منشور (Draft) | منشور/Active |
| مصدر الكومنتات | Webhook `feed` (مش بيشتغل في Development Mode) | **Polling**: `Every Minute` → `Fetch Comments` → `Parse Events` |
| مصدر رسايل الماسنجر | Webhook `messages` (مش بيوصل في Development Mode) | **Polling**: `Every Minute` → `Fetch Conversations` → `Parse Events` |
| Get Product | لو مفيش صف مطابق بيوقف الورك فلو | `alwaysOutputData` → بيرد برسالة عامة بدل ما يسكت |
| التوكن | منتهي / ناقص صلاحيات | Page Token سليم + `pages_read_user_content` + `pages_manage_engagement` |

> ملف الـ JSON المحدّث: `FB Comment to Order Bot.json` — لو استوردته من جديد لازم تتأكد إنه Active.

### شكل الورك فلو بعد التعديل
```
Every Minute (كل دقيقة) ──► Fetch Comments (Graph API) ───────┐
                         └─► Fetch Conversations (Graph API) ─┤
                                                              ├──► Parse Events (كود واحد بيوحّد كل الأشكال)
FB Events (POST) [webhook] ───────────────────────────────────┘
        └──► Is Comment? ──► Get Product ──► Build Comment Reply ──► Reply To Comment
                                                                 └──► Send Private DM ──► Prepare Lead ──► Save Lead
                                                                                       └──► Get Lead ──► Build Prompt ──► AI Agent ...
```
- الفرعين بيدخلوا على نفس نود `Parse Events` وده **مقصود**: الكود بيفهم شكل الويب هوك وشكل رد الـ Graph API (كومنتات + محادثات) ويحوّلهم لنفس الحقول، وبعدها باقي الورك فلو مشترك.
- الكود بيشيل التكرار (بيسجل الـ comment ids في `processedComments` والـ message ids في `processedMessages` جوه static data) وبيتجاهل حاجات الصفحة نفسها.
- أول تشغيل بيسجّل الكومنتات القديمة من غير ما يرد عليها. أما الرسايل، أول تشغيل بيرد على **أحدث رسالة** بس لكل محادثة (عشان مفيش سبام لرسايل قديمة).
- **رسالة واحدة كل دورة:** لو أكتر من عميل بعت في نفس الدقيقة، البوت بيرد على واحد ويسيب الباقي للدقيقة اللي بعدها (عشان الـ AI Agent يشتغل بسياق صح).
- ⚠️ فيسبوك مش بيرجّع حقل `from` (اسم/آي دي صاحب الكومنت) في رد الـ Graph API، ولا `parent` كمان — عشان كده الكود مش بيعتمد عليهم.
  النتيجة: الترحيب بيبقى "أهلاً حضرتك" بدل اسم العميل، لكن الرد والرسالة الخاصة شغالين عادي (لأنهم بيستخدموا `comment_id`).
- الردود بتاعتنا بتبقى nested تحت كومنت العميل، فمش بترجع تاني في لستة الكومنتات الأساسية → مفيش loop.

---

## 2) إعداد فيسبوك (بالترتيب — مهم جدًا)

### أ) Webhooks / Callback URL
- **Callback URL:** `https://dimple-lubricant-unlatch.ngrok-free.dev/webhook/fb-sales-bot`
- **Verify Token:** `my_verify_token_122`
- لازم n8n يكون شغال + ngrok شغال + الورك فلو **Active** وقت الضغط على Verify.
- لو ظهر `The callback URL or verify token couldn't be validated`:
  1. افتح اللينك في المتصفح:
     `https://dimple-lubricant-unlatch.ngrok-free.dev/webhook/fb-sales-bot?hub.mode=subscribe&hub.challenge=123&hub.verify_token=my_verify_token_122`
     لازم يرجع `123` نص عادي. لو رجع 404 → الورك فلو مش Active.
  2. لو رجع `Forbidden` → التوكن مش مطابق.
  3. متستخدمش لينك `/webhook-test/` في فيسبوك نهائيًا (شغال لطلب واحد بس وبعدها بيقف).

### ب) الاشتراك في الأحداث (Webhook Subscription)
بعد ما الـ Verify ينجح:
1. في App Dashboard → Messenger → Settings → Webhooks، اختار الصفحة.
2. اشترك في: `feed` و `messages` (وكمان `messaging_postbacks` لو محتاج).
3. لو ظهر `Webhook subscription failed`:
   - تأكد إن الـ Verify نجح الأول (المشكلة دي غالبًا بتاعة Verify مش الاشتراك).
   - تأكد إنك Admin في الصفحة وفي التطبيق.
   - جرّب من متصفح تاني / Incognito (الخطأ ده بيحصل كتير بسبب كاش الداشبورد).
   - أو اشترك يدوي من Graph API Explorer:
     ```
     POST /{PAGE_ID}/subscribed_apps
       ?subscribed_fields=feed,messages,messaging_postbacks
       &access_token={PAGE_ACCESS_TOKEN}
     ```

### ج) الصلاحيات المطلوبة
- `pages_show_list`
- `pages_read_engagement`
- `pages_read_user_content` ← **مهمة لقراءة كومنتات المستخدمين** (من غيرها بيرجع `(#10)`)
- `pages_manage_engagement` ← **مهمة للرد على الكومنتات** (من غيرها بيرجع `(#200)`)
- `pages_manage_metadata`  ← مهمة للاشتراك في الـ webhook
- `pages_messaging`        ← مهمة لإرسال الرسائل

> **مهم:** الصلاحيات دي مش بتظهر في Graph API Explorer غير لو الـ Use Case بتاعها مضاف في التطبيق.
> `pages_read_user_content` و `pages_manage_engagement` تابعين لـ **Manage everything on your Page**:
> App Dashboard → **Use cases** → **Add use case** → **Pages** → *Manage everything on your Page* → Save،
> وبعدين **Customize** جوه الـ Use Case واضغط **Add** قدام كل صلاحية.
> إضافة الـ Use Case لوحدها مش بتدي الصلاحية — لازم زرار Add.

> في وضع Development، **إرسال** الرسائل بيتم بس لحسابات عندها Role في التطبيق (علشان كده محتاج تضيف الحساب التاني كـ Tester).
> لكن **قراءة الكومنتات والرد عليها وقراءة المحادثات** شغالة عادي بالـ polling مع أي حساب.
> يعني البوت هيشوف رسالة العميل ويرد عليها، لكن لو الحساب مش Tester ممكن فيسبوك يرفض **الإرسال** في Development Mode —
> الحل النهائي للعملاء الحقيقيين: التطبيق يبقى **Live** + App Review على `pages_messaging`.

---

## 3) التوكن المستخدم في n8n
الـ HTTP Header Auth credential (اسمه **FaceBook**) لازم يكون فيه **Page Access Token** (مش User Token)، بالشكل:
```
Authorization: Bearer {PAGE_ACCESS_TOKEN}
```
لأن الرد على الكومنت وإرسال الرسائل بيتم باسم الصفحة.

**الوضع الحالي:** محطوط Page Token لصفحة `Mixed Showy` (id `550829251444940`) و**مش بينتهي** (`expires_at = 0`).
طريقة عمله:
1. User Token طويل المدة (Extend Access Token من Access Token Debugger) — ده اللي بيبقى `expires_at = 0`.
2. `GET /{page-id}?fields=access_token` → بيرجع Page Token بيورث نفس المدة (مش بينتهي).

> لو التوكن اتغير يومًا: اعمل الخطوتين دول وحط الناتج في credential **FaceBook** ثم `docker restart n8n_con`.

---

## 4) ملاحظات مهمة
- **ngrok مش مطلوب** مع الـ Polling: n8n هو اللي بيسأل فيسبوك كل دقيقة، مش فيسبوك اللي بيبعت. (سيب الويب هوك زي ما هو كخطة احتياطية لو التطبيق اتحول Live).
- **التوكن لازم يكون long-lived:** توكن الـ Explorer بيخلص بعد ساعة–ساعتين. الصحيح: خد Long-lived User Token وبعدها `GET /{page-id}?fields=access_token` — التوكن اللي بيرجع من long-lived user token مش بينتهي. ✅ (اتعمل)
- **الرد على كومنت بيشتغل مع أي حساب** (لأنه بيتم باسم الصفحة)، لكن **الرسالة الخاصة** في Development Mode بتوصل بس لأصحاب الـ Role في التطبيق.
- **الشات (استكمال الأوردر) شغال بالـ polling كمان** — البوت بيسحب المحادثات كل دقيقة من `me/conversations` ويرد على رسالة العميل. مش محتاج الـ webhook.
- **الصفحة ثابتة في الكود:** الـ `PAGE_ID` مكتوب في أول نود `Parse Events` (`550829251444940`). لو غيّرت الصفحة، غيّر الرقم ده.
- **رسالة واحدة كل دقيقة:** لو أكتر من عميل بعت في نفس اللحظة، البوت بيرد على واحد والباقي بيتأجل للدقيقة اللي بعدها.
- **Private Reply:** الرد الخاص على كومنت شغال خلال 7 أيام من الكومنت.
- **نافذة الرسائل:** الرد على رسالة العميل متاح خلال 24 ساعة من آخر رسالة منه.
- **الـ Sheet:** تأكد إن أسماء الشيتات `Products` / `Leads` / `Orders` موجودة، وإن عمود `post_id` في Products مطابق لـ post id الحقيقي.
- **ترتيب أعمدة Products مهم:** `post_id | name | price | sizes | colors | details | shipping`.
  `details` = وصف المنتج، و`shipping` = رقم الشحن بس (زي `30`). لو اتبدلوا، الرسالة هتطلع غلط
  (البوت بيضيف «جنيه» تلقائي لو الشحن رقم).
- **لو البوست مش موجود في Products:** البوت لسه بيرد على الكومنت وبيبعت رسالة خاصة **عامة** («قولي حضرتك مهتم بأنهي منتج...») — ده بفضل `alwaysOutputData` على نود `Get Product`. فلو عايز تفاصيل المنتج تطلع تلقائي، لازم صف في Products.
- **الأعمدة في Google Sheets:** لو ظهر خطأ زي `Sheet with name not found` افتح نود Google Sheets واختار الشيت من القائمة تاني واحفظ.

---

## 5) اختبار سريع
```powershell
# 1) اختبار الـ verification (لازم يرجع 123)
curl.exe "https://dimple-lubricant-unlatch.ngrok-free.dev/webhook/fb-sales-bot?hub.mode=subscribe&hub.challenge=123&hub.verify_token=my_verify_token_122"

# 2) اختبار توكن غلط (لازم يرجع 403 Forbidden)
curl.exe "https://dimple-lubricant-unlatch.ngrok-free.dev/webhook/fb-sales-bot?hub.mode=subscribe&hub.challenge=123&hub.verify_token=wrong"
```

---

## 6) خطة الاختبار الكاملة (بالترتيب)

### قبل الاختبار — تأكد من 3 حاجات
1. n8n شغال والورك فلو **Published/Active** (مش لازم ngrok مع الـ Polling).
2. التوكن في n8n هو **Page Access Token** سليم وفيه `pages_read_user_content` + `pages_manage_engagement`.
3. شيت `Products` فيه صف للبوست التجريبي، وعمود `post_id` مكتوب فيه **بالصيغة الكاملة** `{PAGE_ID}_{POST_ID}` (زي `550829251444940_122192203892743121`).

> ملحوظة: أول تشغيل بيسجّل الكومنتات الموجودة من غير رد. فلو عايز البوت يرد على كومنت قديم، امسح `global.processedComments` من static data الورك فلو، أو علّق كومنت جديد.

### اختبار 1 — كومنت عادي (الرد العلني + الرسالة الخاصة)
1. اعمل بوست تجريبي على الصفحة.
2. خد `post_id` بتاعه من Graph API:
   `GET /{PAGE_ID}/posts?fields=id,message`
3. حط الصف في شيت Products:
   `post_id | name | price | sizes | colors | details | shipping`
4. علّق على البوست من حساب شخصي عادي (مش حساب الصفحة، وكومنت جديد مش رد).
5. المتوقع:
   - Execution جديدة في n8n → Executions
   - رد علني على الكومنت
   - رسالة خاصة في Messenger فيها اسم المنتج والسعر
   - صف جديد في شيت **Leads**

### اختبار 2 — إكمال الأوردر من الشات
1. رد على الرسالة الخاصة: `عايز أطلب`
2. المتوقع: البوت يسأل خطوة خطوة (الاسم، الموبايل، المحافظة، العنوان، المقاس، اللون، الكمية).
3. ابعت البيانات + أكّد الملخص.
4. المتوقع: صف جديد في شيت **Orders** بالحساب الكامل (price × qty + shipping) والحالة `جديد`.

### اختبار 3 — التحقق من الـ Webhook (اتعمل ✅)
```
200 + challenge   → سليم
403 Forbidden     → توكن غلط
404               → الورك فلو مش Active
```

### لو حاجة فشلت — اعرف السبب منين
| العرض | السبب المحتمل | الحل |
|---|---|---|
| مفيش Execution خالص | الورك فلو مش Active / الـ Schedule Trigger متوقف | اتأكد إن `Every Minute` شغال وإن الورك فلو Active (كل دقيقة لازم يبقى فيه Execution) |
| `Fetch Comments` بيرجع `(#10)` | التوكن ناقص `pages_read_user_content` | أضف الـ Use Case والصلاحية واعمل توكن جديد |
| مفيش رد على كومنت موجود قبل كده | الكومنت اتسجل في `processedComments` (backlog) | علّق كومنت جديد، أو امسح الـ static data |
| مفيش رد على رسالة ماسنجر | الرسالة اتسجلت في `processedMessages` أو أقدم من 24 ساعة | ابعت رسالة جديدة، أو امسح `processedMessages` من static data |
| `Fetch Conversations` بيرجع `(#10)` | التوكن ناقص `pages_messaging` | راجع صلاحيات التطبيق والتوكن |
| Execution `error` بسبب `The connection cannot be established` أو `ENOTFOUND graph.facebook.com` | DNS بتاع Docker بيفصل لحظيًا | نودات السحب فيها **retry 5 مرات × 5 ثواني** + `On Error: Continue` فالدورة اللي بعدها بتعوّض. لو اتكرر كتير: `docker restart n8n_con` |
| `Send Chat Reply` بيرجع خطأ `(#10) ... outside allowed window` | عدّى 24 ساعة على آخر رسالة من العميل | العميل لازم يبعت رسالة جديدة الأول |
| Execution بتبدأ وتفشل عند `Reply To Comment` | التوكن مش Page Token أو ناقص `pages_manage_engagement` | راجع الصلاحيات والتوكن |
| `Send Private DM` فشل | ناقص `pages_messaging` أو عدّى 7 أيام على الكومنت أو العميل قافل الرسائل | استخدم كومنت جديد وحساب يسمح بالرسائل |
| المنتج مش موجود / رسالة عامة | `post_id` في الشيت مش مطابق | صحّح `post_id` في شيت Products |
| `AI Agent` فشل | مشكلة في Gemini credential أو الكوتة | راجع credential في n8n |
| `Save Order` فشل | اسم الشيت/الأعمدة | افتح النود واختار الشيت من القائمة واحفظ |

### أدوات التشخيص
- **n8n → Executions:** افتح التنفيذ وشوف input/output كل نود.
- **ngrok inspector:** `http://127.0.0.1:4040` — بيوريك كل طلب فيسبوك بعته.
- **Graph API Explorer:**
  - `GET /me?fields=id,name` (للتأكد من التوكن)
  - `GET /{PAGE_ID}/subscribed_apps` (للتأكد إن الصفحة مشتركة)
- **n8n logs:**
  ```powershell
  docker logs --tail 100 n8n_con
  ```
