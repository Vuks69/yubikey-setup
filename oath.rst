####################
 OATH (TOTP / HOTP)
####################

***************
 1. What It Is
***************

The OATH application on the YubiKey stores HMAC-based OTP secrets (TOTP
and HOTP) and generates 6-8 digit codes directly on the device. It is
functionally equivalent to Google Authenticator or Authy, but the
secrets live on the hardware key instead of a phone.

1a. What It Is Used For
=======================

-  **Time-based one-time passwords (TOTP)** - 30-second rotating 2FA
   codes for services like Google, GitHub, Facebook, Dropbox, etc.
-  **Counter-based one-time passwords (HOTP)** - Counter-incrementing
   codes, less common.
-  **Offline code generation** - tap the YubiKey (or touch policy) to
   get codes without network access.
-  **Multiple accounts** - store up to 32 (YubiKey 5) OATH accounts on
   the device.

*************************
 2. Overlaps / Conflicts
*************************

+-----------------------+-----------------------+-----------------------+
| Feature               | Overlaps With         | Notes                 |
+=======================+=======================+=======================+
| TOTP codes            | Phone authenticator   | YubiKey is more       |
|                       | apps                  | secure (secrets never |
|                       |                       | leave key) but less   |
|                       |                       | convenient (limited   |
|                       |                       | accounts, no push).   |
+-----------------------+-----------------------+-----------------------+
| HOTP via OATH         | OTP slot HOTP         | OATH app stores many  |
|                       |                       | HOTP credentials;     |
|                       |                       | slot only one. Use    |
|                       |                       | OATH for              |
|                       |                       | multi-account.        |
+-----------------------+-----------------------+-----------------------+
| Password protection   | YubiKey PINs          | Separate from         |
|                       |                       | FIDO/PIV/OpenPGP PIN. |
|                       |                       | Optional, set         |
|                       |                       | independently.        |
+-----------------------+-----------------------+-----------------------+

**Important**: The OATH secret **can be exported** from the YubiKey if
you have the password — this is by design (backup). FIDO credentials
cannot be exported.

**********************************
 3. Setup for the Two-Device Case
**********************************

Both YubiKeys should store the **same** OATH secrets so B can generate
identical codes.

3a. Full Setup From Scratch
===========================

**Step 1 - Set a password (optional but recommended)**

.. code:: bash

   ykman oath access change
   # You will be prompted for the new password

Repeat on both keys with the **same password**. Store the password in
your encrypted backup.

**Step 2 - Add accounts to YubiKey A**

For each service, you need the TOTP secret (a base32 string from the
service's setup page or QR code).

.. code:: bash

   # Manual add
   ykman oath accounts add "service:username" BASE32SECRET

   # With issuer label
   ykman oath accounts add "username" --issuer "ServiceName" BASE32SECRET

   # Require touch for high-value accounts
   ykman oath accounts add "service:username" BASE32SECRET --touch

   # From an otpauth:// URI (scanned from a QR)
   ykman oath accounts uri "otpauth://totp/..."

**Step 3 - Export all accounts to PSKC for backup and import to B**

The cleanest approach is to save each account to an encrypted PSKC as
you add it, then import that file onto YubiKey B:

.. code:: bash

   # Add to A and save each account to an encrypted PSKC
   ykman oath accounts add "service:username" BASE32SECRET \
     --output /path/to/backup/oath-accounts-A.pskc \
     --pskc-passphrase "your-passphrase"

.. code:: bash

   # Import the PSKC onto YubiKey B (prompts for the PSKC passphrase)
   ykman oath accounts import /path/to/backup/oath-accounts-A.pskc

**Step 4 - Alternative: manual add to both keys**

For each account, add the same secret to both YubiKeys directly. This is
the safest approach but requires double-entry:

.. code:: bash

   # Add to A
   ykman oath accounts add "service:username" BASE32SECRET

   # Add to B
   ykman -d <B-SERIAL> oath accounts add "service:username" BASE32SECRET

**Step 5 - Verify**

.. code:: bash

   # List accounts
   ykman oath accounts list

   # Generate codes
   ykman oath accounts code

   # Generate code for a specific account
   ykman oath accounts code "service"

3b. Recovery After Losing Device A
==================================

#. Retrieve YubiKey B from safe location.
#. If you set a password on B, enter it when prompted.
#. YubiKey B has all the same OATH secrets (if you synced them), so you
   can generate codes immediately.
#. If B was never synced, import from the PSKC backup:

.. code:: bash

   ykman oath accounts import /path/to/backup/oath-accounts.pskc
   # You will be prompted for the PSKC passphrase used during export

5. You're back in business. Services see the same TOTP codes from B.

**Crisis scenario - no backup of secrets:**

-  Use recovery codes for each service to log in.
-  Set up TOTP again with a new key.
-  **This is why you back up OATH secrets.**

3c. Revocation
==============

OATH secrets themselves aren't revocable per se — the service doesn't
know which device generated a code. Steps:

#. **Regenerate TOTP secrets** for each service (disable 2FA, re-enable
   with a new secret). This invalidates the lost key's codes.
#. **Remove accounts from the lost key** (only useful if you later
   recover the key physically):

.. code:: bash

   ykman oath reset

Or delete specific accounts:

.. code:: bash

   ykman oath accounts delete "service:username"

3. **When setting up a replacement key**, restore from PSKC backup or
   re-add accounts manually.

3d. Notes
=========

-  **OATH is the one app whose secrets are recoverable.** The PSKC
   backup is the safety net — keep the PSKC *and* its passphrase in
   encrypted storage alongside the password.

-  **Storage is limited.** YubiKey 5 stores up to **32 OATH accounts**
   (some models may differ). Prioritize critical accounts, offload less
   critical TOTP codes to a password manager, and keep some accounts
   only on your phone.

***********
 4. Backup
***********

-  OATH password (if set)
-  PSKC file(s) with encrypted account data
-  Passphrase for PSKC encryption
