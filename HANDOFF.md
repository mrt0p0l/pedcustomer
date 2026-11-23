# PedImporter — یادداشت انتقال پروژه

## این اپ چیه
یه ابزار WPF (C#, .NET 6, ویندوز) برای مدیریت لباس/پراپ‌های پدهای FiveM/GTA V. کاربر یه پوشه‌ی مبدأ (فایل‌های `.ydd`/`.ytd`) و یه پوشه‌ی مقصد (پوشه‌ی `stream` یه ped، همراه با فایل `.ymt`ش) می‌ده، و اپ:
1. فایل‌ها رو با pattern استاندارد FiveM می‌خونه (`ig_<PedName>^<component>_<index3>_<u/r>.ydd` و دیف‌تکسچرهای `..._diff_<index3>_<letter>_<uni/whi/...>.ytd`).
2. فایل‌های انتخاب‌شده رو با ایندکس درست کپی/rename می‌کنه و **مستقیم باینری `.ymt` واقعی (فرمت RSC7 فشرده‌ی Rockstar) رو ادیت و ذخیره می‌کنه** — نه یه فایل XML واسطه.
3. یه دکمه‌ی «بازسازی کامل» هم داره که کل بخش لباس/پراپ ymt رو از رو فایل‌های واقعیِ روی دیسک از صفر می‌سازه (برای رفع ناهم‌خوانی‌های قدیمی).

## معماری
- `PedImporter/Core/FileNameParser.cs` — پارس کردن اسم فایل‌ها (لباس + پراپ)، ساخت اسم فایل خروجی.
- `PedImporter/Core/YmtComponentEditor.cs` — همه‌ی منطق خوندن/نوشتن ساختار `CPedVariationInfo` (لباس‌ها) و `propInfo` (پراپ‌ها) به‌صورت `XmlDocument`.
- `PedImporter/Core/ImportPlanner.cs` — منطق «Import عادی»: محاسبه‌ی ایندکس بعدی، اعتبارسنجی، اجرا.
- `PedImporter/MainWindow.xaml(.cs)` — کل UI.
- `CodeWalker.Core/` — کتابخونه‌ی متن‌باز dexyfex/CodeWalker (عیناً، بدون تغییر) که خوندن/نوشتنِ باینری RSC7 و تبدیل Meta↔XML رو انجام می‌ده. این پروژه دقیقاً به همین کتابخونه وابسته‌ست، نمی‌شه حذفش کرد.

### چطور ymt واقعاً ادیت می‌شه (نکته‌ی کلیدی فنی)
1. `YmtIo.LoadYmtAsXml` → فایل باینری رو با `YmtFile.Load` + `MetaXml.GetXml` به یه `System.Xml.XmlDocument` معمولی تبدیل می‌کنه.
2. با `XmlDocument` API معمولی ادیتش می‌کنیم.
3. `YmtIo.SaveXmlAsYmt` → با `XmlMeta.GetRSCData` دوباره به بایت باینری تبدیل و ذخیره می‌کنه.

## ساختار XML واقعیِ ymt (از سورس CodeWalker دراومده، نه حدسی)
- ریشه: `<CPedVariationInfo>` با `availComp` (۱۲ عدد، هر کدوم یا `255` یعنی نیست، یا ایندکسی تو `aComponentData3`).
- ترتیب ۱۲ کامپوننت: `HEAD, BERD, HAIR, UPPR, LOWR, HAND, FEET, TEEF, ACCS, TASK, DECL, JBIB` (تو `YmtComponentEditor.ComponentTypeOrder`).
- هر کامپوننت: `aComponentData3/Item` → `aDrawblData3/Item` (هر Item یه drawable با `propMask`: نیمه‌ی بالاش PropType، ۰=u، ۱=r) → `aTexData/Item` با `texId` (۰=uni,۱=whi,۲=bla,...) که با حرف تکسچر (a,b,c...) میاد.
- `compInfos/Item` آرایه‌ی موازیِ متادیتای هر drawable (audioId, vfxComps, ...) — با `pedXml_compIdx`/`pedXml_drawblIdx` خودش رو معرفی می‌کنه.
- پراپ‌ها: `propInfo/aPropMetaData/Item` (anchorId: ۰=head,۱=eyes,۲=ears,۶=lwrist,۷=rwrist) و `propInfo/aAnchors/Item`.
- اسم فایل پراپ (بدون suffix u/r و بدون uni/whi): `ig_<Ped>_p^p_<slot>_<idx3>.ydd` و `..._diff_<idx3>_<letter>.ytd`.

## قوانین کسب‌وکار مهم (کاربر صریح خواسته، عوض نکن بدون تأیید)
1. **ایندکس ۰۰۰ فقط برای کامپوننت‌های اختیاری خالی می‌مونه**: `berd, teef, accs, task, decl` (`YmtComponentEditor.ReservedZeroComponents`). این‌ها می‌تونن اصلاً پوشیده نشن.
2. **بقیه (head, hair, uppr, lowr, hand, feet, jbib) هیچ محدودیتی رو ۰۰۰ ندارن.** این محدودیت رو یه بار برای همه اعمال کرده بودم، باعث شد پد تو بازی غیب بشه — یه باگ واقعی و حل‌شده.
3. **ایندکس واقعی همیشه از خودِ سند ymt گرفته می‌شه**، نه از پیش‌بینیِ preview — چون `AddComponentDrawable` ایندکس رو از رو تعداد واقعیِ drawable های موجود می‌سازه.
4. **کامپوننتی که رو دیسک فایل نداره، نباید از ymt حذف بشه.** باگ جدی بود: قبلاً Rebuild هر کامپوننت بدون فایل مطابق رو (مثل ACCS/DECL) کامل پاک می‌کرد. الان `RebuildFromFiles` قبل از بازسازی وضعیت اصلیِ template رو ذخیره می‌کنه و کامپوننت بدون فایل رو عیناً کپی می‌کنه.
5. **اسم فایل خروجی همیشه با `ig_` شروع می‌شه، فارغ از اینکه ورودیِ «اسم Ped مقصد» خودش `ig_` داشته باشه یا نه** — `NormalizePedName` تو `FileNameParser.cs` این رو هندل می‌کنه (جلوی `ig_ig_De3` رو می‌گیره).
6. **اسکن فایل نباید به پیشوند `ig_` گیر بده** — رجکس‌ها هر پیشوندی رو قبول می‌کنن.
7. **دکمه‌ی Rebuild گپ‌های واقعی رو با renumber کردنِ خودِ فایل‌ها می‌بنده** (دو-فازی، از اسم موقت رد می‌شه که تصادم نشه).
8. **اگه یه آیتم تو یه Import مشکل داشت، فقط همون رد می‌شه، بقیه اجرا می‌شن.**

## چیزی که هنوز ساخته نشده (خواسته‌ی صریح کاربر)
یه بخش/تب کاملاً جدا برای جابجایی/تبدیل لباس‌های از قبل موجود تو پوشه‌ی مقصد (نه اضافه کردن جدید از یه مبدأ):
- مثلاً `jbib_1` رو ببره ایندکس دیگه، یا تبدیلش کنه به `lowr_1`، با تکستچرهاش کامل.
- اگه جای مقصد پر بود، هشدار بده و بپرسه آیتم فعلیِ اونجا کجا بره (wizard تصادم).

## محیط توسعه
- کاربر Visual Studio نداره؛ با CLI می‌سازه:
  `dotnet publish PedImporter/PedImporter.csproj -c Release -r win-x64 --self-contained true -p:PublishSingleFile=true`
- `CodeWalker.Core` از NuGet به SharpDX/SharpDX.Mathematics نیاز داره (فقط بار اول اینترنت لازمه).
- WPF فقط رو ویندوز کامپایل می‌شه — هیچ راهی برای تست رو لینوکس/Mac نیست.

## روش دیباگ که جواب داده
وقتی مشکلی پیش میاد، بهترین راه: کاربر فایل XML خروجیِ CodeWalker/YMTEditor (Export به XML) رو از ymt اصلی *و* ymt بعد از اجرای اپ بفرسته، و با مقایسه‌ی ساختاری (تعداد drawable هر کامپوننت، texcount هر drawable) دقیقاً می‌شه فهمید کجا داده گم شده.
