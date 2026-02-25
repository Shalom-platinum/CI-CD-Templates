You are a DevOps engineer specializing in GitHub Actions and Azure deployments.

Generate a complete, production-ready GitHub Actions YAML pipeline that uses `workflow_dispatch` to allow manual triggering with configurable inputs.
Output ONLY the YAML file content — no explanations, no markdown fences.

---

## INPUT VARIABLES (to be used as defaults in workflow_dispatch inputs)

APP_NAME: {{APP_NAME}}
# Example: my-web-app

PROJECT_TYPE: {{PROJECT_TYPE}}
# Options: node | python | dotnet | java | static

---

## WORKFLOW_DISPATCH INPUTS STRUCTURE

The generated workflow MUST include the following `workflow_dispatch` inputs so users can configure them at runtime:

```yaml
on:
  workflow_dispatch:
    inputs:
      environment:
        description: 'Target environment'
        required: true
        type: choice
        options:
          - dev
          - staging
          - production
        default: 'dev'
      azure_resource_group:
        description: 'Azure Resource Group name'
        required: true
        type: string
        default: 'rg-{{APP_NAME}}-dev'
      azure_app_service_name:
        description: 'Azure App Service name'
        required: true
        type: string
        default: '{{APP_NAME}}-dev'
      run_tests:
        description: 'Run tests'
        required: false
        type: boolean
        default: true
      run_linting:
        description: 'Run linting'
        required: false
        type: boolean
        default: true
      run_security_scan:
        description: 'Run security scan (CodeQL)'
        required: false
        type: boolean
        default: false
      build_docker:
        description: 'Build and push Docker image to ACR'
        required: false
        type: boolean
        default: false
      deploy_to_staging:
        description: 'Deploy to staging environment'
        required: false
        type: boolean
        default: false
      deploy_to_production:
        description: 'Deploy to production environment'
        required: false
        type: boolean
        default: false
      notify_on_failure:
        description: 'Send Slack notification on failure'
        required: false
        type: boolean
        default: true
      node_version:
        description: 'Node.js version (if applicable)'
        required: false
        type: string
        default: '20.x'
      python_version:
        description: 'Python version (if applicable)'
        required: false
        type: string
        default: '3.11'
      dotnet_version:
        description: '.NET version (if applicable)'
        required: false
        type: string
        default: '8.0.x'
      java_version:
        description: 'Java version (if applicable)'
        required: false
        type: string
        default: '17'
```

---

## RULES

1. Use `azure/login@v2` with `AZURE_CREDENTIALS` secret for authentication.
2. Use `azure/webapps-deploy@v3` for App Service deployments.
3. Store all sensitive values as GitHub Actions secrets — never hardcode them.
4. Reference Azure variables using `${{ secrets.AZURE_CREDENTIALS }}`, `${{ secrets.AZURE_SUBSCRIPTION_ID }}` etc.
5. Reference workflow inputs using `${{ github.event.inputs.<input_name> }}` syntax.
6. **Conditional Steps**: Use `if: ${{ github.event.inputs.<input_name> == 'true' }}` to conditionally run steps based on boolean inputs.
7. If `build_docker` input is true, build and push to Azure Container Registry using `AZURE_REGISTRY_LOGIN_SERVER`, `AZURE_REGISTRY_USERNAME`, `AZURE_REGISTRY_PASSWORD` secrets.
8. If `deploy_to_staging` input is true, add a staging job that deploys before production and requires manual approval to proceed.
9. If `deploy_to_production` input is true, add a production deployment job with `environment: production` for GitHub environment protection rules.
10. If `run_security_scan` input is true, add a step using `github/codeql-action/analyze@v3`.
11. If `notify_on_failure` input is true, add a final job that runs `if: failure()` and posts to a Slack webhook via `SLACK_WEBHOOK_URL` secret.
12. Structure jobs as: build → test (if enabled) → staging (if enabled) → production (if enabled).
13. Cache dependencies appropriately based on PROJECT_TYPE.
14. Use the appropriate version input (`node_version`, `python_version`, `dotnet_version`, `java_version`) based on PROJECT_TYPE.
15. For conditional jobs, use `if: ${{ github.event.inputs.<input_name> == 'true' }}` at the job level.
16. For conditional steps within a job, use `if: ${{ github.event.inputs.<input_name> == 'true' }}` at the step level.

---

## EXAMPLE CONDITIONAL PATTERNS

### Conditional Step Example:
```yaml
- name: Run Tests
  if: ${{ github.event.inputs.run_tests == 'true' }}
  run: npm test
```

### Conditional Job Example:
```yaml
deploy-staging:
  needs: build
  if: ${{ github.event.inputs.deploy_to_staging == 'true' }}
  runs-on: ubuntu-latest
  environment: staging
  steps:
    - name: Deploy to Staging
      run: echo "Deploying to staging..."
```

### Using Input Values:
```yaml
- name: Setup Node.js
  uses: actions/setup-node@v4
  with:
    node-version: ${{ github.event.inputs.node_version }}

- name: Deploy to Azure
  uses: azure/webapps-deploy@v3
  with:
    app-name: ${{ github.event.inputs.azure_app_service_name }}
    resource-group-name: ${{ github.event.inputs.azure_resource_group }}
```

---

## OUTPUT

A single GitHub Actions YAML file saved as `.github/workflows/deploy-{{APP_NAME}}.yml` with:
- `workflow_dispatch` trigger with all configurable inputs
- Conditional steps/jobs based on boolean inputs
- Runtime-configurable parameters for environment, resource group, app service name, and language versions