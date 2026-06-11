Yubico OTP
==========

1. What It Is
-------------

Yubico OTP is the original YubiKey application. It provides two
keyboard-emulation “slots” (slot 1 triggered by a short touch, slot 2 by
a long touch) that output a one-time password or static string as USB
keyboard scancodes.

1a. What It Is Used For
~~~~~~~~~~~~~~~~~~~~~~~

- **Yubico OTP credential** - a 44-character one-time password validated
  against Yubico's cloud validation service (or a self-hosted server).
  Used for web login on sites that support it.
- **Challenge-response (HMAC-SHA1 / TOTP)** - the YubiKey signs a
  challenge. Used with offline authenticators like ``pam_yubico`` for
  sudo/login, or with tools like ``ykchalresp``.
- **HOTP** - HMAC-based OATH OTP (RFC 4226). Counter-based one-time
  passwords.
- **Static password** - a fixed string up to 38 chars, emitted on touch.
  Useful for password manager master passwords.
- **NDEF (NFC)** - slot output is wrapped in an NDEF record so tapping
  an NFC reader triggers a URI or text.

2. Overlaps / Conflicts
-----------------------

+-----------------------+-----------------------+-----------------------+
| Feature               | Overlaps With         | Notes                 |
+=======================+=======================+=======================+
| OATH TOTP/HOTP via    | Standalone OATH app   | Slot-based TOTP is    |
| slot                  | (ykman oath)          | trickier to manage;   |
|                       |                       | OATH app stores many  |
|                       |                       | accounts. Prefer OATH |
|                       |                       | app for               |
|                       |                       | multi-account.        |
+-----------------------+-----------------------+-----------------------+
| Static password       | Password manager      | Convenient but the    |
|                       |                       | same string goes to   |
|                       |                       | *any* field; use only |
|                       |                       | as a *prefix* or for  |
|                       |                       | master-password       |
|                       |                       | entry.                |
+-----------------------+-----------------------+-----------------------+
| Challenge-response    | PIV (attestation)     | Different use case:   |
|                       |                       | CR is offline auth    |
|                       |                       | (pam_yubico), PIV is  |
|                       |                       | PKI. Can coexist.     |
+-----------------------+-----------------------+-----------------------+

3. Setup for Two-Device Use Case
--------------------------------

Both YubiKeys (A = daily, B = backup) should share identical slot
credentials so B can replace A seamlessly if lost.

3a. Full Setup From Scratch
~~~~~~~~~~~~~~~~~~~~~~~~~~~

**Step 1 - Generate credential files for Yubico OTP (slot 1)**

.. code:: bash

   # Generate and write credential to slot 1, saving the config to a file
   ykman otp yubiotp 1 --serial-public-id --generate-private-id --generate-key \
     --config-output /path/to/backup/yubico-otp-slot1.txt

Repeat for the **other** YubiKey (B):

.. code:: bash

   ykman otp yubiotp 1 --serial-public-id --generate-private-id --generate-key \
     --config-output /path/to/backup/yubico-otp-slot1.txt

- Each YubiKey gets its *own* unique credential (different private ID &
  key).
- Save the output file securely (e.g., encrypted backup).
- ``--serial-public-id`` uses the YubiKey serial as the public ID so the
  validation server can distinguish the two keys even though they'd both
  be registered to your account.

**Register both public IDs with your Yubico account** at
https://upgrade.yubico.com or your self-hosted validation server.

**Step 2 - Challenge-response (slot 2) for local auth**

Generate a random 20-byte HMAC-SHA1 key on both keys (identical key so
both give same response):

.. code:: bash

   # Generate key ONCE, write it down securely
   KEY=$(openssl rand -hex 20)

   # Program both YubiKeys with the SAME key
   ykman otp chalresp 2 $KEY
   ykman -d <B-SERIAL> otp chalresp 2 $KEY

Or generate per-key (unique keys):

.. code:: bash

   ykman otp chalresp 2 --generate
   ykman -d <B-SERIAL> otp chalresp 2 --generate

**Step 3 - Set access codes (optional)**

Prevent overwriting slots without a 6-byte hex access code:

.. code:: bash

   ACCESS_CODE=$(openssl rand -hex 6)
   ykman otp --access-code $ACCESS_CODE settings 1 --new-access-code $ACCESS_CODE
   ykman otp --access-code $ACCESS_CODE settings 2 --new-access-code $ACCESS_CODE

Store the access code(s) in your encrypted backup.

**Step 4 - Verify**

.. code:: bash

   ykman otp info
   # Both slots should show "programmed"

   # Test challenge-response
   ykman otp calculate 2 $(echo -n "test" | xxd -p)

3b. Recovery After Losing Device A
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

1. Take YubiKey B from safe location.
2. If B was already programmed identically (same challenge-response key,
   same static password), it works immediately for local auth.
3. For Yubico OTP: if you registered *both* public IDs with the
   validation service, B still works. If you only registered A's public
   ID, you need to:

   - Log into the validation service and register B's public ID.
   - Update any service that relies on the Yubico OTP.

4. For challenge-response with *unique* keys: reconfigure ``pam_yubico``
   or the relying app to accept B's key.
5. Remove A from any services/relying parties.
6. Order a replacement key; program it from the stored credential
   backups.

3c. Revocation
~~~~~~~~~~~~~~

- **Yubico OTP**: Log into your Yubico validation dashboard and revoke
  the lost device's public ID.
- **Challenge-response**: Remove the lost key's HMAC secret from
  ``/etc/yubico/`` or ``~/.yubico/``.
- **Slot access codes**: If you set access codes, the replacement key
  can be configured with the same codes. The lost key's codes cannot be
  changed (they're in the lost device), so services relying on it must
  be updated.

Data to Back Up (Encrypted)
---------------------------

- Yubico OTP private ID & key for each slot on each key
- Challenge-response keys (if shared)
- Access codes (if set)
- NDEF configuration (if used)
