# 🚀 Complete CI/CD Pipeline Setup Guide

## 📋 Table of Contents
1. [What is Unity?](#what-is-unity)
2. [What is CI/CD?](#what-is-cicd)
3. [Pipeline Architecture](#pipeline-architecture)
4. [Prerequisites](#prerequisites)
5. [Step-by-Step Setup](#step-by-step-setup)
6. [GitHub Secrets Configuration](#github-secrets-configuration)
7. [Testing the Pipeline](#testing-the-pipeline)
8. [Troubleshooting](#troubleshooting)
9. [Design Decisions](#design-decisions)

---

## 🎮 What is Unity?

**Unity** is a cross-platform game engine used to create 2D and 3D games and interactive experiences. 

### Key Concepts:
- **Unity Project**: A folder containing all game assets, scripts, scenes, and settings
- **Build**: The process of converting your Unity project into a playable application (APK for Android, IPA for iOS, EXE for Windows, etc.)
- **Batchmode**: Running Unity without the graphical interface (headless) - perfect for automated builds
- **APK**: Android Package Kit - the file format for Android apps
- **AAB**: Android App Bundle - Google Play's preferred publishing format (smaller downloads, optimized)

---

## 🔄 What is CI/CD?

**CI/CD** stands for Continuous Integration / Continuous Deployment.

### Continuous Integration (CI):
- Automatically build and test code when changes are pushed
- Catch errors early
- Ensure code quality

### Continuous Deployment (CD):
- Automatically deploy/release builds
- Faster delivery to users
- Consistent deployment process

### Our Pipeline Does:
1. ✅ Detects code changes on `main` branch
2. ✅ Automatically builds Android APK and AAB
3. ✅ Stores build artifacts
4. ✅ Notifies team via Slack
5. ✅ Can be triggered manually

---

## 🏗️ Pipeline Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                    TRIGGER EVENTS                            │
│  • Push to main branch                                       │
│  • Manual workflow dispatch                                  │
└────────────────────┬────────────────────────────────────────┘
                     │
                     ▼
┌─────────────────────────────────────────────────────────────┐
│              GITHUB ACTIONS RUNNER                           │
│              (ubuntu-latest)                                 │
└────────────────────┬────────────────────────────────────────┘
                     │
                     ▼
┌─────────────────────────────────────────────────────────────┐
│                 PIPELINE STEPS                               │
│                                                              │
│  1. Checkout Code                                           │
│     └─> Clone repository with Git LFS                       │
│                                                              │
│  2. Cache Unity Library                                     │
│     └─> Speed up builds by caching dependencies             │
│                                                              │
│  3. Setup Android Keystore                                  │
│     └─> Decode and prepare signing certificate              │
│                                                              │
│  4. Build APK                                               │
│     └─> Unity builds in batchmode (headless)                │
│                                                              │
│  5. Build AAB                                               │
│     └─> Unity builds App Bundle for Play Store             │
│                                                              │
│  6. Upload APK Artifact                                     │
│     └─> Store APK in GitHub Actions                         │
│                                                              │
│  7. Upload AAB Artifact                                     │
│     └─> Store AAB in GitHub Actions                         │
│                                                              │
│  8. Send Slack Notification                                 │
│     └─> Notify team with build details                      │
│                                                              │
│  9. Cleanup                                                 │
│     └─> Remove sensitive files                              │
└────────────────────┬────────────────────────────────────────┘
                     │
                     ▼
┌─────────────────────────────────────────────────────────────┐
│                    OUTPUTS                                   │
│  • APK file (ready to install on Android)                   │
│  • AAB file (ready for Google Play Store)                   │
│  • Slack notification with download links                   │
└─────────────────────────────────────────────────────────────┘
```

---

## ✅ Prerequisites

### 1. Unity Account
- Create account at: https://id.unity.com/
- Required for Unity license

### 2. GitHub Account
- Repository with Unity project
- Admin access to configure secrets

### 3. Slack Workspace
- Admin access to create incoming webhooks
- Channel for build notifications

### 4. Android Keystore
- Required to sign Android builds
- We'll create this in setup steps

---

## 📝 Step-by-Step Setup

### STEP 1: Get Unity License

Unity requires a license even for automated builds. We'll use Unity's free Personal license.

#### Option A: Using Unity Activation (Recommended)

1. **Install Unity locally** (if not already installed):
   - Download Unity Hub: https://unity.com/download
   - Install Unity version `2022.1.10f1` (match your project version)

2. **Get Unity License File**:
   ```bash
   # On Windows
   "C:\Program Files\Unity\Hub\Editor\2022.1.10f1\Editor\Unity.exe" -quit -batchmode -serial YOUR-SERIAL-KEY -username "your@email.com" -password "yourpassword"
   
   # On Mac
   /Applications/Unity/Hub/Editor/2022.1.10f1/Unity.app/Contents/MacOS/Unity -quit -batchmode -serial YOUR-SERIAL-KEY -username "your@email.com" -password "yourpassword"
   
   # On Linux
   /opt/unity/Editor/Unity -quit -batchmode -serial YOUR-SERIAL-KEY -username "your@email.com" -password "yourpassword"
   ```

3. **Find License File**:
   - Windows: `C:\ProgramData\Unity\Unity_lic.ulf`
   - Mac: `/Library/Application Support/Unity/Unity_lic.ulf`
   - Linux: `~/.local/share/unity3d/Unity/Unity_lic.ulf`

4. **Copy License Content**:
   - Open the `.ulf` file in a text editor
   - Copy the entire content
   - This will be your `UNITY_LICENSE` secret

#### Option B: Using Game CI Activation

1. Create a temporary workflow to get activation file:
   ```yaml
   # .github/workflows/get-activation.yml
   name: Get Unity Activation File
   on: workflow_dispatch
   jobs:
     activation:
       runs-on: ubuntu-latest
       steps:
         - uses: game-ci/unity-request-activation-file@v2
           with:
             unityVersion: 2022.1.10f1
         - uses: actions/upload-artifact@v4
           with:
             name: activation-file
             path: Unity_v2022.1.10f1.alf
   ```

2. Run this workflow manually
3. Download the `.alf` file
4. Go to: https://license.unity3d.com/manual
5. Upload the `.alf` file
6. Download the `.ulf` license file
7. Copy its content for `UNITY_LICENSE` secret

---

### STEP 2: Create Android Keystore

The keystore is used to sign your Android app (required for installation).

#### Using Command Line:

```bash
# Install Java JDK if not installed
# Windows: Download from https://www.oracle.com/java/technologies/downloads/
# Mac: brew install openjdk
# Linux: sudo apt-get install openjdk-11-jdk

# Generate keystore
keytool -genkey -v -keystore user.keystore -alias myalias -keyalg RSA -keysize 2048 -validity 10000

# You'll be prompted for:
# - Keystore password (remember this!)
# - Key password (remember this!)
# - Your name, organization, etc.
```

#### Important Information to Save:
- **Keystore file**: `user.keystore`
- **Keystore password**: (what you entered)
- **Key alias**: `myalias` (or what you chose)
- **Key password**: (what you entered)

#### Convert Keystore to Base64:

```bash
# On Linux/Mac
base64 user.keystore > keystore.txt

# On Windows (PowerShell)
[Convert]::ToBase64String([IO.File]::ReadAllBytes("user.keystore")) > keystore.txt
```

The content of `keystore.txt` will be your `ANDROID_KEYSTORE` secret.

---

### STEP 3: Setup Slack Webhook

1. **Go to Slack API**:
   - Visit: https://api.slack.com/apps
   - Click "Create New App"
   - Choose "From scratch"
   - Name: "Unity Build Bot"
   - Select your workspace

2. **Enable Incoming Webhooks**:
   - In app settings, click "Incoming Webhooks"
   - Toggle "Activate Incoming Webhooks" to ON
   - Click "Add New Webhook to Workspace"
   - Select channel (e.g., #builds or #general)
   - Click "Allow"

3. **Copy Webhook URL**:
   - You'll see a URL like: `https://hooks.slack.com/services/T00000000/B00000000/XXXXXXXXXXXXXXXXXXXX`
   - This is your `SLACK_WEBHOOK_URL` secret

4. **Test Webhook** (optional):
   ```bash
   curl -X POST -H 'Content-type: application/json' \
   --data '{"text":"Hello from Unity Build Bot!"}' \
   YOUR_WEBHOOK_URL
   ```

---

### STEP 4: Configure GitHub Secrets

1. **Go to Your Repository**:
   - Navigate to: `https://github.com/YOUR_USERNAME/YOUR_REPO`

2. **Open Settings**:
   - Click "Settings" tab
   - Click "Secrets and variables" → "Actions"

3. **Add Repository Secrets**:
   Click "New repository secret" for each:

   | Secret Name | Value | Description |
   |------------|-------|-------------|
   | `UNITY_LICENSE` | Content of `.ulf` file | Unity license file content |
   | `UNITY_EMAIL` | your@email.com | Unity account email |
   | `UNITY_PASSWORD` | yourpassword | Unity account password |
   | `ANDROID_KEYSTORE` | Base64 encoded keystore | Content from keystore.txt |
   | `ANDROID_KEYSTORE_PASS` | your_keystore_password | Password you set for keystore |
   | `ANDROID_KEY_ALIAS` | myalias | Alias name from keystore |
   | `ANDROID_KEY_ALIAS_PASS` | your_key_password | Password for the key alias |
   | `SLACK_WEBHOOK_URL` | https://hooks.slack.com/... | Slack webhook URL |

4. **Verify Secrets**:
   - All 8 secrets should be listed
   - Values are hidden (shown as ***)

---

### STEP 5: Configure Unity Project for Android

1. **Open Unity Project**:
   - Open Unity Hub
   - Open your `unity2d-prototype` project

2. **Switch to Android Platform**:
   - File → Build Settings
   - Select "Android"
   - Click "Switch Platform" (wait for it to complete)

3. **Configure Player Settings**:
   - In Build Settings, click "Player Settings"
   - Set the following:

   **Identification:**
   - Company Name: `YourCompany`
   - Product Name: `Unity2D Prototype`
   - Package Name: `com.yourcompany.unity2dprototype`
   - Version: `1.0.0`
   - Bundle Version Code: `1`

   **Publishing Settings:**
   - Build → Minify → Release: `None` (or Proguard)
   - Build → Target API Level: `Automatic (highest installed)`
   - Build → Minimum API Level: `Android 5.0 (API level 21)` or higher

4. **Save Project**:
   - File → Save Project
   - Commit changes to Git

---

### STEP 6: Push Workflow to GitHub

The workflow file is already created at `.github/workflows/android-build.yml`.

1. **Commit and Push**:
   ```bash
   cd unity2d-prototype
   git add .github/workflows/android-build.yml
   git commit -m "Add Android CI/CD pipeline"
   git push origin main
   ```

2. **Workflow Will Trigger Automatically** on push to main branch

---

### STEP 7: Monitor Build

1. **Go to Actions Tab**:
   - Visit: `https://github.com/YOUR_USERNAME/YOUR_REPO/actions`

2. **Watch Build Progress**:
   - Click on the running workflow
   - Expand steps to see logs
   - Build takes 15-30 minutes (first time)

3. **Check Slack**:
   - You should receive notification when build completes

---

## 🔐 GitHub Secrets Configuration

### Complete Secrets Checklist:

```
✅ UNITY_LICENSE
   Purpose: Allows GitHub Actions to use Unity
   Format: Multi-line XML content from .ulf file
   Example: <?xml version="1.0" encoding="UTF-8"?><root>...</root>

✅ UNITY_EMAIL
   Purpose: Unity account authentication
   Format: Email address
   Example: developer@example.com

✅ UNITY_PASSWORD
   Purpose: Unity account authentication
   Format: Plain text password
   Example: MySecurePassword123!

✅ ANDROID_KEYSTORE
   Purpose: Sign Android builds
   Format: Base64 encoded binary
   Example: /u3+7QAAAAIAAAABAAAAAQAKbXlhbGlhcy4uLg==...

✅ ANDROID_KEYSTORE_PASS
   Purpose: Unlock keystore file
   Format: Plain text password
   Example: keystorePassword123

✅ ANDROID_KEY_ALIAS
   Purpose: Identify signing key in keystore
   Format: Plain text alias name
   Example: myalias

✅ ANDROID_KEY_ALIAS_PASS
   Purpose: Unlock specific key in keystore
   Format: Plain text password
   Example: keyPassword123

✅ SLACK_WEBHOOK_URL
   Purpose: Send build notifications
   Format: HTTPS URL
   Example: https://hooks.slack.com/services/T00/B00/XXX
```

---

## 🧪 Testing the Pipeline

### Test 1: Automatic Trigger (Push to Main)

```bash
# Make a small change
echo "# CI/CD Pipeline Active" >> README.md

# Commit and push
git add README.md
git commit -m "Test CI/CD pipeline"
git push origin main

# Check GitHub Actions tab
```

### Test 2: Manual Trigger

1. Go to: `https://github.com/YOUR_USERNAME/YOUR_REPO/actions`
2. Click "Unity Android Build Pipeline"
3. Click "Run workflow" button
4. Select branch: `main`
5. Click "Run workflow"

### Test 3: Download Artifacts

1. After build completes, go to workflow run
2. Scroll to "Artifacts" section
3. Download:
   - `Android-APK-{commit-hash}.zip`
   - `Android-AAB-{commit-hash}.zip`
4. Extract and test APK on Android device

### Test 4: Verify Slack Notification

Check your Slack channel for message containing:
- ✅ Repository name
- ✅ Branch name
- ✅ Commit hash with link
- ✅ Author name
- ✅ Download link to artifacts

---

## 🔧 Troubleshooting

### Issue 1: Unity License Error

**Error**: `License activation failed`

**Solutions**:
- Verify `UNITY_LICENSE` secret contains complete `.ulf` file content
- Check `UNITY_EMAIL` and `UNITY_PASSWORD` are correct
- Ensure Unity version matches: `2022.1.10f1`
- Try regenerating license file

### Issue 2: Keystore Error

**Error**: `Keystore file not found` or `Invalid keystore format`

**Solutions**:
- Verify `ANDROID_KEYSTORE` is properly base64 encoded
- Check passwords are correct (no extra spaces)
- Ensure alias name matches exactly
- Regenerate keystore if needed

### Issue 3: Build Fails

**Error**: `Build failed with errors`

**Solutions**:
- Check Unity project builds locally first
- Verify all required packages are in `Packages/manifest.json`
- Check for missing assets or broken references
- Review build logs in GitHub Actions

### Issue 4: Slack Notification Not Sent

**Error**: `Slack notification failed`

**Solutions**:
- Verify webhook URL is correct
- Check webhook is still active in Slack
- Test webhook with curl command
- Ensure channel still exists

### Issue 5: Artifacts Not Uploaded

**Error**: `No artifacts found`

**Solutions**:
- Check build actually completed successfully
- Verify build path is correct: `Builds/Android`
- Look for APK/AAB files in build logs
- Check file permissions

---

## 🎯 Design Decisions

### 1. **Why Ubuntu Runner?**
- **Cost**: Free for public repositories
- **Performance**: Fast build times
- **Compatibility**: Best support for Unity builds
- **Stability**: Most reliable for CI/CD

### 2. **Why Cache Unity Library?**
- **Speed**: Reduces build time from 30min to 10min
- **Cost**: Saves GitHub Actions minutes
- **Efficiency**: Reuses unchanged dependencies
- **Implementation**: Cache invalidates when Assets/Packages change

### 3. **Why Build Both APK and AAB?**
- **APK**: Direct installation for testing
- **AAB**: Required for Google Play Store
- **Flexibility**: Supports both distribution methods
- **Best Practice**: Industry standard approach

### 4. **Why Separate Upload Steps?**
- **Organization**: Clear artifact separation
- **Download**: Users can download only what they need
- **Retention**: Different retention policies possible
- **Debugging**: Easier to identify issues

### 5. **Why Slack Integration?**
- **Visibility**: Team knows build status immediately
- **Convenience**: No need to check GitHub
- **Rich Format**: Detailed information in notification
- **Actionable**: Direct links to artifacts

### 6. **Why Cleanup Step?**
- **Security**: Remove sensitive keystore file
- **Best Practice**: Don't leave credentials on runner
- **Compliance**: Meets security standards
- **Always Runs**: Even if build fails

### 7. **Why Error Handling?**
- **Reliability**: Graceful failure handling
- **Debugging**: Clear error messages
- **Notifications**: Team knows when builds fail
- **Recovery**: Easy to identify and fix issues

---

## 📊 Pipeline Performance

### Expected Timings:

| Step | First Run | Cached Run |
|------|-----------|------------|
| Checkout | 30s | 30s |
| Cache Restore | - | 2min |
| Keystore Setup | 5s | 5s |
| Build APK | 15min | 8min |
| Build AAB | 12min | 6min |
| Upload Artifacts | 1min | 1min |
| Slack Notification | 5s | 5s |
| **Total** | **~30min** | **~17min** |

### Optimization Tips:

1. **Enable Caching**: Already implemented
2. **Parallel Builds**: Not possible (Unity limitation)
3. **Incremental Builds**: Enabled via cache
4. **Smaller Project**: Remove unused assets
5. **Faster Runner**: Use self-hosted runner (advanced)

---

## 🚀 Next Steps

### Enhancements You Can Add:

1. **Automated Testing**:
   ```yaml
   - name: Run Unity Tests
     uses: game-ci/unity-test-runner@v4
   ```

2. **Version Bumping**:
   - Auto-increment version numbers
   - Tag releases automatically

3. **Multiple Platforms**:
   - Add iOS build job
   - Add Windows build job
   - Add WebGL build job

4. **Deploy to Store**:
   - Auto-upload to Google Play
   - Beta testing distribution

5. **Code Quality**:
   - Add linting
   - Add code coverage
   - Add security scanning

---

## 📚 Additional Resources

- **Unity CI/CD**: https://unity.com/solutions/ci-cd
- **Game CI Documentation**: https://game.ci/docs
- **GitHub Actions**: https://docs.github.com/actions
- **Slack API**: https://api.slack.com/messaging/webhooks
- **Android Signing**: https://developer.android.com/studio/publish/app-signing

---

## ✅ Completion Checklist

Before submitting, verify:

- [ ] Workflow file exists at `.github/workflows/android-build.yml`
- [ ] All 8 GitHub secrets are configured
- [ ] Unity project is configured for Android
- [ ] Pipeline runs successfully
- [ ] APK artifact is generated and downloadable
- [ ] AAB artifact is generated and downloadable
- [ ] Slack notification is received
- [ ] This README is complete and clear
- [ ] Screenshot of successful run is captured
- [ ] Repository is public or access is granted

---

## 📸 Screenshot Requirements

Capture screenshot showing:
1. ✅ Green checkmark on workflow run
2. ✅ All steps completed successfully
3. ✅ Artifacts section with APK and AAB
4. ✅ Build duration
5. ✅ Commit information

---

## ⏱️ Estimated Completion Time

- **Setup Time**: 2-3 hours (first time)
- **Build Time**: 30 minutes (first run)
- **Build Time**: 15-20 minutes (subsequent runs with cache)

---

## 📝 Design Decisions Summary

1. **Game CI Actions**: Industry-standard, well-maintained, Unity-specific
2. **Ubuntu Runner**: Cost-effective, reliable, fast
3. **Caching Strategy**: Intelligent cache invalidation based on project changes
4. **Security First**: All credentials in secrets, cleanup after build
5. **Rich Notifications**: Detailed Slack messages with actionable links
6. **Error Handling**: Graceful failures with notifications
7. **Artifact Management**: Organized, versioned, easy to download
8. **Extensibility**: Easy to add more platforms or features

---

**Created by**: CI/CD Pipeline Setup
**Date**: 2026
**Version**: 1.0.0
**License**: MIT
