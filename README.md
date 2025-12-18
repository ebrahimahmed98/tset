# Movie-Manager

## دليل المشروع + بنك أسئلة DevOps (سؤال وجواب)

المحتوى: Terraform + AWS VPC/EKS + Jenkins EC2 + Docker/ECR + Kubernetes + Helm + Ingress/ALB + Monitoring + Mongo/PVC

النسخة: v10 (منظّمة ومُنسّقة)

تاريخ التحديث: 2025-12-18

ملاحظة: كل سؤال تحته إجابة نموذجية + نقاط تركّز عليها في الانترفيو + ASCII diagrams.


## الجزء A: شرح مشروع Movie-Manager بسرعة (Q&A)

```
Flow (high-level)
GitHub -> Jenkins (EC2) -> Docker build/push -> ECR
                           -> aws eks update-kubeconfig
                           -> kubectl apply/set image/rollout
                           -> Ingress -> LBC -> ALB -> App

```

### س1: إيه هدف مشروع Movie-Manager؟

**ج:** تجهيز وتشغيل تطبيق (Frontend + Backend + MongoDB) على AWS باستخدام EKS، مع CI/CD عبر Jenkins لبناء الصور ورفعها على ECR ثم نشرها على Kubernetes، وإضافة Monitoring (Prometheus/Grafana).

### س2: ليه Jenkins يقدر ينشر على EKS مع إن بورت 22 مقفول على النودز؟

**ج:** لأن النشر بيتم عبر Kubernetes API Server على HTTPS/443 باستخدام kubectl، مش عبر SSH للنودز. النودز نفسها بتسحب الصور من ECR عبر 443.

### س3: إزاي Jenkins “بيوصل” للـ EKS؟

**ج:** Jenkins على EC2 بيستخدم AWS CLI لعمل aws eks update-kubeconfig ثم kubectl بيتعامل مع الـ endpoint ويستخدم IAM token (aws eks get-token). بعد كده RBAC جوه الكلاستر يحدد الصلاحيات.

### س4: ليه AWS Load Balancer Controller مهم عندك؟

**ج:** لأنه هو اللي بيحوّل Ingress resources إلى ALB على AWS. لو مش متثبت، هتلاقي Ingress بدون ADDRESS أو الترافيك مش بيوصل.

### س5: ليه Mongo ممكن تبقى Pending؟ وإزاي بتتفاداها؟

**ج:** غالبًا بسبب PVC/StorageClass/CSI Driver. لازم EBS CSI شغال و StorageClass (gp3) موجود و volume binding مناسب (WaitForFirstConsumer) لتجنب AZ mismatch.

### س6: ليه Monitoring معمول في Terraform (infra/monitoring)؟

**ج:** عشان يبقى قابل لإعادة النشر بسهولة وبشكل idempotent، ولأن stack زي kube-prometheus-stack له إعدادات كثيرة ويفضل تثبيته بشكل منظم (Helm/Terraform) بدل أوامر متفرقة.


## الجزء B: بنك أسئلة DevOps مرتبط بهيكلة مشروعك + إجابات نموذجية

## 1. هيكلة الريبو وملكية المكونات (Repo Ownership)

```
Repo
├─ Jenkinsfile          -> CI/CD pipeline
├─ infra/eks/           -> Terraform: VPC + EKS + Jenkins EC2 (+ IAM)
├─ infra/monitoring/    -> Terraform: EBS CSI + StorageClass gp3 + kube-prometheus-stack
├─ infra/addons/        -> Scripts/Helm helpers (مثل تثبيت LBC)
└─ k8s/                 -> Kubernetes manifests: app + mongo + ingress + jobs
```

### س: ليه عندك infra/eks منفصل عن infra/monitoring؟ وإزاي بتمنعهم يبوّظوا على بعض في الـ state؟

**ج:** الفصل بيخلّي كل جزء له دورة حياة مختلفة: infra/eks نادر يتغير (VPC/EKS/Jenkins)، بينما monitoring يتغير أكتر (charts/versions). بتمنع التداخل عن طريق state منفصل لكل مجلد (backend مختلف أو workspace منفصل) + متغيرات أسماء واضحة تمنع موارد متكررة.

✓ نقاط تركّز عليها في الإجابة: (السبب/الاختيار/الـ trade-offs/أوامر التشخيص لو فيه مشكلة).

### س: إيه اللي يعتبر “Infrastructure” في مشروعك وإيه اللي يعتبر “Application deploy”؟ وليه مش كله Terraform؟

**ج:** Infrastructure = موارد AWS طويلة العمر (VPC, EKS, IAM, Jenkins EC2, ECR policy…)، وده Terraform ممتاز فيه. Application deploy = موارد Kubernetes اللي بتتغير كتير (Deployments/Services/Ingress/Jobs) ودي أسرع وأسهل بـ kubectl/manifest لأن تغييراتها متكررة ومش محتاجة Terraform لكل rollout.

✓ نقاط تركّز عليها في الإجابة: (السبب/الاختيار/الـ trade-offs/أوامر التشخيص لو فيه مشكلة).

### س: لو حد جديد فتح الريبو: إيه أول 3 ملفات لازم يقراهم عشان يفهم الـ flow؟

**ج:** 1) Jenkinsfile (بيوضح build/push/deploy). 2) infra/eks (بيوضح البنية). 3) k8s/ (بيوضح تشغيل التطبيق والمونجو والـ ingress).

✓ نقاط تركّز عليها في الإجابة: (السبب/الاختيار/الـ trade-offs/أوامر التشخيص لو فيه مشكلة).

### س: إزاي بتتعامل مع اختلاف البيئات (dev/staging/prod) بنفس الهيكلة؟

**ج:** بتعمل parameterization: terraform.tfvars لكل بيئة + naming prefixes (env-). وفي Kubernetes: namespace لكل بيئة أو kubeconfig لكلاستر مختلف، وبتستخدم tags مختلفة للـ images (commit SHA أو build number) لكل بيئة.

✓ نقاط تركّز عليها في الإجابة: (السبب/الاختيار/الـ trade-offs/أوامر التشخيص لو فيه مشكلة).

—————————————————————————————————————————————

## 2. المعمارية ورحلة التنفيذ من البداية للنهاية (Architecture & End-to-End Flow)

```
git push
  |
  v
Jenkins Pipeline
  |-- docker build (frontend/backend)
  |-- login + push to ECR (tags: <commitSHA>)
  |-- aws eks update-kubeconfig (يوصل للـ API على 443)
  |-- kubectl apply ./k8s
  |-- kubectl set image + rollout status
  |-- (Ingress -> LBC -> ALB)  (Pods pull from ECR on 443)
  v
App Running + Monitoring
```

### س: اشرح الـ flow من أول git push لحد ما الـ pods تبقى Running على EKS.

**ج:** 1) push على GitHub يفعّل Jenkins job. 2) Jenkins يعمل checkout. 3) يبني Docker images للـ frontend/backend. 4) يعمل login على ECR ويعمل push للصور بتاج واضح (يفضل commit SHA). 5) يجهز اتصال EKS عبر aws eks update-kubeconfig. 6) ينشر manifests: kubectl apply -f k8s/. 7) يثبت إن الإصدار الصحيح اتطبق عبر kubectl set image ثم kubectl rollout status. 8) لو فيه Ingress: الـ AWS Load Balancer Controller ينشئ ALB ويربطه بالـ Service/Pods. الـ Pods نفسها بتسحب الصور من ECR عبر 443، مش عبر SSH.

✓ نقاط تركّز عليها في الإجابة: (السبب/الاختيار/الـ trade-offs/أوامر التشخيص لو فيه مشكلة).

### س: ليه فصلت infra عن app deploy؟ وإيه اللي يتعمل Terraform وإيه اللي يتعمل kubectl؟

**ج:** Terraform للموارد الثقيلة والبطيئة (VPC/EKS/IAM/Jenkins/Storage addons/Monitoring) لأنه يضمن state ويمنع اختلافات. kubectl للموارد اليومية (deploy/rollout/config) لأنه أسرع ومناسب للتغييرات المتكررة.

✓ نقاط تركّز عليها في الإجابة: (السبب/الاختيار/الـ trade-offs/أوامر التشخيص لو فيه مشكلة).

### س: إيه الـ “source of truth” عندك؟ Terraform ولا الـ manifests ولا Jenkins؟

**ج:** Terraform هو source of truth للبنية على AWS. ملفات k8s هي source of truth للـ desired state جوه الكلاستر. Jenkins هو “منفّذ” (orchestrator) مش مرجع—دوره يطبق اللي موجود في Git.

✓ نقاط تركّز عليها في الإجابة: (السبب/الاختيار/الـ trade-offs/أوامر التشخيص لو فيه مشكلة).

### س: لو عايز تعمل بيئة جديدة (staging) في ساعة—هتعملها إزاي بنفس الكود؟

**ج:** طريقتين: (أ) كلاستر جديد: terraform vars جديدة (اسم/شبكة/cluster) ثم نفس Jenkinsfile يdeploy على kubeconfig الجديد. (ب) نفس الكلاستر: namespace جديد + قيم env مختلفة + images tags مختلفة. الأهم: naming + isolation + عدم مشاركة secrets بين البيئات.

✓ نقاط تركّز عليها في الإجابة: (السبب/الاختيار/الـ trade-offs/أوامر التشخيص لو فيه مشكلة).

—————————————————————————————————————————————

## 3. شبكات AWS (VPC/Networking)

```
(No SSH needed)
Jenkins EC2  --HTTPS/443-->  EKS API endpoint
Nodes        --HTTPS/443-->  ECR (pull images)
ALB          ----HTTP/HTTPS--> Service/Pods (target port)
Private subnets? -> NAT needed for outbound (ECR/updates)
```

### س: الـ EKS endpoint عندك Public ولا Private؟ وإيه أثر ده على Jenkins EC2؟

**ج:** في مشروعك الحالي: **EKS Public Endpoint (endpointPublicAccess = true)**.  
ده معناه إن Jenkins على EC2 يقدر يتواصل مع **Kubernetes API Server** على **HTTPS/443** مباشرة طالما عنده:
- Route للإنترنت (لو EC2 في Public Subnet) **أو** Outbound عبر NAT (لو EC2 في Private Subnet).
- IAM Role + RBAC جوه الكلاستر (ده جزء الصلاحيات، مش الشبكة).

**مهم:** قفل Port 22 على الـ Nodes طبيعي جدًا ومش له علاقة بالنشر؛ لأن Jenkins **مش بيـ SSH للنودز**. هو بيكلم الـ API Server فقط على 443.

**أوامر تأكيد (مفيدة في المناقشة)**
```bash
aws eks describe-cluster --name depi-eks --region us-east-1   --query 'cluster.resourcesVpcConfig.{public:endpointPublicAccess,private:endpointPrivateAccess}' --output table
```

✓ نقاط تركّز عليها في الإجابة: (443 للـ API مش 22، الفرق بين reachability و RBAC، وإزاي تتأكد بالأمر فوق).
### س: لو حوّلت الـ endpoint إلى Private (Private endpoint): Jenkins يوصل إزاي؟ (subnets/route/NAT/peering/VPN)

**ج:** ده سيناريو *مش اللي شغال عندك دلوقتي* (لأنك شغال Public endpoint)، بس مهم في المناقشة:
- أفضل وضع: Jenkins EC2 يكون **داخل نفس الـ VPC** (داخل subnets تقدر تشوف الـ Control Plane ENIs) فيوصل للـ Private endpoint مباشرة على **443**.
- لو Jenkins خارج الـ VPC: لازم **Site-to-Site VPN / Direct Connect / Transit Gateway / Peering** حسب تصميم الشبكة.
- **NAT مش بيحل الوصول للـ Private API**؛ NAT دوره للـ outbound للإنترنت (مثل تنزيل dependencies أو الوصول لخدمات AWS بدون VPC endpoints).

✓ نقاط تركّز عليها في الإجابة: (Private endpoint = شبكة/VPC connectivity، NAT = internet outbound، والاتصال دايمًا 443).
### س: Security Groups: مين بيسمح لمين على 443 تحديدًا؟

**ج:** Jenkins EC2: outbound 443 إلى EKS endpoint + AWS APIs. Worker nodes: outbound 443 إلى ECR وSTS (لسحب الصور). ALB: inbound 80/443 من الإنترنت، وoutbound إلى target groups على البورت اللي Service بيستخدمه.

✓ نقاط تركّز عليها في الإجابة: (السبب/الاختيار/الـ trade-offs/أوامر التشخيص لو فيه مشكلة).

### س: بتتعامل إزاي مع outbound access للـ nodes؟ ليه محتاج NAT؟

**ج:** لأن النودز غالبًا في private subnets؛ لازم NAT عشان تسحب images من ECR وتوصل لـ AWS endpoints للتحديثات. بديل أرخص/أكثر أمانًا: VPC Endpoints لـ ECR/STS/S3 حسب الحاجة.

✓ نقاط تركّز عليها في الإجابة: (السبب/الاختيار/الـ trade-offs/أوامر التشخيص لو فيه مشكلة).

### س: إزاي بتتأكد إن Ingress/ALB شغال في Subnets صح (public vs private)؟

**ج:** في مشروعك الحالي أنت شغال **Internet-facing ALB** (Public). فالتأكد بيبقى كده:
- الـ **Public Subnets** لازم عليها tag:
  - `kubernetes.io/role/elb=1`
  - + tag الكلاستر (زي `kubernetes.io/cluster/depi-eks=shared` أو `owned`)
- الـ Ingress لازم يبقى واضح إنه public:
  - `alb.ingress.kubernetes.io/scheme: internet-facing`
  - (ولو بتحدد subnets يدويًا) `alb.ingress.kubernetes.io/subnets: subnet-...`

**تشخيص سريع لو الـ ALB ما اتعملش**
```bash
kubectl describe ingress <name>
kubectl -n kube-system logs deploy/aws-load-balancer-controller | tail -n 200
kubectl -n kube-system get sa,deploy | grep -i load-balancer
```

✓ نقاط تركّز عليها في الإجابة: (subnet tags + scheme annotation + controller logs/events).
## 4. IAM وتوثيق/تفويض EKS (AuthN/AuthZ)

```
kubectl
  |
  | (exec) aws eks get-token  -> STS -> IAM identity
  v
EKS API Server
  |
  | RBAC (Role/RoleBinding/ClusterRoleBinding)
  v
Allowed / Forbidden
```

### س: Explain الفرق بين Authentication و Authorization في EKS.

**ج:** Authentication: مين أنت؟ في EKS غالبًا عبر IAM token (aws eks get-token). Authorization: مسموح تعمل إيه؟ ده عبر Kubernetes RBAC (roles/bindings) داخل الكلاستر.

✓ نقاط تركّز عليها في الإجابة: (السبب/الاختيار/الـ trade-offs/أوامر التشخيص لو فيه مشكلة).

### س: Jenkins بيستخدم IAM Role ولا AWS keys؟ وليه؟

**ج:** الأفضل IAM Role via EC2 Instance Profile: أقل مخاطرة، مفيش مفاتيح ثابتة، وسهل تدير الصلاحيات. AWS keys ممكن تشتغل لكن خطر التسريب أعلى وتحتاج rotation.

✓ نقاط تركّز عليها في الإجابة: (السبب/الاختيار/الـ trade-offs/أوامر التشخيص لو فيه مشكلة).

### س: aws eks update-kubeconfig بيعمل إيه بالظبط؟

**ج:** بيكتب/يحدّث kubeconfig (cluster endpoint + certificate) ويضيف user يعتمد على exec plugin. الـ exec بينادي aws eks get-token وقت تنفيذ kubectl عشان يطلع token مؤقت مبني على IAM.

✓ نقاط تركّز عليها في الإجابة: (السبب/الاختيار/الـ trade-offs/أوامر التشخيص لو فيه مشكلة).

### س: إيه معنى authentication_mode = API_AND_CONFIG_MAP؟ وليه اخترته؟

**ج:** معناه EKS يقبل طريقتين لربط IAM بـ Kubernetes: Access Entries (API) و aws-auth ConfigMap. مفيد أثناء الانتقال/التوافق، لكن لازم حوكمة كويسة عشان ما يبقاش في صلاحيات مزدوجة غير مقصودة.

✓ نقاط تركّز عليها في الإجابة: (السبب/الاختيار/الـ trade-offs/أوامر التشخيص لو فيه مشكلة).

### س: إيه الفرق بين EKS Access Entries و aws-auth ConfigMap؟

**ج:** Access Entries: إدارة أحدث وأوضح عبر API/terraform، أسهل في التدقيق. aws-auth: طريقة قديمة لكنها واسعة الانتشار. الخطر فيها إن تعديل configmap غلط ممكن يقفل الإدارة أو يفتح صلاحيات زيادة.

✓ نقاط تركّز عليها في الإجابة: (السبب/الاختيار/الـ trade-offs/أوامر التشخيص لو فيه مشكلة).

### س: أقل صلاحيات (least privilege): Jenkins محتاج IAM permissions إيه بالظبط عشان deploy؟

**ج:** أقل صلاحيات عملية (Minimal working set) غالبًا تشمل:
- **STS**: `sts:GetCallerIdentity` (للتحقق من الهوية)
- **EKS**: `eks:DescribeCluster` (عشان `aws eks update-kubeconfig` يقدر يجيب endpoint و CA data)
- **ECR (Push Images)**: `ecr:GetAuthorizationToken` + `ecr:CreateRepository` (لو بتنشئ repo وقت التشغيل) +
  `ecr:BatchCheckLayerAvailability`, `ecr:InitiateLayerUpload`, `ecr:UploadLayerPart`, `ecr:CompleteLayerUpload`, `ecr:PutImage`
- **اختياري حسب شغلك**: `iam:PassRole` (لو فيه إنشاء موارد بتحتاج تمرير Role) أو صلاحيات ALB/ELB لو بتثبت/تدير LBC من الـ pipeline.

وبعد ده: لازم **صلاحيات Kubernetes RBAC** داخل الكلاستر (مثلاً السماح بـ `kubectl apply` على namespaces المطلوبة) وإلا هتشوف `Forbidden` حتى لو IAM تمام.

✓ نقاط تركّز عليها في الإجابة: (السبب/الاختيار/الـ trade-offs/أوامر التشخيص لو فيه مشكلة).

—————————————————————————————————————————————

## 5. تصميم Terraform والـ State

```
Terraform
  |
  | backend: S3 (state) + DynamoDB (lock)
  v
plan/apply
  |
  +--> drift? -> import / taint / state rm (بحذر)
```

### س: state بتاع Terraform متخزن فين؟ وليه مش local؟

**ج:** المثالي: S3 backend + DynamoDB lock. مش local لأن local يضيع/يتعارض بين الأجهزة ويخلي apply غير آمن في فريق أو CI.

✓ نقاط تركّز عليها في الإجابة: (السبب/الاختيار/الـ trade-offs/أوامر التشخيص لو فيه مشكلة).

### س: بتحل مشكلة drift إزاي؟ وإمتى تستخدم import أو state rm؟

**ج:** لو المورد موجود في AWS بس مش في state: تستخدم import عشان تتبناه. state rm تستخدمه بحذر لو عايز Terraform ينسى المورد (وأنت متأكد مش هيمسحه) أو لإعادة تبنّي/إعادة إنشاء بشكل مقصود.

✓ نقاط تركّز عليها في الإجابة: (السبب/الاختيار/الـ trade-offs/أوامر التشخيص لو فيه مشكلة).

### س: إيه اللي ممكن يحصل لو 2 pipelines عملوا terraform apply في نفس الوقت؟

**ج:** لو فيه locking سليم: واحد هيمسك lock والتاني هيفشل/يستنى. من غير lock: race conditions، state corruption، و drift صعب.

✓ نقاط تركّز عليها في الإجابة: (السبب/الاختيار/الـ trade-offs/أوامر التشخيص لو فيه مشكلة).

### س: بتقسّم Terraform modules إزاي (VPC/EKS/Jenkins/monitoring)؟

**ج:** تقسيم حسب domain: network module، eks module، jenkins module، monitoring module. الفايدة: إعادة استخدام، صيانة أسهل، و boundaries واضحة.

✓ نقاط تركّز عليها في الإجابة: (السبب/الاختيار/الـ trade-offs/أوامر التشخيص لو فيه مشكلة).

### س: إزاي بتدير secrets/variables في Terraform (زي passwords/tokens)؟

**ج:** متخزنهمش في state قدر الإمكان. استخدم AWS Secrets Manager/SSM Parameter Store. ولو لازم متغير حساس: mark sensitive + تشفير backend + تقليل الوصول لملف state.

✓ نقاط تركّز عليها في الإجابة: (السبب/الاختيار/الـ trade-offs/أوامر التشخيص لو فيه مشكلة).

### س: إيه خطتك لترقية EKS version بأمان باستخدام Terraform؟

**ج:** ترقية تدريجية: upgrade control plane، ثم add-ons (مثل CNI/CoreDNS/kube-proxy)، ثم node groups. اختبر على staging أولًا، وتأكد من توافق versions للـ controllers (خصوصًا LBC و CSI).

✓ نقاط تركّز عليها في الإجابة: (السبب/الاختيار/الـ trade-offs/أوامر التشخيص لو فيه مشكلة).

—————————————————————————————————————————————

## 6. Jenkins Pipeline (التجميعة والنشر)

```
Stages (typical)
1) Checkout
2) Build images (docker build)
3) ECR login + push
4) Configure kubeconfig (aws eks update-kubeconfig)
5) Deploy (kubectl apply)
6) Update images (kubectl set image) + rollout status
```

### س: ليه Jenkins على EC2 مش Managed CI؟ وإيه trade-offs؟

**ج:** EC2 يعطيك تحكم كامل في الأدوات والـ plugins والشبكة، ومناسب لو محتاج وصول private لـ EKS. العيب: صيانة (updates/backup/monitoring) وأمان (hardening) أعلى على عاتقك.

✓ نقاط تركّز عليها في الإجابة: (السبب/الاختيار/الـ trade-offs/أوامر التشخيص لو فيه مشكلة).

### س: بتدير credentials إزاي جوه Jenkins؟ وإزاي تمنع تسريب AWS keys؟

**ج:** الأفضل: IAM role على EC2 بدل keys. لو بتستخدم credentials store: mask في logs + أقل صلاحيات + rotate + ممنوع تكتبها في Jenkinsfile.

✓ نقاط تركّز عليها في الإجابة: (السبب/الاختيار/الـ trade-offs/أوامر التشخيص لو فيه مشكلة).

### س: الـ pipeline بيبني images إزاي؟ وإيه استراتيجية الـ caching؟

**ج:** docker build لكل خدمة (frontend/backend) مع Dockerfile مضبوط. caching عبر طبقات docker (إعادة استخدام layers) أو BuildKit، وممكن تحفظ cache على نفس agent أو registry cache.

✓ نقاط تركّز عليها في الإجابة: (السبب/الاختيار/الـ trade-offs/أوامر التشخيص لو فيه مشكلة).

### س: بتعمل tagging للصور إزاي؟ latest ولا commit SHA؟ وليه؟

**ج:** الأفضل commit SHA/build number للتتبع والـ rollback. latest ينفع للاختبار فقط لأنه يضيع traceability.

✓ نقاط تركّز عليها في الإجابة: (السبب/الاختيار/الـ trade-offs/أوامر التشخيص لو فيه مشكلة).

### س: لو push على ECR فشل—إيه اللي يحصل؟ وإزاي تمنع partial deploy؟

**ج:** الـ pipeline لازم يفشل قبل deploy (fail-fast). تعمل stages gates: لا deploy إلا بعد نجاح push لكلا الخدمتين.

✓ نقاط تركّز عليها في الإجابة: (السبب/الاختيار/الـ trade-offs/أوامر التشخيص لو فيه مشكلة).

### س: إزاي تعمل rollback سريع لو deploy كسر production؟

**ج:** إما kubectl rollout undo للـ deployment، أو تعيد set image لتاج أقدم معروف. الأهم: تكون محافظ على history وعلى tags غير قابلة للكتابة فوقها.

✓ نقاط تركّز عليها في الإجابة: (السبب/الاختيار/الـ trade-offs/أوامر التشخيص لو فيه مشكلة).

### س: ليه بتستخدم kubectl set image بعد kubectl apply؟

**ج:** apply يضمن الـ resources موجودة (Deployment/Service/Ingress). set image يسمح بتغيير image tag بسرعة بدون تعديل YAML كل مرة، ويحافظ على نفس manifest baseline.

✓ نقاط تركّز عليها في الإجابة: (السبب/الاختيار/الـ trade-offs/أوامر التشخيص لو فيه مشكلة).

—————————————————————————————————————————————


## 6.5 Helm (إدارة Charts على Kubernetes)

**ليه Helm موجود في مشروعك؟**
- لأن عندك مكوّنات *Add-ons* على الكلاستر (زي **AWS Load Balancer Controller** و **kube-prometheus-stack**) بتتثبت عادةً كـ **Helm charts**.
- Helm بيديك install/upgrade/rollback بشكل **منظّم** بدل ما تكون YAMLs كتير منفصلة.

**إزاي بيظهر في شغلك عمليًا؟ (أمثلة أوامر)**
```bash
# تثبيت/تحديث Chart (الـ pattern الأشهر)
helm repo add eks https://aws.github.io/eks-charts
helm repo update
helm upgrade --install aws-load-balancer-controller eks/aws-load-balancer-controller \
  -n kube-system --create-namespace \
  --set clusterName=depi-eks \
  --set serviceAccount.create=false \
  --set serviceAccount.name=aws-load-balancer-controller

# متابعة الـ releases
helm -n kube-system list
helm -n monitoring list

# حذف (لو محتاج cleanup)
helm -n kube-system uninstall aws-load-balancer-controller
```

**نقطة مناقشة مهمة**
- Helm **مش بديل لـ Terraform**: Terraform بيدير AWS infra و backend/state.
- Helm بيدير **حزم Kubernetes** (charts) وده مناسب جدًا للـ add-ons.
- لو بتستخدم Terraform Helm Provider، يبقى Terraform هو اللي بيشغّل Helm (بس الفكرة واحدة).


## 7. Docker ونظافة الصور (Image Hygiene)

```
Dockerfile (best practice)
multi-stage build
  build stage -> compile
  runtime stage -> minimal image
scan -> push to ECR
```

### س: Multi-stage build: بتستخدمه؟ وليه مهم في frontend/backend؟

**ج:** مهم لأنه يقلل حجم الصورة ويشيل أدوات البناء من runtime، وده يحسن الأمن والسرعة.

✓ نقاط تركّز عليها في الإجابة: (السبب/الاختيار/الـ trade-offs/أوامر التشخيص لو فيه مشكلة).

### س: بتفحص الصور security scanning ازاي؟ (ECR scan/Trivy)

**ج:** تفعّل ECR image scanning أو تشغل Trivy في pipeline كـ gate قبل deploy. أي CVE عالية تخلي pipeline يفشل أو يطلع warning حسب policy.

✓ نقاط تركّز عليها في الإجابة: (السبب/الاختيار/الـ trade-offs/أوامر التشخيص لو فيه مشكلة).

### س: بتتعامل إزاي مع build-time secrets (npm token مثلًا) بدون ما تدخل في image؟

**ج:** استخدم BuildKit secrets أو CI secret mount، وتأكد إن الـ token مش بيتكتب في layer أو logs. ممنوع تحطه ENV ثابت داخل Dockerfile.

✓ نقاط تركّز عليها في الإجابة: (السبب/الاختيار/الـ trade-offs/أوامر التشخيص لو فيه مشكلة).

—————————————————————————————————————————————

## 8. ECR وإدارة الريجستري (Registry Ops)

```
Jenkins -> (docker push) -> ECR
Nodes  -> (docker pull) -> ECR
Tags: <commitSHA> (recommended) + optional latest
Lifecycle policy -> cleanup old images
```

### س: لو الـ ECR repo مش موجود—بتعمله إزاي في pipeline؟ وليه ده مش Terraform؟

**ج:** عمليًا ممكن تعمل create repo في pipeline لتقليل الخطوات اليدوية. لكن IaC purists يفضلوا Terraform عشان يبقى كل شيء declarative. الاتنين ينفع، المهم consistency.

✓ نقاط تركّز عليها في الإجابة: (السبب/الاختيار/الـ trade-offs/أوامر التشخيص لو فيه مشكلة).

### س: إزاي تمنع إن tag يتكتب عليه (immutability)؟

**ج:** فعّل ECR Tag Immutability بحيث أي tag ما يتستبدلش. ده يحمي الـ rollback ويمنع نشر غير مقصود.

✓ نقاط تركّز عليها في الإجابة: (السبب/الاختيار/الـ trade-offs/أوامر التشخيص لو فيه مشكلة).

### س: Lifecycle policies: بتعمل cleanup للـ old images إزاي؟

**ج:** تحط policy تحذف untagged بعد X أيام، وتحتفظ بآخر N tags لكل repo. ده يقلل التكلفة ويحافظ على rollback windows.

✓ نقاط تركّز عليها في الإجابة: (السبب/الاختيار/الـ trade-offs/أوامر التشخيص لو فيه مشكلة).

—————————————————————————————————————————————

## 9. نشر Kubernetes (K8s Deploy)

```
kubectl apply -f k8s/
   |
   v
EKS API Server stores desired state
   |
Scheduler -> chooses node
   |
kubelet pulls image from ECR (443) and runs Pod
```

### س: اشرح Deployment vs StatefulSet—إيه اللي مناسب للـ Mongo؟ وليه؟

**ج:** Deployment مناسب للـ stateless apps. StatefulSet مناسب للـ stateful مثل Mongo لأنه يعطي هوية ثابتة للـ Pod ويدير PVCs بشكل مرتب. في الديمو ممكن تشغل Mongo كـ Deployment، لكن للإنتاج StatefulSet أفضل.

✓ نقاط تركّز عليها في الإجابة: (السبب/الاختيار/الـ trade-offs/أوامر التشخيص لو فيه مشكلة).

### س: إيه readiness/liveness probes اللي حاططها؟ وليه؟

**ج:** readiness تمنع إرسال traffic قبل ما الخدمة تبقى جاهزة. liveness تعيد تشغيل الـ pod لو hung. بدونهم ممكن ALB يبعث traffic لنسخة مش جاهزة أو pod يفضل معلق.

✓ نقاط تركّز عليها في الإجابة: (السبب/الاختيار/الـ trade-offs/أوامر التشخيص لو فيه مشكلة).

### س: Requests/Limits: بتحطهم إزاي؟ وإيه اللي يحصل لو ماحطيتش؟

**ج:** تحط requests لضمان scheduling عادل، وlimits لمنع pod ياكل موارد ويوقع node. بدونها: scheduling غير متوقع و OOM kills ممكن تزيد.

✓ نقاط تركّز عليها في الإجابة: (السبب/الاختيار/الـ trade-offs/أوامر التشخيص لو فيه مشكلة).

### س: RollingUpdate strategy عندك إيه؟ وبتضمن zero downtime إزاي؟

**ج:** RollingUpdate مع maxUnavailable=0 (أو صغير) و readiness probes. وكمان ALB health check مرتبط بـ readiness path عشان ما يمررش traffic إلا للجاهز.

✓ نقاط تركّز عليها في الإجابة: (السبب/الاختيار/الـ trade-offs/أوامر التشخيص لو فيه مشكلة).

### س: Secrets & ConfigMaps: بتفصل config عن code إزاي؟

**ج:** config غير حساس في ConfigMap، والحساس في Secret. وتتجنب hardcoding داخل image أو YAML ثابت.

✓ نقاط تركّز عليها في الإجابة: (السبب/الاختيار/الـ trade-offs/أوامر التشخيص لو فيه مشكلة).

### س: Namespace strategy: شغال default ولا عامل namespaces؟ وليه؟

**ج:** للـ demo ممكن default. للأفضل: namespace لكل بيئة/تطبيق + RBAC scope أسهل + فصل موارد واضح.

✓ نقاط تركّز عليها في الإجابة: (السبب/الاختيار/الـ trade-offs/أوامر التشخيص لو فيه مشكلة).

—————————————————————————————————————————————

## 10. Seed Job وخطوات ما بعد النشر (Mongo Seed)

```
Deploy app + mongo
  |
  v
Run one-time Job (seed)
  |
  v
Validate data exists
(Important: prevent re-seeding on every deploy)
```

### س: ليه اخترت Job للـ seed بدل ما يكون جزء من app startup؟

**ج:** الـ seed خطوة one-time. لو حطيته في startup ممكن يتكرر مع كل restart/scale ويعمل duplicate أو يبوّظ البيانات. Job منفصل سهل مراقبته وإعادته عند الحاجة.

✓ نقاط تركّز عليها في الإجابة: (السبب/الاختيار/الـ trade-offs/أوامر التشخيص لو فيه مشكلة).

### س: إزاي تمنع seed يتكرر كل deploy ويبوّظ البيانات؟

**ج:** تخلي الـ Job idempotent (يتأكد هل البيانات موجودة قبل الإدخال) أو تستخدم اسم job unique وتتحقق من completion. وكمان تربط تشغيله بشرط (أول مرة فقط) أو manual gate في pipeline.

✓ نقاط تركّز عليها في الإجابة: (السبب/الاختيار/الـ trade-offs/أوامر التشخيص لو فيه مشكلة).

### س: إزاي بتراقب نجاح الـ Job؟ وإيه الإجراء لو فشل؟

**ج:** kubectl get jobs/pods + kubectl logs لبوود الـ job. لو فشل: راجع env/secrets واتصالات DB، ثم delete job وأعد تشغيله بعد الإصلاح.

✓ نقاط تركّز عليها في الإجابة: (السبب/الاختيار/الـ trade-offs/أوامر التشخيص لو فيه مشكلة).

### س: لو الـ seed محتاج secrets: بتحطها فين وبأي شكل؟

**ج:** تحطها في Kubernetes Secret وتعمل mount كـ env vars للـ job. تجنب وضعها داخل image أو في Jenkins logs.

✓ نقاط تركّز عليها في الإجابة: (السبب/الاختيار/الـ trade-offs/أوامر التشخيص لو فيه مشكلة).

—————————————————————————————————————————————

## 11. Ingress و AWS Load Balancer Controller (LBC) — (عندك: Internet-facing ALB)

> **الوضع الحقيقي في مشروعك الآن:**  
> - الـ Ingress بيتحوّل إلى **ALB Internet-facing (Public)**.  
> - الدخول من **Internet → ALB (80/443)** ثم يروح لـ **Service/Pods** داخل الـ VPC.

```text
Internet Users
   |
   |  80/443
   v
ALB (internet-facing)  [AWS]
   |
   |  HTTP to targets (Service targetPort)
   v
EKS Worker Nodes (private subnets)
   |
   v
K8s Service -> Pods (frontend/backend)
```

### س: اشرح إزاي Ingress بيتحول لـ ALB على AWS. مين بيعمل إيه؟

**ج:**  
- أنت بتعمل `kubectl apply` لملف الـ Ingress → object يتسجّل في Kubernetes API.  
- **AWS Load Balancer Controller** بيراقب (watch) الـ Ingress/Service ويستدعي AWS APIs لإنشاء:
  - ALB + Listeners + Rules
  - Target Groups + Health checks
  - Security Groups (حسب الإعدادات)
- بعدها الـ ALB يوجّه الـ traffic للبورت الصحيح على الـ Targets (غالبًا nodes/NodePort أو IP targets حسب الـ config).

✓ نقاط تركّز عليها: (controller watch → AWS APIs → ALB resources + rules + health checks).

### س: لما Ingress يبقى “ADDRESS فاضي”—أشهر 3 أسباب عندك؟

**ج:**  
1) **LBC مش متثبت أو مش Running** (وده حصل عندك قبل كده فعلًا).  
2) **IAM permissions/IRSA ناقصة** للـ controller (مش قادر ينشئ ALB/TG/SG).  
3) **Subnet tags غلط/ناقصة** أو الـ scheme غلط (مش قادر يختار subnets أو يعمل internet-facing).

**تشخيص سريع**
```bash
kubectl get ingress
kubectl describe ingress <name>
kubectl -n kube-system get deploy aws-load-balancer-controller
kubectl -n kube-system logs deploy/aws-load-balancer-controller | tail -n 200
```

✓ نقاط تركّز عليها: (controller exists? IAM? subnet tags? events/logs).

### س: أنت عندك Internet-facing ولا Internal ALB؟ وإزاي بتحدد ده؟

**ج:** عندك **Internet-facing**. بتحدده من:
- Ingress annotation:
  - `alb.ingress.kubernetes.io/scheme: internet-facing`
- والـ Subnets اللي هيتحط فيها ALB لازم تكون **Public subnets** وموسومة بـ:
  - `kubernetes.io/role/elb=1`
  - + tag الكلاستر `kubernetes.io/cluster/depi-eks=shared/owned`

**معلومة للمناقشة:** لو كان Internal كنت هتستخدم:
- `alb.ingress.kubernetes.io/scheme: internal`
- و tag `kubernetes.io/role/internal-elb=1` على **Private subnets**.

### س: LBC محتاج IAM permissions إيه؟ وبتديهاله إزاي؟

**ج:** محتاج صلاحيات لإنشاء/تعديل:
- ALB, Target Groups, Listeners/Rules
- Security Groups (EC2)
أفضل طريقة: **IRSA** (ServiceAccount مرتبط بـ IAM Role) عشان الصلاحيات تكون دقيقة، بدل ما تدي الـ Node role صلاحيات واسعة.

✓ نقاط تركّز عليها: (IRSA = least privilege + auditing أسهل).

### س: Target group health checks بتتحدد منين؟ وإزاي تربطها بالـ readiness؟

**ج:** بتتحدد من annotations أو defaults. أفضل ممارسة:
- Health check path = نفس endpoint اللي الـ readiness probe بيستخدمه (مثلاً `/health`).
- كده ALB مايبعتش traffic إلا للنسخ الجاهزة فعلاً.

✓ نقاط تركّز عليها: (health checks لازم تعكس readiness وإلا هتشوف 5xx/unhealthy targets).

### س: لو عندك 2 Ingress على نفس host/path—بيحصل إيه؟

**ج:** يحصل تضارب في الـ rules أو behavior غير متوقع. الأفضل:
- دمج القواعد في Ingress واحد، أو
- تقسيم host/path بوضوح.
وفي AWS LBC تقدر تستخدم **IngressGroup** (group.name) لجمع ingresses في ALB واحد بشكل منظم بدل الصدام.

✓ نقاط تركّز عليها: (conflicts → events/logs، والحل grouping أو توحيد القواعد).


## 12. التخزين (Mongo + PVC)

```
Pod (mongo)
  |
  v
PVC -> StorageClass (gp3)
  |
  v
EBS CSI Driver provisions EBS volume
  |
  v
Volume attached to Node (AZ must match)
```

### س: لما PVC يبقى Pending على EKS—إزاي تشخّصها خطوة خطوة؟

**ج:** 1) kubectl describe pvc وشوف events. 2) تأكد StorageClass موجود و default أو محدد. 3) تأكد EBS CSI driver شغال. 4) راجع volume binding mode و AZ mismatch. 5) راجع IAM permissions للـ CSI لو بيستخدم IRSA.

✓ نقاط تركّز عليها في الإجابة: (السبب/الاختيار/الـ trade-offs/أوامر التشخيص لو فيه مشكلة).

### س: ليه استخدمت EBS CSI + gp3 StorageClass؟

**ج:** EBS CSI هو driver الرسمي لإدارة EBS على Kubernetes. gp3 أفضل من gp2 في الأداء/السعر ويديك تحكم في IOPS/throughput.

✓ نقاط تركّز عليها في الإجابة: (السبب/الاختيار/الـ trade-offs/أوامر التشخيص لو فيه مشكلة).

### س: إيه الفرق بين WaitForFirstConsumer و Immediate في volume binding؟

**ج:** Immediate بيحجز volume فورًا وقد يختار AZ قبل ما pod يتحدد. WaitForFirstConsumer ينتظر scheduling للـ pod ثم ينشئ volume في نفس AZ للنود، وده يقلل AZ mismatch.

✓ نقاط تركّز عليها في الإجابة: (السبب/الاختيار/الـ trade-offs/أوامر التشخيص لو فيه مشكلة).

### س: Backup/restore للـ Mongo على EKS: خطتك إيه؟

**ج:** Backup منطقي: mongodump + تخزين في S3. Backup على مستوى block: EBS snapshots (مع مراعاة consistency). للإنتاج: schedule + اختبار restore دوري.

✓ نقاط تركّز عليها في الإجابة: (السبب/الاختيار/الـ trade-offs/أوامر التشخيص لو فيه مشكلة).

—————————————————————————————————————————————

## 13. المراقبة والرصد (Monitoring/Observability)

```
Prometheus -> scrapes metrics (nodes/pods/kube-state)
   |
   v
Grafana -> dashboards
   |
   v
Alerts (Alertmanager) -> notifications
```

### س: إيه اللي بيراقبه kube-prometheus-stack بالظبط؟

**ج:** Metrics على مستوى node (CPU/mem/disk/network)، وعلى مستوى kube-system (API server/etcd إن وجد)، وكمان kube-state-metrics لحالة objects (deployments/pods).

✓ نقاط تركّز عليها في الإجابة: (السبب/الاختيار/الـ trade-offs/أوامر التشخيص لو فيه مشكلة).

### س: إزاي بتطلع Grafana admin password من secret؟ وليه base64؟

**ج:** Kubernetes secrets محفوظة كـ base64 encoding (مش تشفير). فتقرأ الـ secret ثم تعمل base64 decode للحصول على الباسورد.

✓ نقاط تركّز عليها في الإجابة: (السبب/الاختيار/الـ trade-offs/أوامر التشخيص لو فيه مشكلة).

### س: Alerts: هتنبه على إيه كـ minimum viable monitoring؟

**ج:** Node NotReady، Pod CrashLoop/Restart spikes، CPU/mem saturation، 5xx rate على ALB/ingress، Targets unhealthy.

✓ نقاط تركّز عليها في الإجابة: (السبب/الاختيار/الـ trade-offs/أوامر التشخيص لو فيه مشكلة).

### س: Logs: بتجمع logs إزاي؟ (CloudWatch/ELK) ولو مش موجود—ليه؟

**ج:** ممكن تستخدم CloudWatch agent/Fluent Bit أو EFK stack. لو مش موجود: غالبًا لتقليل التعقيد والتكلفة في demo؛ لكن للإنتاج لازم logging مركزي.

✓ نقاط تركّز عليها في الإجابة: (السبب/الاختيار/الـ trade-offs/أوامر التشخيص لو فيه مشكلة).

—————————————————————————————————————————————

## 14. الأمان و Hardening

```
Where secrets leak:
Jenkins logs -> (bad)
kubeconfig files -> (protect)
Terraform state -> (encrypt + restrict)
RBAC -> limit permissions
```

### س: إيه أخطر 3 نقاط تسريب secrets في مشروعك؟ وإزاي قفلتها؟

**ج:** 1) Jenkins logs: استخدم masking ومنع echo للـ secrets. 2) kubeconfig على Jenkins EC2: صلاحيات ملفات ضيقة وعدم مشاركتها. 3) Terraform state: backend مشفر + IAM policies تقلل الوصول + تجنب تخزين secrets أصلاً.

✓ نقاط تركّز عليها في الإجابة: (السبب/الاختيار/الـ trade-offs/أوامر التشخيص لو فيه مشكلة).

### س: RBAC: مين يقدر يعمل kubectl apply؟ ومين يقدر يقرأ secrets؟

**ج:** المفروض Jenkins role/serviceaccount يقدر يطبق resources في namespace محدد، لكن ميكونش له read للـ secrets إلا لو لازم. Cluster-admin يتجنب إلا للإدارة فقط.

✓ نقاط تركّز عليها في الإجابة: (السبب/الاختيار/الـ trade-offs/أوامر التشخيص لو فيه مشكلة).

### س: Network policies: بتستخدمها؟ ولو لا—إيه المخاطر؟

**ج:** لو مش مستخدمة: أي pod ممكن يكلم أي pod (east-west). ده يزيد مساحة الهجوم. NetworkPolicies تقلل الحركة وتفصل الطبقات (frontend لا يكلم إلا backend… إلخ).

✓ نقاط تركّز عليها في الإجابة: (السبب/الاختيار/الـ trade-offs/أوامر التشخيص لو فيه مشكلة).

—————————————————————————————————————————————

## 15. الاعتمادية و Rollback و Disaster Recovery

```
Deploy tag: <commitSHA>
  |
  v
If broken:
- rollback to previous tag
- or kubectl rollout undo
Disaster:
- rebuild via Terraform
- restore DB backups
```

### س: لو deployment باظ بعد ما اتنشر—هتعمل rollback إزاي وبأي tag؟

**ج:** Rollback سريع: kubectl rollout undo. أو إعادة set image لتاج سابق (commit SHA/build number). لازم تكون محافظ على تاريخ التاجات وما فيش tag بيتكتب فوقه.

✓ نقاط تركّز عليها في الإجابة: (السبب/الاختيار/الـ trade-offs/أوامر التشخيص لو فيه مشكلة).

### س: لو region وقعت/اتمسحت resources بالغلط—إيه recovery plan؟

**ج:** إعادة بناء البنية عبر Terraform (IaC) + استرجاع البيانات من backups (Mongo) + إعادة بناء images من الـ registry أو إعادة build.

✓ نقاط تركّز عليها في الإجابة: (السبب/الاختيار/الـ trade-offs/أوامر التشخيص لو فيه مشكلة).

### س: إزاي تمنع “نشر نسخة غلط” (مثلا frontend جديد مع backend قديم)؟

**ج:** تستخدم release identifier واحد (build number) للـ services الاتنين في نفس pipeline. وتنشرهم كـ atomic release: لو push لأي واحدة فشل، توقف النشر. وممكن تضيف compatibility checks أو contract tests قبل deploy.

✓ نقاط تركّز عليها في الإجابة: (السبب/الاختيار/الـ trade-offs/أوامر التشخيص لو فيه مشكلة).

—————————————————————————————————————————————

## 16. التكلفة والتشغيل (Cost & Operations)

```
Common cost drivers:
NAT gateway(s) + ALB + Node sizes + EBS volumes + Monitoring stack
Cost control:
- right-size nodes
- lifecycle images
- VPC endpoints to reduce NAT
- autoscaling
```

### س: في تصميمك إيه وازاي تقللهم؟ cost drivers أكبر؟

**ج:** الأغلب: NAT gateways، ALB، حجم/عدد الـ nodes، EBS، ومونيتورنج (storage/retention). تقليل: right-sizing، VPC endpoints بدل NAT لبعض الترافيك، تقليل retention، و autoscaling.

✓ نقاط تركّز عليها في الإجابة: (السبب/الاختيار/الـ trade-offs/أوامر التشخيص لو فيه مشكلة).

### س: Auto-scaling: بتستخدم Cluster Autoscaler/Karpenter؟ وليه/ليه لا؟

**ج:** ينفع تضيف Cluster Autoscaler لو الحمل متغير لتقليل التكلفة. لو المشروع demo أو ثابت، ممكن تتجنب التعقيد. Karpenter أكثر مرونة لكن يحتاج ضبط وسياسات.

✓ نقاط تركّز عليها في الإجابة: (السبب/الاختيار/الـ trade-offs/أوامر التشخيص لو فيه مشكلة).

—————————————————————————————————————————————

## 17. Troubleshooting (تشخيص عملي)

```
Quick triage
kubectl get pods -A
kubectl describe pod <name>
kubectl logs <pod> --previous
kubectl get events -A --sort-by=.lastTimestamp
```

### س: ImagePullBackOff: تشخّصها إزاي؟ (repo/tag/permissions/network)

**ج:** تحقق من: image name/tag صحيح، repo موجود، node عنده permissions يسحب من ECR، والـ node عنده outbound (NAT/endpoints). ثم راجع describe pod للأخطاء التفصيلية.

✓ نقاط تركّز عليها في الإجابة: (السبب/الاختيار/الـ trade-offs/أوامر التشخيص لو فيه مشكلة).

### س: Pod CrashLoopBackOff: تعمل إيه أول 5 دقايق؟

**ج:** kubectl logs + kubectl describe + events. تأكد من env vars/secrets، ports، readiness/liveness، و resources (OOMKilled).

✓ نقاط تركّز عليها في الإجابة: (السبب/الاختيار/الـ trade-offs/أوامر التشخيص لو فيه مشكلة).

### س: Service شغال بس مفيش traffic من ALB—تتأكد من إيه؟

**ج:** تأكد من Ingress annotations، إن LBC شغال، target group healthy، health check path صحيح، والـ Service port/targetPort متطابق.

✓ نقاط تركّز عليها في الإجابة: (السبب/الاختيار/الـ trade-offs/أوامر التشخيص لو فيه مشكلة).

### س: Mongo شغال بس app مش قادر يتصل—هل المشكلة DNS ولا Network ولا creds؟

**ج:** ابدأ بـ DNS: nslookup/service name داخل pod. ثم network: هل port مفتوح؟ ثم creds: secret/env صحيح. واختم بالـ logs في backend + mongo logs.

✓ نقاط تركّز عليها في الإجابة: (السبب/الاختيار/الـ trade-offs/أوامر التشخيص لو فيه مشكلة).

## Advanced Discussion Questions

> القسم ده معمول للمناقشة/الانترفيو: **أسئلة زيادة “تقيلة”** مرتبطة مباشرة بهيكلة مشروعك وأدواتك.
> لكل سؤال: **إجابة نموذجية مختصرة (3 سطور)** + أحيانًا **ASCII** يوضح الفكرة.

### CI/CD وعمليات النشر

**س1) إزاي بتضمن إن الـ Pipeline عندك Idempotent (لو اتشغّل مرتين ما يبوّظش الدنيا)؟**  
**إجابة نموذجية (مختصرة):**
- بفصل الـ *build/push* عن *deploy* وبخلي deploy يعتمد على نفس الـ tag (مش latest عشوائي).  
- أي install/addon بيتعمل بطريقة “install-or-upgrade” ومش “create كل مرة”.  
- بحط checks قبل التنفيذ (وجود repo/ingress/namespace) وبمنع steps تسيب partial state.

```text
git push
  |
  v
[Build] -> [Push ECR] -> [Deploy k8s] -> [Verify Rollout]
  |            |             |               |
  |            |             |               +--> succeeds OR fails safely (no partial)
  |            |             +--> idempotent apply + set image
  |            +--> deterministic tags (sha/build)
  +--> repeatable build steps
```

**س2) إيه الـ Quality Gates اللي ينفع تضيفها قبل النشر في نفس Jenkinsfile؟**  
**إجابة نموذجية (مختصرة):**
- Tests + lint قبل docker build، وبعدها image scan (ECR/Trivy) قبل push أو قبل deploy.  
- Manifest validation: `kubectl apply --dry-run=client` و/أو `kubeconform`.  
- Gate على rollout: لو `kubectl rollout status` فشل → stop + rollback.

**س3) لو deploy فشل في النص: هتعمل rollback إزاي “بالضبط”؟**  
**إجابة نموذجية (مختصرة):**
- أولاً: أثبّت إن الفشل من app ولا من infra (events/logs/rollout status).  
- Rollback سريع: `kubectl rollout undo deployment/<name>` أو pin image tag السابق.  
- لازم tagging يكون traceable (commit SHA/build) عشان ترجع لنقطة معروفة.

**س4) إزاي تمنع تكرار الـ seed / migrations كل deploy؟**  
**إجابة نموذجية (مختصرة):**
- سيب seed كـ Job منفصل له “marker” (مثلاً ConfigMap/DB flag) يمنع تكراره.  
- شغّله conditionally (مرة واحدة أو عند version معين).  
- لو لازم يتكرر: خليّه idempotent (upsert بدل insert أعمى).

---

### Terraform وState وإدارة التغييرات

**س5) Terraform state عندك Remote ولا Local؟ وليه ده مهم في مشروع فيه CI؟**  
**إجابة نموذجية (مختصرة):**
- Remote state (S3) + lock (DynamoDB) يمنع `apply` المتوازي ويقلل race conditions.  
- Local state خطير مع CI لأنه بيتكسر بسهولة مع تغيّر الماشينات.  
- Remote state بيسهّل التعاون واسترجاع الحالة لو السيرفر اتغير.

**س6) Drift حصل في monitoring/ingress: إمتى تعمل import وإمتى recreate؟**  
**إجابة نموذجية (مختصرة):**
- Import لو resource “صح” على AWS/K8s لكن Terraform فقد تتبّعه (state mismatch).  
- Recreate لو resource غلط أو فيه إعدادات unsafe/غير قابلة للتعديل.  
- قبل القرار: قارن impact على downtime (ALB/ingress حساس).

**س7) إزاي تمنع `terraform apply` يتنفذ من مكانين؟**  
**إجابة نموذجية (مختصرة):**
- Locking في backend + سياسات CI تمنع parallel apply على نفس env.  
- Environments منفصلة (state per env) + branches أو approvals.  
- Runbook واضح: “terraform changes من pipeline واحد فقط”.

---

### IAM + EKS AuthN/AuthZ

**س8) اشرح فرق Authentication وAuthorization في EKS “في مشروعك”.**  
**إجابة نموذجية (مختصرة):**
- Authentication: Jenkins EC2 يثبت هويته بـ IAM token (AWS CLI exec).  
- Authorization: داخل الكلاستر RBAC يحدد يقدر يعمل إيه (apply/get/secret…).  
- فشل AuthN غالبًا “Unauthorized”، وفشل AuthZ غالبًا “Forbidden”.

```text
Jenkins EC2 (IAM Role)
    |
    |  AuthN (token via aws eks get-token)
    v
EKS API Server
    |
    |  AuthZ (RBAC: Roles/RoleBindings)
    v
Allowed?  -> apply/patch/rollout ...  OR  Forbidden
```

**س9) Jenkins بيستخدم IAM Role ولا Access Keys؟ وإيه القرار الصح؟**  
**إجابة نموذجية (مختصرة):**
- IAM Role على EC2 (Instance Profile) أفضل: rotation تلقائي وأقل risk تسريب.  
- Keys داخل Jenkins credentials شغالة لكن أخطر (leak/rotation/manual).  
- القرار الصح: role + least privilege + auditing.

**س10) Least privilege: إيه أقل Permissions من AWS غالبًا محتاجها للتشغيل؟**  
**إجابة نموذجية (مختصرة):**
- EKS: `DescribeCluster` + `List/Describe` حسب استخدامك، وSTS للهوية.  
- ECR: login + create repo (لو بتعمله runtime) + push/pull.  
- لو بتثبت LBC: permissions خاصة بالـ ELB/EC2/IAM (أو عبر IRSA).

---

### Ingress + AWS Load Balancer Controller (LBC)

**س11) Ingress بيتحول لـ ALB إزاي؟ مين بيعمل إيه؟**  
**إجابة نموذجية (مختصرة):**
- أنت بتعمل `kubectl apply ingress.yaml` → الـ Ingress object يتسجل في API.  
- LBC يشوفه (watch) ويستدعي AWS APIs عشان ينشئ ALB + rules + target groups.  
- Health checks + listeners تتظبط بناءً على annotations/config.

```text
kubectl apply Ingress
       |
       v
K8s API stores desired state
       |
       v
LBC watches Ingress  ---> AWS APIs ---> ALB + TG + Rules
       |
       v
Targets = NodePort/Pod IP (حسب النوع) + health checks
```

**س12) لو Ingress بدون ADDRESS: إيه Checklist سريع؟**  
**إجابة نموذجية (مختصرة):**
- هل LBC موجود Running؟ وهل عنده IAM permissions كفاية؟  
- هل subnets عليها tags المطلوبة للـ ALB؟ (public/private حسب التصميم)  
- هل فيه conflict في IngressClass/annotations أو rules متعارضة؟

**س13) IRSA ولا Node Role للـ LBC؟ وليه؟**  
**إجابة نموذجية (مختصرة):**
- IRSA أفضل: صلاحيات دقيقة على ServiceAccount بدل كل النودز.  
- Node role أسهل لكنه “broad permissions” ومخاطره أعلى.  
- في بيئة production: IRSA عادة هو المعيار.

---

### Storage + Mongo (PVC/SC/EBS CSI)

**س14) PVC Pending: تشخصها خطوة بخطوة إزاي؟**  
**إجابة نموذجية (مختصرة):**
- شوف Events على الـ PVC/Pod: `kubectl describe pvc/pod`.  
- اتأكد من StorageClass وCSI driver شغالين، وbinding mode مناسب.  
- راجع الـ AZ/subnets: EBS volume لازم يتخلق في AZ فيه node.

```text
Pod -> PVC (Pending?)
        |
        v
StorageClass -> CSI Driver -> EBS Volume (AZ must match node)
```

**س15) ليه gp3 + EBS CSI؟**  
**إجابة نموذجية (مختصرة):**
- gp3 غالبًا أفضل في الأداء/السعر وبيسمح بتخصيص IOPS/throughput.  
- CSI driver هو الطريقة الحديثة المدعومة لإدارة volumes على EKS.  
- بيحل مشاكل dynamic provisioning بدل manual volumes.

**س16) Backup/restore للـ Mongo على EKS: تعمل إيه؟**  
**إجابة نموذجية (مختصرة):**
- Snapshot على EBS كويس كـ DR سريع، لكن لازم consistency strategy.  
- Logical backup (mongodump/restore) مهم للـ point-in-time على مستوى البيانات.  
- اكتب runbook واضح: frequency + retention + restore test.

---

### Monitoring & Observability

**س17) kube-prometheus-stack بيراقب إيه عمليًا؟ وإيه أول 5 Alerts؟**  
**إجابة نموذجية (مختصرة):**
- Metrics للنودز والبودز (CPU/Mem), وkube-state-metrics لحالة objects.  
- Alerts: Node not ready، Pod crashloop، High error rate، ALB unhealthy targets، Disk pressure.  
- المهم: alerts قابلة للتنفيذ مش noise.

**س18) Grafana admin password جوه secret: ليه base64؟**  
**إجابة نموذجية (مختصرة):**
- Kubernetes secrets غالبًا “encoded” بـ base64 مش “encrypted” افتراضيًا.  
- base64 ده تنسيق نقل، مش حماية.  
- الحماية الحقيقية: RBAC + encryption at rest (لو متفعل) + external secret manager.

```text
Secret (base64) != Encryption
RBAC + Encryption-at-rest + Audit = real protection
```

**س19) Logs مش متجمعة: ده هيأثر إزاي؟ وإيه أقل حل؟**  
**إجابة نموذجية (مختصرة):**
- Troubleshooting هيبقى بطيء لأنك هتلف على pods/nodes يدوي.  
- أقل حل: CloudWatch Container Insights أو Fluent Bit إلى CloudWatch.  
- بعد كده: ELK/OpenSearch لو احتجت بحث وتحليلات أعمق.

---

### Security & Hardening

**س20) أخطر 3 نقاط تسريب secrets في تصميمك؟**  
**إجابة نموذجية (مختصرة):**
- Jenkins console logs (لو اتطبع token/creds بالغلط).  
- Terraform state (ممكن يخزن values حساسة).  
- kubeconfig على Jenkins EC2 (لو اتسحب/اتنسخ).

**س21) RBAC: Jenkins يقدر يعمل إيه؟ وإزاي تقلل المخاطر؟**  
**إجابة نموذجية (مختصرة):**
- ادّيه permissions للـ namespaces المطلوبة فقط، وامنعه من قراءة secrets لو مش لازم.  
- استخدم ServiceAccount/RoleBinding محدد بدل cluster-admin.  
- Audit logs + review دوري للصلاحيات.

**س22) NetworkPolicies: لو مش موجودة… إيه الخطر الحقيقي؟**  
**إجابة نموذجية (مختصرة):**
- أي pod ممكن يكلم أي pod/service داخليًا (east-west unrestricted).  
- ده بيزود blast radius لو خدمة اتاخترقت.  
- الأقل: policies تمنع الوصول للـ DB إلا من backend فقط.

---

### Reliability / Rollback / Cost

**س23) إزاي تمنع “نشر نسخة غلط” (frontend جديد مع backend قديم)؟**  
**إجابة نموذجية (مختصرة):**
- Versioning واضح + pin tags (SHA) + deploy نفس الـ release bundle.  
- Gate يراجع compatibility (contract tests أو minimum API version).  
- Prefer “single pipeline release” يطلع version واحد للاتنين.

**س24) أكبر cost drivers غالبًا في تصميمك؟ وإزاي تقلل؟**  
**إجابة نموذجية (مختصرة):**
- NAT + ALB + node sizes + monitoring storage/retention.  
- قلل عبر right-sizing + spot (بحذر) + lifecycle للـ images/metrics.  
- راقب usage بالأرقام قبل ما تقلل (metrics-driven cost cutting).

**س25) Autoscaling: إمتى تضيف HPA/Cluster Autoscaler؟**  
**إجابة نموذجية (مختصرة):**
- لو load متغير وبتشوف saturation على CPU/Mem أو latency.  
- HPA للتطبيق، وCluster Autoscaler/Karpenter للنودز.  
- لازم limits/requests تكون مظبوطة الأول وإلا scaling هيبقى عشوائي.

---

### Helm (متقدم) — أسئلة مربوطة بمشروعك فعلًا

**س26) ليه استخدمت Helm في الـ LBC و kube-prometheus-stack بدل kubectl apply؟**  
**إجابة نموذجية (مختصرة):**
- Helm مناسب للـ *packaged apps* اللي فيها templates + values كثيرة (زي controllers و monitoring stacks).  
- بيديك versioning/upgrade/rollback واضح (`helm upgrade --install`, `helm rollback`).  
- أسهل في إدارة dependencies و parameters مقارنة بـ YAMLs كبيرة يدويًا.

**س27) إزاي بتضمن إن Helm releases Idempotent ومش بتعمل “duplicate resources”؟**  
**إجابة نموذجية (مختصرة):**
- دايمًا استخدم `helm upgrade --install` بنفس **release name** ونفس **namespace**.  
- ثبّت chart version (pin) بدل “latest chart” عشان التكرار يبقى deterministic.  
- راجع ownership conflicts: لو في resources معمول لها apply يدوي، يا إمّا تتبنّاها بالـ Helm أو تمنع overlap.

**س28) لو Helm upgrade بوّظ الـ Controller/Monitoring — تعمل rollback إزاي؟**  
**إجابة نموذجية (مختصرة):**
- `helm history <release> -n <ns>` عشان تشوف revisions.  
- `helm rollback <release> <revision> -n <ns>` للرجوع السريع.  
- وبعدها verify: pods/CRDs/events + functional check (ALB created / Grafana up).

```text
helm upgrade --install
      |
      v
revision N created
      |
      +--> bad? -> helm rollback -> revision N-1
```

---

### Public EKS Endpoint + Internet-facing ALB — أسئلة “Networking واقعية”

**س29) بما إن EKS endpoint عندك Public: إزاي بتقلّل المخاطر بدل ما تسيبه مفتوح؟**  
**إجابة نموذجية (مختصرة):**
- استخدم **public access CIDRs** (تقييد الـ API على IPs محددة: Jenkins/office/VPN).  
- خَلّي Security posture: least-privilege IAM + RBAC، وممنوع kubeconfig يتسرب.  
- (اختياري) راقب CloudTrail + EKS audit logs لو متاحة.

**س30) بما إن ALB عندك Internet-facing: إزاي بتتأكد إنه راكب على الـ public subnets الصح؟**  
**إجابة نموذجية (مختصرة):**
- Subnet tags الصح: `kubernetes.io/role/elb=1` + tag الكلاستر (اللي الـ LBC بيستخدمه).  
- Ingress annotation بتحدد الـ scheme: `alb.ingress.kubernetes.io/scheme: internet-facing`.  
- Verify من AWS: ALB subnets + route tables (لازم IGW موجود للـ public subnets).

**س31) Security Groups “مين يسمح لمين” في الحالة دي؟ (مختصر عملي)**  
**إجابة نموذجية (مختصرة):**
- **ALB SG**: inbound 80/443 من الإنترنت، outbound على targetPort للنودز/targets.  
- **Nodes SG**: inbound من ALB SG على ports بتاعة الـ Service/NodePort (أو targetGroup bindings)، outbound 443 لـ ECR/STS.  
- **Jenkins SG**: outbound 443 لـ EKS API + AWS APIs؛ inbound 22/8080 حسب إدارة Jenkins (من IPs مقفولة).
