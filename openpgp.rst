OpenPGP
=======

**Deep reference**:
`drduh/YubiKey-Guide <https://github.com/drduh/YubiKey-Guide>`__ — the
definitive upstream guide dedicated solely to OpenPGP on YubiKey. Covers
air-gapped generation, subkey rotation, SSH agent forwarding, and
troubleshooting in depth. This doc condenses the essentials for a
two-device setup.

1. What It Is
-------------

The OpenPGP application on the YubiKey implements the OpenPGP Card
specification (version 3.4). It provides three key slots (signature,
decryption, authentication) and stores the corresponding private keys on
the device. Private keys never leave the YubiKey — they are generated
on-device or imported and then deleted from the host.

1a. What It Is Used For
~~~~~~~~~~~~~~~~~~~~~~~

- **Git commit & tag signing** - sign commits with
  ``gpg --sign``/``git commit -S``.
- **SSH authentication** - use ``gpg-agent`` as an SSH agent
  (``gpg --export-ssh-key``).
- **Email encryption & signing** - OpenPGP (GPG) for email with
  Thunderbird or Mutt.
- **File encryption** - ``gpg -e/-d`` for symmetric or asymmetric
  encryption.
- **Package signing** - sign ``.deb``, ``.rpm``, or other packages.
- **Authentication to services** - some services accept GPG-signed
  tokens.

Key slots:\n\n+------------+----------------+-------------------------------+\n| Slot       | Usage          | Example                       |\n+============+================+===============================+\n| SIG (1)    | Signing        | Git commits, emails, files    |\n+------------+----------------+-------------------------------+\n| DEC (2)    | Decryption     | Email decryption, file        |\n|            |                | decryption                    |\n+------------+----------------+-------------------------------+\n| AUT (3)    | Authentication | SSH, TLS client cert          |\n+------------+----------------+-------------------------------+

2. Overlaps / Conflicts
-----------------------

+-----------------------+-----------------------+-----------------------+
| Feature               | Overlaps With         | Notes                 |
+=======================+=======================+=======================+
| SSH auth via AUT key  | PIV slot 9a / 9e      | gpg-agent vs PKCS#11. |
|                       |                       | Pick one agent.       |
+-----------------------+-----------------------+-----------------------+
| Git commit signing    | PIV signature slot 9c | GPG is more common in |
|                       |                       | open-source.          |
+-----------------------+-----------------------+-----------------------+
| Encryption            | PIV slot 9d (key      | OpenPGP uses its own  |
|                       | mgmt)                 | PKI; PIV uses X.509.  |
+-----------------------+-----------------------+-----------------------+
| PINs                  | PIV PIN               | Completely separate.  |
|                       |                       | Different PINs,       |
|                       |                       | different retries.    |
+-----------------------+-----------------------+-----------------------+

3. Setup for Two-Device Use Case
--------------------------------

Both YubiKeys get subkeys from the **same master key**. The master key
(certify-only) stays offline in encrypted storage. Subkeys are loaded
onto both YubiKeys.

**Critical**: Subkeys are identical on both keys. B can decrypt data
intended for A.

3a. Full Setup From Scratch
~~~~~~~~~~~~~~~~~~~~~~~~~~~

**Step 1 — Create hardened GPG config**

.. code:: bash

   mkdir -p ~/.gnupg && chmod 700 ~/.gnupg

   cat > ~/.gnupg/gpg.conf << 'EOF'
   personal-cipher-preferences AES256 AES192 AES
   personal-digest-preferences SHA512 SHA384 SHA256
   personal-compress-preferences ZLIB BZIP2 ZIP Uncompressed
   default-preference-list SHA512 SHA384 SHA256 AES256 AES192 AES ZLIB BZIP2 ZIP Uncompressed
   cert-digest-algo SHA512
   s2k-digest-algo SHA512
   s2k-cipher-algo AES256
   charset utf-8
   no-comments
   no-emit-version
   no-greeting
   keyid-format 0xlong
   list-options show-uid-validity
   verify-options show-uid-validity
   with-fingerprint
   require-cross-certification
   require-secmem
   no-symkey-cache
   armor
   use-agent
   # throw-keyids  # Uncomment to hide recipient key ID (WARNING: breaks Mailvelope)
   EOF

**Step 2 — Create gpg-agent config**

.. code:: bash

   cat > ~/.gnupg/gpg-agent.conf << 'EOF'
   enable-ssh-support
   default-cache-ttl 600
   max-cache-ttl 7200
   # pinentry-program /usr/bin/pinentry  # adjust for your system
   EOF

Add this to your shell rc:

.. code:: bash

   export GPG_TTY=$(tty)
   export SSH_AUTH_SOCK=$(gpgconf --list-dirs agent-ssh-socket)
   gpgconf --launch gpg-agent

**Step 3 — Create an offline master key**

.. code:: bash

   # Generate master key (certify only)
   gpg --full-generate-key
   # Type: 9 (ECC sign+encrypt), Curve: 1 (Ed25519/Cv25519)
   # Expiration: do NOT set one on the certify key (set it on subkeys only)
   # Real name: Your Name
   # Email: your@email.com

   # List the key and note the ID
   gpg --list-secret-keys --keyid-format LONG

**Step 4 — Create subkeys (sign, encrypt, auth)**

.. code:: bash

   gpg --expert --edit-key <KEYID>

   # Add signing subkey (Ed25519)
   addkey
   # Type: 8 (ECC sign only), Curve: Ed25519
   # Set expiration (e.g., 2y)

   # Add encryption subkey (Cv25519)
   addkey
   # Type: 10 (ECC encrypt only), Curve: Cv25519
   # Set expiration

   # Add authentication subkey
   addkey
   # Type: 11 (Existing + auth) then pick the Ed25519 key
   # Set expiration

   save

   drduh recommends RSA/4096 for compatibility (some systems reject
   Ed25519 for SSH). If you hit issues, use ``KEY_TYPE=rsa4096``
   instead.

**Step 5 — Backup and export keys**

Before touching the YubiKey, back everything up:

.. code:: bash

   gpg --export --armor <KEYID> > public-key.asc
   gpg --export-secret-keys --armor <KEYID> > master-key.asc
   gpg --export-secret-subkeys --armor <KEYID> > subkeys.asc
   gpg --gen-revoke --armor <KEYID> > revoke.asc

Store these files in **encrypted offline storage** (LUKS USB,
age-encrypted archive, etc.). The master key must never live on an
online machine outside of this setup step.

**Step 6 — Enable KDF on the YubiKey (critical — do this first!)**

KDF (Key Derived Function) hashes the PIN on the YubiKey so it is never
sent as plaintext. This must be done **before** changing PINs or moving
keys.

.. code:: bash

   gpg --card-edit
   admin
   kdf-setup
   12345678  # default Admin PIN
   quit

**Warning**: KDF breaks compatibility with older GnuPG versions
(especially mobile clients). If you need those, skip this step.

**Step 7 — Configure YubiKey A**

.. code:: bash

   # Change PIN and Admin PIN from defaults
   # Default: PIN=123456, Admin PIN=12345678
   ykman openpgp access change-pin
   ykman openpgp access change-admin-pin

   # Set retry counts (default 3 — increase to 5)
   ykman openpgp access set-retries 5 5 5

   # Set touch policies (optional)
   # Cached/Cached-Fixed are better for email clients — 15s cache
   ykman openpgp keys set-touch sig on
   ykman openpgp keys set-touch dec on
   ykman openpgp keys set-touch aut on

   # Set card holder info
   gpg --card-edit
   admin
   login
   Your Name <your@email.com>
   quit

Use the **same PIN and Admin PIN** on both keys.

**Step 8 — Transfer subkeys to YubiKey A**

.. code:: bash

   # Switch to a temporary GNUPGHOME
   mkdir -p /tmp/gpg-transfer
   export GNUPGHOME=/tmp/gpg-transfer
   gpg --import /path/to/backup/master-key.asc

   # Insert YubiKey A
   gpg --card-status

   # Transfer subkeys (this deletes them from the temp dir)
   gpg --expert --edit-key <KEYID>
   key 1        # select sig
   keytocard
   1            # signature key
   key 1        # deselect sig
   key 2        # select dec
   keytocard
   2            # encryption key
   key 2        # deselect dec
   key 3        # select auth
   keytocard
   3            # authentication key
   save

**Step 9 — Prevent repeated insert prompts**

.. code:: bash

   echo "disable-ccid" >> ~/.gnupg/scdaemon.conf

This stops GnuPG from repeatedly asking you to insert an
already-inserted YubiKey.

**Step 10 — Transfer subkeys to YubiKey B**

.. code:: bash

   # Insert YubiKey B
   gpg --card-status

   # Transfer subkeys (same process)
   gpg --expert --edit-key <KEYID>
   key 1 ; keytocard ; 1 ; key 1
   key 2 ; keytocard ; 2 ; key 2
   key 3 ; keytocard ; 3
   save

**Step 11 — Set up SSH**

.. code:: bash

   # SSH public key
   ssh-add -L
   # Or: gpg --export-ssh-key <KEYID> > ~/.ssh/id_ed25519_gpg.pub

   # Add to authorized_keys
   ssh-copy-id user@host

When using ``IdentityFile`` in ``~/.ssh/config``, point it to the
**public** key, not private:

::

   Host github.com
     IdentitiesOnly yes
     IdentityFile ~/.ssh/id_ed25519_gpg.pub

**Step 12 — Configure Git signing**

.. code:: bash

   git config --global user.signingkey <SUBKEY-ID>!
   git config --global commit.gpgsign true

**Step 13 — Verify**

.. code:: bash

   # Check card — note 'sec#' means certify key is offline
   gpg -K                  # 'ssb>' means key is on smart card
   gpg --card-status
   echo "test" | gpg --clearsign
   ssh user@host

4. Daily Use Notes
------------------

- **Switch between YubiKeys**: Remove key A, run
  ``gpg-connect-agent updatestartuptty /bye``, insert key B.
- **PIN caching**: ``cache-ttl`` in ``gpg-agent.conf`` does **not**
  apply to smart cards — the YubiKey caches the PIN itself. Remove the
  key to clear the cache.
- **``throw-keyids``**: Enables ``throw-keyids`` in gpg.conf for
  privacy, but **breaks Mailvelope**. If you use Mailvelope, keep it
  commented out.
- **Thunderbird**: The ``armor`` option in gpg.conf can cause decryption
  failures. If messages fail to decrypt, remove ``armor`` from gpg.conf.

4a. Recovering Public Key From YubiKey
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

If you lose the public key but still have the YubiKey:

.. code:: bash

   gpg --card-status                    # shows key fingerprints
   gpg --export -a <KEYID> > key.asc   # from another machine that has the key
   # Or fetch from a keyserver if you uploaded it
   gpg --recv-key <KEYID>

5. Recovery After Losing Device A
---------------------------------

1. Take YubiKey B from safe location.

2. Insert B and enter PIN.

3. **SSH**: B has same auth subkey — works immediately.

4. **Git signing**: Same signing subkey — ``git commit -S`` works.

5. **Decryption**: Same encryption subkey — can decrypt old data.

6. **Stub refresh** (if you get “Please insert card with serial
   number”): run

   .. code:: bash

      gpg-connect-agent "scd serialno" "learn --force" /bye

   This recreates the correct stub for YubiKey B.

6. Revocation / Rotation
------------------------

+-----------------------------------+-----------------------------------+
| Action                            | Effect                            |
+===================================+===================================+
| **Renew subkeys**                 | Extend expiration date. Requires  |
|                                   | master key. Subkeys unchanged.    |
+-----------------------------------+-----------------------------------+
| **Rotate subkeys**                | Generate new subkeys. Old ones    |
|                                   | can't sign/encrypt but can still  |
|                                   | decrypt old data.                 |
+-----------------------------------+-----------------------------------+
| **Full revocation**               | Publish revocation cert. Abandon  |
|                                   | the identity.                     |
+-----------------------------------+-----------------------------------+

**Renew** (needs master key):

.. code:: bash

   gpg --quick-set-expire <KEYFP> 2028-06-01 \
     $(gpg -K --with-colons | awk -F: '/^fpr:/ {print $10}' | tail -n+2 | tr '\n' ' ')

**Rotate**: Generate new subkeys under the same master key (same process
as Step 4), then transfer to both YubiKeys. Old subkeys are deleted from
the YubiKeys and cannot be recovered.

**Revoke** (nuclear):

.. code:: bash

   gpg --import revoke.asc
   gpg --send-key <KEYID>

Warnings
--------

- **3 wrong User PINs** = PIN blocked. Unblock with Admin PIN or Reset
  Code.
- **3 wrong Admin/Reset Code** = data destroyed. YubiKey must be reset
  and re-provisioned.
- **Stub overwrite**: When you transfer subkeys to YubiKey B, the stub
  in your GPG keyring points to B's serial number. Both keys have the
  same subkeys, but the stub only remembers the last YubiKey written.
  See "Recovery After Losing Device A" for the fix.

Data to Back Up (Offline, Encrypted)
------------------------------------

- Master key (certify-only, encrypted passphrase)
- Public key (``gpg --export --armor <KEYID>``)
- Revocation certificate (``gpg --gen-revoke``)
- Subkeys: **only exist on the YubiKeys** (you backed up the stub
  versions above)
- PIN, Admin PIN, Reset Code
