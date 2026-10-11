# Oracle Datafile Recovery Platform

منصة مؤسسية لقراءة ملفات بيانات أوراكل (.dbf) التالفة أو غير القابلة للفتح واستخراج البيانات منها دون الحاجة إلى instance أوراكل — بوضع القاموس وبدونه، مع دعم ASM وLOB والضغط والصفوف المتسلسلة، وتقرير شرعي قياسي.

Enterprise-grade, DUL-class tool for extracting data from corrupted / unopenable Oracle datafiles without a running Oracle instance.

## Repository layout

| Path | Content |
|---|---|
| `docs/research/01_foundational_research_ar.md` | البحث التأسيسي: الأدوات القائمة، الأبحاث، بنية الكتل، الفجوات، خارطة الطريق (كل مرجع تم التحقق منه) |
| `docs/` | Design documents (to be added) |
| `src/` | Core engine (to be added) |
| `tests/fixtures/` | Reference datafiles with controlled corruption (to be added) |

## Status

Phase 0 — research complete; design & lab setup next. See `docs/research/01_foundational_research_ar.md` §10 for the proposed roadmap.
