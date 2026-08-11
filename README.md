# StoreSwitcher Vault 0.6.9

A clean-room, rootless Theos tweak for Dopamine on iOS 15–16. It lists App Store
sessions exposed by StoreServices, switches a still-saved session, and keeps
user-entered fallback passwords in Keychain. If no saved session matches, it
opens the App Store sign-in UI, selects the different-Apple-ID row when present,
fills the email/password fields, and advances the native two-step form. It may
choose Apple's optional "Other Options" followed by "Don't Upgrade/Not Now"
response when the post-login screen asks to upgrade 2FA. It never fills or
submits a 2FA code and does not bypass Apple's checks.

At the top of Store accounts there is an immediate switch for **Cho phép
mua/tải app yêu cầu iOS cao hơn**, a separate **Áp dụng cho cập nhật** switch,
an **iOS Version to Spoof** field, and an **Áp dụng** button. This follows the
open-source AppStoreTroller model in two stages: the request hook runs in
`appstored` so Apple records the purchase, and the `installd` hook runs in
`installd` and changes only MIBundle's applicable OS argument for the native
minimum-version check. Preferences are read directly from the mobile plist by
both daemons, so a switch or version change is effective for the next request.
The updates option is intentionally off by default because Apple may return a
very large update list.

For the direct-download workflow, enable the purchase switch, choose a
reasonable spoof value (for example `17.6` or `18.0` rather than `99.0`), tap
**Get** once, then keep the switch enabled while the download/install is
started. Tap **Áp dụng** after changing the policy; it refreshes the account
screen and makes a best-effort request to reload `appstored` and `installd`.
If the device's App Store sandbox cannot signal a daemon, the preference still
takes effect on its next request and the result message tells you that one
Userspace Reboot may be needed after installing the package. The button does
not claim to reboot launchd without jailbreak/root authority.

The 0.6.9 sign-in path recognizes UIKit primary actions, navigation-bar
`UIBarButtonItem` actions, and the native controller's sign-in selectors in
addition to ordinary buttons. It keeps retrying until the native form actually
advances, instead of treating a keyboard return event as a successful login. The
manager is retained while the sheet is being filled, so dismissing the manager
cannot cancel the automation. It also waits briefly after StoreServices sign-out
before opening the new sign-in sheet. The `installd` bypass only removes the metadata compatibility gate. It cannot
add APIs or frameworks missing from the current iOS, so a current binary may
still fail to launch or there may be no usable build for the device. If Apple
has a last-compatible version, turn the higher-OS switches off, tap **Áp dụng**,
and request that version instead.

Version 0.6.9 keeps the saved-session direct switch and adds repeated native
submit attempts, one-time sign-in coordination, and broader Apple ID password
autofill for clearly identified App Store password prompts. When a password
prompt is a normal sign-in sheet, the tweak also submits the native Sign In
action after filling; it never fills verification codes and does not submit a
purchase-history confirmation. Version 0.6.2 added SpringBoard quick actions;
they now append to the existing App Store menu without removing or truncating
Apple's built-in items. Version 0.6.0 extended the filter to
`installd`. After the first installation, use **Áp dụng** once; if the old
daemon was already running and cannot be signalled from the sandbox, perform a
single Dopamine Userspace Reboot (or reboot the device) so the new hook is
loaded. Restarting SpringBoard alone is not sufficient for `appstored` or
`installd`.

Version 0.6.9 fixes a sign-in regression where automation could select an
unrelated App Store or keyboard window and clear the pending password before
the native password field appeared. The manager now prefers the visible
sign-in form and keeps the Keychain password pending until that field has been
observed and submitted. If the managed coordinator is interrupted, a
conservative fallback may fill the password only when the visible prompt
explicitly contains the matching managed email.

Version 0.6.9 adds a Debian `postinst` maintainer script. Sileo/Zebra uses it
to reload only `SpringBoard`, `AppStore`, `appstored`, and `installd` after an
install or upgrade, so the account button and Home Screen quick actions do not
depend on a separate manual respring. If the jailbreak hides `killall` from
maintainer scripts, perform one Dopamine Userspace Reboot.

The account editor is intentionally compact: one account/email field, quick
`@gmail.com`, `@icloud.com`, and `@yahoo.com` suffix buttons, a preview of the
normalized email, and a Keychain password field. Display name is
inferred from the part before `@`; country and note metadata are retained for
compatibility but are not required when adding an account.

Adding an account requires the password once and stores it in Keychain. Later
selections reuse that password without prompting; a still-valid `SSAccountStore`
session never prompts. A one-time fallback prompt remains only for managed
entries created by older versions that never had a stored password.

When the target Apple ID is already present in `SSAccountStore`, switching uses
`setActive:` plus `saveAccount:verifyCredentials:NO` directly; it does not sign
out or ask for the password again. This matches the StoreSwitcher2 community
workflow, where accounts are logged in once and then selected from the saved
session list. If the target has never been saved by App Store, the tweak first
attempts to sign out/deactivate the currently active StoreServices account and
opens the native sign-in screen. When the sheet offers a row such as "Not <account>?" or
"Use a different Apple ID", that row is selected before the saved email is
entered. The email field receives a simulated Return/Continue action, then the
password field is filled and the native Sign In action is submitted. If Apple
offers an optional 2FA-upgrade prompt, the tweak first selects "Other Options"
when present and then only an explicit "Don't Upgrade", "Not Now", "Later", or
Vietnamese equivalent. When Apple presents a
verification-code/trusted-phone screen, automation stops and waits for the user
to enter the code. After the code is accepted, the tweak watches for the target
account to appear in `SSAccountStore` and reuses that saved session on later
switches. Apple can still ask for 2FA again if it invalidates or re-evaluates
the session; no tweak can safely suppress that server decision. Terms, device
approval, and any other security checks remain interactive.

Long-pressing a row in the manager shows an `Kích hoạt` menu (plus edit and
delete for vault entries). The tweak also registers two Home Screen quick
actions in the App Store icon menu: `Chuyển tài khoản` and `Thêm tài khoản`.
They are appended to the existing system actions, and **Chuyển tài khoản** opens
the manager immediately so a saved session can be selected without another
login. Open App Store once after installation so iOS records the dynamic item.
The currently active session is labeled `Active` in green in both sections. An
Apple ID password is filled automatically whenever the visible App Store prompt
clearly identifies an account/password request; purchase-history confirmations
remain user-submitted, while a normal sign-in sheet receives the native Sign In
action after the password is filled.

Managed Apple IDs can also be removed by swiping from right to left and
tapping **Xóa**; this removes that ID's Vault password and metadata only.

## Profile containers

Version 0.6.0 assigns every managed Apple ID a stable profile UUID. The UUID
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
  about twenty-one seconds and recognizes localized account-choice,
  Continue/Next, Sign In, Other Options, Don't Upgrade/Not Now, and 2FA labels.
  Generic autofill is limited to a clearly identified Apple ID/App Store
  password prompt; it never fills a device passcode or a verification code.
- The Home Screen menu exposes **Chuyển tài khoản** and **Thêm tài khoản**.
  The App Store process registers dynamic `UIApplicationShortcutItem` values and
  the SpringBoard `SBIconView` fallback supplies the same actions for the system
  App Store icon. **Chuyển tài khoản** opens the manager; **Thêm tài khoản** opens
  the compact editor directly. The hook preserves the original system actions
  and only appends these two entries. A SpringBoard restart is required once
  after installing this version so the new process hook is loaded.
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
9. Long-press the App Store icon and confirm the native actions remain present
   alongside **Chuyển tài khoản** and **Thêm tài khoản**.
