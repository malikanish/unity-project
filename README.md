# Unity Android CI/CD Pipeline

Automated CI/CD pipeline for Unity Android builds using GitHub Actions with APK/AAB generation, artifact storage, and Slack notifications.

##  Pipeline Architecture
```
Push to main / Manual Trigger
         ↓
┌────────────────────────────┐
│   Checkout & Setup         │
│   • Git LFS                │
│   • Cache Unity Library    │
│   • Free Disk Space        │
└────────────┬───────────────┘
             ↓
┌────────────────────────────┐
│   Build Phase              │
│   • APK (Direct Install)   │
│   • AAB (Play Store)       │
└────────────┬───────────────┘
             ↓
┌────────────────────────────┐
│   Upload & Notify          │
│   • GitHub Artifacts       │
│   • Slack Notification     │
└────────────────────────────┘
```

**Workflow File**: `.github/workflows/android-build.yml`

##  Technical Stack

- **Runner**: ubuntu-latest
- **Unity Version**: 2022.3.20f1 (LTS)
- **Build Mode**: Batchmode (headless)
- **Output**: `build/Android/`
- **Artifacts**: 30-day retention

##  Required GitHub Secrets

| Secret | Description |
|--------|-------------|
| `UNITY_LICENSE` | Unity license file (.ulf) content |
| `UNITY_EMAIL` | Unity account email |
| `UNITY_PASSWORD` | Unity account password |
| `ANDROID_KEYSTORE` | Base64 encoded keystore |
| `ANDROID_KEYSTORE_PASS` | Keystore password |
| `ANDROID_KEY_ALIAS` | Key alias name |
| `ANDROID_KEY_ALIAS_PASS` | Key alias password |
| `SLACK_WEBHOOK_URL` | Slack webhook (optional) |

**Setup**: `Repository Settings → Secrets and variables → Actions → New repository secret`

##  Quick Setup

### 1. Configure Secrets
Add all 8 secrets to your GitHub repository settings.

### 2. Trigger Build
```bash
# Automatic: Push to main
git push origin main

# Manual: Actions tab → Run workflow
```

### 3. Download Artifacts
Navigate to: `Actions → Workflow Run → Artifacts`

##  Performance

| Metric | First Build | Cached Build |
|--------|-------------|--------------|
| Total Time | ~30 min | ~18 min |
| Cache Savings | - | 40% faster |

**Optimizations**:
- Unity Library caching
- Disk space cleanup (~14GB freed)
- Parallel artifact uploads

##  Slack Notifications

**Success**: ✅ Status, repo, branch, commit, artifact link  
**Failure**: ❌ Status, error context, logs link

**Setup Webhook**:
1. https://api.slack.com/apps → Create App
2. Enable "Incoming Webhooks"
3. Copy URL to `SLACK_WEBHOOK_URL` secret

##  Design Decisions

### Why Cache Unity Library?
Reduces build time by 40% (30min → 18min) by reusing unchanged dependencies.

### Why Build Both APK & AAB?
- **APK**: Direct device testing
- **AAB**: Google Play Store requirement

### Why Free Disk Space?
Unity builds need significant storage. Cleanup prevents out-of-space failures.

### Why Separate Artifacts?
Clear organization, selective downloads, version tracking via commit SHA.

### Why continue-on-error for Slack?
Build succeeds even if Slack fails - notifications are optional, not critical.

##  Troubleshooting

**License Failed**: Verify `UNITY_LICENSE` contains complete .ulf content  
**Keystore Error**: Check base64 encoding and passwords  
**Build Failed**: Test local build first, check Unity version match  
**No Slack**: Verify webhook URL and channel access

##  Build Success Evidence

Latest successful build:
- ✅ Status: Success
- ⏱️ Duration: 15m 8s
- 📦 APK: 25.4 MB
- 📦 AAB: 25.5 MB
- 🔗 Commit: f16e026

## ⏱ Completion Time

- **Pipeline Setup**: 2-3 hours
- **First Build**: 30 minutes
- **Subsequent Builds**: 18 minutes

##  Requirements Checklist

- [x] Triggers: Push to main + manual dispatch
- [x] Runner: ubuntu-latest
- [x] Unity Version: Explicitly defined (2022.3.20f1)
- [x] Batchmode: Headless builds
- [x] Outputs: APK + AAB in structured directory
- [x] Artifacts: Uploaded with actions/upload-artifact
- [x] Slack: Repo, branch, commit, artifact link
- [x] Secrets: All credentials secured
- [x] Bonus: Caching + error handling

---

**Repository**: [unity2d-prototype](https://github.com/malikanish/unity-project)  
**Workflow**: `.github/workflows/android-build.yml`
