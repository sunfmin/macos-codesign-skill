---
name: macos-codesign
description: >
  Set up a self-signed code signing certificate for macOS app development so that TCC permissions
  (Screen Recording, Microphone, Accessibility) persist across rebuilds. Use this skill whenever
  working on a macOS app that needs persistent permissions, setting up code signing for local
  development, fixing "permission revoked after rebuild" issues, or when the user mentions
  TCC permissions, ad-hoc signing problems, CODE_SIGN_IDENTITY, self-signed certificate,
  codesign, or persistent macOS permissions for development builds.
---

# macOS Self-Signed Code Signing Certificate

Create and manage a self-signed code signing certificate for macOS app development, so that
system permissions survive across rebuilds without requiring the user to re-grant them every time.

## Why This Exists

macOS TCC (Transparency, Consent, and Control) grants permissions like Screen Recording and
Microphone based on the app's **bundle ID + code signature**. With ad-hoc signing
(`CODE_SIGN_IDENTITY="-"`), every rebuild produces a different signature hash, so macOS treats
it as a new app and revokes permissions. This forces the user to re-grant permissions after
every build — a painful experience during iterative development.

**The fix:** Create a self-signed certificate and use it consistently. Same bundle ID + same
certificate = same app identity to TCC. Permissions persist across rebuilds. The user grants
Screen Recording / Microphone / Accessibility once, and it works forever.

## Setup (One-Time)

### Step 1: Create the Certificate

Run this early in the project — during initial setup or when the first permission-dependent
feature is implemented.

```bash
# Choose a certificate name (use your project name)
CERT_NAME="MyApp Dev"

# Check if it already exists
if security find-identity -v -p codesigning 2>/dev/null | grep -q "$CERT_NAME"; then
  echo "Certificate '$CERT_NAME' already exists."
else
  echo "Creating self-signed code signing certificate '$CERT_NAME'..."

  # Generate OpenSSL config for code signing
  cat > /tmp/cert.cfg <<CERT_EOF
[ req ]
distinguished_name = req_dn
[ req_dn ]
CN = $CERT_NAME
[ extensions ]
keyUsage = digitalSignature
extendedKeyUsage = codeSigning
CERT_EOF

  # Generate self-signed cert (valid for 10 years)
  openssl req -x509 -newkey rsa:2048 \
    -keyout /tmp/dev.key -out /tmp/dev.crt \
    -days 3650 -nodes \
    -config /tmp/cert.cfg -extensions extensions \
    -subj "/CN=$CERT_NAME" 2>/dev/null

  # Import into login keychain with codesign trust
  security import /tmp/dev.crt -k ~/Library/Keychains/login.keychain-db -T /usr/bin/codesign 2>/dev/null
  security import /tmp/dev.key -k ~/Library/Keychains/login.keychain-db -T /usr/bin/codesign 2>/dev/null

  # Trust the certificate for code signing
  security add-trusted-cert -d -r trustRoot -k ~/Library/Keychains/login.keychain-db /tmp/dev.crt 2>/dev/null

  # Clean up temp files
  rm -f /tmp/cert.cfg /tmp/dev.key /tmp/dev.crt

  echo "Certificate '$CERT_NAME' created and trusted."
fi

# Verify
security find-identity -v -p codesigning | grep "$CERT_NAME"
```

The user may see a Keychain Access dialog to confirm — this is the one-time approval.

### Step 2: Configure the Project

**XcodeGen (`project.yml`):**
```yaml
settings:
  base:
    CODE_SIGN_IDENTITY: "MyApp Dev"   # Must match CERT_NAME above
    CODE_SIGNING_ALLOWED: "YES"
    CODE_SIGN_STYLE: "Manual"
    ENABLE_HARDENED_RUNTIME: "NO"     # MUST be NO for dev builds — hardened runtime
                                      # blocks self-signed certs from loading test bundles
                                      # and causes "bundle format unrecognized" codesign errors
```

**CRITICAL — UI test target (`bundle.ui-testing`) requires extra settings:**

UI test targets must NOT have `BUNDLE_LOADER` or `TEST_HOST` (those are for unit tests only).
XcodeGen auto-adds them when it sees a dependency on an app target. Override them to empty:

```yaml
  MyAppUITests:
    type: bundle.ui-testing
    dependencies:
      - target: MyApp
    settings:
      base:
        BUNDLE_LOADER: ""              # MUST be empty — UI tests run in a separate runner process
        TEST_HOST: ""                  # MUST be empty — not injected into the app
        CODE_SIGN_IDENTITY: "MyApp Dev"
        CODE_SIGNING_ALLOWED: "YES"
        CODE_SIGN_STYLE: "Manual"
```

Without these overrides:
- `BUNDLE_LOADER` causes Xcode to embed `PercevTests.xctest` inside `MyApp.app/Contents/PlugIns/`
- `codesign` then fails with "bundle format unrecognized, invalid, or unsuitable" on the embedded `.xctest`
- UI tests never run because the build itself fails

**Xcode project (manual):**
1. Select target > Build Settings
2. Set "Code Signing Identity" to the certificate name (e.g., "MyApp Dev")
3. Ensure "Code Signing Allowed" is YES
4. Set "Enable Hardened Runtime" to NO
5. For UI test targets: clear "Bundle Loader" and "Test Host" fields

### Step 3: First Run — Grant Permissions Once

After building with the new certificate, launch the app and grant permissions:

```bash
# Build
xcodebuild build -scheme MyApp -destination 'platform=macOS' -quiet

# Launch (adjust scheme name)
open "$(xcodebuild -scheme MyApp -showBuildSettings 2>/dev/null | \
  grep ' BUILT_PRODUCTS_DIR' | xargs | cut -d= -f2)/MyApp.app"
```

Go to **System Settings > Privacy & Security** and grant the needed permissions
(Screen Recording, Microphone, Accessibility, etc.). This is the last time you'll need to do it.

## Integrating Into init.sh

If your project has an `init.sh` setup script, add the certificate check before the build step:

```bash
# Ensure self-signed certificate exists for persistent TCC permissions
CERT_NAME="MyApp Dev"  # Change to match your project
if ! security find-identity -v -p codesigning 2>/dev/null | grep -q "$CERT_NAME"; then
  echo "Creating self-signed code signing certificate '$CERT_NAME'..."
  echo "You may be prompted to allow Keychain access — this is a ONE-TIME setup."

  cat > /tmp/cert.cfg <<CERT_EOF
[ req ]
distinguished_name = req_dn
[ req_dn ]
CN = $CERT_NAME
[ extensions ]
keyUsage = digitalSignature
extendedKeyUsage = codeSigning
CERT_EOF

  openssl req -x509 -newkey rsa:2048 -keyout /tmp/dev.key -out /tmp/dev.crt \
    -days 3650 -nodes -config /tmp/cert.cfg -extensions extensions -subj "/CN=$CERT_NAME" 2>/dev/null
  security import /tmp/dev.crt -k ~/Library/Keychains/login.keychain-db -T /usr/bin/codesign 2>/dev/null
  security import /tmp/dev.key -k ~/Library/Keychains/login.keychain-db -T /usr/bin/codesign 2>/dev/null
  security add-trusted-cert -d -r trustRoot -k ~/Library/Keychains/login.keychain-db /tmp/dev.crt 2>/dev/null
  rm -f /tmp/cert.cfg /tmp/dev.key /tmp/dev.crt

  echo "Certificate '$CERT_NAME' created. Update project.yml: CODE_SIGN_IDENTITY: \"$CERT_NAME\""
fi
```

## Troubleshooting

### Certificate not found after creation
```bash
# List all code signing identities
security find-identity -v -p codesigning
```
If empty, the import or trust step may have failed. Re-run the setup commands.

### "bundle format unrecognized, invalid, or unsuitable" during codesign
This happens when a `.xctest` unit test bundle gets embedded inside the app's `PlugIns/` directory.
- **Cause:** `BUNDLE_LOADER`/`TEST_HOST` set on a UI test target, or stale DerivedData
- **Fix 1:** Clear `BUNDLE_LOADER` and `TEST_HOST` on UI test targets (see Step 2 above)
- **Fix 2:** Clean DerivedData: `rm -rf ~/Library/Developer/Xcode/DerivedData/MyApp-*`
- **Fix 3:** Set `ENABLE_HARDENED_RUNTIME: "NO"` — hardened runtime with self-signed certs causes codesign to reject embedded bundles

### Permissions still revoked after rebuild
- Verify `CODE_SIGN_IDENTITY` in build settings matches the certificate name exactly
- Check that `CODE_SIGNING_ALLOWED` is `YES` (not `NO`)
- Confirm you're not accidentally overriding with `-` in a scheme or CI config

### "User interaction is not allowed" during build
The keychain may be locked. Unlock it:
```bash
security unlock-keychain ~/Library/Keychains/login.keychain-db
```

### Removing the certificate
```bash
# Find and delete from keychain (use Keychain Access.app for GUI)
security delete-identity -c "MyApp Dev"
# Then revert project to ad-hoc signing:
# CODE_SIGN_IDENTITY: "-"
```

## Why This Matters for Automated Testing

Without a stable certificate:
- Every `xcodebuild build` invalidates TCC permissions
- XCUITests that trigger permission dialogs block indefinitely or fail
- The developer must manually re-grant permissions after every build
- CI/CD and autonomous agent workflows are impossible for permission-dependent features

With a stable certificate:
- Build > test > rebuild > test cycles work without interruption
- Automated agents can implement and verify permission-dependent features
- The developer only interacts once during initial setup
