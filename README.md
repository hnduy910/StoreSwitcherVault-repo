# StoreSwitcher Vault 0.4.2

A clean-room, rootless Theos tweak for Dopamine on iOS 15–16. It lists App Store
sessions exposed by StoreServices, switches a still-saved session, and keeps
user-entered fallback passwords in Keychain. If no saved session matches, it
opens the App Store sign-in UI, selects the different-Apple-ID row when present,
fills the email/password fields, and advances the native two-step form. It may
choose Apple's optional "Other Options" followed by "Don't Upgrade/Not Now"
response when the post-login screen asks to upgrade 2FA. It never fills or
submits a 2FA code and does not bypass Apple's checks.

At the top of Store accounts there is an immediate switch for **Cho phép
mua/tải app yêu cầu iOS cao hơn**. The request hook now runs in both the App
Store UI and Apple's `appstored` daemon, matching the open-source AppStoreTroller
filter. When enabled, the purchase request User-Agent is temporarily presented
as iOS 99.0.0. The switch is read for every request, so turning it off takes
effect immediately without a respring. The reliable workflow is: enable it,
tap Get once to record the app to the Apple ID, turn it off, refresh the page,
and then download the latest compatible build if Apple has one. It cannot make
a newer binary run on the current iOS.

The account editor is intentionally compact: one account/email field, quick
`@gmail.com`, `@icloud.com`, and `@yahoo.com` suffix buttons, a preview of the
normalized email, and a Keychain password field. Display name is
inferred from the part before `@`; country and note metadata are retained for
compatibility but are not required when adding an account.

Adding an account requires the password once and stores it in Keychain. Later
selections reuse that password without prompting; a still-valid `SSAccountStore`
session never prompts. A one-time fallback prompt remains only for managed
entries created by older versions that never had a stored password.

After a new account is saved, the tweak first attempts to sign out/deactivate
the currently active StoreServices account, then immediately opens the App
Store sign-in screen. When the sheet offers a row such as "Not <account>?" or
"Use a different Apple ID", that row is selected before the saved email is
entered. The email field receives a simulated Return/Continue action, then the
password field is filled and the native Sign In action is submitted. If Apple
offers an optional 2FA-upgrade prompt, the tweak first selects "Other Options"
when present and then only an explicit "Don't Upgrade", "Not Now", "Later", or
Vietnamese equivalent. When Apple presents a
verification-code/trusted-phone screen, automation stops and waits for the user
to enter the code. Terms, device approval, and any other security checks remain
interactive.

Long-pressing a row in the manager shows an `Kích hoạt` menu (plus edit and
delete for vault entries). The tweak also registers a dynamic Home Screen quick
action named `Chuyển tài khoản` in the App Store icon menu; open App Store once
after installation so iOS records the dynamic item. The currently active
session is labeled `Active` in green in both sections. When Apple's purchase
history screen displays a password field, the active account's Keychain
password is filled when the prompt is clearly identified; the user still
controls submission and any required 2FA/confirmation.

Managed Apple IDs can also be removed by swiping from right to left and
tapping **Xóa**; this removes that ID's Vault password and metadata only.

## Profile containers

Version 0.4.2 assigns every managed Apple ID a stable profile UUID. The UUID
namespaces the tweak's Keychain service and per-profile `NSUserDefaults` suite;
old email-keyed Keychain entries are migrated once on first load. The manager
shows a short container label so profiles can be distinguished and deleting a
profile removes its profile-scoped password and metadata. Long-pressing a
managed profile now offers **Đặt lại dữ liệu profile**. After an explicit
confirmation this deletes only Vault-owned password/metadata, rotates the
profile UUID, and keeps the managed Apple ID entry. It does not sign out of
the App Store, wipe another app, or claim to reset the physical device.

This is a safe data/session boundary for StoreSwitcher Vault, not a fake
hardware device. `SSAccountStore` sessions remain Apple's device-wide
StoreServices sessions, and the tweak does not change `identifierForVendor`,
Secure Enclave, App Attest/DeviceCheck, receipts, push tokens, or server-side
license records. Apps that bind licenses to a physical device may therefore
still see the same device; use the app vendor's account/device management or a
separate physical device for that requirement.

The profile reset is intentionally limited to this tweak's own namespaces. It
is a diagnostic convenience for accounts the user owns or is authorized to
test; it is not a way to manufacture additional device-bound licenses. No
arbitrary third-party app wipe or hardware/device-identifier spoofing is
included.

## Asspp reference

The latest `Lakr233/Asspp` source was reviewed as a reference. Its account
screen asks for email and password once, then its authentication service stores
the returned account data; its rotation path reuses the stored cookie and sends
an empty 2FA code instead of forcing a fresh code on every operation. This
tweak follows the equivalent safe rule for the native StoreServices path:
refresh and reuse `SSAccountStore` sessions first, and only open sign-in for a
new account with no saved session. Asspp's `ApplePackage` HTTP cookies cannot
be transplanted into Apple's private `SSAccountStore` objects. No code from
Asspp is bundled. Apple can still require 2FA whenever it invalidates or
re-evaluates a session; that server decision is not bypassed here.

## Build

Requirements: macOS, Xcode, a current Theos checkout, and an iOS SDK containing
iOS 15 or newer.

```sh
export THEOS=/path/to/theos
make clean package FINALPACKAGE=1 THEOS_PLATFORM_DEB_COMPRESSION_TYPE=xz
```

The package is `iphoneos-arm64` and uses `THEOS_PACKAGE_SCHEME=rootless`.

## Source provenance and licensing

The upstream repositories reviewed were:

- `subdiox/StoreSwitcher2` (master, commit `0b90799`)
- `0xkuj/StoreSwitcher2IconFix` (main, commit `dd5050d`)

Neither repository contained a LICENSE file nor a license notice when reviewed
on 2026-08-10. Public source availability is not a copyright license. Therefore
this distribution copies no upstream source or artwork. It reimplements the
small interoperability surface independently and is released under MIT. Do not
replace this notice with a claim that the upstream is MIT/GPL without written
permission or a later explicit license from its owner.

## File map

No upstream file is shipped unchanged.

| File | Status | Purpose |
|---|---|---|
| `Tweak.xm` | new | Hooks the App Store account screen, `appstored` purchase compatibility request, and manager button |
| `SSVAccountBridge.h/.m` | new | Runtime-checked StoreServices/StoreKit private API adapter |
| `SSVKeychain.h/.m` | new | Generic-password Keychain storage |
| `SSVProfileStore.h/.m` | new | Stable profile UUIDs and per-profile preferences |
| `SSVManagerController.h/.m` | new | Session list, credential list, switching and fallback fill UI |
| `Makefile`, `control`, plist | new | Dopamine/rootless Theos packaging |
| `LICENSE`, `README.md` | new | License, provenance, build and risk notes |

Concepts retained because they are required for interoperability: resolve
`AppStore.AccountViewController`, enumerate `SSAccountStore`, mark an account
active, save it without credential verification, reload the storefront, and ask
the App Store UI to reset after a storefront change. Names of private Objective-C
selectors are facts needed for compatibility; their use does not copy the
upstream implementation.

## Compatibility and security risks

- Every App Store/StoreServices class and selector used here is private and may
  disappear or change on any iOS update. The tweak checks availability and fails
  closed, but only device testing can prove a specific build.
- Sign-out/deactivation is best-effort. Runtime signature checks prevent calls
  into incompatible no-argument selectors; the fallback only clears the
  active flag and saves the old session so it can be restored later.
- Profile separation covers only data owned by this tweak. It cannot isolate
  another app's sandbox or its hardware-backed licensing keys, and it should
  not be described as a second device.
- `AppStore.AccountViewController`, sign-in action selectors, and the sign-in
  view hierarchy are especially fragile. Sign-in automation is bounded to
  about twenty-one seconds and only recognizes localized account-choice,
  Continue/Next, Other Options, Don't Upgrade/Not Now, and 2FA labels; purchase-password autofill
  never auto-submits.
- The Home Screen quick action is a dynamic `UIApplicationShortcutItem` added
  from the App Store process and handles both app-delegate and scene-delegate
  shortcut callbacks. It may not appear until App Store has been opened once
  after installation, and iOS can replace or reorder Apple's built-in
  Redeem/Purchased actions.
- Apple may invalidate a saved session or require password, 2FA, updated terms,
  or a storefront review. The tweak cannot and should not bypass these checks.
- Keychain items use `kSecAttrAccessibleWhenUnlockedThisDeviceOnly`. They do not
  migrate in backups. Because the dylib runs inside App Store, access ultimately
  depends on the host process entitlements and the jailbreak's code-signing
  behavior; validate add/read/delete on each Dopamine/ElleKit combination.
- Account metadata is non-secret and stored in the custom preferences domain
  `com.example.storeswitchervault`; passwords are never placed there.
- Any tweak injected into App Store can conflict with other App Store hooks.
  Test without StoreSwitcher2, Crane, or UI-modifying tweaks before reporting a
  crash.

## Device test checklist

1. Install on a spare/test device; launch App Store and open its account page.
2. Confirm the people icon appears only once after relaunches.
3. Switch between two accounts that already exist in `SSAccountStore`.
4. Add a vault-only account, lock/unlock the device, then verify fallback fill.
5. Long-press a managed profile, confirm **Đặt lại dữ liệu profile**, and verify
   that the container label changes while the App Store session and other apps
   remain untouched; re-enter the password once when signing in again.
6. Verify cancel, wrong password, 2FA, expired session, terms, and offline paths.
7. Delete a vault entry and confirm its Keychain password can no longer be read.
8. Repeat on at least one iOS 15 and one iOS 16 device under Dopamine/ElleKit.
