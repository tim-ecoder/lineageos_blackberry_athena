Domain Verification Auto-Approve (microG / degoogled ROM compat)

Without Google Play Services, Android's domain verification agent is absent.
App Links with autoVerify="true" never get verified, showing "none" or "legacy_failure".
This breaks in-app link handling for banking apps, social media, etc.

Two fixes:

1. IntentFilter.java — allow mixed-scheme intent filters (e.g. vtb:// + https://)
   to be collected for domain verification. Stock Android rejects filters with
   non-web schemes even if https is also declared.

2. DomainVerificationService.java — auto-approve all auto-verify domains to
   STATE_SUCCESS at package install time, and in sendBroadcast() for re-verify.
   Replaces the verification agent broadcast (which nobody receives without GMS).

Files modified:
  frameworks/base/core/java/android/content/IntentFilter.java
  frameworks/base/services/core/java/com/android/server/pm/verify/domain/DomainVerificationService.java

To apply:
  cd /data/LOS22/frameworks/base
  git apply domain-verification-auto-approve.patch
