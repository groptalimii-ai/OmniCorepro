# نشر EnterpriseAI-OS على Kubernetes

## ما هذا المجلد وما الذي أُصلح فيه

كان `backend.yaml` قبل هذه الجولة يحتوي على أعطال حرجة كانت ستمنع أي نشر فعلي من
العمل إطلاقًا:

- **متغيرات بيئة ناقصة (الأخطر):** `ENCRYPTION_KEY`/`SECRET_KEY`/`ENVIRONMENT`/
  `REDIS_URL`/`ALLOWED_ORIGINS`/`ALLOWED_HOSTS` كانت غائبة تمامًا. بما أن
  `core/config.py` يرفض الإقلاع فورًا في `ENVIRONMENT=production` بأسرار
  افتراضية/قصيرة (منذ الجولة 11 من `FIX_STATUS.md`)، كان أي pod يُنشَر بالملف
  السابق سيدخل **CrashLoopBackOff فوري ودائم**.
- **`livenessProbe`/`readinessProbe` كانتا نفس المسار** (`/health`)، والذي كان
  بدوره يُرجع `200 OK` دائمًا مهما كانت حالة قاعدة البيانات الفعلية (أُصلح في
  `backend/main.py` أيضًا - راجع `FIX_STATUS.md`). النتيجة المزدوجة: لا فحص حقيقي
  أصلاً، وحتى بعد إصلاحه، اعتماد liveness على تبعية خارجية كان سيسبب "عاصفة إعادة
  تشغيل" عند أي انقطاع DB مؤقت.
- **الصورة بوسم `:latest`** - غير قابل للتتبع أو الرجوع الموثوق.
- **لا `securityContext`، لا `startupProbe`، لا `HorizontalPodAutoscaler`، لا
  `PodDisruptionBudget`.**
- **لا مانيفست frontend إطلاقًا**، ولا `Ingress` للوصول الخارجي لأي شيء.

كل ما سبق أُصلح. الملفات الآن:

| الملف | الغرض |
|---|---|
| `namespace.yaml` | مساحة الاسم `enterpriseai` |
| `configmap.yaml` | إعدادات غير سرّية (`ENVIRONMENT`, `REDIS_URL`, ...) |
| `secrets-template.yaml` | **توثيقي فقط** - لا تُطبِّقه كما هو، راجع التعليمات بداخله |
| `backend.yaml` | Deployment + Service + HPA + PodDisruptionBudget لخدمة API |
| `frontend.yaml` | Deployment + Service للواجهة الأمامية (nginx) |
| `ingress.yaml` | وصول خارجي بـ TLS (يتطلب ingress-nginx + cert-manager مثبَّتين مسبقًا) |

## ⚠️ ما لم يُبنَ في هذا المجلد - ولماذا عمدًا

**لا توجد هنا مانيفستات Postgres/Redis/Kafka/ClickHouse.** هذا قرار مقصود وليس
نسيانًا: كتابة `StatefulSet` يدوية لقواعد بيانات إنتاجية (خصوصًا Kafka متعدد
الوسطاء، وPostgres مع النسخ الاحتياطي/replication) بشكل صحيح وآمن هو عمل كبير
ومعقّد بحد ذاته (تخزين دائم، نسخ احتياطي، استرجاع، ترقيات بلا توقف)، وإعادة اختراعه
هنا كان سيُنتج غالبًا بنية هشة تبدو كاملة لكنها غير جاهزة فعليًا لبيانات عملاء
حقيقيين. **التوصية الفعلية لنشر إنتاجي متعدد العملاء:**

- **الخيار الأفضل:** خدمات مُدارة من مزوّد السحابة (RDS/Cloud SQL لـ Postgres،
  ElastiCache/Memorystore لـ Redis، MSK/Confluent Cloud لـ Kafka) - تتولى النسخ
  الاحتياطي والتوفر العالي والترقيات نيابة عنك.
- **إن كان لا بد من التشغيل داخل الكلاستر:** استخدم Helm charts ناضجة ومُختبَرة
  مجتمعيًا (Bitnami: `postgresql`, `redis`, `kafka`) بدل كتابة StatefulSets يدويًا.

`docker-compose.full.yml` يبقى المسار الموثَّق والمُختبَر لتشغيل المكدس الكامل
(بما فيه Postgres/Redis/Kafka/ClickHouse) دفعة واحدة لأغراض التطوير أو نشر شركة
واحدة بحجم صغير/متوسط على خادم واحد.

## خطوات النشر (بعد أن تصبح Postgres/Redis/Kafka جاهزة عبر أحد الخيارين أعلاه)

```bash
kubectl apply -f namespace.yaml
kubectl apply -f configmap.yaml

# أنشئ الأسرار الفعلية - راجع التعليمات الكاملة داخل secrets-template.yaml
kubectl create secret generic app-secrets -n enterpriseai \
  --from-literal=jwt-secret="$(openssl rand -base64 48)" \
  --from-literal=encryption-key="$(openssl rand -base64 48)" \
  --from-literal=secret-key="$(openssl rand -base64 48)"
kubectl create secret generic db-credentials -n enterpriseai \
  --from-literal=url="postgresql://enterpriseai:<REAL_PASSWORD>@<YOUR_DB_HOST>:5432/enterpriseai"

# عدّل __BUILD_TAG__ في backend.yaml/frontend.yaml لوسم صورة حقيقي من CI/CD
# (وليس :latest)، وعدّل النطاقات في configmap.yaml/ingress.yaml لنطاقات العميل
# الفعلية، ثم:
kubectl apply -f backend.yaml
kubectl apply -f frontend.yaml
kubectl apply -f ingress.yaml

kubectl get pods -n enterpriseai -w
```

## ⚠️ تنبيه صادق

لم تُختبَر هذه المانيفستات على كلاستر Kubernetes حقيقي (لا وصول لبيئة كهذه أثناء
المراجعة) - فقط تحقَّق يدويًا من صحة بنية YAML واتساقها مع الكود الفعلي في
`backend/` (مسارات `/health/live`، `/health/ready`، أسماء متغيرات البيئة، منافذ
الحاويات). شغّلها في بيئة اختبار (staging) أولاً، وراقب `kubectl describe pod`/
`kubectl logs` عن كثب عند أول نشر فعلي.
