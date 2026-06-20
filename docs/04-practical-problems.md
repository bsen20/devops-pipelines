# 04 — Practical Problems

> 15 real-world industry problem statements with complete GitHub Actions pipeline solutions. Each problem is based on actual scenarios from production environments.

---

## How to Use

1. Read the problem statement
2. Try designing the pipeline yourself
3. Compare with the provided solution
4. Deploy to your own repo and test it

---

## Problem 1: Pull Request Quality Gates

**Context:** Your team of 20 developers merges 15-20 PRs daily to a monorepo. Bugs regularly slip through because reviews miss edge cases.

**Requirements:**
- Every PR must pass linting, unit tests, and type checking
- Test coverage must not decrease below 80%
- No secrets or API keys in code
- PRs with changes to `services/payments/` require additional security scanning
- PR author must be able to request a re-run without pushing new code

<details>
<summary>Solution</summary>

```yaml
name: PR Quality Gates

on:
  pull_request:
    types: [opened, synchronize, reopened]

concurrency:
  group: pr-${{ github.ref }}
  cancel-in-progress: true

env:
  NODE_VERSION: "20"
  MIN_COVERAGE: 80

jobs:
  lint:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: ${{ env.NODE_VERSION }}
          cache: 'npm'
      - run: npm ci
      - run: npm run lint
      - run: npm run type-check

  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: ${{ env.NODE_VERSION }}
          cache: 'npm'
      - run: npm ci
      - run: npm test -- --coverage
      - name: Check coverage
        run: |
          COVERAGE=$(npx jest --coverage --silent 2>&1 | grep "Lines" | awk '{print $3}' | sed 's/%//')
          echo "Coverage: $COVERAGE%"
          if [ "$(echo "$COVERAGE < $MIN_COVERAGE" | bc)" -eq 1 ]; then
            echo "Coverage $COVERAGE% is below threshold $MIN_COVERAGE%"
            exit 1
          fi

  secret-scan:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: aquasecurity/trivy-action@master
        with:
          scan-type: "fs"
          scan-ref: "."
          format: "sarif"
          output: "trivy-results.sarif"
          scanners: "secret"

  payments-security:
    if: contains(github.event.pull_request.changed_files, 'services/payments/')
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - run: |
          echo "Payment service changed — running enhanced security scan"
      - uses: github/codeql-action/analyze@v3
        with:
          category: "/language:javascript"
```
</details>

---

## Problem 2: Multi-Environment Deployment Pipeline

**Context:** Your SaaS product has dev, staging, and production environments. Dev deploys automatically on PR merge to main. Staging deploys on tag creation. Production deploys require manual approval from the release manager.

**Requirements:**
- Dev: auto-deploy on merge to main (every commit)
- Staging: deploy when a `v*` tag is pushed
- Production: deploy only after manual approval, requires 2 reviewers
- Rolling back must be possible by reverting the commit
- Each environment uses different secrets

<details>
<summary>Solution</summary>

```yaml
name: Multi-Environment Deploy

on:
  push:
    branches: [main]
    tags: ['v*']

env:
  REGISTRY: ghcr.io
  IMAGE_NAME: ${{ github.repository }}

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - run: docker build -t ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}:${{ github.sha }} .
      - run: docker push ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}:${{ github.sha }}

  deploy-dev:
    needs: build
    if: github.ref == 'refs/heads/main'
    runs-on: ubuntu-latest
    environment:
      name: development
      url: https://dev.app.example.com
    steps:
      - run: echo "Deploying ${{ github.sha }} to dev"
      - run: |
          curl -X POST https://api.example.com/deploy \
            -H "Authorization: Bearer ${{ secrets.DEV_API_KEY }}" \
            -d '{"version":"${{ github.sha }}","env":"dev"}'

  deploy-staging:
    needs: build
    if: startsWith(github.ref, 'refs/tags/v')
    runs-on: ubuntu-latest
    environment:
      name: staging
      url: https://staging.app.example.com
    steps:
      - run: echo "Deploying ${{ github.sha }} (${{ github.ref_name }}) to staging"

  deploy-production:
    needs: [build, deploy-staging]
    if: startsWith(github.ref, 'refs/tags/v')
    runs-on: ubuntu-latest
    environment:
      name: production
      url: https://app.example.com
    steps:
      - run: echo "Deploying ${{ github.ref_name }} to production"
```
</details>

---

## Problem 3: Docker Build & Push with Vulnerability Scanning

**Context:** Your team builds and publishes Docker images to a private registry. Security team requires zero critical vulnerabilities in production images. The pipeline must block deployment if critical CVEs are found.

**Requirements:**
- Build optimized multi-stage Docker image
- Push to GitHub Container Registry
- Scan image for vulnerabilities
- Block push if critical or high severity CVEs found
- Tag with both `latest` and commit SHA
- Sign images with Cosign for supply chain security

<details>
<summary>Solution</summary>

```yaml
name: Docker Build & Push

on:
  push:
    branches: [main]
    tags: ['v*']

env:
  REGISTRY: ghcr.io
  IMAGE_NAME: ${{ github.repository }}

jobs:
  build-and-scan:
    runs-on: ubuntu-latest
    permissions:
      contents: read
      packages: write
      id-token: write

    steps:
      - uses: actions/checkout@v4

      - uses: docker/setup-buildx-action@v3

      - name: Log in to registry
        uses: docker/login-action@v3
        with:
          registry: ${{ env.REGISTRY }}
          username: ${{ github.actor }}
          password: ${{ secrets.GITHUB_TOKEN }}

      - name: Build and push
        uses: docker/build-push-action@v5
        with:
          push: true
          tags: |
            ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}:${{ github.sha }}
            ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}:latest
          cache-from: type=gha
          cache-to: type=gha,mode=max

      - name: Scan image
        id: scan
        uses: aquasecurity/trivy-action@master
        with:
          image-ref: ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}:${{ github.sha }}
          format: 'sarif'
          output: 'trivy-results.sarif'
          severity: 'CRITICAL,HIGH'
          exit-code: '1'

      - name: Sign image with Cosign
        uses: sigstore/cosign-installer@v3
      - run: |
          cosign sign --yes ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}:${{ github.sha }}
```
</details>

---

## Problem 4: Database Migration Pipeline

**Context:** Your application uses PostgreSQL with Alembic/Prisma migrations. Applying them at the wrong time causes downtime. You need a safe, automated migration pipeline.

**Requirements:**
- Run migrations as a separate step before deployment
- Never run migrations automatically on production (require manual approval)
- Rollback migration if deployment fails
- Run read-only migrations on PR (validate they work)
- Notify team if migration takes longer than 5 minutes

<details>
<summary>Solution</summary>

```yaml
name: Database Migration Pipeline

on:
  pull_request:
    paths:
      - 'prisma/**'
      - 'migrations/**'
  push:
    branches: [main]

jobs:
  validate-migration:
    if: github.event_name == 'pull_request'
    runs-on: ubuntu-latest

    services:
      postgres:
        image: postgres:16
        env:
          POSTGRES_DB: testdb
          POSTGRES_PASSWORD: testpass
        ports:
          - 5432:5432
        options: >-
          --health-cmd pg_isready
          --health-interval 10s
          --health-timeout 5s
          --health-retries 5

    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: '20'
      - run: npm ci

      - name: Run migration (dry run)
        run: npx prisma migrate resolve --applied
        env:
          DATABASE_URL: postgres://postgres:testpass@localhost:5432/testdb

      - name: Apply migration (read-only check)
        run: npx prisma migrate deploy
        env:
          DATABASE_URL: postgres://postgres:testpass@localhost:5432/testdb

  apply-migration:
    if: github.ref == 'refs/heads/main'
    runs-on: ubuntu-latest
    environment:
      name: production

    steps:
      - uses: actions/checkout@v4

      - name: Run pre-deploy backup
        run: |
          pg_dump $DATABASE_URL > pre-migration-backup.sql
        env:
          DATABASE_URL: ${{ secrets.PROD_DATABASE_URL }}

      - name: Apply migration with timeout
        run: |
          timeout 300 npx prisma migrate deploy
        env:
          DATABASE_URL: ${{ secrets.PROD_DATABASE_URL }}

      - name: Verify migration
        run: npx prisma migrate status
        env:
          DATABASE_URL: ${{ secrets.PROD_DATABASE_URL }}

      - name: Rollback on failure
        if: failure()
        run: |
          psql $DATABASE_URL < pre-migration-backup.sql
        env:
          DATABASE_URL: ${{ secrets.PROD_DATABASE_URL }}
```
</details>

---

## Problem 5: Monorepo — Selective CI

**Context:** Your monorepo contains 10 microservices. Every PR triggers all 10 services' tests, taking 45 minutes. Most PRs only change one service.

**Requirements:**
- Only test services that changed
- If a shared library changes, test ALL services that depend on it
- Run lint on the entire repo regardless (cheap check)
- Timeout per service test at 10 minutes
- Report which services were tested in the PR comment

<details>
<summary>Solution</summary>

```yaml
name: Monorepo CI

on:
  pull_request:

jobs:
  detect:
    runs-on: ubuntu-latest
    outputs:
      services: ${{ steps.filter.outputs.changes }}

    steps:
      - uses: actions/checkout@v4

      - uses: dorny/paths-filter@v3
        id: filter
        with:
          filters: |
            auth:
              - 'services/auth/**'
            payments:
              - 'services/payments/**'
            shipping:
              - 'services/shipping/**'
            shared:
              - 'packages/**'

    # If shared libs change → test everything
    - name: Check shared changes
      id: check-shared
      run: |
        if echo '${{ steps.filter.outputs.changes }}' | grep -q 'shared'; then
          echo "All services affected by shared lib change"
          echo "affected=all" >> $GITHUB_OUTPUT
        fi

  lint:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - run: npm ci
      - run: npm run lint

  test:
    needs: [detect, lint]
    if: needs.detect.outputs.services != '[]'
    strategy:
      matrix:
        service: ${{ fromJSON(needs.detect.outputs.services) }}
      fail-fast: false

    runs-on: ubuntu-latest
    timeout-minutes: 10

    steps:
      - uses: actions/checkout@v4
      - run: |
          cd services/${{ matrix.service }}
          npm ci
          npm test

  report:
    needs: [detect, test]
    if: always()
    runs-on: ubuntu-latest
    steps:
      - name: Comment PR
        uses: actions/github-script@v7
        with:
          script: |
            const services = '${{ needs.detect.outputs.services }}';
            const message = `## CI Results\n\n**Tested services:** ${services || 'none'}\n**Status:** ${{ needs.test.result }}`;
            github.rest.issues.createComment({
              issue_number: context.issue.number,
              owner: context.repo.owner,
              repo: context.repo.repo,
              body: message
            });
```
</details>

---

## Problem 6: Scheduled Security Compliance Scan

**Context:** Your SOC 2 compliance requires weekly vulnerability scans of all production dependencies. Results must be archived for audit. Critical findings must be auto-ticketed.

**Requirements:**
- Run every Sunday at midnight
- Scan all npm/pip/go dependencies
- Archive scan results as build artifacts (90-day retention)
- Create GitHub issue if critical vulnerability found
- Send Slack alert for critical findings
- Generate SBOM (Software Bill of Materials)

<details>
<summary>Solution</summary>

```yaml
name: Weekly Security Compliance Scan

on:
  schedule:
    - cron: '0 0 * * 0'  # Sunday midnight
  workflow_dispatch:       # Manual trigger

jobs:
  security-scan:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Generate SBOM
        uses: anchore/sbom-action@v0
        with:
          path: ./
          format: spdx-json
          output-file: sbom-report.json

      - name: Dependency scan
        uses: aquasecurity/trivy-action@master
        with:
          scan-type: 'fs'
          scan-ref: '.'
          format: 'sarif'
          output: 'dependency-scan.sarif'
          severity: 'CRITICAL,HIGH'

      - name: Upload scan results
        uses: actions/upload-artifact@v4
        with:
          name: compliance-scan-${{ github.run_id }}
          path: |
            sbom-report.json
            dependency-scan.sarif
          retention-days: 90

      - name: Check for critical findings
        id: check-critical
        run: |
          if grep -q '"severity": "CRITICAL"' dependency-scan.sarif; then
            echo "critical_found=true" >> $GITHUB_OUTPUT
          else
            echo "critical_found=false" >> $GITHUB_OUTPUT
          fi

      - name: Create GitHub issue for critical findings
        if: steps.check-critical.outputs.critical_found == 'true'
        uses: actions/github-script@v7
        with:
          script: |
            github.rest.issues.create({
              owner: context.repo.owner,
              repo: context.repo.repo,
              title: 'Critical vulnerability found - ${{ github.run_id }}',
              body: 'A critical vulnerability was found in the weekly compliance scan.\n\nSee scan: ${{ github.run_id }}\n\nAction: Resolve within 7 days per SLA.',
              labels: ['security', 'critical', 'compliance']
            })

      - name: Send Slack alert
        if: steps.check-critical.outputs.critical_found == 'true'
        uses: slackapi/slack-github-action@v1
        with:
          payload: |
            {
              "channel": "#security-alerts",
              "text": "CRITICAL: Vulnerability found in ${{ github.repository }}. Scan: ${{ github.server_url }}/${{ github.repository }}/actions/runs/${{ github.run_id }}"
            }
        env:
          SLACK_WEBHOOK_URL: ${{ secrets.SLACK_WEBHOOK_URL }}
```
</details>

---

## Problem 7: Blue/Green Deployment with Automated Rollback

**Context:** Your e-commerce platform handles $1M/day in transactions. A bad deploy could cost $40k/hour in lost revenue. You need zero-downtime deployments with automated rollback.

**Requirements:**
- Deploy new version alongside existing (blue/green)
- Run health checks on new version before switching traffic
- Monitor error rates for 5 minutes after switch
- Auto-rollback if error rate exceeds 1%
- Notify on-call via PagerDuty on rollback

<details>
<summary>Solution</summary>

```yaml
name: Blue/Green Deploy

on:
  push:
    branches: [main]

jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Determine deployment color
        id: color
        run: |
          CURRENT=$(kubectl get service app -n production -o jsonpath='{.spec.selector.version}')
          if [ "$CURRENT" == "blue" ]; then
            echo "new=v2" >> $GITHUB_OUTPUT
            echo "old=v1" >> $GITHUB_OUTPUT
            echo "target=blue" >> $GITHUB_OUTPUT
          else
            echo "new=v1" >> $GITHUB_OUTPUT
            echo "old=v2" >> $GITHUB_OUTPUT
            echo "target=green" >> $GITHUB_OUTPUT
          fi

      - name: Deploy to inactive environment
        run: |
          kubectl set image deployment/app-${{ steps.color.outputs.target }} \
            app=${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}:${{ github.sha }} \
            -n production
          kubectl rollout status deployment/app-${{ steps.color.outputs.target }} \
            -n production --timeout=300s

      - name: Health check
        run: |
          for i in {1..30}; do
            STATUS=$(curl -s -o /dev/null -w "%{http_code}" http://${{ steps.color.outputs.target }}.app.internal/health)
            if [ "$STATUS" == "200" ]; then
              echo "Health check passed"
              exit 0
            fi
            sleep 10
          done
          echo "Health check failed"
          exit 1

      - name: Switch traffic
        run: |
          kubectl patch service app -n production -p \
            '{"spec":{"selector":{"version":"${{ steps.color.outputs.new }}"}}}'

      - name: Monitor for 5 minutes
        run: |
          sleep 300
          ERROR_RATE=$(curl -s http://metrics.app.internal/api/v1/query?query=error_rate)
          if (( $(echo "$ERROR_RATE > 1.0" | bc -l) )); then
            echo "Error rate $ERROR_RATE% exceeds threshold. Rolling back."
            kubectl patch service app -n production -p \
              '{"spec":{"selector":{"version":"${{ steps.color.outputs.old }}"}}}'
            exit 1
          fi

      - name: Notify on rollback
        if: failure()
        run: |
          curl -X POST https://events.pagerduty.com/v2/enqueue \
            -H "Content-Type: application/json" \
            -d '{
              "routing_key": "${{ secrets.PAGERDUTY_KEY }}",
              "event_action": "trigger",
              "payload": {
                "summary": "Blue/green deploy rolled back",
                "severity": "critical",
                "source": "GitHub Actions"
              }
            }'
```
</details>

---

## Problem 8: Infrastructure as Code Pipeline (Terraform)

**Context:** Your infrastructure is managed with Terraform. Every change must be planned, reviewed, and applied safely. Terraform state is stored remotely with locking.

**Requirements:**
- On PR: run `terraform plan` and comment results on PR
- On merge to main: run `terraform apply`
- State stored in S3 with DynamoDB locking
- Pin Terraform and provider versions
- Validate formatting with `terraform fmt`

<details>
<summary>Solution</summary>

```yaml
name: Terraform Pipeline

on:
  pull_request:
    paths:
      - 'infra/**'
      - '*.tf'
  push:
    branches: [main]
    paths:
      - 'infra/**'
      - '*.tf'

env:
  TF_VERSION: '1.9.0'
  TF_ROOT: './infra'

jobs:
  validate:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - uses: hashicorp/setup-terraform@v3
        with:
          terraform_version: ${{ env.TF_VERSION }}

      - name: Terraform fmt
        run: terraform fmt -check -recursive
        working-directory: ${{ env.TF_ROOT }}

      - name: Terraform init
        run: terraform init
        working-directory: ${{ env.TF_ROOT }}

      - name: Terraform validate
        run: terraform validate
        working-directory: ${{ env.TF_ROOT }}

  plan:
    needs: validate
    if: github.event_name == 'pull_request'
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - uses: hashicorp/setup-terraform@v3
        with:
          terraform_version: ${{ env.TF_VERSION }}

      - name: Terraform init
        run: terraform init
        working-directory: ${{ env.TF_ROOT }}
        env:
          AWS_ACCESS_KEY_ID: ${{ secrets.AWS_ACCESS_KEY_ID }}
          AWS_SECRET_ACCESS_KEY: ${{ secrets.AWS_SECRET_ACCESS_KEY }}

      - name: Terraform plan
        id: plan
        run: terraform plan -no-color
        working-directory: ${{ env.TF_ROOT }}
        continue-on-error: true
        env:
          AWS_ACCESS_KEY_ID: ${{ secrets.AWS_ACCESS_KEY_ID }}
          AWS_SECRET_ACCESS_KEY: ${{ secrets.AWS_SECRET_ACCESS_KEY }}

      - name: Comment PR
        uses: actions/github-script@v7
        with:
          script: |
            const output = `#### Terraform Plan
            <details><summary>Show Plan</summary>
            \`\`\`
            ${{ steps.plan.outputs.stdout }}
            \`\`\`
            </details>`;
            github.rest.issues.createComment({
              issue_number: context.issue.number,
              owner: context.repo.owner,
              repo: context.repo.repo,
              body: output
            });

  apply:
    needs: validate
    if: github.ref == 'refs/heads/main' && github.event_name == 'push'
    runs-on: ubuntu-latest
    environment:
      name: production

    steps:
      - uses: actions/checkout@v4

      - uses: hashicorp/setup-terraform@v3
        with:
          terraform_version: ${{ env.TF_VERSION }}

      - name: Terraform init
        run: terraform init
        working-directory: ${{ env.TF_ROOT }}
        env:
          AWS_ACCESS_KEY_ID: ${{ secrets.AWS_ACCESS_KEY_ID }}
          AWS_SECRET_ACCESS_KEY: ${{ secrets.AWS_SECRET_ACCESS_KEY }}

      - name: Terraform apply
        run: terraform apply -auto-approve
        working-directory: ${{ env.TF_ROOT }}
        env:
          AWS_ACCESS_KEY_ID: ${{ secrets.AWS_ACCESS_KEY_ID }}
          AWS_SECRET_ACCESS_KEY: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
```
</details>

---

## Problem 9: Mobile App CI/CD (iOS + Android)

**Context:** Your team builds a React Native app for both iOS and Android. Building for both platforms takes 30+ minutes. You need a unified pipeline that handles both.

**Requirements:**
- Build both iOS and Android on every PR (just compile check)
- Full build + sign on push to main
- Distribute to TestFlight (iOS) and Play Console (Android) internal testing
- Increment build numbers automatically
- Code sign iOS with a certificate stored in GitHub Secrets

<details>
<summary>Solution</summary>

```yaml
name: Mobile CI/CD

on:
  pull_request:
  push:
    branches: [main]

jobs:
  lint:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: '20'
          cache: 'npm'
      - run: npm ci
      - run: npm run lint

  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: '20'
          cache: 'npm'
      - run: npm ci
      - run: npm test

  android-build:
    needs: [lint, test]
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: '20'
      - uses: actions/setup-java@v4
        with:
          distribution: 'temurin'
          java-version: '17'
      - run: npm ci
      - run: cd android && ./gradlew assembleRelease
      - uses: actions/upload-artifact@v4
        with:
          name: android-release
          path: android/app/build/outputs/apk/release/*.apk

  ios-build:
    needs: [lint, test]
    runs-on: macos-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: '20'
      - run: npm ci
      - run: |
          cd ios
          pod install --repo-update
      - name: Build iOS
        run: |
          xcodebuild -workspace ios/App.xcworkspace \
            -scheme App \
            -configuration Release \
            -archivePath build/App.xcarchive \
            archive
      - uses: actions/upload-artifact@v4
        with:
          name: ios-build
          path: build/App.xcarchive
```
</details>

---

## Problem 10: Multi-Branch Hotfix Pipeline

**Context:** You maintain v1.x (stable) and v2.x (current) releases. A critical security fix must be deployed to BOTH versions immediately, then merged forward to main.

**Requirements:**
- Hotfix branches follow `hotfix/v*.*.*` naming
- Pipeline runs tests and deploys to the matching version
- After deploy, auto-create PR to merge hotfix into next version up
- Tag the release automatically
- Notify security team of the fix

<details>
<summary>Solution</summary>

```yaml
name: Hotfix Pipeline

on:
  push:
    branches:
      - 'hotfix/v*.*.*'

jobs:
  extract-version:
    runs-on: ubuntu-latest
    outputs:
      version: ${{ steps.extract.outputs.version }}
    steps:
      - name: Extract version from branch name
        id: extract
        run: |
          BRANCH=${GITHUB_REF_NAME}
          VERSION=$(echo $BRANCH | sed 's/hotfix\/v//')
          echo "version=$VERSION" >> $GITHUB_OUTPUT

  test:
    needs: extract-version
    runs-on: ubuntu-latest
    strategy:
      matrix:
        node: [18, 20]
    steps:
      - uses: actions/checkout@v4
        with:
          ref: v${{ needs.extract-version.outputs.version }}
      - uses: actions/setup-node@v4
        with:
          node-version: ${{ matrix.node }}
      - run: npm ci
      - run: npm test

  deploy-and-tag:
    needs: [extract-version, test]
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
        with:
          ref: v${{ needs.extract-version.outputs.version }}

      - run: |
          echo "Deploying hotfix to v${{ needs.extract-version.outputs.version }}"

      - name: Tag hotfix release
        run: |
          git config user.name "CI Bot"
          git config user.email "ci@example.com"
          git tag hotfix-v${{ needs.extract-version.outputs.version }}-${{ github.run_number }}
          git push origin --tags

  forward-merge:
    needs: deploy-and-tag
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Create forward-merge PR
        run: |
          NEXT_VERSION=$(( $(echo "${{ needs.extract-version.outputs.version }}" | cut -d. -f1) + 1 ))
          git checkout -b forward-merge/${{ github.ref_name }}
          git push origin forward-merge/${{ github.ref_name }}
      
      - name: Open PR
        uses: actions/github-script@v7
        with:
          script: |
            github.rest.pulls.create({
              owner: context.repo.owner,
              repo: context.repo.repo,
              title: 'Forward-merge hotfix: ${{ github.ref_name }}',
              head: 'forward-merge/${{ github.ref_name }}',
              base: 'main',
              body: 'Automated forward-merge of hotfix ${{ github.ref_name }}'
            });
```
</details>

---

## Problem 11: Performance Regression Detection

**Context:** Your API's p99 latency crept up from 200ms to 2s over 3 months without anyone noticing. You need performance tests as a CI gate.

**Requirements:**
- Run k6 load test on every PR deploy preview
- Compare p50, p95, p99 latency against baseline
- Block PR if p99 increased by more than 20%
- Store performance history as artifact
- Trend analysis for monthly performance reporting

<details>
<summary>Solution</summary>

```yaml
name: Performance Regression

on:
  pull_request:
  push:
    branches: [main]

jobs:
  performance-test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Start app
        run: |
          docker compose -f docker-compose.test.yml up -d
          sleep 10

      - name: Install k6
        run: |
          curl -L https://github.com/grafana/k6/releases/download/v0.50.0/k6-v0.50.0-linux-amd64.tar.gz | tar xz
          sudo mv k6-v0.50.0-linux-amd64/k6 /usr/local/bin/

      - name: Run load test
        id: loadtest
        run: |
          k6 run --out json=k6-results.json tests/load.js
          P99=$(cat k6-results.json | grep '"metric":"http_req_duration"' | grep '"type":"Point"' | tail -1 | jq '.data.attributes["p(99)"]')
          echo "p99=$P99" >> $GITHUB_OUTPUT

      - name: Get baseline
        id: baseline
        run: |
          if [ -f .performance-baseline.json ]; then
            BASELINE=$(cat .performance-baseline.json | jq '.p99')
            echo "value=$BASELINE" >> $GITHUB_OUTPUT
          fi

      - name: Check regression
        if: steps.baseline.outputs.value != ''
        run: |
          THRESHOLD=$(echo "${{ steps.baseline.outputs.value }} * 1.2" | bc)
          if (( $(echo "${{ steps.loadtest.outputs.p99 }} > $THRESHOLD" | bc -l) )); then
            echo "Performance regression detected!"
            echo "Baseline p99: ${{ steps.baseline.outputs.value }}ms"
            echo "Current p99: ${{ steps.loadtest.outputs.p99 }}ms"
            exit 1
          fi

      - name: Update baseline on main
        if: github.ref == 'refs/heads/main'
        run: |
          echo '{"p99": ${{ steps.loadtest.outputs.p99 }}}' > .performance-baseline.json

      - uses: actions/upload-artifact@v4
        with:
          name: k6-results
          path: k6-results.json
```
</details>

---

## Problem 12: GitOps — Auto-Deploy from Image Tag Update

**Context:** Your team follows GitOps: the Git repo is the source of truth. When CI builds a new Docker image, it should update the deployment manifest in a separate GitOps repo, and ArgoCD will sync it to the cluster.

**Requirements:**
- CI builds image and pushes to registry
- CI updates the image tag in `gitops-config` repo
- ArgoCD detects the change and deploys to Kubernetes
- If ArgoCD sync fails, CI pipeline should show as failed
- Support both staging and production environments

<details>
<summary>Solution</summary>

```yaml
name: GitOps Pipeline

on:
  push:
    branches: [main]

env:
  REGISTRY: ghcr.io
  IMAGE_NAME: myapp
  GITOPS_REPO: org/gitops-config
  GITOPS_BRANCH: main

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Build and push
        run: |
          docker build -t ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}:${{ github.sha }} .
          docker push ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}:${{ github.sha }}

  update-gitops:
    needs: build
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
        with:
          repository: ${{ env.GITOPS_REPO }}
          token: ${{ secrets.GITOPS_TOKEN }}

      - name: Update staging manifest
        run: |
          sed -i "s|image: ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}:.*|image: ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}:${{ github.sha }}|" \
            environments/staging/deployment.yaml

      - name: Commit and push
        run: |
          git config user.name "CI Bot"
          git config user.email "ci@example.com"
          git add environments/staging/deployment.yaml
          git commit -m "chore(staging): bump image to ${{ github.sha }}"
          git push

  wait-for-argocd:
    needs: update-gitops
    runs-on: ubuntu-latest
    steps:
      - name: Wait for ArgoCD sync
        run: |
          for i in {1..30}; do
            STATUS=$(curl -s -H "Authorization: Bearer ${{ secrets.ARGOCD_TOKEN }}" \
              "https://argocd.example.com/api/v1/applications/myapp?fields=status.sync.status" | jq -r '.status.sync.status')
            if [ "$STATUS" == "Synced" ]; then
              echo "ArgoCD sync completed"
              exit 0
            fi
            sleep 10
          done
          echo "ArgoCD sync timed out"
          exit 1
```
</details>

---

## Problem 13: Release Drafter — Automated Changelog

**Context:** Your team spends 2 hours every release manually writing changelogs. You need automated release notes generated from PR titles, with proper categorization.

**Requirements:**
- On every PR merge to main, update a draft release
- Categorize changes by type (feature, fix, chore)
- Auto-increment version based on Conventional Commits
- Include PR author and links
- Publish release when maintainer clicks "Publish"

<details>
<summary>Solution</summary>

```yaml
name: Release Drafter

on:
  push:
    branches: [main]

jobs:
  draft-release:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - uses: release-drafter/release-drafter@v6
        with:
          config-name: release-drafter.yml
        env:
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
```

```yaml
# .github/release-drafter.yml
name-template: 'v$NEXT_PATCH_VERSION'
tag-template: 'v$NEXT_PATCH_VERSION'

categories:
  - title: ' Features'
    labels:
      - 'feature'
      - 'enhancement'
  - title: ' Bug Fixes'
    labels:
      - 'fix'
      - 'bugfix'
      - 'bug'
  - title: ' Maintenance'
    labels:
      - 'chore'
      - 'dependencies'
      - 'refactor'
  - title: ' Documentation'
    labels:
      - 'docs'
      - 'documentation'

change-template: '- $TITLE (#$NUMBER) by @$AUTHOR'
change-title-escapes: '\<*_&'

template: |
  ## Changes

  $CHANGES

  ## Contributors

  $CONTRIBUTORS

  **Full Changelog:** https://github.com/$OWNER/$REPOSITORY/compare/$PREVIOUS_TAG...v$NEXT_PATCH_VERSION
```
</details>

---

## Problem 14: Ephemeral Environments (Preview Deployments)

**Context:** Reviewing frontend PRs by reading code is ineffective. You need a live preview environment for every PR that auto-destroys when merged.

**Requirements:**
- Deploy a unique preview environment for each PR
- Environment URL posted as PR comment
- Database seeded with test data
- Auto-destroy environment when PR is closed or merged
- Label PR with environment status

<details>
<summary>Solution</summary>

```yaml
name: Preview Environment

on:
  pull_request:
    types: [opened, synchronize, reopened]
  pull_request_target:
    types: [closed]

jobs:
  deploy-preview:
    if: github.event.action != 'closed'
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Deploy to preview
        run: |
          ENV_NAME="pr-${{ github.event.number }}"
          echo "Deploying $ENV_NAME"

          kubectl create namespace $ENV_NAME --dry-run=client -o yaml | kubectl apply -f -

          sed "s/{{NAMESPACE}}/$ENV_NAME/g" k8s/preview-template.yaml | kubectl apply -f -

          kubectl rollout status deployment/app -n $ENV_NAME --timeout=120s

          URL="https://$ENV_NAME.preview.app.example.com"
          echo "url=$URL" >> $GITHUB_OUTPUT

      - name: Comment PR
        uses: actions/github-script@v7
        with:
          script: |
            github.rest.issues.createComment({
              issue_number: context.issue.number,
              owner: context.repo.owner,
              repo: context.repo.repo,
              body: ` Preview Environment Ready\n\nVisit: ${{ steps.deploy.outputs.url }}\n\n_Environment will be auto-destroyed when PR is closed._`
            });

      - name: Add label
        uses: actions/github-script@v7
        with:
          script: |
            github.rest.issues.addLabels({
              issue_number: context.issue.number,
              owner: context.repo.owner,
              repo: context.repo.repo,
              labels: ['preview-deployed']
            });

  destroy-preview:
    if: github.event.action == 'closed'
    runs-on: ubuntu-latest
    steps:
      - name: Destroy preview
        run: |
          ENV_NAME="pr-${{ github.event.number }}"
          kubectl delete namespace $ENV_NAME --ignore-not-found

      - name: Remove label
        uses: actions/github-script@v7
        with:
          script: |
            github.rest.issues.removeLabel({
              issue_number: context.issue.number,
              owner: context.repo.owner,
              repo: context.repo.repo,
              name: 'preview-deployed'
            }).catch(() => {});

      - name: Comment closure
        uses: actions/github-script@v7
        with:
          script: |
            github.rest.issues.createComment({
              issue_number: context.issue.number,
              owner: context.repo.owner,
              repo: context.repo.repo,
              body: ' Preview environment destroyed.'
            });
```
</details>

---

## Problem 15: Enterprise-Grade Deployment with Canary + Observability

**Context:** Your fintech app processes millions of transactions daily. Deployments must be gradual, monitored, and instantly reversible. The CTO wants a full audit trail of every deployment.

**Requirements:**
- Canary deploy: 2% → 25% → 100% over 30 minutes
- Automated canary analysis (error rate, latency, business metrics)
- Auto-promote if healthy, auto-rollback if degraded
- Full audit log with SHA, approver, duration, metrics
- Send deployment events to SIEM system
- PagerDuty alert if rollback occurs

<details>
<summary>Solution</summary>

```yaml
name: Enterprise Canary Deploy

on:
  push:
    branches: [main]

jobs:
  canary:
    runs-on: ubuntu-latest
    environment:
      name: production
      url: https://app.example.com

    steps:
      - uses: actions/checkout@v4

      - name: Build and push
        run: |
          docker build -t app:${{ github.sha }} .
          docker push app:${{ github.sha }}

      - name: Canary 2% for 5 minutes
        run: |
          kubectl scale deployment/app-stable --replicas=98
          kubectl scale deployment/app-canary --replicas=2
          kubectl set image deployment/app-canary app=app:${{ github.sha }}

          sleep 300

          # Check metrics
          ERROR_RATE=$(curl -s http://prometheus:9090/api/v1/query \
            --data-urlencode 'query=sum(rate(http_requests_total{version="canary",status=~"5.."}[5m]))/sum(rate(http_requests_total{version="canary"}[5m])))')
          if (( $(echo "$ERROR_RATE > 0.001" | bc -l) )); then
            echo "Canary error rate $ERROR_RATE% exceeds 0.1%. Rolling back."
            kubectl scale deployment/app-canary --replicas=0
            kubectl scale deployment/app-stable --replicas=100
            exit 1
          fi

      - name: Canary 25% for 10 minutes
        run: |
          kubectl scale deployment/app-canary --replicas=25
          kubectl scale deployment/app-stable --replicas=75

          sleep 600

          # Check p99 latency
          P99=$(curl -s http://prometheus:9090/api/v1/query \
            --data-urlencode 'query=histogram_quantile(0.99, sum(rate(http_request_duration_seconds_bucket{version="canary"}[10m])) by (le))')
          BASELINE=$(curl -s 'http://prometheus:9090/api/v1/query' \
            --data-urlencode 'query=histogram_quantile(0.99, sum(rate(http_request_duration_seconds_bucket{version="stable"}[10m])) by (le))')
          THRESHOLD=$(echo "$BASELINE * 1.2" | bc)
          if (( $(echo "$P99 > $THRESHOLD" | bc -l) )); then
            echo "p99 latency degraded. Rolling back."
            kubectl scale deployment/app-canary --replicas=0
            kubectl scale deployment/app-stable --replicas=100
            exit 1
          fi

      - name: Promote to 100%
        run: |
          kubectl scale deployment/app-canary --replicas=100
          kubectl scale deployment/app-stable --replicas=0
          kubectl set image deployment/app-stable app=app:${{ github.sha }}
          echo "Deployment ${{ github.sha }} promoted to 100%"

      - name: Audit log
        run: |
          curl -X POST https://siem.example.com/events \
            -H "Authorization: Bearer ${{ secrets.SIEM_TOKEN }}" \
            -d '{
              "event": "deployment",
              "service": "payment-api",
              "version": "${{ github.sha }}",
              "author": "${{ github.actor }}",
              "duration": "'"$(($(date +%s) - $(date -d "${{ github.event.head_commit.timestamp }}" +%s)))"'s",
              "status": "success",
              "run_url": "${{ github.server_url }}/${{ github.repository }}/actions/runs/${{ github.run_id }}"
            }'

      - name: PagerDuty on rollback
        if: failure()
        uses: pagerduty/pagerduty-action@v2
        with:
          pagerduty_api_token: ${{ secrets.PAGERDUTY_API_TOKEN }}
          incident_title: 'Canary deploy rolled back: ${{ github.sha }}'
          incident_body: 'Automated rollback triggered due to metrics degradation'
          incident_severity: critical
```
</details>

---

## Bonus: Pipeline Decision Matrix

| Problem Size | Team Size | Deploy Frequency | Recommended Pattern |
|-------------|-----------|-----------------|-------------------|
| Small (<50 files) | 1-5 | Daily | GitHub Flow + simple CI |
| Medium | 5-20 | Multiple/day | Matrix builds + canary |
| Large (monorepo) | 20-100 | Continuous | Selective CI + GitOps |
| Enterprise | 100+ | Scheduled | GitFlow + blue/green |
