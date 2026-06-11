PIV (Personal Identity Verification)
====================================

1. What It Is
-------------

PIV is a smart-card application based on NIST SP 800-73. The YubiKey can
store up to 24 key pairs / certificates in numbered slots and perform
cryptographic operations (signature, decryption, authentication) without
the private key ever leaving the device.

1a. What It Is Used For
~~~~~~~~~~~~~~~~~~~~~~~

- **SSH authentication** - use the PIV 9a key to authenticate via OpenSC
  PKCS#11 (see Step 8).
- **Code signing** - sign Git commits, binaries, or documents.
- **S/MIME email encryption & signing** - store certificates for email
  security.
- **TLS client certificates** - authenticate to web services.
- **Local login (smart-card PAM)** - log in to Linux/macOS/Windows with
  a smart card.
- **Disk encryption** - unlock LUKS with a PIV key (via ``yubikey-luks``
  or similar).
- **Windows logon** - native PIV support in Windows.
- **Wi-Fi authentication (EAP-TLS)** - enterprise network
  authentication.

PIV Slots (common assignments):\n\n+----------+---------------------------+---------------+\n| Slot     | Usage                     | Key           |\n+==========+===========================+===============+\n| 9a       | Authentication            | PIN required  |\n+----------+---------------------------+---------------+\n| 9c       | Digital Signature         | PIN required  |\n+----------+---------------------------+---------------+\n| 9d       | Key Management            | PIN required  |\n|          | (encryption)              |               |\n+----------+---------------------------+---------------+\n| 9e       | Card Authentication       | No PIN (touch |\n|          |                           | only)         |\n+----------+---------------------------+---------------+\n| 82-95    | Retired / alternate keys  | Various       |\n+----------+---------------------------+---------------+

2. Overlaps / Conflicts
-----------------------

+-----------------------+-----------------------+-----------------------+
| Feature               | Overlaps With         | Notes                 |
+=======================+=======================+=======================+
| SSH auth via PIV      | SSH auth via OpenPGP  | Both work. PIV uses   |
|                       | (authentication key)  | PKCS#11, OpenPGP uses |
|                       |                       | gpg-agent. Choose     |
|                       |                       | one.                  |
+-----------------------+-----------------------+-----------------------+
| Git commit signing    | OpenPGP signing       | PIV can do it via     |
|                       |                       | PKCS#11; OpenPGP via  |
|                       |                       | gpg. Different        |
|                       |                       | keyrings.             |
+-----------------------+-----------------------+-----------------------+
| Encryption            | OpenPGP encryption    | PIV uses X.509,       |
|                       |                       | OpenPGP uses its own  |
|                       |                       | PKI.                  |
+-----------------------+-----------------------+-----------------------+
| Slot 9a (auth)        | Card slot 9e          | Different PIN         |
|                       |                       | policies; 9e is       |
|                       |                       | touch-only for        |
|                       |                       | low-security.         |
+-----------------------+-----------------------+-----------------------+

3. Setup for Two-Device Use Case
--------------------------------

Each YubiKey should have its **own** private keys (they never leave the
hardware). X.509 certificates for the backup key should be signed by the
same CA (or self-signed with the same subject) so the relying party
trusts either key.

3a. Full Setup From Scratch
~~~~~~~~~~~~~~~~~~~~~~~~~~~

**Step 1 - Reset PIV to factory defaults (new keys)**

.. code:: bash

   ykman piv reset

**Step 2 - Change PIN and PUK from defaults**

Defaults: PIN=123456, PUK=12345678.

.. code:: bash

   # Change PIN
   ykman piv access change-pin --pin 123456 --new-pin <NEW-PIN>

   # Change PUK
   ykman piv access change-puk --puk 12345678 --new-puk <NEW-PUK>

Use the **same PIN/PUK on both keys** for consistency.

**Step 3 - Generate or store the management key**

Use a PIN-protected, YubiKey-stored management key (recommended):

.. code:: bash

   ykman piv access change-management-key --generate --protect

This generates a random management key stored on the YubiKey, unlocked
by the PIN.

Repeat on both keys (each gets a unique random key).

**Step 4 - Set retry counts**

.. code:: bash

   ykman piv access set-retries 5 5

The management key from Step 3 is used automatically when PIN-protected.

**Step 5 - Generate key pairs on each YubiKey**

For each key slot you need, generate on YubiKey A, then on YubiKey B.
Add ``--pin-policy`` and ``--touch-policy`` flags to override defaults:

.. code:: bash

   # Example: Authentication key (slot 9a) with ECC P-256
   ykman piv keys generate --algorithm ECCP256 9a pubkey-9a-A.pem

   # Example: Signature key (slot 9c) with ECC P-256
   # Require PIN every time, cache touch for 15s
   ykman piv keys generate --algorithm ECCP256 --pin-policy always --touch-policy cached 9c pubkey-9c-A.pem

   # Example: Encryption/Key Management key (slot 9d) with RSA 2048
   ykman piv keys generate --algorithm RSA2048 9d pubkey-9d-A.pem

Now do the same for YubiKey B, saving to different filenames:

.. code:: bash

   ykman -d <B-SERIAL> piv keys generate --algorithm ECCP256 9a pubkey-9a-B.pem
   ykman -d <B-SERIAL> piv keys generate --algorithm ECCP256 --pin-policy always --touch-policy cached 9c pubkey-9c-B.pem
   ykman -d <B-SERIAL> piv keys generate --algorithm RSA2048 9d pubkey-9d-B.pem

**Step 6 - Card auth key (optional)**

Generate a low-security card auth key (slot 9e) for touch-only operations
(no PIN required):

.. code:: bash

   ykman piv keys generate --algorithm ECCP256 --pin-policy never --touch-policy always 9e pubkey-9e.pem

**Step 7 - Generate self-signed certificates**

Create self-signed certificates for each slot on each key:

.. code:: bash

   # YubiKey A
   ykman piv certificates generate --subject "CN=User Auth, O=YubiKey-A" 9a pubkey-9a-A.pem
   ykman piv certificates generate --subject "CN=Digital Sig, O=YubiKey-A" 9c pubkey-9c-A.pem
   ykman piv certificates generate --subject "CN=Key Mgmt, O=YubiKey-A" 9d pubkey-9d-A.pem

   # YubiKey B (note: same CN so services see equivalent certs)
   ykman -d <B-SERIAL> piv certificates generate --subject "CN=User Auth, O=YubiKey-B" 9a pubkey-9a-B.pem
   ykman -d <B-SERIAL> piv certificates generate --subject "CN=Digital Sig, O=YubiKey-B" 9c pubkey-9c-B.pem
   ykman -d <B-SERIAL> piv certificates generate --subject "CN=Key Mgmt, O=YubiKey-B" 9d pubkey-9d-B.pem

**Step 8 - Set up SSH authentication**

The PIV 9a key generated in Step 5 can be used for SSH authentication.
Convert the PEM public key to SSH format and add it to ``authorized_keys``:

.. code:: bash

   # Convert PEM to SSH one-line format and add to authorized_keys
   ssh-keygen -i -m PKCS8 -f pubkey-9a-A.pem >> ~/.ssh/authorized_keys

   # Configure SSH to use the PIV key via OpenSC PKCS#11
   # In ~/.ssh/config:
   #   Host *
   #     PKCS11Provider /usr/lib64/opensc-pkcs11.so

For YubiKey B, convert its public key and add to ``authorized_keys``:

.. code:: bash

   ssh-keygen -i -m PKCS8 -f pubkey-9a-B.pem | \
     ssh <user>@<host> 'cat >> ~/.ssh/authorized_keys'

**Step 9 - Generate CHUID and CCC (optional)**

.. code:: bash

   ykman piv objects generate CHUID
   ykman piv objects generate CCC

**Step 10 - Verify**

.. code:: bash

   ykman piv info
   # Should show: PIN/PUK tries, management key status, and slot info

3b. Recovery After Losing Device A
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

1. Retrieve YubiKey B from safe location.
2. Insert B and enter its PIN.
3. **SSH**: B's public key was added to ``authorized_keys`` during setup
   — it works immediately.
4. **Code signing / S/MIME**: If B's certificates were signed by the
   same CA and have the same subject names, relying parties should
   accept them. If self-signed, you need to distribute B's certificates
   to each relying party.
5. **TLS client cert**: Install B's certificate on each server that
   needs it.
6. Remove A's public keys/certificates from all services.

3c. Revocation
~~~~~~~~~~~~~~

Private keys on the lost YubiKey A cannot be deleted remotely.
Mitigations:

1. **SSH**: Remove A's public key from all ``authorized_keys`` files.

2. **Certificates**: If signed by an internal CA, publish a Certificate
   Revocation List (CRL) or use OCSP stapling for A's certificates.

3. **Self-signed certs**: Manually replace trust anchors everywhere A's
   certificate was imported.

4. **Management key**: If A is later recovered physically, you can do a
   full PIV reset:

   .. code:: bash

      ykman piv reset

   This wipes all keys and certificates and restores factory PIN/PUK.

5. **Attestation**: Use attestation certs to prove B's keys were
   generated on a YubiKey (not copied):

   .. code:: bash

      ykman piv keys attest 9a attest-9a-B.pem

Attestation
-----------

PIV attestation proves a key was generated on the YubiKey (not
imported). It requires the attestation key, which is present on YubiKeys
manufactured after 2020.

.. code:: bash

   # Generate attestation certificate for slot 9a
   ykman piv keys attest 9a /path/to/backup/attest-9a-A.pem

Attestation certificates can be verified against Yubico's public CA.

Data to Back Up
---------------

- PIN, PUK (store in encrypted password manager)
- Public keys for all slots on both keys (for easy recovery, add to
  services)
- Attestation certificates (for proof of hardware generation)
- Any CSRs / CA-signed certificates
- Management key (if not using PIN-protected storage)
