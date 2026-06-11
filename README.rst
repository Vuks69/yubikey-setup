YubiKey Two-Device Setup Guide
==============================

YubiKey A (daily driver) and YubiKey B (backup, safe location).

**Hardware**: Two YubiKeys (5th gen or later recommended), here called A
(daily) and B (backup).

**Principle**: Both keys are configured to be interchangeable. For most
applications, the backup produces identical credentials. For FIDO, both
keys are registered independently with every service.

--------------

Prerequisites
-------------

Install required packages:

.. code:: bash

   # Find and install yubikey-manager, opensc, pcsc-lite, pinentry
   dnf search yubikey
   dnf info yubikey-manager
   sudo dnf install yubikey-manager opensc pcsc-lite pinentry gnupg2
   # pinentry-backend packages: pinentry-gtk, pinentry-qt, pinentry-curses

   # For PAM U2F (local auth)
   sudo dnf install pam-u2f pamu2fcfg

   # Enable and start pcscd
   sudo systemctl enable --now pcscd

Verify both keys are detected:

.. code:: bash

   ykman list

Use ``ykman -d <SERIAL>`` to target a specific key when both are plugged in.

--------------

Many commands below also need ``opensc`` installed for PKCS#11
operations (SSH with PIV) and ``pcsc-lite`` + ``pcscd`` service running
for smart card communication. Verify with:

.. code:: bash

   systemctl status pcscd

Step-by-Step Setup
------------------

0. Initial Configuration (both keys)
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

Apply matching configuration to both keys before setting up individual
applications.

1. **Set a configuration lock code** (optional — prevents unauthorized
   config changes):

   .. code:: bash

      ykman config set-lock-code --generate

   Repeat on both keys (each gets a unique code). Store codes in
   encrypted backup.

2. **Disable unused applications** (e.g., YubiHSM Auth if you don't own
   a YubiHSM):

   .. code:: bash

      ykman config usb --disable hsmauth
      ykman config nfc --disable hsmauth

3. **Review enabled interfaces**:

   .. code:: bash

      ykman config usb --list

--------------

1. FIDO (U2F / WebAuthn / Passkeys)
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

**Priority: HIGH** — this is your primary 2FA method for most services.

The only correct multi-device FIDO setup is to **register both keys
independently** with every service.

- `Full FIDO guide → <fido.rst>`__
- Set a **FIDO2 PIN** (same on both keys)
- **Register both keys** with every service that supports FIDO
- Set up ``pam-u2f`` for local Linux login

**Recovery**: As long as both keys are registered, you're fine. If only
A was registered, use backup codes to log in and register B.

--------------

2. OATH (TOTP 2FA Codes)
~~~~~~~~~~~~~~~~~~~~~~~~

**Priority: HIGH** — second-factor for services without FIDO support.

Store the same TOTP secrets on both keys so B generates identical codes.

- `Full OATH guide → <oath.rst>`__
- Set a password on both keys (same password)
- Add accounts to A, export with ``--output PSKC``, import to B
- Or add manually to both keys

**Recovery**: B has the same secrets. Works immediately. If secrets
weren't synced, restore from PSKC backup.

--------------

3. OpenPGP (GPG Signing, SSH, Encryption)
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

**Priority: MEDIUM** — needed for Git commit signing and SSH via
gpg-agent.

Create subkeys from an offline master key and load identical subkeys
onto both keys.

- `Full OpenPGP guide → <openpgp.rst>`__
- Deep reference:
  `drduh/YubiKey-Guide <https://github.com/drduh/YubiKey-Guide>`__ (the
  definitive OpenPGP-on-YubiKey resource)
- Generate offline master key (certify only), back up encrypted +
  revocation cert
- Create sign/encrypt/auth subkeys, transfer to both YubiKeys
- **Enable KDF** before changing PINs (see openpgp.rst)
- Set touch policies and PINs
- Write hardened ``gpg.conf`` and ``gpg-agent.conf``

**Recovery**: B has identical subkeys. SSH, signing, decryption work
immediately. If GPG asks for a specific serial number, run
``gpg-connect-agent "scd serialno" "learn --force" /bye`` to refresh the
stub.

--------------

4. PIV (Smart Card / PKI)
~~~~~~~~~~~~~~~~~~~~~~~~~

**Priority: MEDIUM** — needed for SSH via PKCS#11, TLS client certs,
S/MIME.

Each key gets its own unique key pairs, but certificates use coordinated
subjects so relying parties accept both.

- `Full PIV guide → <piv.rst>`__
- Reset PIV to factory defaults
- Set PIN/PUK/management key (same PIN on both)
- Generate key pairs on each key independently
- Create self-signed certificates (or get CA-signed)
- Add both SSH public keys to ``authorized_keys``

**Recovery**: If both SSH keys are in ``authorized_keys``, B works
immediately. For S/MIME/TLS, distribute B's certificate.

--------------

5. Yubico OTP
~~~~~~~~~~~~~

**Priority: LOW** — increasingly superseded by FIDO. Useful for legacy
services and local challenge-response auth (pam_yubico).

- `Full OTP guide → <otp.rst>`__
- Program Yubico OTP credentials (slot 1) on each key (unique per key)
- Register both public IDs with Yubico validation service
- Program challenge-response (slot 2, same key on both) for local auth

**Recovery**: Challenge-response works immediately if keys share the
same secret. Yubico OTP needs the public ID registered with the
validation service.

--------------

6. YubiHSM Auth
~~~~~~~~~~~~~~~

**Priority: SKIP** — only if you own a YubiHSM 2.

- `Full HSMAuth guide → <hsmauth.rst>`__

--------------

Backup and Storage
------------------

Store the following in an **encrypted** backup (e.g., VeraCrypt volume,
age-encrypted archive, password manager):

============================== ========================================
Item                           Source
============================== ========================================
Configuration lock codes       ``ykman config set-lock-code``
FIDO2 PIN                      Chosen during setup
OATH password                  ``ykman oath access change``
OATH PSKC file                 ``ykman oath accounts add --output``
PIV PIN / PUK                  Chosen during setup
OpenPGP master key (encrypted) ``gpg --export-secret-keys --armor``
OpenPGP revocation cert        ``gpg --gen-revoke``
OpenPGP Admin PIN              Chosen during setup
Yubico OTP config files        ``ykman otp yubiotp --config-output``
OTP access codes               ``ykman otp settings --new-access-code``
============================== ========================================

--------------

Recovery Summary (Lost YubiKey A)
---------------------------------

+-----------------------------------+------------------------------------------------------------+
| Application                       | What to do                                                 |
+===================================+============================================================+
| **FIDO**                          | Use B to log in. Remove A from each service.               |
+-----------------------------------+------------------------------------------------------------+
| **OATH**                          | B has same secrets. Works immediately.                     |
+-----------------------------------+------------------------------------------------------------+
| **OpenPGP**                       | B has same subkeys. SSH/signing/decrypt works. If GPG asks |
|                                   | for serial number, run                                     |
|                                   | ``gpg-connect-agent "scd serialno" "learn --force" /bye``. |
+-----------------------------------+------------------------------------------------------------+
| **PIV**                           | B's SSH key in authorized_keys → immediate. For PKI,       |
|                                   | distribute B's cert.                                       |
+-----------------------------------+------------------------------------------------------------+
| **Yubico OTP**                    | Register B's public ID with validation service.            |
|                                   | Challenge-response works if shared key.                    |
+-----------------------------------+------------------------------------------------------------+

--------------

Reference
---------

- ``ykman info`` — device status
- ``ykinfo -a`` — hardware info
- ``ykman <app> info`` — application status
- ``ykman <app> --help`` — command reference
- ``ykman config usb --list`` — enabled USB interfaces
- ``ykman config nfc --list`` — enabled NFC interfaces
