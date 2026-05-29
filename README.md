# cicd-gitops-101

`cicd-gitops-101` เป็นโปรเจกต์ตัวอย่าง (demo) ที่สาธิต **CI/CD + GitOps pipeline
ระดับโปรดักชัน** อย่างครบวงจร ครอบคลุมตั้งแต่การ commit โค้ดไปจนถึงการนำขึ้นใช้งานบน
production ประกอบด้วยด่านตรวจสอบคุณภาพ (quality gate), การสแกนความปลอดภัยของ
supply chain ในคอนเทนเนอร์, การทดสอบประสิทธิภาพด้วย k6, การจัดการ environment ด้วย
Kustomize และการ deploy แบบ GitOps ผ่าน ArgoCD ซึ่งกำหนดให้ production ต้องผ่าน
การอนุมัติโดยบุคคล

ตัวแอปพลิเคชันถูกออกแบบให้เรียบง่ายโดยตั้งใจ เนื่องจากจุดสำคัญของโปรเจกต์นี้อยู่ที่
**delivery pipeline** มิใช่ตัวแอปพลิเคชัน ทั้งระบบสามารถรันเพื่อทดลองได้จริงบนเครื่อง
ของผู้ใช้ผ่านคำสั่ง `make demo` (kind + ArgoCD)

> มีเวอร์ชันที่ใช้ **Jenkins** แบบ **ไม่ใช้ Kubernetes** (deploy เป็น Docker container)
> เป็น repo คู่กัน: [**cicd-jenkins-101**](https://github.com/PonPond/cicd-jenkins-101)
> — แอปชุดเดียวกัน คนละปลายทาง deploy (Kubernetes/GitOps ↔ Docker/push)

## การเดินทางของโค้ด 1 commit (ภาพรวม)

<p align="center">
  <img src="docs/pipeline.gif" alt="ภาพรวม CI/CD pipeline แนวนอน — commit เดินทางผ่านแต่ละด่านจนถึงผู้ใช้งาน" width="760">
  <br><sub>ภาพรวมแนวนอน — commit เดินทางผ่านแต่ละด่าน (ตรวจคุณภาพ → build → deploy) จนถึงผู้ใช้งาน</sub>
</p>

<p align="center">
  <img src="docs/pipeline-flow.gif" alt="แอนิเมชัน CI/CD + GitOps pipeline แบบละเอียด (git graph) รวมการ rollback ด้วย git revert" width="500">
  <br><sub>ฉบับละเอียด (git graph) — รวมขั้นตอน rollback ด้วย <code>git revert</code> เมื่อ production เกิดปัญหา</sub>
</p>

ภาพรวมของโปรเจกต์สามารถมองเป็น **การเดินทางของโค้ดหนึ่ง commit** ตั้งแต่เขียนเสร็จ
จนถึงผู้ใช้งาน โดยแต่ละ "ด่าน" คือกลไกที่จำเป็นต่อการ deploy ขึ้น production ได้อย่าง
ถี่และมั่นใจ:

```
เขียนโค้ด → เปิด PR → [ด่านตรวจคุณภาพ] → merge → [สร้าง artifact] → [ส่งขึ้น env] → ผู้ใช้งาน
```

### 1) เส้นทางปกติ — จาก commit สู่ผู้ใช้งาน

| ด่าน | ไฟล์ | หน้าที่ | กรณีไม่ผ่าน |
| --- | --- | --- | --- |
| 🚧 **ตรวจคุณภาพ** | [`ci.yaml`](.github/workflows/ci.yaml) | lint · unit test · `npm audit` · Trivy · k6 smoke | **PR ไม่สามารถ merge ได้** |
| 👤 **Code Review** | branch protection | ต้องได้รับการอนุมัติก่อน merge | merge ไม่ได้ |
| 📦 **สร้าง artifact** | [`build.yaml`](.github/workflows/build.yaml) | build image (distroless) → สแกน CVE → push ขึ้น GHCR → บันทึก image tag ใหม่ลง git | build ล้มเหลว ไม่มี image |
| 🚀 **ส่งขึ้น environment** | [ArgoCD](argocd/) | ArgoCD ตรวจสอบ git แล้วปรับ cluster ให้ตรงกัน | sync ล้มเหลว / ต้อง rollback |

> **staging** จะ sync โดยอัตโนมัติ (รวดเร็ว เหมาะกับการทดสอบ) ส่วน **production** ต้องอาศัยการสั่งงานโดยบุคคล ([`promote-prod.yaml`](.github/workflows/promote-prod.yaml)) เพื่อป้องกันการ deploy ขึ้น production โดยไม่ได้ตั้งใจ

### 2) กรณีเกิดปัญหา — การย้อนกลับด้วย `git revert`

หาก release ใหม่เกิดปัญหาบน production การ rollback จะไม่ใช่การ SSH เข้าไปแก้ไขที่
เครื่องโดยตรง แต่เป็น **git operation** ที่ตรวจสอบย้อนหลังได้:

```
🔴 release มีปัญหา  →  git revert  →  commit ใหม่ที่ชี้กลับไปยัง image tag เวอร์ชันเดิม
                                              │
                                              ▼
                      🟢 ArgoCD ตรวจพบการเปลี่ยนแปลงใน git  →  ปรับ cluster กลับสู่เวอร์ชันเดิมอัตโนมัติ
```

- `revert` คือการ **สร้าง commit ใหม่** (ไม่ใช่การลบของเดิม) ประวัติจึงยังคงครบถ้วน ตรวจสอบได้ว่าผู้ใดเป็นผู้ย้อน เมื่อใด และด้วยเหตุผลใด
- การแก้ไขเกิดขึ้นที่ **git** (source of truth) มิใช่ที่เครื่อง cluster จึงปรับตามโดยอัตโนมัติ ผ่านเส้นทางเดียวกับการ deploy ตามปกติ
- ดำเนินการได้รวดเร็วและปลอดภัย เนื่องจาก image เวอร์ชันเดิมยังคงอยู่ใน registry เพียงชี้ tag กลับเท่านั้น

### 3) หลักการสำคัญ 3 ข้อที่ทำให้ deploy ได้บ่อยอย่างมั่นใจ

| หลักการ | ความหมาย |
| --- | --- |
| **อัตโนมัติ** | ไม่มีการ SSH เข้า server เพื่อ `git pull` ทุกขั้นตอนดำเนินการโดยระบบ ลดความผิดพลาดจากบุคคล |
| **ตรวจสอบได้** | git เป็นแหล่งข้อมูลที่ถูกต้องเพียงหนึ่งเดียว หากต้องการทราบว่า production รันเวอร์ชันใด ตรวจสอบได้จาก git โดยตรง |
| **ย้อนกลับได้** | การ rollback ทำได้ด้วย `git revert` จากนั้น ArgoCD จะ sync กลับให้โดยอัตโนมัติ |

## เจาะลึกแต่ละขั้น (อ้างอิงไฟล์จริง)

สำหรับผู้ที่ต้องการเข้าใจกลไกเบื้องหลังภาพรวมจาก GIF ส่วนนี้อธิบายสิ่งที่เกิดขึ้นจริง
ในแต่ละขั้นตอน โดยอ้างอิงไฟล์จริงภายใน repo:

### ขั้นที่ 1 — เปิด PR เข้า `main` แล้ว CI ทำงานทันที ([`ci.yaml`](.github/workflows/ci.yaml))

- **เงื่อนไขการทำงาน:** ทุกครั้งที่เปิดหรืออัปเดต Pull Request ที่ยิงเข้า branch `main`
- **ทำงานพร้อมกัน 4 job (ขนาน):**
  - `lint` — รัน `npm ci` แล้ว `npm run lint` (ESLint) เพื่อตรวจรูปแบบโค้ดและข้อผิดพลาดเบื้องต้น
  - `test` — รัน `npm run test:ci` (unit test พร้อม coverage)
  - `security` — ตรวจสอบ 2 ชั้น:
    - `npm audit --omit=dev --audit-level=high` — ตรวจ dependency ที่ใช้งานจริง (ไม่รวม dev) หากพบช่องโหว่ระดับ high ขึ้นไป จะถือว่าไม่ผ่าน
    - **Trivy** สแกนไฟล์ (`scan-type: fs`) ที่ระดับ `HIGH,CRITICAL` เมื่อพบจะ `exit-code: 1` (ข้ามรายการที่ยังไม่มีแพตช์ด้วย `ignore-unfixed`)
  - `k6` — ทดสอบบน container จริง:
    - build image → `docker run` → ตรวจสอบ `/readyz` เป็นรอบ ๆ จนกว่า service จะพร้อม (สูงสุด 30 วินาที)
    - รัน `k6 run smoke.js` (ยืนยันว่ารับ request ได้) จากนั้น `k6 run load.js` (**ด่านวัดประสิทธิภาพ** — ไม่ผ่านหากค่า p95 หรืออัตรา error เกิน SLO)
- **ผลลัพธ์:** หาก job ใดไม่ผ่าน PR จะมีสถานะไม่ผ่าน และหากตั้ง branch protection ไว้ จะ **ไม่สามารถ merge ได้**

### ขั้นที่ 2 — การรีวิวและ merge

- ต้องมีผู้รีวิวกด **Approve** บน PR (บังคับผ่าน branch protection)
- เมื่อ merge เข้า `main` ถือเป็นจุดที่ผ่านด่านคุณภาพเรียบร้อยแล้ว

### ขั้นที่ 3 — push เข้า `main` แล้วสร้าง artifact ([`build.yaml`](.github/workflows/build.yaml))

- **เงื่อนไขการทำงาน:** ทุกครั้งที่มี commit ใหม่บน `main` (รวมถึงเมื่อ merge)
- `test` (ทำงานซ้ำอีกครั้งเป็นด่านก่อน build) → `lint` + `test:ci`
- `build-and-push`:
  - กำหนดชื่อ image เป็นตัวพิมพ์เล็ก `ghcr.io/<owner>/<repo>` และ **tag = `sha-` ตามด้วย 7 อักขระแรกของ commit** (เช่น `sha-abc1234`) ทำให้ทุก build มี tag เฉพาะตัว ไม่ทับกัน
  - เข้าสู่ระบบ **GHCR** ด้วย `GITHUB_TOKEN`
  - build จากโฟลเดอร์ `app/` แล้ว push 2 tag คือ `:sha-xxxxxxx` และ `:latest` (มี layer cache แบบ gha เพื่อความรวดเร็ว)
  - **Trivy** สแกน image ที่เพิ่ง build (`HIGH,CRITICAL`) หากพบช่องโหว่จะไม่ปล่อยต่อ
- `promote-staging` — **หัวใจของ GitOps:**
  - ใช้ `kustomize edit set image` แก้ไข image tag ใน `gitops/overlays/staging`
  - `ci-bot` **commit และ push การเปลี่ยนแปลง tag กลับเข้า git** ซึ่งคือขั้นตอน "บันทึกเวอร์ชันใหม่ลง git" ตามที่ปรากฏใน GIF

### ขั้นที่ 4 — ArgoCD ตรวจพบ commit แล้ว deploy staging อัตโนมัติ ([`argocd/staging.yaml`](argocd/staging.yaml))

- ArgoCD Application `staging` ตรวจสอบ repo ที่ path `gitops/overlays/staging` บน branch `main`
- เปิดใช้ `syncPolicy.automated`:
  - `prune: true` — ลบ resource ที่ถูกนำออกจาก git ออกจาก cluster ด้วย
  - `selfHeal: true` — หากมีการแก้ไข cluster โดยตรง ArgoCD จะปรับกลับให้ตรงกับ git
  - `CreateNamespace=true` — สร้าง namespace `demo-staging` ให้โดยอัตโนมัติ
- สรุป: เมื่อ tag ใน git เปลี่ยน ArgoCD จะปรับ cluster ให้ตรงกันภายในไม่กี่นาที โดยไม่ต้อง `kubectl apply` ด้วยบุคคล

### ขั้นที่ 5 — การ promote ขึ้น production (โดยบุคคล) ([`promote-prod.yaml`](.github/workflows/promote-prod.yaml))

- **ไม่ทำงานอัตโนมัติ** — ต้องสั่งงานที่แท็บ **Actions → Promote to Production** (`workflow_dispatch`) พร้อมระบุ `image_tag` ที่ผ่าน staging มาแล้ว
- `environment: production` — หากตั้ง **required reviewers** ไว้ workflow จะหยุดรอการอนุมัติจากบุคคลก่อน (คือด่านอนุมัติที่ปรากฏใน GIF)
- เมื่ออนุมัติแล้ว: `kustomize edit set image` ใน `gitops/overlays/production` จากนั้น `ci-bot` commit และ push
- ArgoCD ฝั่ง production ตั้งเป็น **manual-sync** จึงต้องสั่ง **Sync** ใน ArgoCD อีกครั้ง (เป็นการป้องกันเพิ่มอีกชั้นหนึ่ง)

### ขั้นที่ 6 — กรณี production เกิดปัญหา: rollback ด้วย `git revert`

- ใช้ `git revert` กับ commit ที่ปรับ tag (หรือแก้ tag กลับเป็นเวอร์ชันเดิม) แล้ว `push`
- ArgoCD ตรวจพบการเปลี่ยนแปลงใน git แล้ว sync cluster **กลับสู่เวอร์ชันเดิม**
- ไม่จำเป็นต้อง build ใหม่ เนื่องจาก image เวอร์ชันเดิมยังคงอยู่ใน GHCR เพียงชี้ tag กลับ
- ประวัติครบถ้วน สามารถดู git log เพื่อตรวจสอบได้ว่าผู้ใดเป็นผู้ย้อน เมื่อใด และด้วยเหตุผลใด

## สิ่งที่ repo นี้ครอบคลุม

| หัวข้อ | รายละเอียด |
| --- | --- |
| ด่านคุณภาพ CI | lint, unit test + coverage, `npm audit`, Trivy (สแกนไฟล์) |
| ด่านวัดประสิทธิภาพ | **k6** smoke + load พร้อม SLO threshold (p95 < 200ms, error < 1%) |
| คอนเทนเนอร์ | Dockerfile แบบหลาย stage, distroless, ไม่รันด้วย root, rootfs แบบอ่านอย่างเดียว |
| ความปลอดภัย supply chain | สแกน image ด้วย Trivy, push ขึ้น GHCR, ติด tag ตาม commit SHA |
| การจัดการ config | **Kustomize** base พร้อม overlay `staging`/`production` |
| GitOps | **ArgoCD** Applications; staging sync อัตโนมัติ, production สั่งโดยบุคคล |
| การ promote | เลื่อนข้าม environment ผ่าน GitHub Environment ที่ต้องมีผู้อนุมัติ |
| การทำซ้ำ | `make demo` สร้าง kind + ArgoCD บนเครื่องได้ |

## ภาพรวม pipeline (ฉบับย่อ)

```
PR ─▶ [lint · test · security · k6] ──merge──▶ main
main ─▶ build image ─▶ Trivy scan ─▶ push GHCR ─▶ bump staging overlay (git commit)
        ArgoCD ตรวจพบ commit ─▶ auto-sync STAGING ─▶ ด่าน k6 (perf)
        อนุมัติโดยบุคคล ─▶ promote-prod ─▶ bump prod overlay ─▶ ArgoCD manual-sync PRODUCTION
```

ไดอะแกรมฉบับเต็มและเหตุผลการออกแบบ: [docs/architecture.md](docs/architecture.md)

## โครงสร้างโปรเจกต์

```
app/                  Node (Express) service + เทส + Dockerfile
tests/k6/             k6 smoke / load / stress (ใช้ SLO threshold ร่วมกัน)
gitops/
  base/               Kustomize base (Deployment, Service, HPA)
  overlays/staging/    config staging (CI ปรับ image tag ที่นี่อัตโนมัติ)
  overlays/production/ config production (promote โดยบุคคล)
argocd/               ArgoCD Application manifests (staging + production)
.github/workflows/    ci.yaml · build.yaml · promote-prod.yaml
Makefile              คำสั่งสำหรับ dev และรันเดโม kind/ArgoCD บนเครื่อง
```

## การเริ่มต้นใช้งาน (บนเครื่อง)

```bash
# 1. แอปพลิเคชัน: ติดตั้ง, ทดสอบ, รัน
make install
make test
make run            # http://localhost:3000/api/hello?name=you

# 2. คอนเทนเนอร์ + k6  (host port 3001 -> container 3000)
make docker-build                       # build image ชื่อ cicd-gitops-101:dev
docker run -d -p 3001:3000 --name demo cicd-gitops-101:dev
curl localhost:3001/healthz
BASE_URL=http://localhost:3001 make k6-smoke
docker rm -f demo

# 3. รันระบบ GitOps แบบครบวงจรบน cluster ในเครื่อง (ต้องมี kind + kubectl + argocd)
make demo           # สร้าง kind cluster + ArgoCD + ลงทะเบียน Applications
make argocd-ui      # จากนั้นเปิด https://localhost:8080
make argocd-password
```

## การนำไปใช้ต่อ (fork)

manifest ทั้งหมด (ArgoCD / Kustomize) ตั้งค่าไว้สำหรับบัญชี **`ponpond`** ได้แก่
image `ghcr.io/ponpond/cicd-gitops-101` และ repo `github.com/ponpond/cicd-gitops-101`
หากนำไปใช้กับบัญชีของท่านเอง ให้ปรับให้ชี้มายังบัญชีของท่าน:

```bash
grep -rl ponpond . --exclude-dir=.git | xargs sed -i '' 's/ponpond/<your-username>/g'   # macOS
```

จากนั้นตั้งค่าใน Settings ของ repo:

1. **สร้าง lockfile**: รัน `make install` แล้ว commit `app/package-lock.json`
   (CI ใช้ `npm ci` ซึ่งต้องมีไฟล์นี้)
2. **Environments → `production`**: เพิ่มผู้ใช้เป็น required reviewer เพื่อเปิดใช้ด่านอนุมัติ production
3. **Branch protection บน `main`**: กำหนดให้ CI ต้องผ่านก่อน merge

## เหตุผลเชิงออกแบบ (design decisions)

- **GitOps แทนการ deploy แบบสั่งด้วยมือ** — git เป็น source of truth; CI ไม่เข้าไป
  แก้ไข cluster โดยตรง; การ rollback ทำผ่าน `git revert`
- **build ครั้งเดียวแล้ว promote artifact เดิม** — image ที่ผ่าน staging คือ image
  ตัวเดียวกับที่นำขึ้น production (ไม่ build ใหม่แยกตาม environment)
- **การป้องกันแบบหลายชั้นบน supply chain** — ตรวจสอบ dependency, สแกนไฟล์,
  สแกน image และใช้ runtime แบบ distroless + non-root
- **กำหนดให้ประสิทธิภาพเป็นด่านหนึ่ง มิใช่เรื่องที่พิจารณาภายหลัง** — k6 threshold
  ทำให้ pipeline ไม่ผ่านเมื่อ latency หรือ error เกิน SLO
- **วางด่านอนุมัติโดยบุคคลเฉพาะจุดที่จำเป็น** — staging อัตโนมัติเพื่อความรวดเร็ว
  ส่วน production ต้องผ่านการอนุมัติ

## สัญญาอนุญาต (License)

MIT
