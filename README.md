# Nexus Demo: Maestro tests on BrowserStack App Automate (with App Accessibility)

Maestro flows for the Nexus Demo Android app (`com.demobank.app`), run on BrowserStack App Automate with App Accessibility reports (WCAG 2.2 AA).

```
.maestro/
  config.yaml          # runs android/*.yaml
  android/             # flows 00 to 17
  subflows/open_hub.yaml
.github/workflows/browserstack-maestro-a11y.yml   # GitHub Actions run
```

## Prerequisites

- A BrowserStack account with App Automate and App Accessibility.
- Your **username** and **access key**: BrowserStack dashboard, Account & Profile.
- The app (`NexusDemo-videofix.apk`). An app uploaded to a different BrowserStack account is rejected with `BROWSERSTACK_INVALID_APP_URL`, so upload it to **your** account.
- `curl` and `zip`.

> Never commit your access key. Use environment variables locally and GitHub secrets in CI.

---

## 1. Run from the command line (zip the flows and call the API)

### 1.1 Set your credentials (current shell only)

```sh
export BROWSERSTACK_USERNAME="<your username>"
export BROWSERSTACK_ACCESS_KEY="<your access key>"
```

### 1.2 Upload the app

```sh
curl -u "$BROWSERSTACK_USERNAME:$BROWSERSTACK_ACCESS_KEY" \
  -X POST "https://api-cloud.browserstack.com/app-automate/maestro/v2/app" \
  -F "file=@/path/to/NexusDemo-videofix.apk"
```

Copy `app_url` (`bs://...`) from the response.

### 1.3 Zip the flows and upload the suite

BrowserStack needs **one visible parent folder** inside the zip. A zip of the hidden `.maestro` folder is rejected, so copy the flows into `flows/` first.

```sh
rm -rf suite && mkdir -p suite/flows
cp -R .maestro/android .maestro/subflows .maestro/config.yaml suite/flows/
(cd suite && zip -r flows.zip flows)

curl -u "$BROWSERSTACK_USERNAME:$BROWSERSTACK_ACCESS_KEY" \
  -X POST "https://api-cloud.browserstack.com/app-automate/maestro/v2/test-suite" \
  -F "file=@suite/flows.zip"
```

Copy `test_suite_url` (`bs://...`) from the response.

### 1.4 Start a build with accessibility enabled

```sh
curl -u "$BROWSERSTACK_USERNAME:$BROWSERSTACK_ACCESS_KEY" \
  -X POST "https://api-cloud.browserstack.com/app-automate/maestro/v2/android/build" \
  -H "Content-Type: application/json" \
  -d '{
    "app": "<app_url>",
    "testSuite": "<test_suite_url>",
    "project": "NexusDemo A11y",
    "devices": ["Samsung Galaxy S23-13.0"],
    "networkLogs": true,
    "testObservability": false,
    "accessibility": true,
    "accessibilityOptions": {
      "wcagVersion": "wcag22aa",
      "includeIssueType": { "bestPractice": true, "needsReview": false }
    }
  }'
```

- `testObservability` **must be `false`**, otherwise accessibility reports are not generated.
- Omit `execute` to run every flow, or add `"execute": ["android/01_hub_tiles.yaml"]` to run chosen flows.
- The response contains a `build_id`. Check it with:

```sh
curl -u "$BROWSERSTACK_USERNAME:$BROWSERSTACK_ACCESS_KEY" \
  "https://api-cloud.browserstack.com/app-automate/maestro/v2/builds/<build_id>"
```

### 1.5 Flows that need a device capability

Some flows only pass when the build sets a capability. Run them as their own builds by adding these fields to the body above.

| Flow | Add to the build body |
|---|---|
| `03_biometric`, `12_biometric_negative` | `"enableBiometric": true` |
| `08_network_low` | `"networkProfile": "2g-gprs-lossy"` |
| `09_gps` | `"gpsLocation": "48.8566,2.3522"` |
| `02_image_injection` | `"enableCameraImageInjection": true, "cameraInjectionMedia": "media://<IMAGE_ID>"` |
| `05_barcode`, `13_barcode_decode` | `"enableCameraImageInjection": true, "cameraInjectionMedia": "media://<QR_ID>"` |
| `16_video_record_play` | `"enableCameraImageInjection": true` |

Get a `media://` id by uploading an image or QR code:

```sh
curl -u "$BROWSERSTACK_USERNAME:$BROWSERSTACK_ACCESS_KEY" \
  -X POST "https://api-cloud.browserstack.com/app-automate/upload-media" \
  -F "file=@/path/to/image.png"
```

`07_network_offline` should run **without** a `networkProfile`.

---

## 2. Run with GitHub Actions

The workflow `.github/workflows/browserstack-maestro-a11y.yml` uploads the app and flows, starts one build per capability group with accessibility and network logs on, waits for the results and writes a summary.

It runs on:
- pushes and pull requests to `main`
- **Actions > Maestro on BrowserStack App Automate > Run workflow**, with an optional `app_url` input

Capability groups:

| Group | Flows | Capability |
|---|---|---|
| core | 00, 01, 04, 06, 07, 10, 11, 14, 15, 17 | none |
| biometric | 03, 12 | `enableBiometric` |
| network-low | 08 | `networkProfile: 2g-gprs-lossy` |
| gps | 09 | `gpsLocation` |
| video | 16 | `enableCameraImageInjection` |
| image-injection | 02 | needs `BROWSERSTACK_IMAGE_MEDIA_ID` |
| barcode | 05, 13 | needs `BROWSERSTACK_QR_MEDIA_ID` |

If a media id is not set, those flows run in the core group without injection and the run prints a warning.

The job fails if any group fails. Each build is named `NexusDemo A11y - <group>` in App Automate and App Accessibility.

> The workflow only runs from a repo root. Put this folder in its own repository (do not push it from a parent folder such as your home directory).

---

## 3. Add your own credentials and test

### 3.1 GitHub Actions

1. In your repo go to **Settings > Secrets and variables > Actions**.
2. Under **Secrets**, add:
   - `BROWSERSTACK_USERNAME`
   - `BROWSERSTACK_ACCESS_KEY`
3. Under **Variables**, add one of:
   - `BROWSERSTACK_APP_URL`: the `bs://...` id of the app on **your** account (section 1.2), or
   - commit the APK at `app/NexusDemo-videofix.apk` and remove `*.apk` from `.gitignore`.
4. Optional variables for camera flows: `BROWSERSTACK_IMAGE_MEDIA_ID`, `BROWSERSTACK_QR_MEDIA_ID` (the id after `media://`).
5. Start a run from the **Actions** tab, or push to `main`.

---

## Known issues

- `00_explore` fails at the `standalone_item_network` scroll step.
- `08_network_low` has failed at `assertVisible "Low network.*"` even with the 2G profile.
- `03_biometric` and `12_biometric_negative` fail unless `enableBiometric` is set.

