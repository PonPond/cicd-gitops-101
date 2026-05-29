# cicd-gitops-101

A small Node service wrapped in a **production-grade CI/CD + GitOps pipeline**.
The application is intentionally simple — the point of this repo is the
**delivery pipeline**: quality gates, container supply-chain scanning, k6
performance gates, Kustomize environments, and ArgoCD GitOps with a manual
production approval gate.

## การเดินทางของโค้ด 1 commit (ดูภาพรวมแบบเห็นภาพ)

<p align="center">
  <img src="docs/pipeline.gif" alt="ภาพรวม CI/CD pipeline แนวนอน — commit วิ่งผ่านแต่ละด่านจนถึงผู้ใช้" width="760">
  <br><sub>ภาพรวมแนวนอน — commit วิ่งผ่านแต่ละด่าน (ตรวจคุณภาพ → build → deploy) จนถึงผู้ใช้</sub>
</p>

<p align="center">
  <img src="docs/pipeline-flow.gif" alt="แอนิเมชัน CI/CD + GitOps pipeline แบบละเอียด (git graph) รวมการ rollback ด้วย git revert" width="500">
  <br><sub>ฉบับละเอียด (git graph) — รวมฉาก rollback ด้วย <code>git revert</code> เมื่อ production พัง</sub>
</p>

ลองคิดว่ามันคือ **การเดินทางของโค้ด 1 commit** ตั้งแต่เขียนเสร็จจนถึงมือผู้ใช้ —
ทุก "ด่าน" คือสิ่งที่ทีมจริงต้องมีเพื่อกล้าปล่อยของขึ้น production บ่อย ๆ โดยไม่พัง:

```
เขียนโค้ด → เปิด PR → [ด่านตรวจคุณภาพ] → merge → [สร้าง artifact] → [ส่งขึ้น env] → ผู้ใช้
```

### องก์ 1 — เดินทางไปข้างหน้า (commit → ผู้ใช้)

| ด่าน | ไฟล์ | ทำอะไร | ถ้าไม่ผ่าน |
| --- | --- | --- | --- |
| 🚧 **ตรวจคุณภาพ** | [`ci.yaml`](.github/workflows/ci.yaml) | lint · unit test · `npm audit` · Trivy · k6 smoke | **PR เข้าไม่ได้** (ปิดประตู) |
| 👤 **Code Review** | branch protection | คนอนุมัติก่อน merge | merge ไม่ได้ |
| 📦 **สร้าง artifact** | [`build.yaml`](.github/workflows/build.yaml) | build image (distroless) → scan CVE → push GHCR → เขียน image tag ใหม่ลง git | build ล้ม ไม่มี image |
| 🚀 **ส่งขึ้น env** | [ArgoCD](argocd/) | ArgoCD เฝ้าดู git → ปรับ cluster ให้ตรง | sync ล้ม / rollback |

> **staging** sync อัตโนมัติ (เร็ว ทดลองได้) แต่ **production** ต้องมีคนกด ([`promote-prod.yaml`](.github/workflows/promote-prod.yaml)) — กันของหลุดขึ้น prod โดยไม่ตั้งใจ

### องก์ 2 — ของพังแล้วย้อนกลับ (rollback ด้วย `git revert`)

ถ้าเวอร์ชันใหม่มีปัญหาบน production การ rollback **ไม่ใช่** การ ssh เข้าไปกู้ของที่เครื่อง
แต่เป็น **git operation** ที่ตรวจสอบได้:

```
🔴 ของพัง  →  git revert  →  commit ใหม่ที่ชี้ image tag "เวอร์ชันเดิม"
                                   │
                                   ▼
                       🟢 ArgoCD เห็น git เปลี่ยน  →  ปรับ cluster กลับเวอร์ชันเดิมอัตโนมัติ
```

- `revert` = **สร้าง commit ใหม่** (ไม่ใช่ลบของเก่า) → ประวัติยังครบ ตรวจสอบได้ว่าใครย้อน ตอนไหน เพราะอะไร
- ย้อนที่ **git** (source of truth) ไม่ใช่ที่เครื่อง → cluster ปรับตามเอง เส้นทางเดียวกับตอน deploy
- เร็วและปลอดภัย เพราะ image เก่ายังอยู่ใน registry แล้ว — แค่ชี้ tag กลับ

### 3 หัวใจที่ทำให้กล้าปล่อยของบ่อย ๆ

| หลักการ | หมายความว่า |
| --- | --- |
| **อัตโนมัติ** | ไม่มีใคร ssh เข้า server แล้ว `git pull` — ทุกขั้นเป็นเครื่องทำ ไม่พลาดจากมือคน |
| **ตรวจสอบได้** | git คือความจริงเพียงหนึ่งเดียว อยากรู้ prod รันอะไร? ดู git ได้เลย |
| **ย้อนกลับได้** | rollback = `git revert` → ArgoCD sync กลับให้เอง |

## What it demonstrates

| Area | What's here |
| --- | --- |
| CI quality gate | lint, unit tests + coverage, `npm audit`, Trivy filesystem scan |
| Performance gate | **k6** smoke + load with SLO thresholds (p95 < 200ms, error < 1%) |
| Container | multi-stage Dockerfile, distroless, non-root, read-only rootfs |
| Supply chain | Trivy image scan, images pushed to GHCR, tagged by commit SHA |
| Config management | **Kustomize** base + `staging`/`production` overlays |
| GitOps | **ArgoCD** Applications; staging auto-syncs, production is manual |
| Promotion | environment promotion with a GitHub Environment approval gate |
| Reproducible demo | `make demo` spins up kind + ArgoCD locally |

## Pipeline at a glance

```
PR ─▶ [lint · test · security · k6] ──merge──▶ main
main ─▶ build image ─▶ Trivy scan ─▶ push GHCR ─▶ bump staging overlay (git commit)
        ArgoCD sees the commit ─▶ auto-sync STAGING ─▶ k6 perf gate
        manual approval ─▶ promote-prod ─▶ bump prod overlay ─▶ ArgoCD manual-sync PRODUCTION
```

Full diagram + design rationale: [docs/architecture.md](docs/architecture.md).

## Repository layout

```
app/                  Node (Express) service + tests + Dockerfile
tests/k6/             k6 smoke / load / stress scenarios (shared SLO thresholds)
gitops/
  base/               Kustomize base (Deployment, Service, HPA)
  overlays/staging/   staging config (CI auto-bumps the image tag here)
  overlays/production/ production config (promoted manually)
argocd/               ArgoCD Application manifests (staging + production)
.github/workflows/    ci.yaml · build.yaml · promote-prod.yaml
Makefile              local dev + kind/ArgoCD demo targets
```

## Quickstart (local)

```bash
# 1. App: install, test, run
make install
make test
make run            # http://localhost:3000/api/hello?name=you

# 2. Container + k6  (host port 3001 -> container 3000)
make docker-build                       # builds image tagged cicd-gitops-101:dev
docker run -d -p 3001:3000 --name demo cicd-gitops-101:dev
curl localhost:3001/healthz
BASE_URL=http://localhost:3001 make k6-smoke
docker rm -f demo

# 3. Full GitOps demo on a local cluster (needs kind + kubectl + argocd)
make demo           # kind cluster + ArgoCD + register Applications
make argocd-ui      # then open https://localhost:8080
make argocd-password
```

## Before you push it to GitHub

The ArgoCD / Kustomize manifests are configured for **`ponpond`** — image
`ghcr.io/ponpond/cicd-gitops-101`, repo `github.com/ponpond/cicd-gitops-101`.
If you fork this, point them at your own account:

```bash
grep -rl ponpond . --exclude-dir=.git | xargs sed -i '' 's/ponpond/<your-username>/g'   # macOS
```

Then, in the GitHub repo settings:

1. **Generate the lockfile**: `make install` and commit `app/package-lock.json`
   (CI uses `npm ci`, which requires it).
2. **Environments → `production`**: add yourself as a required reviewer to arm
   the production approval gate.
3. **Branch protection on `main`**: require the CI checks to pass before merge.

## Talking points (interview)

- **GitOps over imperative deploys** — git is the source of truth; CI never
  touches the cluster; rollback is `git revert`.
- **Build once, promote the artifact** — the exact image that passed staging is
  what reaches production (no per-env rebuilds).
- **Defense in depth on the supply chain** — dependency audit, filesystem scan,
  image scan, distroless + non-root runtime.
- **Performance as a gate, not an afterthought** — k6 thresholds fail the
  pipeline when latency/error SLOs regress.
- **Human gate where it matters** — staging is automated for speed; production
  requires explicit approval.

## License

MIT
