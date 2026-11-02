# AGENTS.md

Guidance for AI coding agents working in this repository.

## What this is

Showroom lab and cluster bootstrap for the Parasol Insurance workshop: Red Hat OpenShift AI on Azure Red Hat OpenShift (ARO). Attendees follow a claims-processing story (LLM, RAG, image models, a small web app) on a multi-user ARO cluster ordered from RHPDS.

This repo is the lab guide and the Kubernetes manifests GitOps applies. It does not contain the Ansible that builds the cluster. That lives in AgnosticD. The RHPDS order form lives in AgnosticV.

Remote: `https://github.com/rh-mobb/parasol-insurance`. Default branch: `dev`.

## Layout

```
content/                  Antora showroom book (nav.adoc, modules/ROOT/pages)
content/utilities/        lab-build, lab-serve, lab-stop, lab-clean
default-site.yml          Antora playbook Showroom renders
bootstrap/                Kustomize apps Argo CD syncs
bootstrap/applicationset/ ApplicationSet that registers those apps
app/                      Claims web app (React frontend, Python backend)
lab-materials/            Notebooks and exercise assets cloned into workbenches
references/agnosticv/     Local clone of rhpds/agnosticv. Gitignored. Not part of this repo.
```

Catalog item (only present after `references/agnosticv` is cloned):

```
references/agnosticv/azure_gpte/ocp-rhoai-aro-parasol-mobb/
  common.yaml    workloads, ARO size, Showroom repo, __meta__
  dev.yaml       deployer scm_ref, dev access group, purpose
  description.adoc
  info-message-template.adoc
  user-message-template.adoc
```

## Three git sources

A change in this working tree does not automatically become what a lab order runs. Keep these apart:

| Consumer | Repo and ref | What it uses |
|---|---|---|
| Showroom lab guide | `rh-mobb/parasol-insurance` `dev` | This repo. Antora playbook `default-site.yml`. |
| RHPDS GitOps apply | `paulczar/parasol-insurance` `dev` | `ocp4_workload_oai_lab_insurance_claim` clones this fork and applies its ApplicationSet. That file syncs bootstrap paths from the same fork at `dev`. |
| Gitea mirror | `paulczar/parasol-insurance` | What attendees see inside the cluster. |
| This tree's ApplicationSet | `rh-mobb/parasol-insurance` `stage` | Used only if someone applies `bootstrap/applicationset/applicationset-bootstrap.yaml` from this checkout. |

`content/antora.yml` still has `git-clone-repo-url` pointed at `rh-aiservices-bu/parasol-insurance`. The static site in the README is `https://rh-aiservices-bu.github.io/parasol-insurance/`.

Do not retarget the ApplicationSet, the catalog `*_repo_url` vars, or the Gitea mirror unless that retarget is the task. Say which of the three consumers the edit is for.

## How RHPDS deploys

AgnosticV `env_type` is `open-environment-azure-subscription` with `install_aro: true`. `dev.yaml` pins the deployer to AgnosticD tag `ocp4_workload_oai_parasol_insurance-1.1.9` and image `quay.io/agnosticd/ee-multicloud:v0.1.2`.

AgnosticD then:

1. Allocates the Azure sandbox subscription from `__meta__.sandboxes`.
2. Deploys a bastion and ARO. Version pin: `az_aro_version: "4.21.22"`.
3. Runs `infra_workloads` on the bastion. `student_workloads` is empty.

Workload order in `common.yaml`:

1. `ocp4_workload_authentication` — htpasswd, users `user1`…`userN`, password `openshift`, kubeadmin removed.
2. `ocp4_workload_openshift_data_foundation` — channel `stable-4.21`, default storage class `ocs-storagecluster-ceph-rbd`.
3. `ocp4_workload_pipelines` — `pipelines-1.21`.
4. `ocp4_workload_openshift_gitops` — `gitops-1.20`.
5. `ocp4_workload_gitea_operator`.
6. `ocp4_workload_serverless` — operator only; serving and eventing are off.
7. `ocp4_workload_ossm` — Service Mesh v2 for OpenShift AI 2.x KServe. Elasticsearch, Jaeger, and Kiali are off.
8. `ocp4_workload_openshift_ai` — channel `stable-2.25`. ModelMesh is required. Do not move this lab to RHOAI 3.x without replacing ModelMesh.
9. `ocp4_workload_web_terminal`.
10. `ocp4_workload_showroom` — Helm, `content_only: true` (guide only, no terminal pane).
11. `ocp4_workload_oai_lab_insurance_claim` — clone and `oc apply` the ApplicationSet. It does not wait for Argo CD. The newer role `ocp4_workload_oai_parasol_insurance` (Argo CD patches and a health wait) is on the same AgnosticD tag and is not in `infra_workloads`.

`num_users` defaults to 2, minimum 2. Worker count is `max(4, ceil(num_users * 0.3) + 3)` of `Standard_D16s_v3`. Masters are `Standard_D8s_v3` through 10 users and `Standard_D16s_v3` above that. The NVIDIA GPU workload is commented out.

`azure_ai_info_url` is a required catalog parameter. It is passed through Showroom user data and used in the lab pages (`{azure_ai_info_url}`) for Azure OpenAI, Azure AI Search, and Azure Database for PostgreSQL. Those services are not created by this repo.

Ansible roles are in `redhat-cop/agnosticd` under `ansible/roles_ocp_workloads/` and `ansible/configs/open-environment-azure-subscription/`. Edit them in an AgnosticD checkout, then bump `dev.yaml` `scm_ref` in the AgnosticV catalog item. A tag bump belongs in `rhpds/agnosticv`, not in a commit to this repo.

## GitOps apps

`bootstrap/applicationset/applicationset-bootstrap.yaml` registers:

- `bootstrap/ic-shared-minio`
- `bootstrap/ic-shared-llm` — shared Hugging Face TGI server
- `bootstrap/ic-shared-img-det`
- `bootstrap/ic-rhoai-configuration` — Azure File CSI, workbench image streams, dashboard config
- `bootstrap/ic-user-projects` — one project, workbench, and pipeline server per Showroom namespace
- `bootstrap/ic-shared-app` — shared claims web app

`bootstrap/ic-shared-database` and `bootstrap/ic-shared-milvus` are not in the ApplicationSet. Leave them out unless the task is to wire them in on every repo the ApplicationSet points at.

`ic-user-projects` counts namespaces named `showroom*`. Showroom is installed before this job, so that count is the attendee count. A zero count takes a different path that fabricates users; do not rely on that path for a real order.

## Editing the lab guide

- Pages: `content/modules/ROOT/pages/`.
- Navigation: `content/modules/ROOT/nav.adoc`. Modules run 1 background, 2 connect, 3 LLM, 4 image, 5 web app, 6 productization, 7 close. 2.2 and 2.3 use the pre-created project and workbench. The DIY variants are commented out in the nav.
- Shared AsciiDoc attributes: `content/antora.yml` and `content/modules/ROOT/pages/_attributes.adoc`.
- Preview from the repo root: `./content/utilities/lab-build`, then `./content/utilities/lab-serve` (http://localhost:8443/). Stop with `./content/utilities/lab-stop`.
- Showroom injects `{user}`, `{password}`, console URLs, and `{azure_ai_info_url}` at render time. Keep those as Antora attributes. Local preview uses the placeholders in `content/antora.yml`.
- GitHub Pages publishes `main` via `.github/workflows/publish.yml` (`npx antora default-site.yml`). This fork's default branch is `dev`; do not assume a `main` branch exists here.

## Editing the app or notebooks

- `app/` is the claims UI. Frontend is PatternFly/React. Backend is Python (`app/backend`, Pipfile). Local run from `app/`: `npm install`, then `npm run dev` or `./start-dev.sh`. Frontend `:9000`, backend `:5000`.
- The image build runs the npm build. See `app/README.md`.
- Notebooks and datasets are under `lab-materials/`. `lab-materials/04/datasets/` and `lab-materials/04/runs/` are gitignored.

## Catalog and secrets

- `references/` is gitignored. Clone `rhpds/agnosticv` there when catalog work is in scope. Commit catalog changes in that clone, not here.
- Do not commit `.env`, kubeconfigs, Azure credentials, or generated `common_password` values.
- Catalog AsciiDoc templates use single-brace attributes (`{openshift_console_url}`). `common.yaml` values are Jinja2 (`{{ guid }}`). `#include /includes/...` lines are AgnosticV merge directives, not YAML.

## Branches on this remote

`dev` (default, Showroom ref), `stage` (this tree's ApplicationSet `targetRevision`), `azure-build`.

The README's `main` promotion list (`applicationset-bootstrap.yaml`, `content/antora.yml`, `05-03-web-app-deploy-application.adoc`) describes the upstream `rh-aiservices-bu` flow. Follow it only when the task is a promotion onto a branch that exists.
