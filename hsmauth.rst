YubiHSM Auth
============

1. What It Is
-------------

YubiHSM Auth is a YubiKey application that stores credentials for
authenticating to a YubiHSM 2 hardware security module. The YubiKey
serves as a “key to the HSM” — you touch the YubiKey to authorize
operations on the YubiHSM.

1a. What It Is Used For
~~~~~~~~~~~~~~~~~~~~~~~

- **YubiHSM 2 authentication** - the primary and intended use case.
- **Asymmetric credential storage** - generate and store Ed25519 or ECC
  P-256 key pairs for custom authentication schemes.
- **Symmetric credential storage** - store HMAC-SHA256 shared secrets
  derived from a password.
- **Credential management** - list, add, delete, and change passwords
  for stored credentials.

**Note**: Unless you own a YubiHSM 2, this application is unused. It is
enabled by default on new YubiKeys.

2. Overlaps / Conflicts
-----------------------

+-----------------------+-----------------------+-----------------------+
| Feature               | Overlaps With         | Notes                 |
+=======================+=======================+=======================+
| Asymmetric keys       | PIV / OpenPGP         | Different protocol,   |
|                       |                       | different use case    |
|                       |                       | (HSM auth vs          |
|                       |                       | PKI/OpenPGP). Can     |
|                       |                       | coexist.              |
+-----------------------+-----------------------+-----------------------+
| Symmetric secrets     | OATH                  | OATH is for TOTP/HOTP |
|                       |                       | codes. HSMAuth is for |
|                       |                       | HSM authentication.   |
+-----------------------+-----------------------+-----------------------+

3. Setup for Two-Device Use Case
--------------------------------

Both YubiKeys should store the **same** credentials so either can
authenticate to the YubiHSM.

3a. Full Setup From Scratch
~~~~~~~~~~~~~~~~~~~~~~~~~~~

**Step 1 - Set management key**

.. code:: bash

   # Change the management key password
   ykman hsmauth access change-management-password

Use the **same password** on both keys. Store in encrypted backup.

**Step 2 - Add credentials**

Symmetric credential (derived from password):

.. code:: bash

   ykman hsmauth credentials derive "my-hsm-cred" --password <PASSWORD>

Asymmetric credential (generate on-device):

.. code:: bash

   ykman hsmauth credentials generate "my-asym-cred" --algorithm ECCP256

Symmetric credential (import existing key):

.. code:: bash

   ykman hsmauth credentials symmetric "my-sym-cred" --key HEXKEY

**Step 3 - Repeat on YubiKey B**

.. code:: bash

   # For derived/symmetric: use the same password/key
   ykman -d <B-SERIAL> hsmauth credentials derive "my-hsm-cred" --password <PASSWORD>

   # For asymmetric: generate is unique per device (cannot be duplicated)
   ykman -d <B-SERIAL> hsmauth credentials generate "my-asym-cred" --algorithm ECCP256

**Step 4 - Verify**

.. code:: bash

   ykman hsmauth credentials list

3b. Recovery After Losing Device A
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

1. Take YubiKey B from safe storage.
2. For **symmetric/derived** credentials: B has the same shared secret —
   the YubiHSM accepts it immediately.
3. For **asymmetric** credentials: if B created its own unique key pair,
   the HSM must be configured to trust B's public key.
4. If B was never synced: use backup credential material to re-add them.
5. Remove A's public keys from the YubiHSM's allowed key list.

3c. Revocation
~~~~~~~~~~~~~~

Since credentials are standalone (no CA/issuer):

1. **Symmetric**: Change the shared secret in the YubiHSM. Old
   credential on lost A is useless.

2. **Asymmetric**: Remove A's public key from the HSM's authorized keys
   list.

3. **YubiHSM Auth reset** (if you recover the key physically):

   .. code:: bash

      ykman hsmauth reset

When to Disable
---------------

If you don't own a YubiHSM, disable HSMAuth to reduce attack surface and
free up NFC/USB bandwidth:

.. code:: bash

   ykman config usb --disable hsmauth
   ykman config nfc --disable hsmauth
