# GitHub Actions Workflow Explained (Beginner Friendly)

This page explains every section of this workflow file:

- `.github/workflows/static.yml`

The workflow publishes your DocFX site to GitHub Pages.

---

## Full workflow at a glance

```yml
name: Publish DocFX to Pages

on:
  push:
    branches: [ main ]
  workflow_dispatch:

permissions:
  contents: read
  pages: write
  id-token: write

concurrency:
  group: pages
  cancel-in-progress: false

jobs:
  publish-docs:
    runs-on: ubuntu-latest
    environment:
      name: github-pages
      url: ${{ steps.deployment.outputs.page_url }}

    steps:
      - name: Checkout
        uses: actions/checkout@v4

      - name: Setup .NET
        uses: actions/setup-dotnet@v4
        with:
          dotnet-version: 8.x

      - name: Install DocFX
        run: |
          dotnet tool install --global docfx || dotnet tool update --global docfx

      - name: Build docs
        run: docfx ./docfx.json

      - name: Configure Pages
        uses: actions/configure-pages@v5

      - name: Upload artifact
        uses: actions/upload-pages-artifact@v3
        with:
          path: ./_site

      - name: Deploy to GitHub Pages
        id: deployment
        uses: actions/deploy-pages@v4
```

---

## What a workflow is

A **workflow** is an automated process that GitHub runs for your repository.

- It is defined in a YAML file inside `.github/workflows/`.
- It can run on events like a push, pull request, or manual trigger.
- It runs one or more jobs made of ordered steps.

In your case, the workflow builds documentation and deploys it to GitHub Pages.

---

## Section-by-section explanation

## 1) `name`

```yml
name: Publish DocFX to Pages
```

- This is the display name you see in the Actions tab.
- It helps humans quickly understand what this workflow does.

---

## 2) `on` (triggers)

```yml
on:
  push:
    branches: [ main ]
  workflow_dispatch:
```

This defines **when** the workflow should run.

- `push` on `main`: runs automatically every time commits are pushed to `main`.
- `workflow_dispatch`: adds a **Run workflow** button in GitHub Actions so you can run it manually.

Why this is useful:

- Automatic publishing after updates.
- Manual reruns for testing or recovery.

---

## 3) `permissions`

```yml
permissions:
  contents: read
  pages: write
  id-token: write
```

This gives the workflow token only the access it needs.

- `contents: read`: can read repository files.
- `pages: write`: can publish content to GitHub Pages.
- `id-token: write`: allows the workflow to request a short-lived OIDC identity token from GitHub.

Why this specific grant is needed:

- The Pages deploy action uses this OIDC token to prove "this run is a trusted GitHub Actions run from this repository."
- GitHub Pages checks that token before accepting a deployment artifact.
- Without `id-token: write`, the deploy step may fail because it cannot mint the token needed for that trust check.

Why this is safer than long-lived secrets:

- The token is temporary and created only during the workflow run.
- You do not have to store a permanent deploy credential in repository secrets.

Security note:

- Keeping permissions minimal is a best practice.

---

## 4) `concurrency`

```yml
concurrency:
  group: pages
  cancel-in-progress: false
```

This controls how many runs can happen at the same time for this group.

- `group: pages`: all runs in this group are related.
- `cancel-in-progress: false`: if one run is already active, GitHub waits instead of canceling older runs.

Why it matters:

- Prevents conflicting deployments.
- Keeps deployment order predictable.

---

## 5) `jobs`

```yml
jobs:
  publish-docs:
```

A workflow can contain multiple jobs. You currently have one job named `publish-docs`.

---

## 6) `runs-on`

```yml
runs-on: ubuntu-latest
```

- This chooses the virtual machine image GitHub uses to run your job.
- `ubuntu-latest` is a Linux runner maintained by GitHub.

---

## 7) `environment`

```yml
environment:
  name: github-pages
  url: ${{ steps.deployment.outputs.page_url }}
```

- `name: github-pages`: associates the deployment with the GitHub Pages environment.
- `url`: captures the deployed site URL from a later step (`deployment`) and shows it in the UI.

Why the `url` field is important:

- It links the environment record directly to the live site address for that deployment.
- In the Actions and Deployments UI, people can click straight from the deployment entry to the published docs.
- It makes each deployment easier to verify because reviewers can immediately open the exact page that was just released.
- It improves traceability: if you check deployment history later, each entry includes where that version was published.

Is `environment.url` strictly required to deploy?

- Usually no. Deployment can still work without this line.
- But adding it is strongly recommended because it improves visibility, reviewability, and operational clarity.

Expression note:

- `${{ ... }}` is GitHub Actions expression syntax.
- `steps.deployment.outputs.page_url` means: read the `page_url` output from the step whose `id` is `deployment`.

---

## 8) `steps`

Steps run in order from top to bottom.

### Step A: Checkout repository

```yml
- name: Checkout
  uses: actions/checkout@v4
```

- Downloads your repository files into the runner.
- Required before build commands can access your docs.

`uses` means this step uses a prebuilt action from GitHub Marketplace.

---

### Step B: Install .NET SDK

```yml
- name: Setup .NET
  uses: actions/setup-dotnet@v4
  with:
    dotnet-version: 8.x
```

- Installs and configures .NET 8 on the runner.
- Needed because DocFX is installed as a .NET tool.

`with` passes configuration values to an action.

---

### Step C: Install or update DocFX

```yml
- name: Install DocFX
  run: |
    dotnet tool install --global docfx || dotnet tool update --global docfx
```

- `run` executes shell commands.
- The command tries to install DocFX globally.
- If DocFX is already installed, `install` fails and `||` runs `update` instead.

This pattern makes the step resilient across runner states.

---

### Step D: Build the documentation site

```yml
- name: Build docs
  run: docfx ./docfx.json
```

- Runs DocFX using your config file.
- Generates static site output in the `_site` folder (as configured in `docfx.json`).

---

### Step E: Configure GitHub Pages deployment settings

```yml
- name: Configure Pages
  uses: actions/configure-pages@v5
```

- Prepares the job for Pages deployment.
- Sets up metadata and deployment configuration expected by Pages actions.

---

### Step F: Upload built site as an artifact

```yml
- name: Upload artifact
  uses: actions/upload-pages-artifact@v3
  with:
    path: ./_site
```

- Uploads the generated `_site` folder as a deployment artifact.
- That artifact is what the deploy step publishes.

---

### Step G: Deploy to GitHub Pages

```yml
- name: Deploy to GitHub Pages
  id: deployment
  uses: actions/deploy-pages@v4
```

- Publishes the uploaded artifact to GitHub Pages.
- `id: deployment` gives this step a name that other expressions can reference.
- This is why `environment.url` can read `steps.deployment.outputs.page_url`.

---

## Helpful mental model

You can think of this workflow as a simple pipeline:

1. Trigger on push/manual run.
2. Set up tools.
3. Build docs.
4. Upload output.
5. Deploy to Pages.

---

## Common beginner questions

### Why are there version tags like `@v4` and `@v5`?

Those pin the major version of each action. It gives stability while still allowing minor/patch updates.

### Why not commit `_site` to the repository?

This workflow builds `_site` automatically, so source files stay clean and generated output is published from CI.

### What happens if build fails?

The workflow stops and deployment does not run. You can check logs in the Actions tab to see which step failed.

---

## Glossary

- **Workflow**: automation definition in YAML.
- **Job**: a group of steps that run on one runner.
- **Step**: one action or command.
- **Runner**: GitHub-hosted machine that executes jobs.
- **Artifact**: packaged files passed between build and deploy phases.
- **GitHub Pages**: static site hosting for repositories.
