# GitHub Actions setup for Google Play releases

This repository contains the reusable workflow:

`.github/workflows/reusable-android-play-release.yml`

It builds a signed Android App Bundle, runs Gradle checks, uploads the AAB as a short-lived artifact, and publishes to the requested Google Play track. Production releases require approval through the GitHub `production` environment.

## What is automated

- Java 17 and Gradle setup
- Optional JavaScript dependency installation for React Native/Expo repositories
- Android checks (`./gradlew check`)
- Signed release AAB build
- Artifact upload for 14 days
- Google Play upload to `internal`, `alpha`, `beta`, or `production`
- A mandatory GitHub environment approval before production publication

The workflow **does not auto-submit production releases without the configured environment reviewer**.

## Required GitHub secrets

Configure these as repository or organization secrets. Never commit them to source control.

| Secret | Purpose |
|---|---|
| `GOOGLE_PLAY_SERVICE_ACCOUNT_JSON` | Full JSON credential for the Play service account already linked in Play Console |
| `ANDROID_KEYSTORE_BASE64` | Base64-encoded signing keystore for this app |
| `ANDROID_KEYSTORE_PASSWORD` | Keystore password |
| `ANDROID_KEY_ALIAS` | Signing key alias |
| `ANDROID_KEY_PASSWORD` | Signing key password |

The Play JSON credential must be added to GitHub Actions secrets, not pasted into an issue, pull request, log, or chat message.

Example command, run locally after downloading the JSON and setting the file permissions:

```bash
gh secret set GOOGLE_PLAY_SERVICE_ACCOUNT_JSON --repo OWNER/REPO < service-account.json
gh secret set ANDROID_KEYSTORE_BASE64 --repo OWNER/REPO < <(base64 -w0 release.keystore)
gh secret set ANDROID_KEYSTORE_PASSWORD --repo OWNER/REPO
gh secret set ANDROID_KEY_ALIAS --repo OWNER/REPO
gh secret set ANDROID_KEY_PASSWORD --repo OWNER/REPO
```

## Required GitHub environment

Create two environments in each app repository:

- `internal-testing`: no required reviewer; used for internal-track releases.
- `production`: add the owner as a required reviewer. Optionally add a deployment branch rule such as `release/**`.

GitHub settings path:

`Repository → Settings → Environments`

The production reviewer is the final approval gate. A workflow can build and validate an AAB automatically, but it cannot publish production until the reviewer approves the deployment job.

## Calling the reusable workflow from an app repository

Create `.github/workflows/android-play-release.yml` in each app repository:

```yaml
name: Android Play release

on:
  push:
    branches: [main]
    paths-ignore: ['**/*.md']
  workflow_dispatch:
    inputs:
      track:
        description: Play track
        required: true
        default: internal
        type: choice
        options: [internal, alpha, beta, production]
      production:
        description: Request production publication and wait for approval
        required: true
        default: false
        type: boolean

jobs:
  release:
    uses: georgemall1986-spec/google-console/.github/workflows/reusable-android-play-release.yml@main
    with:
      package_name: com.example.replace_me
      track: ${{ inputs.track || 'internal' }}
      production: ${{ inputs.production || false }}
    secrets: inherit
```

Before enabling a repository, replace `com.example.replace_me` with the exact Play package name and verify that the repository contains an executable `gradlew` or `android/gradlew`.

## Current repository audit

Only the following accessible repositories were found in the account at setup time:

- `georgemall1986-spec/google-console` — native Android sample; reusable workflow source
- `georgemall1986-spec/google-play-apps` — placeholder workflow with a hard-coded package name; needs migration to the reusable workflow
- `georgemall1986-spec/google` — Expo/React Native source, but no executable Gradle wrapper in the repository yet
- `georgemall1986-spec/merge-magic-enchanted-glade-` — README only
- `georgemall1986-spec/merge-magic` — empty repository
- `georgemall1986-spec/love-affairs` — empty repository
- `georgemall1986-spec/love-affairs-marketing-videos-` — marketing assets, not an Android source repository

This is **not yet a 12-app inventory**. The remaining six app repositories or their GitHub URLs, package names, and signing keys are still required before all twelve can be enabled.

## Safety notes

- Builds and checks may run automatically.
- Internal testing can be automated after secrets and environment setup.
- Production publication is approval-gated.
- Google Play review/approval is controlled by Google; GitHub Actions can submit and monitor releases but cannot guarantee Google approval.
- Service-account JSON and signing keys must be rotated if exposed.
