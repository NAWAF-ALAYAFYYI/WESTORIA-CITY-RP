# WESTORIA CITY RP — خطة تنفيذ منصة الإدارة

## نطاق المنتج (من طلب المستخدم)

منصة ويب متكاملة لإدارة مجتمع WESTORIA CITY RP (سيرفر FiveM سعودي): لوحة تحكم عربية RTL بهوية بصرية داكنة فاخرة (أسود / فضي معدني / أزرق كهربائي) تستخدم شعار WESTORIA المرفوع، مع إدارة ترقيات تفاعلية بالكامل (نموذج بصري للاعبين + محرك أهلية تلقائي + موافقة جماعية للمشرفين)، إدارة أقسام عسكرية وحكومية ووظائف مدنية قابلة للتهيئة، سجلات مواطنين، سجلات تدقيق، أدوار وصلاحيات، إعدادات نظام، وربط Discord آمن عبر OAuth2 وبوت على الواجهة الخلفية — مع قاعدة بيانات حقيقية ومصادقة وتخويل من جهة الخادم، ودليل إعداد.

## التصميم (Design)

المستخدم حدد الهوية بوضوح (شعار + أسود/فضي/أزرق كهربائي + واجهة داكنة فاخرة عربية RTL)، لذا نعيد استخدام هويته بدل اقتراح أنماط بديلة.

- **Design Movement**: لوحات قيادة الألعاب الفاخرة (Gaming Command Center) بلمسة معدنية سعودية.
- **Core Principles**: عمق داكن متعدد الطبقات، لمعة معدنية مقتصدة، أزرق كهربائي كلون تفاعل وحيد، وضوح جدولي كثيف.
- **Color Philosophy**: الأسود أساس الفخامة والسلطة، الفضي المعدني للحدود والشعارات، والأزرق الكهربائي (≈ #3B9EFF / #0A84FF نطاق) يُحفظ للأزرار والحالات النشطة والترقيات — يوحي بالتقنية والثقة.
- **Layout Paradigm**: شريط جانبي أيمن (RTL) ثابت مع ترويسة عليا رفيعة؛ بطاقات بإطارات حدّية رفيعة وتدرجات خافتة بدل شبكات مركزية.
- **Signature Elements**: إطار "أجزاء معدنية" (حواف بحدود فضية شفافة)، توهج أزرق خافت على العناصر النشطة، وخط فاصل تدرجي أزرق-فضي تحت العناوين.
- **Interaction Philosophy**: استجابة فورية هادئة؛ كل فعل إداري حساس (موافقة جماعية) يُؤكَّد بنافذة صريحة.
- **Animation**: انتقالات 150–250ms (fade/slide خفيف عبر framer-motion)، نبض توهج على الحالة النشطة، بدون حركات ثقيلة.
- **Typography System**: "Cairo" (عناوين وأرقام) + "Tajawal" أو Cairo نفسه للنصوص؛ أرقام جداول بـ tabular-nums.
- **Brand Essence**: مقر قيادة رقمي لمجتمع لعب جاد — فاخر، منضبط، حاسم.
- **Brand Voice**: صوت رسمي قيادي قصير. أمثلة: "قادة المستقبل يُرقّون من هنا" / "موافقة جماعية جاهزة للتنفيذ".
- **Wordmark & Logo**: شعار WESTORIA الدائري المرفوع في الشريط الجانبي وشاشة الدخول والفافيكون.

## البنية التقنية

قوالب web-db-user المُهيّأة: React 19 + Vite + Tailwind 4 + shadcn/ui + wouter + tRPC + Express + Drizzle (MySQL مُدارة، DATABASE_URL متوفر). المصادقة عبر Manus OAuth المدمج (cookie-based) — تُضمن مواصفات SameSite=None; Secure للـ Preview.

### قاعدة البيانات (drizzle/schema.ts)

- `users` (توسعة): openId, name, email, discordId, discordUsername, discordAvatar, characterName, role enum(player, staff, admin), createdAt, lastSignedIn.
- `departments`: key فريد, name, type enum(military, police, traffic, civil_defense, medical, government, civilian), discordRoleId, ranks (JSON مصفوفة رتب مرتبة تصاعدياً), minDaysInRank int, managerUserIds (JSON), isActive, timestamps.
- `promotionRequests`: id, userId, discordId, discordUsername, characterName, departmentId, currentRank, requestedRank, supportingInfo, status enum(pending, eligible, ineligible, approved, rejected, completed), eligibilityReason, timeInRankDays, reviewedByUserId, reviewedAt, processedAt, timestamps; فهرس فريد (userId, departmentId, status in pending/eligible/ineligible) منطقياً لمنع الطلبات المكررة (تحقق في الكود + فهرس على الأعمدة).
- `citizens`: discordId فريد, characterName, departmentId, currentRank, joinedAt, notes, createdByUserId (سجلات المواطنين).
- `applications`: id, discordId, characterName, age, timezone, experience, motivation, departmentId, status enum(pending, accepted, rejected), reviewedByUserId, timestamps.
- `auditLogs`: id, actorUserId, actorName, action, entityType, entityId, details JSON, createdAt.
- `systemSettings`: key فريد, value JSON (إعدادات Discord OAuth/bot/guild — تُخزَّن مراجع غير سرية فقط؛ التوكنات في Secrets بيئية).

### الخادم (server/)

- `discord.ts`: طبقة تكامل Discord رسمية عبر REST API (OAuth2 token exchange، تعيين رتب عبر BOT token، إشعار DM) — تُعطَّل بأمان عند غياب التوكنات وتُسجّل الحالة، ولا يُدَّعى اكتمالها قبل التهيئة والاختبار. التوكنات تُقرأ من متغيرات بيئة (Discord secrets) فقط.
- `eligibility.ts`: محرك الأهلية — يتحقق من: الترتيب في هرم الرتب (الرتبة المطلوبة = التالية مباشرة حسب إعدادات القسم)، مدة اللقب (أيام منذ آخر ترقية معتمدة/تاريخ الانضمام ≥ minDaysInRank)، عدم وجود طلب معلق/مكرر، وخضوع القسم لنفس نطاق المشرف.
- `routers.ts`: 
  - `departments` (list/create/update/delete — تعديل للمدراء فقط)
  - `promotions` (submit للاعب، my، staffList بحالات، evaluate، bulkApprove بمعاملة DB + تسجيل تدقيق + استدعاء Discord + منع المعالجة المكررة عبر حالة مشروطة في UPDATE)
  - `citizens`, `applications`, `audit`, `settings` (قراءة عامة لإعدادات العرض/كتابة admin)، `users` (إدارة الأدوار).
- `auth.ts` مساعدات: requireRole(player/staff/admin) على مستوى tRPC procedures (protectedProcedure قائم + staffProcedure/adminProcedure جديدان).

### الواجهة (client/src/)

- `index.html`: lang="ar" dir="rtl"، خط Cairo من Google Fonts، فافيكون الشعار.
- `index.css`: لوحة الألوان الداكنة (أسود oklch عميق، فضي، أزرق كهربائي) — الوضع الداكن افتراضي.
- صفحات (wouter):
  - `/login` — شاشة دخول بالشعار (Manus OAuth + زر Discord عند التهيئة).
  - `/` نظرة عامة (إحصاءات + رسم بياني recharts لأحدث الترقيات).
  - `/promotions` — نموذج طلب ترقية البصري (اللاعب) + طلباتي.
  - `/staff/promotions` — تبويبات: مؤهلون / معلقون / غير مؤهلين / مكتملة + اختيار checkboxes + مراجعة + نافذة تأكيد + موافقة جماعية.
  - `/departments/military`, `/departments/civilian`, `/departments/government` — نفس مكون الأقسام بفلتر نوع؛ تحرير هرم رتب وقواعد ومعرفات رتب Discord والمدراء.
  - `/citizens` — سجلات المواطنين (جدول + بحث).
  - `/applications` — طلبات الانضمام.
  - `/audit` — سجلات النشاط (admin/staff).
  - `/roles` — الأدوار والصلاحيات (ترقية/تنزيل مستخدمين).
  - `/settings` — إعدادات النظام + حالة تكامل Discord (تظهر "غير مُهيأ" بصدق حتى إدخال بيانات حقيقية).
- `components/` : AppLayout (شريط جانبي أيمن RTL + ترويسة)، StatusBadge، ConfirmDialog، PageHeader، StatCard.
- حماية المسارات: توجيه لـ /login عند عدم المصادقة، ورسالة عدم صلاحية لغير المشرفين.

### النشر

- `Dockerfile` (Node 22: pnpm install → build → node dist/index.js) + `deploy` config مع healthPath `/api/health`.
- `routes`: `/api/*` و `/trpc/*` → server؛ الباقي static مع spaFallback.
- `manus-routes.json` في public/.

### التسليمات الإضافية

- `SETUP.md`: دليل إعداد عربي كامل (Discord OAuth2، بوت، Guild/Role IDs، الأسرار، النشر).

## هيكل المشروع

```
westoria/
├── client/src/
│   ├── pages/ (Login, Dashboard, Promotions, StaffPromotions, Departments, Citizens, Applications, Audit, Roles, Settings, NotFound)
│   ├── components/ (AppLayout, ConfirmDialog, StatusBadge, StatCard, PageHeader)
│   ├── lib/ (trpc, utils)
│   └── App.tsx / index.css
├── server/
│   ├── _core/ (الموجود)
│   ├── db.ts, routers.ts, eligibility.ts, discord.ts, audit.ts
├── drizzle/schema.ts
├── public/manus-routes.json
├── Dockerfile, SETUP.md, plan.md, TODO.md
```

## تسلسل التنفيذ

1. توسعة schema + دفع الترحيلات إلى قاعدة البيانات المُدارة.
2. الخادم: إجراءات tRPC (أقسام، ترقيات، مواطنون، طلبات، تدقيق، إعدادات، مستخدمون) + محرك الأهلية + طبقة Discord + تدقيق.
3. الواجهة: الهوية (RTL + ألوان + خط) ثم التخطيط والصفحات بالترتيب: لوحة → ترقيات → لوحة المشرفين → أقسام → بقية الصفحات.
4. التحقق: `pnpm check` + اختبار تدفق كامل عبر tRPC (إنشاء قسم، طلب ترقية، تقييم، موافقة جماعية، تدقيق).
5. النشر: Dockerfile + routes + checkpoint + دليل SETUP.md.
