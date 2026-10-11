# ملف البحث التأسيسي: قراءة ملفات بيانات أوراكل التالفة واستخراج البيانات منها

إعداد: الباحث — بتاريخ 2026-10-11
الغرض: تجميع كل ما يلزم (مراجع، أدوات قائمة، أبحاث، بنية الملفات، فجوات السوق، خارطة طريق) قبل بدء التطوير مع وكيل التدوير.
قاعدة العمل: كل مصدر في هذا الملف تم فتحه والتحقق من وجوده (رمز HTTP 200 أو DOI يحلّ) في 2026-10-11. ما لم أتمكن من التحقق منه مذكور صراحةً في القسم 11.

---

## 0. الملخص التنفيذي (ما يهمّ القرار)

1. **المجال ليس فارغاً**: توجد 4 أدوات ناضجة تعمل منذ 15–25 سنة (DUL الداخلية لأوراكل، DBRECOVER/PRM-DUL التجارية، ODU، AUL)، لكنها كلها **مغلقة المصدر**، ومعظمها بواجهات قديمة ونماذج ترخيص لكل قاعدة بيانات (DBRECOVER: 1,350 دولار لكل DB_NAME).
2. **لا توجد أداة مفتوحة المصدر مكتملة** لاستخراج بيانات أوراكل من ملفات تالفة. أقرب شيء: `oradul` (بايثون، ~1,100 سطر، يتطلب من المستخدم كتابة تعريف الأعمدة يدوياً، بلا دعم قاموس) و`RICO2` (محرر كتل بديل لـ BBED، لا يستخرج جداول كاملة).
3. **الأساس العلمي موجود ومنشور**: سلسلة DBCarver (Wagner & Rasin, جامعة DePaul، 2015–2019) تصف بالضبط منهجية "نحت" صفحات قواعد البيانات بدون القاموس أو السجلات، وأوراكل أحد الأنظمة المدعومة فيها. وسلسلة Litchfield (2007–2008) تشرح تشريح ملفات أوراكل (redo، undo، كتل البيانات، SCN) لأغراض الطب الشرعي.
4. **بنية الكتلة والصف وأنواع البيانات موثّقة رسمياً** من أوراكل بما يكفي للبدء (Concepts Guide ch.14، SQL Reference Data Types، OCI/OCCI internal datatypes) ويكمّلها عمل Julian Dyke وJonathan Lewis وKamil Stawiarski (خوارزمية checksum، ترميز NUMBER).
5. **الفجوة التي يمكن أن تكون "ما لم يصنعه أحد"**: أداة مؤسسية مفتوحة/حديثة تجمع: (أ) استخراج بالقاموس وبدونه، (ب) استدلال آلي لأنواع الأعمدة وأسماء الجداول في وضع بدون قاموس (كل الأدوات الحالية تجعل هذا يدوياً)، (ج) دعم 12c–23ai/26ai متعدد المستأجرين (PDB/CDB)، (د) دعم ASM وLOB والضغط والصفوف المتسلسلة، (هـ) تقرير تدقيق وشرعي (forensic-grade report) بصيغة قياسية مثل DB3F، (و) واجهة حديثة وAPI لأتمتة المؤسسات، (ز) معالجة سيناريو برامج الفدية (تشفير جزئي للملفات) الذي أصبح أكثر سيناريو طلباً في 2024–2026.

---

## 1. تعريف المشكلة والسيناريوهات

ملف بيانات أوراكل (.dbf) هو تسلسل كتل ثابتة الحجم (2K–32K، الافتراضي 8K). "التلف" الذي نستهدفه يشمل:

| السيناريو | ما يتلف عادةً | هل يمكن الاستخراج؟ |
|---|---|---|
| تلف كتل متفرقة (ORA-01578، checksum/tailchk) | كتل بيانات منفردة | نعم، نتجاوز الكتل التالفة ونستخرج الباقي |
| فقدان/تلف ملف التحكم (controlfile) أو redo | لا تُفتح القاعدة، الملفات سليمة غالباً | نعم، "قراءة قذرة" (dirty read) من الكتل مباشرة |
| ملف "fuzzy" (ORA-01194، نقص recovery) | عدم اتساق SCN بين الكتل | نعم، مع فقد المعاملات غير المكتملة |
| تلف SYSTEM.DBF (القاموس) | لا نعرف أسماء الجداول والأعمدة | نعم بوضع بدون قاموس (Non-Dictionary) مع استدلال |
| حذف جدول/truncate/drop | إشارات القاموس فقط؛ الكتل لا تُمسح فوراً | نعم إن لم تُكتب فوقها |
| تلف مجموعة أقراص ASM | لا نرى ملفات .dbf أصلاً | نعم بقراءة بيانات ASM الوصفية (AMDU-like) |
| برامج الفدية (تشفير جزئي) | رأس الملف + أجزاء متفرقة | جزئياً: الكتل غير المشفرة تُستخرج بمسح كامل |
| حذف ملف من نظام الملفات | لا ملف | نحت الكتل من القرص الخام بتوقيع كتلة أوراكل |

المرجعية لهذه السيناريوهات: دليل DBRECOVER FAQ (2025-12) وسيناريوهات ORA-01194/ORA-00600/ASM [T2]، وصفحة أوراكل لخطأ ORA-01578 [O9]، وأداة OraScan من xifenfei لنحت الكتل من القرص [T7].

---

## 2. خريطة الأدوات القائمة (المنافسون والمراجع التطبيقية)

### 2.1 أدوات تجارية/مغلقة متخصصة في "Unload" بدون instance

| الأداة | المطوّر | اللغة/المنصة | الترخيص والسعر | أبرز القدرات الموثّقة | نقاط الضعف المعلنة |
|---|---|---|---|---|---|
| **DUL** (Data UnLoader) [T1] | Oracle Support (هولندا، Bernard van Duijnen) | C، سطر أوامر، ملفات init.dul/control.dul | **داخلية لأوراكل، غير متاحة للعموم ولا مدعومة** | مسح الملف، التعرف على كتل رؤوس المقاطع (segment header)، قراءة الـ extents، تجاوز الكتل التالفة، تصدير بصيغة SQL*Loader أو exp. لا تقرأ المعاملات غير الملتزمة | لا يحصل عليها إلا عبر أوراكل؛ ضبط يدوي للمنصة عند النقل عبر الأنظمة |
| **DBRECOVER for Oracle** (سابقاً PRM-DUL) [T2] | ParnassusData / DBRECOVER Team | Java، واجهة رسومية، Windows/Linux | تجاري: 1,350$ لكل DB_NAME؛ نسخة مجتمعية مجانية بحد 10,000 صف/جدول | وضع قاموس + وضع بدون قاموس، 8i–23ai/26, PDB/CDB (نسخة 2512)، ASM (مع PRMSCAN)، تحويل endianness تلقائي (AIX→Windows)، Cluster/LOB/Partition، نسخة 2512 تضمّ JRE | وضع بدون قاموس يدوي بالكامل (تحديد الجدول بمعاينة العينات)؛ تعليق أثناء المسح على التلف الشديد؛ ترخيص مرتبط بـ DB_NAME |
| **ODU** (Oracle Database Unloader) [T3] | "老熊" (laoxiong) وdbsnake (الصين) | C، سطر أوامر | كان مجانياً ثم أصبح مدفوعاً | مماثل لـ DUL؛ منذ الإصدار 3.0.2 يستخرج **الصفوف المحذوفة** (unload_deleted yes) | الموقع الرسمي laoxiong.net/oracleodu.com لم يستجب أثناء التحقق (انظر §11) |
| **AUL / MyDUL** [T4] | anysql.net (الصين) | C | نسخة مجانية محدودة، ونسخة مدفوعة | يستخرج من ملفات البيانات مباشرة، يعيد بناء القاموس بـ `UNLOAD TABLE DICT$`، دليل مستخدم PDF منشور | توثيق صيني فقط؛ واجهة أوامر قديمة |
| **Oracle Recovery Tools / OBET / OraScan / OraFHR** [T7] | xifenfei.com (惜分飞) | Windows GUI | أدوات مجانية صغيرة | OBET: محرر كتل بديل لـ BBED (إصلاح checksum/tailchk، نسخ كتل، تعديل SCN في رأس الملف)؛ OraScan: نحت كتل أوراكل من القرص الخام؛ OraFHR: إعادة بناء رأس الملف بعد تشفير برامج الفدية | أدوات نقطية وليست منصة استخراج متكاملة |
| Stellar Phoenix Oracle Recovery، Kernel for Oracle، Recovery Toolbox | شركات أدوات إصلاح عامة | Windows GUI | تجاري | تدّعي إصلاح .dbf واستعادة الجداول/الإجراءات | توثيق تقني ضعيف؛ إصدارات أوراكل قديمة (9i–12c) في صفحاتها الرسمية؛ لا أعتبرها مرجعاً تقنياً |

### 2.2 أدوات مفتوحة المصدر ذات صلة مباشرة (قابلة للدراسة والتعلم منها)

| المشروع | اللغة | النجوم | آخر تحديث | الترخيص | ما يقدّمه لنا |
|---|---|---|---|---|---|
| **jieguolove/oradul** [G1] | Python 3 (ملف واحد ~1,133 سطر، بلا تبعيات) | 0 | 2026-08-31 | MIT | أقرب نقطة انطلاق مفتوحة: `info/scan/dump/find/extract`؛ فكّ ترميز NUMBER/DATE/TIMESTAMP/ROWID؛ تحليل kcbh وkdbh؛ تصدير CSV/INSERT. **عيبه**: يتطلب من المستخدم تمرير تعريف الأعمدة (`-c "id:NUMBER,name:VARCHAR2(100)"`)، أي لا يقرأ القاموس ولا يستدلّ على الأنواع |
| **ora600pl/rico2** [G2] | Python (~2,018 سطر + 426 اختبار) | 16 | 2026-07-15 | غير محدد | بديل مفتوح لـ BBED: يفكّ kcbh/ktbbh/kdbh/kdbt/kdxle/kdxbr، دليل الصفوف kdbr، صفوف heap وcluster ومدخلات B-tree، حساب/تطبيق/تحقق checksum، البحث بـ DATA_OBJECT_ID. **مرجع ممتاز لبنية الكتل في بايثون** |
| **ora600pl/odbv** [G3] | C++ | 13 | 2019 | غير محدد | Oracle Database Block Visualizer لتتبع "البيانات الشبحية" (ghost data) |
| **bersler/OpenLogReplicator** [G4] | C++ | 383 | 2026-07-19 | AGPL-3.0 | أنضج محلّل مفتوح لسجلات redo الثنائية لأوراكل (OpCodes 05.01، 0B.02…، 0B.0C، 1A.02…) — مرجع لفهم تنسيق redo ولتوسيع الأداة لاحقاً لاستعادة التغييرات من سجلات الأرشيف. **تنبيه ترخيص AGPL** (انظر §9) |
| **averemee-si/oracdc** [G5] | Java | 130 | 2026-10-09 | NOASSERTION | CDC لأوراكل؛ يضم فئات لفك ترميز بعض البنى الداخلية (`ojdbc-types`) |
| **zhihuiFan/OracleRedoParser** [G6] | C | 16 | 2017 (ميت) | — | محلل archive log قديم؛ قيمة تاريخية فقط |
| **wublabdubdub/PDU-PostgreSQLDataUnloader** [G7] | C | 119 | 2026-08-22 | Apache-2.0 | ليس أوراكل، لكنه **النموذج المعماري الأقرب لما نريد** لقاعدة أخرى (PostgreSQL 14–18): استخراج كامل دون اتصال، استرجاع عبر WAL، استرجاع جداول محذوفة. مفيد لتصميم الواجهة وهيكل المشروع |
| **xuji755/bic-dmdul** [G8] | Python | 18 | 2026-07-04 | GPL-3.0 | "DUL" لقاعدة DM8 الصينية (2026): يستعيد القاموس، يحدد كائنات التخزين، يصدّر الصفوف وLOB وSQL قابل للتحميل — نموذج حديث لنفس الفكرة |

### 2.3 أدوات أوراكل الرسمية (مرجعية ومكمّلة، لا تعمل على قاعدة لا تُفتح)

- **DBVERIFY (dbv)** [O7]: فحص سلامة الكتل دون instance — نستخدمه كمرجع للتحقق من صحة الكتل في اختباراتنا.
- **RMAN Block Media Recovery** [O8] و**DBMS_REPAIR / SKIP_CORRUPT_BLOCKS / event 10231** [O10]: المسار الرسمي عندما تكون القاعدة قابلة للفتح — نذكره في التوثيق كـ"جرّب هذا أولاً".
- **AMDU / kfed** [O11]: أدوات أوراكل لاستخراج ملفات من مجموعة أقراص ASM غير mounted (موثّق في ASM Administrator's Guide 19c وCERN TWiki) — مرجعنا لدعم ASM.
- **LogMiner / DBMS_LOGMNR_D** [O12]: يحتاج قاموساً؛ مرجع لفهم ما ننافسه في قراءة redo.
- **BBED**: محرر كتل داخلي لم يعد يُشحن منذ 11g؛ RICO2 وOBET بديلان له.

---

## 3. الأسس التقنية الموثّقة (ما سنحتاجه فعلياً لبناء المحلّل)

### 3.1 بنية ملف البيانات والكتلة
- **رأس الملف (كتلة 0 + كتلة 1 = kcvfh)**: يحوي الإصدار، حجم الكتلة، المنصة (endianness)، DB name، checkpoint SCN، resetlogs SCN، علامة fuzzy. مرجع: V$DATAFILE_HEADER [O3]، شرح x$kcvfh [W5]. أداة OBET تعدّل هذه الحقول تحديداً [T7].
- **رأس الكتلة (kcbh، 20 بايت)**: type، format، RDBA، SCN، seq، flag، **checksum عند البايت 16**؛ خوارزمية checksum هي XOR على كلمات الكتلة (شرح وتجربة بـ BBED) [W3]. ثم **ktbbh** (قائمة ITL للمعاملات) ثم **kdbh** (رأس كتلة البيانات) ثم **kdbt** (جداول الكتلة) ثم **kdbr** (دليل الصفوف) ثم الصفوف من نهاية الكتلة ثم **tailchk** (4 بايت). مصدر عملي: RICO2 README [G2] وDBRECOVER FAQ وقائمة أنواع الكتل عند Julian Dyke [W2].
- **المصطلحات الرسمية**: Oracle Database Concepts 21c، الفصل 14 "Logical Storage Structures": الكتل والـ extents والمقاطع، تنسيق الصف (رأس صف ≥3 بايت، قطع الصفوف row pieces)، ASSM وبِتماب L1/L2 [O1].

### 3.2 تنسيق الصف
- كل قطعة صف: **flag byte** (بتات H/F/L/P/K/C/D)، **lock byte** (رقم ITL)، **column count**، ثم لكل عمود بايت طول (1 بايت إذا ≤250، أو 0xFE + بايتين إذا أطول، 0xFF = NULL) ثم البايتات. الصفوف المتسلسلة/المهاجرة تحمل **nrid** (مؤشر إلى القطعة التالية). مراجع: Concepts [O1]، شرح Jonathan Lewis لبتات H/F/L وnrid [W6]، مقال Widlake عن row pieces [W7].

### 3.3 ترميز أنواع البيانات الداخلية (أهم جزء لدقة الاستخراج)
- **NUMBER**: 1–22 بايت؛ البايت الأول أسّ + إشارة، ثم 1–20 بايت mantissa بالأساس 100 (الموجب: digit = byte−1، السالب: digit = 101−byte مع بايت ختامي 0x66) [O2][O4][W4].
- **DATE** 7 بايت (قرن+100، سنة+100، شهر، يوم، ساعة+1، دقيقة+1، ثانية+1)؛ **TIMESTAMP** 7–11 بايت (+ أجزاء الثانية 4 بايت big-endian)؛ **TIMESTAMP WITH TZ** 13 بايت؛ **VARCHAR2/CHAR/RAW** كما هي؛ **ROWID** 10 بايت. المرجع الرسمي لرموز الأنواع الداخلية (1=VARCHAR2، 2=NUMBER، 12=DATE، 23=RAW، 180=TIMESTAMP…) في OCI/OCCI Data Types [O2][O4] وقائمة Julian Dyke [W2]. تطبيق بايثون جاهز للمقارنة في oradul [G1].

### 3.4 القاموس (Data Dictionary) وإعادة بنائه
- جداول الأساس: `bootstrap$` → `obj$`, `tab$`, `col$`, `seg$`, `user$`, `ts$`, `lob$`, `ind$`, `tabpart$`… وكلها في SYSTEM.DBF. المرجع الرسمي: Concepts ch.8 "Data Dictionary" [O5]. طريقة AUL: `UNLOAD TABLE DICT$` ثم `UNLOAD USER/TABLE` [T4]. طريقة DUL: تحميل obj$/tab$/col$ أولاً من ملف SYSTEM (`bootstrap` command) [T1].
- **بدون قاموس**: كل كتلة بيانات تحمل `DATA_OBJECT_ID` في kdbh؛ نجمع الكتل حسب هذا المعرّف ونستدلّ على بنية الأعمدة إحصائياً — هذه بالضبط منهجية DBCarver [A1][A2]، وهي ما تجعله DBRECOVER يدوياً [T2] = **فرصتنا**.

### 3.5 LOB والضغط والتقسيم
- **BasicFile LOB**: inline حتى ~3,960 بايت، وإلا في مقطع LOB منفصل مع LOB index وchunk ثابت. **SecureFile**: inode وchunks متغيرة، يبدأ المقطع بـ16 كتلة. مراجع: Oracle 19c SecureFiles and LOBs Guide [O6]، Jonathan Lewis "Basicfile LOBs" [W8]، NoCOUG 2014 "LOB Internals" [W9].
- **الضغط (Basic/OLTP = Advanced Row Compression)**: على مستوى الكتلة بجدول رموز (symbol table) داخل الكتلة؛ المرجع الرسمي مدونة Oracle DB Storage [O13] وشرح red-gate [W10]. فكّ هذا ضروري لأن معظم مستودعات البيانات المؤسسية مضغوطة.
- **المنصات وendianness**: رأس الملف يحدد المنصة؛ V$TRANSPORTABLE_PLATFORM وRMAN CONVERT [O14] — نحتاج قراءة big-endian (AIX/Solaris SPARC/HP-UX) وlittle-endian.

### 3.6 Redo/Undo (للمرحلة الثانية: استرجاع ما حُذف)
- بنية سجل redo (record → change vectors، OpCodes 5.1 undo، 11.x table ops) موثّقة عند Julian Dyke "Redo Internals" [W1] وLitchfield Part 1 [A5] وOracle Admin Guide "Managing the Redo Log" [O15]؛ تنفيذ عملي كامل في OpenLogReplicator [G4].
- Undo segments وflashback وRecycle Bin كمصدر للبيانات المحذوفة: Litchfield Part 6 [A6]؛ وLitchfield Part 2 "Locating Dropped Objects" [A7]؛ وبحث Choi et al. 2013 لاسترجاع السجلات المحذوفة في أوراكل [A4].

### 3.7 TDE (التشفير الشفاف)
- يُشفَّر الـ tablespace بمفتاح داخل الملف مشفّر بدوره بالمفتاح الرئيسي في wallet؛ فك التشفير دون wallet مستحيل بتصميمه. مرجع: Oracle Advanced Security Guide [O16]. **قرار**: ندعم TDE فقط إذا وفّر العميل الـ wallet وكلمة مرورها (أمر مؤسسي مشروع)؛ ونوثّق ذلك بوضوح.

---

## 4. الأدبيات الأكاديمية الموثّقة (DOI يحلّ)

| # | المرجع | ما يفيدنا |
|---|---|---|
| A1 | Wagner, J., Rasin, A., & Grier, J. (2015). Database forensic analysis through internal structure carving. *Digital Investigation, 14*, S106–S115. https://doi.org/10.1016/j.diin.2015.05.013 | الورقة التأسيسية: أداة عامة تعيد بناء الجداول من صور التخزين لثمانية أنظمة منها **أوراكل**، دون سجلات أو قاموس. 55+ استشهاداً |
| A2 | Wagner, J., Rasin, A., & Grier, J. (2016). Database image content explorer: Carving data that does not officially exist. *Digital Investigation, 18*, S97–S107. https://doi.org/10.1016/j.diin.2016.04.015 | استرجاع الصفوف المحذوفة وغير المخصصة؛ دراسة بقاء البيانات عبر أنظمة مختلفة |
| A3 | Wagner, J., Rasin, A., Glavic, B., Heart, K., Furst, J., Bressan, L., & Grier, J. (2017). Carving database storage to detect and trace security breaches. *Digital Investigation, 22*, S127–S136. https://doi.org/10.1016/j.diin.2017.06.006 | استخدام النحت لاكتشاف العبث؛ مفيد لميزة "تقرير شرعي" |
| A3b | Wagner, J., Rasin, A., Malik, T., Heart, K., Jehle, H., & Grier, J. (2017). Database Forensic Analysis with DBCarver. *CIDR 2017*. (شرائح: https://www.cidrdb.org/cidr2017/slides/p128-wagner-cidr17-slides.pdf) | عرض DBCarver الكامل: معلمات تنسيق الصفحة تُكتشف آلياً، استخراج محتوى قابل للاستعلام من صورة |
| A3c | Wagner, J., Rasin, A., Malik, T., Heart, K., Furst, J., & Grier, J. (2018). Detecting Database File Tampering through Page Carving. *EDBT 2018*. https://doi.org/10.5441/002/edbt.2018.12 | اكتشاف التلاعب على مستوى الصفحة |
| A3d | Wagner, J., Rasin, A., Heart, K., Jacob, R., & Grier, J. (2019). DB3F & DF-Toolkit: The Database Forensic File Format and the Database Forensic Toolkit. *Digital Investigation, 29*, S42–S50. https://doi.org/10.1016/j.diin.2019.04.010 | **صيغة ملف قياسية لمخرجات النحت** — مرشّح لاعتماده كصيغة تصدير مؤسسية/شرعية |
| A3e | Lenard, B., Rasin, A., & Wagner, J. (2026). SoK: What does it mean to benchmark database forensics? *Forensic Science International: Digital Investigation, 57*, 302126. https://doi.org/10.1016/j.fsidi.2026.302126 | أحدث مراجعة منهجية (2026) لكيفية قياس أدوات الطب الشرعي لقواعد البيانات — نستخدمها لتصميم مجموعة اختبارات ومعايير قياس الأداة |
| A4 | Choi, J., Jeong, D., & Lee, S. (2013). The method of recovery for deleted record in Oracle Database. *Journal of the Korea Institute of Information Security and Cryptology, 23*(5), 947–955. https://doi.org/10.13089/jkiisc.2013.23.5.947 | طريقة استرجاع السجلات المحذوفة في أوراكل بتحليل حجم الكتلة ودليل الصفوف (بالكورية مع ملخص إنجليزي) |
| A5 | Litchfield, D. (2007). *Oracle Forensics Part 1: Dissecting the Redo Logs*. NGSSoftware. http://www.davidlitchfield.com/oracle_forensics_part_1._dissecting_the_redo_logs.pdf | رأس ملف redo وبنية السجلات |
| A6 | Litchfield, D. (2007). *Oracle Forensics Part 6: Examining Undo Segments, Flashback and the Oracle Recycle Bin*. http://www.davidlitchfield.com/oracle_forensics_part_6._examining_undo_segments__flashback_and_the_oracle_recycle_bin.pdf | أين تبقى البيانات المحذوفة |
| A7 | Litchfield, D. (2007). *Oracle Forensics Part 2: Locating Dropped Objects*. http://www.davidlitchfield.com/oracle_forensics_part_2.locating_dropped_objects.pdf | إيجاد كائنات محذوفة في ملفات البيانات |
| A8 | Litchfield, D. (2007). *Oracle Forensics Part 5: Finding Evidence of Data Theft in the Absence of Auditing*. http://www.davidlitchfield.com/oracle_forensics_part_5._finding_evidence_of_data_theft_in_the_absence_of_auditing.pdf | قراءة كتل البيانات مباشرة |
| A9 | Litchfield, D. (2008). *Oracle Forensics Part 7: Using the Oracle System Change Number in Forensic Investigations*. http://www.davidlitchfield.com/oracle_forensics_part_7._using_the_oracle_system_change_number__in_forensic_investigations.pdf | SCN كخط زمني للكتل — مهم لترتيب الإصدارات عند التعارض |
| A10 | Litchfield, D. (2007). *The Oracle Hacker's Handbook: Hacking and Defending Oracle*. Wiley. ISBN 978-0-470-08022-1. https://www.wiley.com/en-br/The+Oracle+Hacker%27s+Handbook%3A+Hacking+and+Defending+Oracle-p-9780470133705 | كتاب مرجعي |
| A11 | Lewis, J. (2011). *Oracle Core: Essential Internals for DBAs and Developers*. Apress. https://link.springer.com/book/10.1007/978-1-4302-3955-0 | الكتاب الأهم لفهم الكتل وundo/redo وITL |
| A12 | Lee, S., Park, S., Lee, I., & Choi, J. (2025). A comprehensive analysis and evaluation of SQLite deleted record recovery techniques: A survey. *FSI: Digital Investigation, 55*, 302031. https://doi.org/10.1016/j.fsidi.2025.302031 | ليس أوراكل، لكنه أحدث مسح لتقنيات استرجاع السجلات المحذوفة (تصنيف الطرق يصلح لنا) |
| A13 | Shin, J., & Moon, B. (2026). Forensic recoverability of deleted records under database shrink in Microsoft SQL Server 2025. *Applied Sciences, 16*(13), 6416. https://doi.org/10.3390/app16136416 | منهجية تجريبية حديثة لقياس قابلية الاسترجاع — نموذج لتجاربنا |
| A14 | Nissan, M. I., Wagner, J., & Rasin, A. (2025). ANOC: Automated NoSQL database carver. *FSI: Digital Investigation, 53*, 301929. https://doi.org/10.1016/j.fsidi.2025.301929 | أحدث امتداد لمنهجية النحت الآلي (2025) |

ملاحظة: الأبحاث المتخصصة في أوراكل تحديداً قليلة وقديمة (2007–2015)؛ **هذا بحد ذاته مؤشر على فجوة بحثية** يمكن أن يُنشر عنها لاحقاً.

---

## 5. المراجع الرسمية من أوراكل (تم التحقق)

| # | المرجع |
|---|---|
| O1 | Oracle. (2021). *Oracle Database Concepts 21c*, Ch. 14 Logical Storage Structures. https://docs.oracle.com/en/database/oracle/oracle-database/21/cncpt/logical-storage-structures.html |
| O2 | Oracle. (2021). *Oracle C++ Call Interface Programmer's Guide 21c*, Ch. 5 Data Types (internal datatypes, NUMBER format). https://docs.oracle.com/en/database/oracle/oracle-database/21/lncpp/data-types.html |
| O3 | Oracle. (2019). *Database Reference 19c*: V$DATAFILE_HEADER. https://docs.oracle.com/en/database/oracle/oracle-database/19/refrn/V-DATAFILE_HEADER.html |
| O4 | Oracle. (2019). *SQL Language Reference 19c*: Data Types. https://docs.oracle.com/en/database/oracle/oracle-database/19/sqlrf/Data-Types.html |
| O5 | Oracle. (2021). *Concepts 21c*, Ch. 8 Data Dictionary and Dynamic Performance Views. https://docs.oracle.com/en/database/oracle/oracle-database/21/cncpt/data-dictionary-and-dynamic-performance-views.html |
| O6 | Oracle. (2019). *SecureFiles and Large Objects Developer's Guide 19c*, Ch. 13 LOB Storage with Applications. https://docs.oracle.com/en/database/oracle/oracle-database/19/adlob/LOB-storage-with-applications.html |
| O7 | Oracle. (2019). *Utilities 19c*: DBVERIFY. https://docs.oracle.com/en/database/oracle/oracle-database/19/sutil/oracle-dbverify-utility.html |
| O8 | Oracle. (2019). *Backup and Recovery User's Guide 19c*: Performing Block Media Recovery. https://docs.oracle.com/en/database/oracle/oracle-database/19/bradv/rman-block-media-recovery.html |
| O9 | Oracle. *Error Help*: ORA-01578 ORACLE data block corrupted. https://docs.oracle.com/en/error-help/db/ora-01578/ |
| O10 | Oracle. (2019). *PL/SQL Packages 19c*: DBMS_REPAIR. https://docs.oracle.com/en/database/oracle/oracle-database/19/arpls/DBMS_REPAIR.html |
| O11 | Oracle. (2019). *Automatic Storage Management Administrator's Guide 19c* (AMDU, kfed). https://docs.oracle.com/en/database/oracle/oracle-database/19/ostmg/automatic-storage-management-administrators-guide.pdf ؛ وأمثلة CERN: https://twiki.cern.ch/twiki/bin/view/PDBService/ASM_utilities |
| O12 | Oracle. (2019). *PL/SQL Packages 19c*: DBMS_LOGMNR_D. https://docs.oracle.com/en/database/oracle/oracle-database/19/arpls/DBMS_LOGMNR_D.html |
| O13 | Oracle DB Storage Blog. When to use the various types of Oracle data compression. https://blogs.oracle.com/dbstorage/when-to-use-the-various-types-of-oracle-data-compression |
| O14 | Oracle. (2018). *Transporting Data Across Platforms* (endianness). https://docs.oracle.com/en/database/oracle/oracle-database/18/spucd/transporting-data-across-platforms.html |
| O15 | Oracle. *Database Administrator's Guide*: Managing the Redo Log. https://docs.oracle.com/en/database/oracle/oracle-database/26/admin/managing-the-redo-log.html |
| O16 | Oracle. (2021). *Advanced Security Guide 21c*: Introduction to Transparent Data Encryption. https://docs.oracle.com/en/database/oracle/oracle-database/21/asoag/introduction-to-transparent-data-encryption.html |

## 6. مراجع تقنية لخبراء معروفين (مواقع رسمية للمؤلفين، تم التحقق)

| # | المرجع |
|---|---|
| W1 | Dyke, J. *Redo Internals* / Redo Operations (Level 4/10/11/23/24). http://www.juliandyke.com/Internals/Redo/Redo.php |
| W2 | Dyke, J. *Oracle Internals*: Internal Data Types, Block Types, Block Classes. http://www.juliandyke.com/Internals/Internals.php ؛ http://www.juliandyke.com/Internals/InternalDataTypes.php ؛ http://www.juliandyke.com/Internals/BlockTypes.php |
| W3 | Stawiarski, K. (2018). Oracle database block checksum XOR algorithm explained. https://blog.ora-600.pl/2018/01/28/oracle-database-block-checksum-xor-algorithm-explained/ |
| W4 | Stawiarski, K. (2017). How Oracle stores numbers internally. https://blog.ora-600.pl/2017/08/21/how-oracle-stores-numbers-internally/ |
| W4b | Stawiarski, K. (2023). Hacking database writer (قراءة كتل مشفّرة من ذاكرة VM). https://blog.ora-600.pl/2023/05/24/hacking-database-writer/ |
| W5 | atoracle.cn. Understanding Datafile Header kcvfh structure. http://www.atoracle.cn/index.php?g=Home&m=Article&a=show&id=54 |
| W6 | Lewis, J. (2011). I Wish (3) — بتات H/F/L وnrid في الصفوف المتسلسلة. https://jonathanlewis.wordpress.com/2011/12/16/i-wish-3/ |
| W7 | Widlake, M. (2022). Chained Rows, Migrated Rows, Row Pieces – The Basics. https://mwidlake.wordpress.com/2022/10/11/chained-rows-migrated-rows-row-pieces-the-basics/ |
| W8 | Lewis, J. (2016). Basicfile LOBs 2. https://jonathanlewis.wordpress.com/2016/08/18/basicfile-lobs-2/ |
| W9 | NoCOUG. (2014). *LOB Internals and Best Practices*. http://www.nocoug.org/download/2014-11/NoCOUG-2014_11_LOB_Internals.pdf |
| W10 | Red Gate Simple Talk. Compression in Oracle – Part 3: OLTP Compression. https://www.red-gate.com/simple-talk/databases/oracle-databases/compression-in-oracle-part-3-oltp-compression/ |
| W11 | Finnigan, P. *Oracle Forensics* (شرائح). https://www.petefinnigan.com/Oracle_Forensics_scotland6slides.pdf |
| W12 | NCC Group. Oracle Forensics Part 5 (نسخة الناشر). https://www.nccgroup.com/research/oracle-forensics-part-5-finding-evidence-of-data-theft-in-the-absence-of-auditing/ |

## 7. مراجع الأدوات (تم التحقق)

| # | المرجع |
|---|---|
| T1 | AskMaclean. *Oracle DUL Data recovery UnLoader — what you need to know* (نسخة من دليل DUL الداخلي). https://askmaclean.com/archives/oracle-dul-data-recovery-unloader-what-you-need-to-know.html ؛ ومعلومات المطوّر: https://askmaclean.com/archives/dul-oracle-data-unloader-download.html |
| T2 | DBRECOVER Team. (2025-12-11). *DBRECOVER for Oracle FAQ: Licensing & Technical Operations*. https://www.dbrecover.com/blog/dbrecover-oracle-faq.html ؛ الصفحة الرسمية: https://parnassusdata.com/en/oracle-dul ؛ دليل برامج الفدية: https://www.dbrecover.com/blog/database-ransomware-recovery-guide.html ؛ SourceForge: https://sourceforge.net/projects/prm-dul-oracle/ |
| T3 | ODU: dbform.com (2009) إصدار 3.0.2 مع unload_deleted. https://www.dbform.com/2009/05/20/odulaoxiong/ ؛ مقارنة DUL/AUL/ODU: https://www.cnblogs.com/tianlesoftware/archive/2011/09/14/3609565.html |
| T4 | anysql. *AUL (MyDUL) Oracle 异常数据恢复指南* (PDF). https://www.anysql.net/AUL_User_Guide.pdf ؛ GitHub docs: https://github.com/anysql/AUL |
| T7 | xifenfei. *自研Oracle恢复小工具* (Oracle Recovery Tools, OBET, OraScan, OraFHR). https://www.xifenfei.com/oracle-tools ؛ دليل OraScan: https://www.xifenfei.com/wp-content/uploads/recover_check/OraScan.pdf |
| G1 | https://github.com/jieguolove/oradul |
| G2 | https://github.com/ora600pl/rico2 |
| G3 | https://github.com/ora600pl/odbv |
| G4 | https://github.com/bersler/OpenLogReplicator (+ https://www.bersler.com/openlogreplicator/) |
| G5 | https://github.com/averemee-si/oracdc |
| G6 | https://github.com/zhihuiFan/OracleRedoParser |
| G7 | https://github.com/wublabdubdub/PDU-PostgreSQLDataUnloader |
| G8 | https://github.com/xuji755/bic-dmdul |

---

## 8. تحليل الفجوات: ما الذي لم يصنعه أحد بعد؟

| الفجوة | الوضع الحالي | ما نقترحه |
|---|---|---|
| **استدلال آلي للمخطط بدون قاموس** | DBRECOVER/DUL/AUL: يعرضون `OBJ(12345)` ويتركون التحديد للمستخدم يدوياً [T2] | محرّك استدلال إحصائي للأنواع (توزيع بايتات الطول، أنماط NUMBER/DATE الصالحة، نسبة NULL)، ثم مطابقة الكتل مع بقايا `col$`/`obj$` إن وُجدت جزئياً، ومع مخططات معروفة لتطبيقات شائعة (EBS، SAP على أوراكل…) |
| **مفتوح المصدر وحديث (12c–26ai، PDB/CDB)** | oradul حتى 19c، بلا قاموس؛ البقية مغلقة | نواة مفتوحة (Rust أو Go للأداء + واجهة بايثون) مع نموذج مؤسسي مدفوع للدعم والميزات المتقدمة |
| **صيغة مخرجات شرعية قياسية** | لا أحد يصدّر بصيغة قياسية؛ تصدير CSV/SQL فقط | اعتماد DB3F [A3d] + تقرير سلسلة الحيازة (hash لكل كتلة وملف) |
| **برامج الفدية (تشفير جزئي)** | DBRECOVER يوثّق السيناريو؛ xifenfei لديه OraFHR نقطي | وضع "Scavenge": مسح كامل بتوقيع kcbh، إعادة بناء رأس ملف افتراضي، تقرير نسبة الكتل السليمة لكل جدول قبل الشراء/العمل |
| **ASM مدمج** | DBRECOVER يحتاج أداة مرافقة PRMSCAN؛ أوراكل AMDU لا تعمل على كل التلف | محلل بيانات ASM الوصفية (allocation table, file directory) مدمج في نفس الأداة |
| **LOB + ضغط + صفوف متسلسلة في وضع بدون قاموس** | غير مدعوم بوضوح في المفتوحة؛ التجارية تدّعيه | أولوية عالية في المرحلة 2 |
| **قابلية القياس (benchmark) وتقييم موضوعي** | لا معيار متفق عليه؛ ورقة SoK 2026 تطالب بذلك [A3e] | بناء مجموعة ملفات اختبار بتلف مصطنع ونشر النتائج — يميّزنا تسويقياً وأكاديمياً |
| **واجهة/API للمؤسسات** | GUI سطح مكتب (Java Swing) أو CLI | CLI + REST API + واجهة ويب، تشغيل في حاوية، تسجيل تدقيق كامل |

---

## 9. اعتبارات قانونية وترخيصية (مهمة للمؤسسات)

1. **DUL** ملكية أوراكل الداخلية؛ لا نستخدم أي كود أو ثنائيات منه. ما نستفيده هو **التوثيق المنشور للمفاهيم** فقط.
2. **OpenLogReplicator** مرخّص AGPL-3.0 [G4]: أي ربط برمجي مباشر يلزمنا بفتح مصدر ما نربطه. نستخدمه مرجعاً للفهم أو كعملية منفصلة، لا كمكتبة مدمجة، ما لم نقرر AGPL لمشروعنا.
3. **oradul** MIT [G1]: يمكن الاستفادة من كوده بحرية مع الإسناد.
4. **RICO2/odbv** بلا ترخيص معلن [G2][G3]: نعاملها كمرجع للقراءة فقط، لا ننسخ منها.
5. **TDE**: لا نحاول كسر التشفير؛ ندعم فقط فك التشفير بمفاتيح يملكها العميل [O16].
6. **بيانات العملاء**: الأداة تقرأ بيانات حساسة؛ يلزم وضع سياسة معالجة محلية (on-premises) وعدم رفع ملفات.

---

## 10. خارطة الطريق المقترحة للتطوير (للنقاش مع وكيل التدوير)

**المرحلة 0 — المختبر (أسبوع):**
- حاوية Oracle Database Free 23ai (صورة `gvenzl/oracle-free` على Docker Hub أو container-registry.oracle.com) لإنشاء ملفات بيانات مرجعية بكل الأنواع (NUMBER سالب/كسري، DATE، TIMESTAMP TZ، CLOB/BLOB inline وout-of-line، جداول مضغوطة، صفوف متسلسلة، partitions).
- توليد تلف مضبوط: الكتابة فوق كتل محددة بـ `dd`، حذف SYSTEM.DBF، تشفير 10% من الملف عشوائياً (محاكاة فدية)، قطع الملف.
- مجموعة اختبارات ذهبية: لكل جدول نعرف الصفوف الحقيقية → نقيس الاسترجاع (precision/recall) وفق توصيات SoK 2026 [A3e].

**المرحلة 1 — نواة القراءة (3–4 أسابيع):**
- قارئ ملف: اكتشاف حجم الكتلة والمنصة (endianness) من kcvfh؛ التحقق من checksum (XOR) وtailchk لكل كتلة وتصنيفها (سليمة/تالفة/فارغة/مشفّرة).
- محلل كتل: kcbh → ktbbh → kdbh → kdbt → kdbr → صفوف؛ تجميع حسب DATA_OBJECT_ID.
- فكّ أنواع البيانات الأساسية (مقارنة مرجعية مع oradul وRICO2 والتوثيق الرسمي).
- وضع القاموس: قراءة bootstrap$/obj$/tab$/col$/seg$/user$/ts$ من SYSTEM وربطها.
- تصدير CSV/SQL/Parquet.

**المرحلة 2 — ما يميّزنا (4–6 أسابيع):**
- وضع بدون قاموس مع **استدلال آلي للأنواع والأسماء**.
- LOB (BasicFile ثم SecureFile)، الضغط الأساسي/OLTP، الصفوف المتسلسلة (nrid)، الجداول العنقودية، partitions.
- وضع "Scavenge" لبرامج الفدية والملفات المحذوفة (نحت بتوقيع kcbh من قرص/صورة خام).
- تقرير شرعي (DB3F + hashes).

**المرحلة 3 — المؤسسات (4 أسابيع):**
- ASM metadata reader.
- استرجاع الصفوف المحذوفة من الكتل (row directory مع flags محذوفة) ومن undo، ثم قراءة archive logs (بالاستفادة من معرفة OpenLogReplicator مع احترام الترخيص).
- CLI + REST API + واجهة ويب، حاوية، RBAC، سجل تدقيق.
- دعم TDE بمفاتيح العميل.

**اختيار اللغة**: Rust أو Go للنواة (أداء على ملفات تيرابايت، أمان ذاكرة)، مع ربط بايثون للتحليل والاستدلال. بايثون وحده (كما في oradul) بطيء على الملفات الكبيرة.

---

## 11. ما لم أتمكن من التحقق منه (شفافية)

- الموقع الرسمي لأداة **ODU** (laoxiong.net / oracleodu.com) لم يستجب (HTTP 000) — معلومات ODU أعلاه من مصادر ثانوية موثوقة (dbform.com 2009، cnblogs 2011) وليست من موقع الناشر.
- لم أعثر على **ورقة علمية محكّمة حديثة (2020–2026) مخصصة لأوراكل تحديداً** في استرجاع البيانات من ملفات تالفة؛ الأحدث المتخصص هو Choi et al. 2013. الأوراق الحديثة (2025–2026) تغطي SQLite وSQL Server وNoSQL ومنهجية القياس العامة.
- مواصفات **DUL الرسمية** (دليل المستخدم v10.2.4.27) موجودة فقط كنسخ على مواقع طرف ثالث (askmaclean/sourceforge wiki)؛ لا توجد صفحة أوراكل عامة لها.
- لم أختبر الأدوات التجارية فعلياً؛ القدرات المذكورة منقولة من توثيقها الرسمي.
- Stellar/Kernel/Recovery Toolbox: لم أجد توثيقاً تقنياً يستحق الاعتماد، فاستبعدتها من المراجع التقنية.

---

## 12. ملفات العمل المحلية

- `workspace/oracle-recovery/repos/oradul` و`repos/rico2` و`repos/OpenLogReplicator`: نسخ محلية للدراسة.
- `workspace/oracle-recovery/search*.log`: سجلات البحث الخام.
- `workspace/oracle-recovery/s.sh`: سكربت البحث.
