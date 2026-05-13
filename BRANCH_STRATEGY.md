# Branch Strategy — Salesforce Multi-Developer Isolated Sandbox Flow

## Branch Map

```
main          ← production-ready code (protected: 2 approvals)
  └── uat     ← user acceptance testing (protected: 2 approvals)
        └── dev  ← integration / QA (protected: 1 approval)
              ├── feature/dev1  ← Developer 1 (Devorg1)
              ├── feature/dev2  ← Developer 2 (Devorg2)
              └── feature/dev3  ← Developer 3 (Devorg3)
```

## Developer Workflow

### Day-to-day
```bash
# 1. Pull latest dev
git checkout dev && git pull origin dev

# 2. Create your feature branch
git checkout -b feature/dev1        # or dev2 / dev3

# 3. Develop in your Personal Sandbox (Devorg1/2/3)
#    sf project retrieve start --target-org devorg1
#    ... make changes ...
#    npm run lint && npm run test:unit

# 4. Push and open PR → dev
git add . && git commit -m "feat: my change"
git push -u origin feature/dev1
# Open PR on GitHub targeting 'dev'
```

### PR Requirements
| Target branch | Required approvals | Status checks |
|---|---|---|
| `dev`  | 1 | validate-pr |
| `uat`  | 2 | validate-pr |
| `main` | 2 | validate-pr |

## Pipeline Flow

```
feature/dev* → PR → dev
                      ↓ (auto)
                 dev-qa-deploy → QA Sandbox
                      ↓ (PR + 2 approvals)
                    uat
                      ↓ (auto)
                 uat-deploy → UAT Sandbox
                      ↓ (PR + 2 approvals + UAT sign-off)
                    main
                      ↓ (auto dry-run)
               validate-prod
                      ↓ (MANUAL APPROVAL GATE — 2 reviewers)
                 deploy-prod → Production Org + release tag
```

## GitHub Secrets Required

Add these in **Settings → Secrets and variables → Actions**:

| Secret name | Description |
|---|---|
| `SFDX_AUTH_URL_QA`   | `sf org display --target-org qa-org --json` sfdxAuthUrl |
| `SFDX_AUTH_URL_UAT`  | `sf org display --target-org uat-org --json` sfdxAuthUrl |
| `SFDX_AUTH_URL_PROD` | `sf org display --target-org prod-org --json` sfdxAuthUrl |

## Production Environment Setup

1. Go to **Settings → Environments → New environment**
2. Name: `production`
3. Enable **Required reviewers** — add 2 reviewers (Tech Lead + DevOps)
4. Set **Deployment branches** → Protected branches only
