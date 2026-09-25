# Useful Travel Clock: Windows → GitHub Actions → TestFlight

## Current milestone: get a successful build

GitHub's macOS runner supplies Xcode. You do not need to own a Mac.
The `Build and test iOS` workflow builds the app and widget extension and runs
regression tests. It requires no Apple secrets. It saves logs even on failure.

After these changes are on GitHub, open Actions → Build and test iOS → Run workflow.
Select the branch containing the changes. A green run confirms compilation and
the included tests, not visual parity with the screenshots or App Store readiness.
The Simulator ZIP cannot be installed on a physical iPhone or run on Windows.

The workflow uses `macos-26` and the runner's default Xcode. The runner prints
Xcode and SDK versions so a toolchain change can be diagnosed. XcodeGen generates
the Xcode project and entitlements from `project.yml` on every run.

## Apple setup after the first green build

1. Sign in to https://developer.apple.com/account and find your Team ID under
   membership details. This identifier is not a password.
2. In Certificates, Identifiers & Profiles, register explicit App IDs for:
   - Main app: `com.usefultravelclock.app`
   - Widget extension: `com.usefultravelclock.app.widget`
   If already registered to your account, reuse them. If either is unavailable,
   choose identifiers you own and update the project consistently before proceeding.
3. Register the App Group `group.com.usefultravelclock.app` and enable it for both
   App IDs. The identifier must match the Swift source and both entitlement files.
4. In https://appstoreconnect.apple.com, create a new iOS app using the main app's
   Bundle ID. Use Useful Travel Clock if available, choose the primary language,
   and a unique internal SKU such as `useful-travel-clock-ios`.

## Signing from Windows

The signed pipeline is a separate next step; it is not implemented or enabled by
the Simulator workflow in this package.

For a predictable first release, use an Apple Distribution certificate and two
App Store Connect distribution provisioning profiles, one for each App ID.
Both profiles must include the App Group entitlement and use the same certificate.
The extension needs its own profile even though it ships inside the app.

A certificate request can be made on Windows with OpenSSL. There is no requirement
to generate it in Keychain Access. Keep the generated private key on your machine.
The following are templates to run only during the certificate setup step:

```powershell
openssl genrsa -aes256 -out distribution-private.key 2048
openssl req -new -sha256 -key distribution-private.key -out distribution.certSigningRequest
```

Choose an encryption passphrase when prompted. Supply your own name and email in
the request prompts. Upload only the `.certSigningRequest` to Apple's certificate
creation page and select Apple Distribution. Download the resulting `.cer` file.
Package it with the same private key using:

```powershell
openssl x509 -inform DER -in distribution.cer -out distribution.pem
openssl pkcs12 -export -inkey distribution-private.key -in distribution.pem -out distribution.p12
```

Store these in GitHub Actions secrets when the signed workflow is prepared:

| Secret | Value |
| --- | --- |
| `APPLE_TEAM_ID` | Developer membership Team ID |
| `BUILD_CERTIFICATE_BASE64` | Base64-encoded `.p12` containing certificate and private key |
| `P12_PASSWORD` | Export password for that `.p12` |
| `APP_PROFILE_BASE64` | Base64-encoded app provisioning profile |
| `WIDGET_PROFILE_BASE64` | Base64-encoded widget provisioning profile |
| `APP_STORE_CONNECT_KEY_ID` | App Store Connect API key ID |
| `APP_STORE_CONNECT_ISSUER_ID` | Issuer ID for a team API key |
| `APP_STORE_CONNECT_PRIVATE_KEY` | Downloaded `.p8` API key contents |

Use App Store Connect → Users and Access → Integrations to set up the team API key
with access appropriate for build uploads. Apple may require the account holder
to request API access first. Put private key material in GitHub secrets, never in
the repository, build artifacts, screenshots, or chat.

The eventual manually triggered release job will import the certificate into a
temporary keychain, install both profiles, archive for `generic/platform=iOS`,
export an App Store Connect IPA, upload it, and remove temporary signing material.
It must run only on trusted code and never expose signing secrets to pull requests.

## Before the first TestFlight upload

- Resolve any compiler or test failures reported by the first workflow run.
- Add the final app icon asset catalog and configure it on the app target.
- Set the Developer Team ID, version, and a new build number for each upload.
- Configure and validate the separate signing/upload workflow described above.
- Upload to App Store Connect, complete applicable export-compliance questions,
  and add your account to an internal TestFlight testing group.
- Install Apple's TestFlight app on your iPhone and install the processed build.
- Test the screenshot layouts, city picker, converter, appearance persistence,
  home time zone, and Home/Lock Screen widgets on the actual device.

Uploading a build does not publish the app to the public App Store. Public release
is a later step with store metadata, privacy information, screenshots, and review.

## Sources

- GitHub macOS runners: https://docs.github.com/en/actions/reference/runners/github-hosted-runners
- Signing in Actions: https://docs.github.com/en/actions/how-tos/deploy/deploy-to-third-party-platforms/sign-xcode-applications
- Apple uploads: https://developer.apple.com/help/app-store-connect/manage-builds/upload-builds
- App Groups: https://developer.apple.com/help/account/identifiers/register-an-app-group
- TestFlight: https://developer.apple.com/help/app-store-connect/test-a-beta-version/testflight-overview/
