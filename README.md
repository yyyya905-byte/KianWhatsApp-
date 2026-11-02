# Kian WhatsApp Reminder Service

خدمة مستقلة لتذكير زبائن كيان بالديون عبر WhatsApp Cloud API الرسمي.
هذه الحزمة لا تعدّل مشروع Kian Cashier ولا تقرأ قاعدة بيانات الهاتف مباشرة.

## البنية
- Cloudflare Worker: API خاص + جدولة دورية عبر Cron.
- Cloudflare D1: يخزّن أقل قدر لازم من بيانات التذكير وسجل الإرسال.
- WhatsApp Cloud API: الإرسال الرسمي.
- خادم كيان الحالي (Express): يكون وسيطاً موثوقاً بين واجهة كيان وهذه الخدمة. لا تضع سر الخدمة في JavaScript المتصفح.

## متطلبات قبل التشغيل
1. حساب Meta Business مع WhatsApp Business Platform ورقم مفعّل.
2. اعتماد قالب رسالة Utility في WhatsApp Manager. اسم القالب الافتراضي `kian_debt_reminder` واللغة `ar`.
3. حساب Cloudflare مع Worker وD1.
4. خادم موثوق لنسخة كيان (أو خدمة backend مستقلة) يستطيع حفظ سر `KIAN_BACKEND_SECRET`.
5. تأكد أن الزبائن وافقوا على استقبال رسائل واتساب، ووفّر طريقة لإيقاف التذكير.

## إعداد Cloudflare
```bash
npm install
npx wrangler d1 create kian-whatsapp
```
انسخ `database_id` الذي يظهر، وضعه في `wrangler.toml`.
ثم:
```bash
npx wrangler d1 migrations apply kian-whatsapp --remote
npx wrangler secret put KIAN_BACKEND_SECRET
npx wrangler secret put WHATSAPP_ACCESS_TOKEN
npx wrangler secret put WHATSAPP_PHONE_NUMBER_ID
npx wrangler secret put WHATSAPP_TEMPLATE_NAME
npx wrangler secret put WHATSAPP_TEMPLATE_LANGUAGE
npx wrangler deploy
```
للقيم النصية:
- `WHATSAPP_TEMPLATE_NAME`: `kian_debt_reminder`
- `WHATSAPP_TEMPLATE_LANGUAGE`: `ar`

اضبط Cron من `wrangler.toml` (كل 15 دقيقة) أو من لوحة Cloudflare Triggers.

## قالب الرسالة المطلوب اعتماده
أنشئ Utility template في Meta بالاسم واللغة أعلاه، مع ثلاثة متغيرات body:
1. اسم الزبون
2. المبلغ المستحق
3. تاريخ الاستحقاق

مثال نص القالب:
مرحباً {{1}}، نذكّركم بأن المبلغ المستحق هو {{2}}، وموعد السداد {{3}}. إذا تم السداد يرجى تجاهل الرسالة. لإيقاف التذكيرات يرجى التواصل مع المتجر.

قد تختلف الموافقة/التسمية حسب إعدادات Meta. الرسائل التي تبدأها المنشأة خارج نافذة خدمة العملاء تحتاج قالباً معتمداً، وقد تُطبق رسوم Meta حسب البلد ونوع المحادثة.

## API
كل المسارات التالية تتطلب:
`Authorization: Bearer <KIAN_BACKEND_SECRET>`
ويجب استدعاؤها من خادم كيان فقط، وليس من المتصفح.

### مزامنة أو تحديث تذكير زبون
`POST /api/reminders/upsert`
```json
{
  "customerId": "CUS-1001",
  "phone": "+353...",
  "customerName": "اسم الزبون",
  "amount": "125.00",
  "currency": "EUR",
  "dueAt": "2026-11-10T09:00:00.000Z",
  "intervalDays": 30,
  "consent": true,
  "active": true
}
```
`intervalDays`: 7 أو 30 أو 60 أو 90.
إذا سدّد الزبون، أرسل `active:false` أو استخدم `/api/reminders/settle`.

### إيقاف تذكير بعد السداد
`POST /api/reminders/settle`
```json
{ "customerId": "CUS-1001" }
```

### إرسال يدوي فوري
`POST /api/reminders/send-now`
```json
{ "customerId": "CUS-1001" }
```

### سجل الإرسال
`GET /api/reminders/logs?customerId=CUS-1001&limit=30`

### فحص الخدمة
`GET /health` (لا يعرض أسراراً).

## ربط زر داخل كيان
زر الواجهة يستدعي endpoint موجوداً في خادم كيان Express، مثل:
`POST /api/kian-whatsapp/send-now` مع جلسة المستخدم المعتادة.
خادم Express يتحقق من صلاحية الموظف ثم يرسل الطلب إلى Worker مع `Authorization: Bearer KIAN_BACKEND_SECRET` المحفوظ في متغير بيئة الخادم.
لا تستدعِ Worker مباشرة من المتصفح ولا تضع السر في ملفات HTML/JS أو LocalStorage.

للتذكيرات التلقائية، على خادم كيان مزامنة بيانات الدين عند إنشائه/تعديله/تسديده إلى Worker. يمكن إرسال الحد الأدنى فقط: معرف داخلي، رقم، اسم، مبلغ، عملة، موعد، فترة، وموافقة.

## ملاحظات مهمة
- قاعدة البيانات المحلية في الهاتف لا تعمل كخدمة جدولة عندما يكون الهاتف أو المتصفح مغلقاً؛ لذلك تُزامن بيانات التذكير الضرورية إلى D1.
- لا ترفع مفاتيح Meta أو أسرار Cloudflare إلى GitHub.
- لا يتم اعتبار الرسالة مرسلة إلا بعد استجابة قبول ناجحة من Meta. حالة `accepted` تعني أن Meta قبلت الطلب، وليست ضماناً بأن المستلم قرأ الرسالة.
- أضف سياسة خصوصية وموافقة واضحة للزبائن قبل التفعيل.
