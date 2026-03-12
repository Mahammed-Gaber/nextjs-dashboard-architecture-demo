# نظرة عامة على المشروع

المشروع الحالي هو نسخة معدّلة بالكامل من قالب TailAdmin بحيث يخدم سيناريو **Landing + Dashboardات متعددة (Admin / Provider)** على نفس التطبيق. تم الاعتماد على **Next.js App Router** لكننا أبقينا منطق اللوحات منفصلًا داخل مجلد مستقل `src/dashboards` يشبه أسلوب العمل الذي كنا نتبعه في Previous enterprise SaaS project (هيكلة Feature-Based Modular).  
الهدف الأساسي: مسار واحد نظيف (`/`) للصفحة العامة، ومسار واحد (`/dashboard`) يختار لوحة التحكم المناسبة بحسب الـ role المخزّن في الكوكيز، مع إبقاء كل UI الخاص بالداشبورد خارج `app/` لسهولة التطوير وإعادة الاستخدام.

## البنية الحالية (Feature-Based Modular)

```
src/
├─ app/                     ← مسئول عن التوجيه فقط
│  ├─ page.tsx              ← Landing + منطق التوجيه حسب الكوكيز
│  ├─ dashboard/
│  │   ├─ layout.tsx        ← يحمّل DashboardLayout (الهيدر + السايدبار)
│  │   └─ [[...slug]]/      ← catch-all route يشغّل DashboardOutlet
│  ├─ (marketing)/...       ← الصفحات العامة
│  └─ login/...             ← صفحات الدخول (لاحقًا)
│
├─ dashboards/              ← كل منطق الداشبورد كما لو كان React مستقل
│  ├─ adminDashboard/       ← layouts + features (users, plans, ...إلخ)
│  ├─ providerDashboard/
│  ├─ routes/
│  │   ├─ adminRoutes.ts    ← خريطة المسارات الديناميكية للأدمن
│  │   └─ providerRoutes.ts ← خريطة المزود
│  ├─ layout/DashboardLayout.tsx
│  └─ shared/Error404Page.tsx
│
├─ components/, context/, hooks/, i18n/ … ← الموارد المشتركة
└─ messages/                            ← ملفات next-intl
```

### لماذا هذا التقسيم؟
1. **فصل الـ UI عن الـ routing**: كل ما يتعلق باللوحات يعيش تحت `src/dashboards` بنفس تنظيم React الكلاسيكي، مما يسهل النقل من مشاريع قديمة أو مشاركة المكونات بين أكثر من لوحة.
2. **مرونة إضافة لوحات جديدة**: لإضافة Dashboard ثالث يكفي إنشاء مجلد جديد + ملف routes وربطه في `DashboardOutlet`.
3. **التزام كامل بـ App Router** داخل `app/` دون التضحية بتنظيم Feature-Based.

> لمزيد من التفاصيل راجع [docs/app-flow.md](docs/app-flow.md) الذي يوضح التسلسل الكامل للطلبات وطريقة الربط بين `app/` و `dashboards/`.

## منطق التوجيه والدخول

### 1. `app/page.tsx`
يقرأ الكوكيز (`token`, `role`) ويقرر:
- يوجد `token + role` → `redirect("/dashboard")`.
- يوجد token بدون role → `redirect("/login")`.
- لا يوجد token → يعرض Landing (Marketing).

> أثناء التطوير يمكن تعيين الكوكيز يدويًا من DevTools أو إضافة Server Action صغيرة لضبط قيم تجريبية مثل `token=demo-token` و`role=admin`.

### 2. `app/dashboard/[[...slug]]/page.tsx`
- يستقبل جميع المسارات تحت `/dashboard`.
- بالنسبة لـ Next.js يُعد مسارًا ديناميكيًا واحدًا فقط، لكننا داخليًا نمرر جميع المسارات المحتملة عبر `slug` إلى طبقة الداشبورد، مما يمنع تضخم ملفات `[slug]`.
- يتحقق من الكوكيز مرة أخرى للحماية.
- بعد تحديد الدور، يستدعي `<DashboardOutlet role={role} slug={params.slug} />`.

### 3. `DashboardOutlet`
ملف `src/dashboards/DashboardOutlet.tsx` هو المسؤول عن:
- قراءة المسار الحالي (أو `slug` القادم من App Router).
- اختيار الخريطة المناسبة (`adminRoutes` أو `providerRoutes`).
- تحميل الشاشة المطلوبة Lazy عبر `next/dynamic`.

### لماذا لم نستخدم Parallel Routes أو Route Groups؟
- Parallel Routes تحتاج أن تكون كل لوحة داخل `(dashboard)/@admin`، وهذا يكسِر شرطنا الأساسي بأن يظل landing على `/` وأن تظهر الروابط نظيفة.
- استخدام Route Groups فقط (`(dashboard)`) كان سيمنعنا من امتلاك URL حقيقي `/dashboard` يمكن إعادة التوجيه له؛ وبالتالي كنا سنحتاج Middleware معقد أو إعادة كتابة لكل الروابط.
- الحل الحالي (مسار واحد catch-all + DashboardOutlet) يسمح بإدارة لوحات متعددة على مسار واحد، وبنفس الوقت يبقي كل الملفات خارج `app/` كما نحب.

## مزايا هذا النهج
- **Modular**: كل Dashboard عبارة عن React Module مستقل يمكن نقله أو تطويره دون لمس `app/`.
- **Clean URLs**: زائر الداشبورد يرى `/dashboard/profile` بدل `/dashboard/admin/profile` أو `/admin/profile`.
- **سهل التوسع**: إضافة Dashboard جديد = ملف routes جديد + ربط في `roleRoutes`.
- **متوافق مع نهج Previous enterprise SaaS project**: نفس التقسيم (layout + features + shared) مما يسهّل دمج الكود القديم في حالة تحويل أي مشروع قديم.

## العيوب أو الأمور التي يجب الانتباه لها
1. **الاعتماد على الكوكيز**: بدون backend حقيقي يجب ضبط الكوكيز يدويًا؛ وإلا ستظهر تحذيرات redirect أو أخطاء 401.
2. **عدم وجود Middleware**: يعني أن حماية المسارات تعتمد على فحص الكوكيز داخل الصفحات نفسها. بمجرد توفر backend يُفضل إعادة تفعيل middleware أو RSC guards.
3. **catch-all slug**: في Next.js 16 params أصبحت Promise، لذا يجب جعل الصفحة `async` أو عمل `await params`. نحتاج الحرص على هذه النقطة عند تعديل الكود.
4. **عدم استخدام Parallel Routes** يعني أننا نفقد بعض مزايا Next (مثل عرض أكثر من لوحة في نفس الوقت)، لكن هذا قرار واعٍ للحفاظ على بساطة المسارات.

## كيفية العمل في بيئة التطوير

1. **تشغيل السيرفر**:
   ```bash
   npm install
   npm run dev
   ```
2. **ضبط كوكيز تجريبية** (من DevTools → Application → Cookies):
   - `token = demo-token`
   - `role = admin` أو `provider`
3. **تبديل اللوحة**: غيّر قيمة `role` ثم حدّث `/dashboard`.
4. **استرجاع landing**: احذف الكوكيز من DevTools أو نفّذ `document.cookie="token=; Max-Age=0"`.

## أسئلة شائعة

| السؤال | الإجابة |
| --- | --- |
| لماذا الـ redirect أحيانًا يدخل في Loop؟ | يحدث فقط إذا تم وضع قيم ثابتة (مثل `"undefined"`) بدل قراءة الكوكيز. يجب إبقاء `token`/`role` كما هي أو تهيئتها عبر dev-only logic. |
| هل يمكن إضافة Dashboard ثالث؟ | نعم. أنشئ مجلدًا جديدًا تحت `src/dashboards/<name>`، أضف routes في `routes/<name>Routes.ts`، ثم عدّل `DashboardOutlet` لقراءة role جديدة. |
| هل يمكن نقل اللوحات إلى `app/(dashboard)`؟ | ممكن لكن سنفقد فكرة انفصال الـ UI عن App Router، وسنضطر للتضحية بالمسار `/dashboard` أو استخدام middleware/rewrites معقدة. |
| ماذا عن i18n؟ | ما زالت موجودة تحت `src/messages` و`src/i18n`، ويتم تحميلها في `app/layout.tsx` عبر `NextIntlClientProvider`. |
| ما المخاطر الحالية؟ | الاعتماد على كوكيز وهمية في dev، وعدم وجود حماية على مستوى API (يُفترض إضافتها عندما يُربط النظام بالباك). |

## الخطوات التالية المقترحة
1. ربط الـ login الفعلي بحيث يكتب الكوكيز بدل التعديل اليدوي.
2. إضافة Server Action لتغيير الدور من داخل الواجهة (لأغراض QA).
3. نقل أي صفحات مشتركة (مثل Forms/Alerts) إلى `src/dashboards/shared-ui` ثم استيرادها من جميع اللوحات لتقليل التكرار.
4. عند اقتراب الإطلاق، يمكن تفعيل Middleware بسيط لحماية `/dashboard` وإعادة كتابة الروابط النظيفة دون الحاجة إلى تعديل `DashboardOutlet`.

---

هذا التوثيق هو المرجع الرسمي للبنية الحالية. في حال إجراء أي تغييرات جذرية (إضافة Dashboard جديد، تعديل منطق الكوكيز، أو إعادة تفعيل Middleware) يُفضّل تحديث هذا الملف بنفس الأسلوب لضمان أن كل من يعمل على المشروع يفهم السبب قبل التنفيذ.

</div>
