# PedImporter — HANDOFF #2 (کارنامه‌ی کامل از ابتدا تا الان)

نسخه: build 2026-08-18 · net6.0-windows / WPF · self-contained win-x64 single-file
تست‌ها: **76 / 76 PASS روی لینوکس** با .NET SDK 8.0.418

> این فایل ادامه‌ی `HANDOFF.md` است. اگر AI جدیدی این پروژه را می‌گیرد:
> اول `HANDOFF.md` (معماری و ۸ قانون کسب‌وکار) و بعد همین فایل (تغییرات جدید) را بخواند.

---

## 1) ساختار پروژه

```
desktop/
  CodeWalker.Core/            # کتابخانه‌ی خواندن/نوشتن RSC7 (ymt باینری) — دست‌نخورده
  PedImporter/                # اپ WPF
    App.xaml(.cs)
    MainWindow.xaml(.cs)      # UI: تب ۱ ایمپورت، تب ۲ جابجایی/تبدیل/حذف
    Core/
      FileNameParser.cs       # پارس نام فایل‌ها (لباس + پراپ _p)
      ImportPlanner.cs        # برنامه‌ریزی ایمپورت (ایندکس‌دهی، گپ، قانون 000)
      YmtComponentEditor.cs   # ادیت XML سند CPedVariationInfo + ذخیره به ymt
      YmtSchemaNormalizer.cs  # ⭐ نرمال‌سازی کانونیک طبق اسکیمای YMTEditor 1.4.1
      MoveManager.cs          # ⭐ جابجایی/تبدیل/حذف لباس‌های موجود + wizard تصادم
  PedImporter.Tests/          # هارنس کنسولی (روی لینوکس اجرا می‌شود، بدون WPF)
    Program.cs Fixtures.cs MoveTests.cs SchemaTests.cs
```

نکته‌ی مهم: `PedImporter.Tests.csproj` فایل‌های `PedImporter/Core/*.cs` را **مستقیم کامپایل** می‌کند
(`EnableDefaultCompileItems=false`)، پس منطق روی لینوکس هم قابل تست است و هیچ کد Coreای نباید به WPF وابسته شود.

---

## 2) کارهایی که در این دوره انجام شد (به ترتیب)

### الف) هارنس تست روی لینوکس
- پروژه‌ی `PedImporter.Tests` ساخته شد؛ `Fixtures.cs` سند مصنوعی `CPedVariationInfo` و پوشه‌ی stream جعلی می‌سازد.
- پوشش: نام‌گذاری، ایندکس‌دهی، قانون `000`، پر کردن گپ، rename تکسچرها، round-trip باینری RSC7.
- **باگ رفع‌شده:** `NullReferenceException` در `YmtComponentEditor.AddPropDrawable` وقتی نود `numAvailProps` در `propInfo` وجود نداشت.

### ب) تب دوم: جابجایی / تبدیل لباس‌های موجود (`MoveManager.cs`)
- تبدیل بین کامپوننت‌ها (مثلاً `feet` → `jbib`) و جابجایی ایندکس.
- **wizard تصادم** با ۴ سیاست: `Shift` (مزاحم برود اولین جای آزاد)، `Swap` (تعویض)، `ManualMove` (کاربر مقصد مزاحم را می‌گوید)، `SkipMove` (رد شدن با هشدار).
- rename **دو-فازی** (اول همه به `.movetmp`، بعد به نام نهایی) → سواپ و چرخه‌ها بدون برخورد فایلی امن‌اند.
- تکسچرها (`ytd`) همراه drawable منتقل و حروفشان از موقعیت جدید re-letter می‌شوند.
- قانون `000` کامپوننت‌های اختیاری (BERD/TEEF/ACCS/TASK/DECL) حفظ می‌شود؛ مقصد `accs_000` خطا می‌دهد.
- بکاپ `ymt` قبل از هر عملیات.
- `YmtComponentEditor.RebuildFromFiles` پارامتر `renumber` گرفت. `MoveManager` با `renumber:false` صدا می‌زند
  تا ایندکس‌های روی دیسک عیناً حفظ شوند و گپ‌ها با ورودی خالی پر شوند (renumber ناخواسته = باگ قبلی).

### ج) پراپ‌ها (`_p`) و حذف
- `FileNameParser`: فایل‌های `_p` به‌عنوان پراپ شناخته می‌شوند و کلیدشان پیشوند `p_` می‌گیرد تا با لباس تداخل نکند.
  رجکس‌ها شل‌تر شدند تا پک‌های با پسوند `_u` / `_uni` هم پارس شوند.
- `MoveManager`: فضای اشغال (occupancy) پراپ‌ها **جدا** از لباس است و پراپ فقط به anchorهای مجاز (head/eyes/ears/…) می‌رود.
- `MoveManager.DeleteDrawables`: حذف کامل — `ydd`/`ytd` به زیرپوشه‌ی زمان‌دار `_deleted_<timestamp>` منتقل می‌شوند
  (سطل بازیافت، برگشت‌پذیر) و ymt بدون renumber بازسازی می‌شود تا رفرنس مرده نماند.

### د) UI
- `MainWindow.xaml` بازنویسی شد: `FlowDirection=RightToLeft`، `WindowState=Maximized`،
  گریدهای responsive با `GridSplitter`، سرچ/فیلتر، سوییچ «نمایش پراپ‌ها»، دکمه‌ی «حذف انتخاب‌شده‌ها»، پنل لاگ تاشو.
- `MoveRow` بین لباس و پراپ تفکیک می‌کند (لیست کامپوننت مقصد متفاوت است).

### ه) ⭐ تطبیق کامل ymt با YMTEditor 1.4.1 (`YmtSchemaNormalizer.cs`)
ابزار مرجع دیکامپایل و `XMLHandler.cs` آن خوانده شد. قواعد استخراج‌شده و پیاده‌سازی‌شده:
1. **ترتیب ریشه:** flagها → `availComp` → `aComponentData3` → `aSelectionSets` → `compInfos` → `propInfo` → `dlcName`.
2. `availComp` رشته‌ی ۱۲ عددی؛ ایندکس‌های **متوالی ۰..n-1** بدون سوراخ؛ نبودن کامپوننت = `255`؛ ایندکس خراب پاک می‌شود.
3. `numAvailTex` هر کامپوننت = مجموع تکسچرِ همه‌ی drawableهایش **mod 256**.
4. هر تکسچر: `distribution = 255` (هم لباس هم پراپ).
5. `compInfos`: **دقیقاً یک Item برای هر drawable**، گروه‌شده به‌ترتیب کامپوننت، با `pedXml_compIdx`/`pedXml_drawblIdx` درست.
   متادیتای قبلی (audioID/audioID2/expressionMods/vfxComps/flags/inclusions/exclusions) حفظ و اگر نبود پیش‌فرضِ YMTEditor
   (`none`, `0 0 0 0 0`, `PV_COMP_<NAME>`, `0`) گذاشته می‌شود. نام‌های هش‌شده‌ی CodeWalker (`hash_2FD08CEF` …) alias می‌شوند.
6. `aSelectionSets` همیشه هست (آرایه‌ی خالی `CPedSelectionSet`).
7. `numAvailProps` = تعداد کل پراپ‌ها **mod 256**؛ `aPropMetaData` گروه‌شده به anchor با `propId` متوالی از صفر.
8. **باگ اصلیِ خراب‌کننده‌ی پراپ‌ها در بازی:** `aAnchors/Item/props` باید **لیست تعداد تکسچرِ هر پراپ** باشد
   (مثلاً `3 1`)، نه تعداد پراپ‌ها. رفع شد.
9. نرمال‌سازی **idempotent** است (دوبار اجرا = همان خروجی) و در `YmtComponentEditor.SaveXmlAsYmt` هوک شده،
   یعنی **هر** ذخیره‌ای از آن رد می‌شود.

---

## 3) وضعیت تست‌ها (این بیلد)

```
FileNameParser / ImportPlanner / YmtComponentEditor ... PASS
MoveManager (تبدیل، تصادم، سواپ، 000، rename دیسک، بکاپ) ... PASS
RebuildFromFiles (renumber true/false، گپ، round-trip باینری) ... PASS
YmtSchemaNormalizer (ترتیب ریشه، mod 256، aAnchors، idempotency) ... PASS
---- 76 passed, 0 failed ----
```

### اجرای تست‌ها روی لینوکس
```bash
dotnet run --project desktop/PedImporter.Tests/PedImporter.Tests.csproj
```
### بیلد ویندوز از لینوکس
```bash
dotnet publish desktop/PedImporter/PedImporter.csproj -c Release -r win-x64 \
  --self-contained true -p:PublishSingleFile=true -p:EnableWindowsTargeting=true -o out
```

---

## 4) کارهای باقی‌مانده / نکات برای نفر بعد

- **ارورِ اولین باز شدن اپ:** کاربر گزارش کرده بار اول یک خطا می‌دهد. متن دقیق خطا لازم است؛
  پیشنهاد: لاگ‌گیری `AppDomain.CurrentDomain.UnhandledException` + `DispatcherUnhandledException` در `App.xaml.cs`
  و نوشتن استک‌تریس در `%LOCALAPPDATA%\PedImporter\error.log`. (روی لینوکس قابل بازتولید نیست چون WPF اجرا نمی‌شود.)
- ویزارد تصادم فقط برای لباس/پراپ تک‌تایی تست شده؛ عملیات دسته‌ای بزرگ (۵۰+ آیتم) بار تست ندارد.
- `aSelectionSets` همیشه خالی نوشته می‌شود؛ اگر روزی selection set واقعی لازم شد باید حفظ‌شان کنیم.
- `net6.0-windows` از پشتیبانی خارج شده (warning NETSDK1138)؛ ارتقا به `net8.0-windows` بی‌دردسر است.
- روش دیباگِ جواب‌داده همان قبلی است: ymt را با CodeWalker به XML باز کن، قبل/بعد را diff کن.
