# DocETL Auto-Update Guide

This guide explains how to set up automated tracking of DocETL releases and receive notifications when new versions are available.

## Overview

The `auto-update-docetl.yaml` workflow automatically:
- Monitors [ucbepic/docetl](https://github.com/ucbepic/docetl) for new releases
- Compares with current `appVersion` in Chart.yaml
- Sends notifications via GitHub Issues/Pull Requests or email
- Optionally creates pull requests or auto-commits chart updates

## Prerequisites

- GitHub repository with DocETL Helm chart
- GitHub Actions enabled
- Appropriate permissions configured

## Notification Methods

### Method 1: GitHub Native (Recommended)

Uses GitHub Issues and Pull Requests for notifications - **zero configuration required**.

**How it works:**
1. Workflow runs daily (or on-demand)
2. If new DocETL release found:
   - Creates Issue with changelog and release notes
   - @mentions repository watchers
   - Creates Pull Request with chart version bump (optional)

**Advantages:**
- No secrets/credentials needed
- Native GitHub notifications (email, mobile, web)
- Integrated with GitHub workflow
- Audit trail in Issues/PRs

**Setup:** Enable GitHub Actions (already done if workflows run)

### Method 2: Direct Email (Optional)

Sends email directly to external addresses via SMTP.

**When to use:**
- Need to notify external stakeholders
- Want emails to non-GitHub addresses
- Prefer direct email over GitHub notifications

**Setup:** See [Email Configuration](#email-configuration) below

## Workflow Configuration

The workflow is located at `.github/workflows/auto-update-docetl.yaml`.

### Key Configuration Options

Edit the workflow file to customize:

```yaml
on:
  schedule:
    - cron: '0 0 * * *'  # Daily at midnight UTC
  workflow_dispatch:      # Manual trigger
```

**Cron schedule examples:**
- `0 0 * * *` - Daily at midnight UTC
- `0 0 * * 1` - Weekly on Monday
- `0 */6 * * *` - Every 6 hours
- `0 9 * * 1-5` - Weekdays at 9 AM UTC

### GitHub Actions Variables

Configure in repository Settings → Secrets and variables → Actions → Variables:

| Variable | Description | Required | Default |
|----------|-------------|----------|---------|
| `AUTO_UPDATE_EMAIL` | Email address to notify | No | - |
| `AUTO_UPDATE_ENABLED` | Enable auto-update (`true`/`false`) | No | `true` |
| `AUTO_MERGE` | Auto-commit and tag without PR (`true`/`false`) | No | `false` |

### GitHub Actions Secrets

Configure in repository Settings → Secrets and variables → Actions → Secrets:

| Secret | Description | Required | When Needed |
|--------|-------------|----------|-------------|
| `SMTP_HOST` | SMTP server hostname | Only for email | Method 2 (Direct Email) |
| `SMTP_PORT` | SMTP server port | Only for email | Method 2 (Direct Email) |
| `SMTP_USERNAME` | SMTP username | Only for email | Method 2 (Direct Email) |
| `SMTP_PASSWORD` | SMTP password | Only for email | Method 2 (Direct Email) |
| `SMTP_FROM` | Sender email address | Only for email | Method 2 (Direct Email) |

## Setup Instructions

### Option A: GitHub Native Only (Recommended)

**No configuration needed!** The workflow works out-of-the-box.

1. Verify workflow file exists: `.github/workflows/auto-update-docetl.yaml`

2. Enable GitHub Actions if not already enabled:
   - Go to repository Settings → Actions → General
   - Ensure "Allow all actions" is selected

3. Grant workflow permissions:
   - Settings → Actions → General → Workflow permissions
   - Select "Read and write permissions"
   - Check "Allow GitHub Actions to create and approve pull requests"

4. Test the workflow:
   ```bash
   gh workflow run auto-update-docetl.yaml
   gh run watch
   ```

5. Check for notification:
   - Go to Issues tab
   - Look for "New DocETL Release: vX.Y.Z" (if update available)

**That's it!** You'll receive GitHub notifications when new DocETL versions are released.

### Option B: Add Direct Email Notifications

If you want emails sent to external addresses:

1. **Get SMTP credentials:**

   **Gmail:**
   - Enable 2FA: https://myaccount.google.com/security
   - Create App Password: https://myaccount.google.com/apppasswords
   - Use `smtp.gmail.com:587` as SMTP server

   **Outlook/Office365:**
   - Use `smtp.office365.com:587`
   - Use your email and password

   **SendGrid:**
   - Get API key: https://app.sendgrid.com/settings/api_keys
   - Use `smtp.sendgrid.net:587`
   - Username: `apikey`, Password: Your API key

   **AWS SES:**
   - Get SMTP credentials: https://console.aws.amazon.com/ses/
   - Use region-specific endpoint (e.g., `email-smtp.us-east-1.amazonaws.com:587`)

2. **Add GitHub Secrets:**

   Via GitHub UI (Settings → Secrets and variables → Actions → New repository secret):
   - `SMTP_HOST`: `smtp.gmail.com`
   - `SMTP_PORT`: `587`
   - `SMTP_USERNAME`: `your-email@gmail.com`
   - `SMTP_PASSWORD`: `your-app-password`
   - `SMTP_FROM`: `your-email@gmail.com`

   Or via CLI:
   ```bash
   gh secret set SMTP_HOST --body "smtp.gmail.com"
   gh secret set SMTP_PORT --body "587"
   gh secret set SMTP_USERNAME --body "your-email@gmail.com"
   gh secret set SMTP_PASSWORD --body "your-app-password"
   gh secret set SMTP_FROM --body "your-email@gmail.com"
   ```

3. **Add GitHub Variable for recipient:**

   ```bash
   gh variable set AUTO_UPDATE_EMAIL --body "recipient@example.com"
   ```

4. **Test email delivery:**

   ```bash
   gh workflow run auto-update-docetl.yaml
   ```

   Check your email for "New DocETL Release" notification.

## Behavior Modes

### Review Mode (Default)

**Configuration:**
```bash
gh variable set AUTO_MERGE --body "false"
```

**Behavior:**
1. New release detected → Issue created
2. Manual review of changes
3. Manual chart update and release

**Use when:**
- Want to review changes before updating
- Need approval process
- Testing auto-update setup

### Auto-Commit Mode

**Configuration:**
```bash
gh variable set AUTO_MERGE --body "true"
```

**Behavior:**
1. New release detected → Issue created
2. Chart automatically updated (appVersion bumped)
3. Git tag created automatically
4. Release workflow triggered

**Use when:**
- Trust upstream releases
- Want fully automated updates
- Have good testing in place

**Warning:** Auto-commit mode requires robust testing and monitoring.

### Pull Request Mode

**Configuration:** Edit workflow file:

```yaml
# Change line ~240
- name: Commit and push changes
  run: |
    # Comment out auto-commit section
    # git commit -m "..."
    # git tag ...

    # Add PR creation instead
    gh pr create --title "chore: update DocETL to $NEW_VERSION" \
      --body "Auto-generated PR for DocETL $NEW_VERSION" \
      --base main \
      --head auto-update-docetl-$NEW_VERSION
```

**Behavior:**
1. New release detected → Issue created
2. Branch created with chart updates
3. Pull Request opened for review
4. Manual merge triggers release

**Use when:**
- Want automated PRs but manual approval
- Need CI/CD checks before merge
- Balancing automation and control

## Manual Trigger

Manually check for updates without waiting for schedule:

```bash
# Trigger workflow
gh workflow run auto-update-docetl.yaml

# Watch execution
gh run watch

# View results
gh run list --workflow=auto-update-docetl.yaml
```

## Monitoring

### Check Workflow Status

```bash
# List recent runs
gh run list --workflow=auto-update-docetl.yaml

# View specific run
gh run view <run-id>

# View logs
gh run view <run-id> --log
```

### Notification Examples

**GitHub Issue created:**
```
Title: 🚀 New DocETL Release: v0.3.0

Body:
A new version of DocETL is available!

Current version: v0.2.6
New version: v0.3.0
Released: 2024-01-15

Changelog:
- Feature: Add batch processing support
- Fix: Improve error handling
- Performance: Optimize LLM calls

Release Notes: https://github.com/ucbepic/docetl/releases/tag/v0.3.0

@repository-maintainers
```

**Email notification:**
```
Subject: 🚀 New DocETL Release Available: v0.3.0

A new version of DocETL (v0.3.0) was released on 2024-01-15.
Current chart version: v0.2.6

[View Release Notes]
[View Repository]
```

## Troubleshooting

### Workflow Not Running

**Check schedule:**
```yaml
on:
  schedule:
    - cron: '0 0 * * *'
```

**Verify workflow is enabled:**
- Settings → Actions → Workflows → auto-update-docetl.yaml
- Ensure not disabled

**Check recent runs:**
```bash
gh run list --workflow=auto-update-docetl.yaml
```

### No Notifications Received

**GitHub notifications:**
- Check Settings → Notifications → Actions
- Ensure "Email" is checked
- Check spam folder

**Email notifications:**
- Verify SMTP secrets are set: `gh secret list`
- Check workflow logs: `gh run view --log`
- Test SMTP credentials:
  ```bash
  curl --url 'smtp://smtp.gmail.com:587' \
    --ssl-reqd \
    --mail-from 'your-email@gmail.com' \
    --mail-rcpt 'recipient@example.com' \
    --user 'your-email@gmail.com:your-app-password' \
    --upload-file - <<EOF
  From: your-email@gmail.com
  To: recipient@example.com
  Subject: Test

  Test email
  EOF
  ```

### False Positives

If workflow triggers unnecessarily:

1. **Check version comparison logic** in workflow file
2. **Verify appVersion format** in Chart.yaml matches DocETL releases
3. **Review workflow logs** for version detection issues

### Auto-Commit Fails

**Check permissions:**
- Settings → Actions → General → Workflow permissions
- Must have "Read and write permissions"

**Check branch protection:**
- Settings → Branches → Branch protection rules
- May need to allow Actions to bypass rules

## Advanced Configuration

### Custom Version Detection

Edit workflow to customize version extraction:

```yaml
# Around line 40
NEW_VERSION=$(echo "$LATEST_RELEASE" | jq -r '.tag_name' | sed 's/^v//')

# For different tag formats:
# NEW_VERSION=$(echo "$LATEST_RELEASE" | jq -r '.tag_name')  # Keep 'v' prefix
# NEW_VERSION=$(echo "$LATEST_RELEASE" | jq -r '.name')      # Use release name
```

### Filter Pre-releases

Ignore pre-release versions:

```yaml
# Around line 40
LATEST_RELEASE=$(curl -s https://api.github.com/repos/ucbepic/docetl/releases | \
  jq -r '[.[] | select(.prerelease == false)] | .[0]')
```

### Customize Notifications

Edit notification templates in workflow file:

```yaml
# Issue body (around line 75)
A new version of DocETL is available!

**Current version:** $CURRENT_VERSION
**New version:** $NEW_VERSION
**Released:** $(date)

## Changelog
$CHANGELOG

## Actions Required
- [ ] Review release notes
- [ ] Update chart
- [ ] Test deployment
```

## Security Considerations

1. **SMTP Credentials:**
   - Always use GitHub Secrets (never commit credentials)
   - Use app-specific passwords (not account passwords)
   - Rotate credentials regularly

2. **Auto-Commit Mode:**
   - Only enable for trusted upstream repositories
   - Implement testing before auto-merge
   - Monitor for unexpected changes

3. **Workflow Permissions:**
   - Grant minimum necessary permissions
   - Review permission requirements regularly

## Comparison with Obsidian Auto-Update

DocETL auto-update differs from Obsidian:

| Aspect | Obsidian | DocETL |
|--------|----------|--------|
| Upstream | LinuxServer.io Docker releases | GitHub releases (ucbepic/docetl) |
| Version format | `v1.10.6-ls102` | `v0.3.0` |
| Update frequency | Docker image updates | Application releases |
| Complexity | Docker tag tracking | GitHub API |

## Support

For issues with auto-update workflow:

1. Check [Troubleshooting](#troubleshooting) section
2. Review [workflow logs](https://github.com/X-tructure/helm-charts/actions)
3. Open [GitHub Issue](https://github.com/X-tructure/helm-charts/issues)

For DocETL application issues:
- [DocETL Issues](https://github.com/ucbepic/docetl/issues)
- [DocETL Documentation](https://ucbepic.github.io/docetl)
