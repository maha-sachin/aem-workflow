# Edge Delivery Services Development and Deployment Workflow

Edge Delivery Services uses a scaled trunk-based development model. Because there is no required build process, customers add their own processes through GitHub Actions and Workflows to run the checks and automation needed during pull requests.

Adobe handles deployment. When code is pushed to any GitHub branch, the AEM Code Sync app automatically syncs it to the Codebus, making it available on both `.aem.page` and `.aem.live`. Changes merged into `main` are automatically deployed to production by the AEM Code Sync GitHub App.

GitHub Actions do not deploy the site. They act as quality gates and handle automation around deployment.

## 1. Connect the repository

1. Create the repository from Adobe's boilerplate template.
2. Install the AEM Code Sync GitHub App.
3. In the app's repository access settings, select **Only select repositories**, not **All repositories**.
4. If using GitHub Enterprise with IP filtering, add [3.227.118.73](https://3.227.118.73/) to the allow list.

### Hosting requirements

- Private GitHub repositories are supported when the AEM Code Sync bot is installed on the repository.
- GitHub Enterprise Cloud is supported.
- GitHub Enterprise Server is not supported because Edge Delivery Services requires access to a public GitHub API.
- Cloud Manager can act as an intermediary for GitHub Enterprise Server, Bitbucket Cloud, GitLab Cloud or self-hosted, and Azure DevOps Cloud. This setup uses a webhook with an API key and signing secret.

### AEM Universal Editor

If authors use AEM with Universal Editor, use the `xwalk` boilerplate. In `fstab.yaml`, replace the default URL with the AEM as a Cloud Service author URL in this form:

```text
https://<aem-author>/bin/franklin.delivery/<owner>/<repository>/main
```

If the project also has custom code on the AEM author instance, that code continues to deploy through Cloud Manager pipelines as usual.

## 2. Branches are your environments

Every branch gets its own site:

- Preview: `https://<branch>--<repo>--<owner>.aem.page/`
- Published: `https://<branch>--<repo>--<owner>.aem.live/`

A separate staging environment is usually unnecessary. To keep feature branches current with `main` before merging, enable these GitHub branch protection rules:

- Require a pull request before merging.
- Require status checks to pass before merging.
- Require branches to be up to date before merging.

Keep these rules in mind:

- Do not use `*.aem.page` as a staging environment for code. It is a preview environment for content.
- Production must always use the `main` branch so caching and push invalidation work as intended.
- A dedicated staging setup is advised only for the CDN layer when applying complex configurations, rewrite rules, or custom edge code.
- The combined length of the branch, repository, and owner names, including the two separators, cannot exceed 63 characters.

## 3. GitHub Actions as quality gates

Two checks are built in:

- The standard boilerplate runs ESLint and Stylelint for each pull request change.
- The AEM Code Sync app runs Google PageSpeed Insights on each pull request to assess performance, accessibility, SEO, and best practices.

### DevOps caveats

- Automatic PSI checks are not enabled by default for private repositories.
- Enabling site authentication also prevents automatic PSI checks from running on pull requests. They can be re-enabled through a feature flag by contacting the Adobe team.
- For private repositories, branch protection rules and CI/CD minutes may require a paid GitHub plan.

You can add your own jobs, such as unit tests and accessibility checks, and mark them as required status checks. This is a minimal example of the pattern; it is not an Adobe-provided workflow:

```yaml
name: PR quality gates

on:
  pull_request:
    branches: [main]

jobs:
  checks:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: 20
      - run: npm ci
      - run: npm run lint
```

### Build tooling

Adobe's guidance is to keep the entire project build-less whenever possible. If the team adds TypeScript or SCSS, the compiled files must be committed to the repository because Code Sync serves what is committed.

> The requirement to commit compiled files is an inference from how Code Sync works, not a documented Adobe rule.

## 4. Automating EDS operations with the Admin API

For pipelines that need to publish, check status, or change configuration, use Admin API keys. Include the key in the `X-Auth-Token` or `Authorization` header of HTTP requests.

- Create keys by sending a `POST` request to the organization, profile, or site configuration endpoint, then assign roles such as `publish`.
- Creating, updating, or deleting keys requires a request authenticated with an administrator role.
- The key value is shown only once, so store it immediately as a GitHub secret.
- Keys expire. Generate replacement keys and update the old secret before expiration.

A useful post-merge step is checking the Code Sync job through the documented Code Sync job endpoint:

```yaml
- run: |
    curl -sf "https://admin.hlx.page/job/$ORG/$SITE/main/code" \
      -H "x-auth-token: ${{ secrets.AEM_ADMIN_API_KEY }}"
```

The Admin API can also publish a resource by copying it from preview to live. This also purges the live CDN and your own CDN, if one is configured.

## 5. Domains, CDN, and edge logic

Cloud Manager's Edge Delivery onboarding checklist covers:

- Adding the site.
- Connecting an external Git repository.
- Adding a domain.
- Adding an SSL certificate.
- Configuring the CDN.
- Setting up push invalidation.
- Going live.

CDN rules, including traffic filters, origin selectors, and redirects, are deployed through Cloud Manager's Edge Delivery Config Pipeline, not through GitHub Actions. The Config Pipeline is available for AEM as a Cloud Service environments and, for Edge environments, through a limited beta program.

If using your own CDN instead of Adobe's managed CDN, configure it on the `aem.live` platform.

### AEM Edge Functions

AEM Edge Functions are the part deployed from GitHub Actions directly. Deploying from a CI/CD system requires an Adobe Developer Console project for API credentials.

The `aio` CLI plugin supports non-interactive runs. Supply the required values as masked environment variables, such as `AEM_EDGE_FUNCTIONS_ADC_CONFIG`, and use the `--batch` flag. For Edge Delivery sites on Adobe Managed CDN, you need the Cloud Manager Deployment Manager product profile.

## Recommended documentation order

1. Adobe's Developer Tutorial.
2. Development Collaboration and Good Practices.
3. Staging & Environments.
4. Admin API Keys.
5. Introduction to Edge Delivery Services in Cloud Manager.

## Setup decisions

The exact setup depends on three choices:

1. Authoring source: Document Authoring, SharePoint/Google, or AEM Universal Editor.
2. Git host.
3. CDN: Adobe-managed or your own.
