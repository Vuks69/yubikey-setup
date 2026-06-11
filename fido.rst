FIDO (U2F / FIDO2 / WebAuthn)
=============================

1. What It Is
-------------

FIDO encompasses two related protocols:

- **FIDO U2F** (Universal 2nd Factor) - the original standard for
  hardware-backed second-factor authentication. Uses a single key pair
  per origin. No PIN required.
- **FIDO2 / WebAuthn** - modern standard supporting passwordless
  (resident keys), 2FA, and discoverable credentials. PIN required for
  resident key operations.

Both live in the same application space on the YubiKey. Resetting one
resets the other.

1a. What It Is Used For
~~~~~~~~~~~~~~~~~~~~~~~

- **Second-factor (U2F / 2FA)** - tap to approve login after password.
- **Passwordless (FIDO2 resident keys)** - the private key is stored on
  the YubiKey; the user authenticates with PIN + touch, no server-side
  password hash needed.
- **Passkeys** - Apple/Google/Microsoft's evolution of FIDO2 for
  cross-device syncing (optional). On YubiKey, passkeys are resident
  credentials.
- **Platform authentication** - Windows Hello, macOS Touch ID
  alternatives, Linux ``pam-u2f``.

2. Overlaps / Conflicts
-----------------------

+-----------------------+-----------------------+-----------------------+
| Feature               | Overlaps With         | Notes                 |
+=======================+=======================+=======================+
| FIDO2 PIN             | PIV PIN               | Completely separate.  |
|                       |                       | Different PINs,       |
|                       |                       | different retry       |
|                       |                       | counters.             |
+-----------------------+-----------------------+-----------------------+
| Resident credentials  | OATH (stored secrets) | Different protocols.  |
|                       |                       | FIDO2 secrets never   |
|                       |                       | leave the key. OATH   |
|                       |                       | secrets can be backed |
|                       |                       | up.                   |
+-----------------------+-----------------------+-----------------------+
| Multiple keys         | Multiple              | You register **both** |
|                       | registrations         | YubiKeys with each    |
|                       |                       | service. This is the  |
|                       |                       | only correct way.     |
+-----------------------+-----------------------+-----------------------+

3. Setup for Two-Device Use Case
--------------------------------

FIDO is inherently multi-device friendly: you register each YubiKey
independently with every service that supports it. No keys are shared
between devices.

3a. Full Setup From Scratch
~~~~~~~~~~~~~~~~~~~~~~~~~~~

**Step 1 - Set a FIDO2 PIN (same on both keys)**

.. code:: bash

   ykman fido access change-pin
   # Enter current PIN (default: none on new key — just press Enter)
   # Enter new PIN (4+ chars, min length configurable)

Repeat on both YubiKeys. Use the **same PIN** for convenience so muscle
memory works on the backup.

**Step 2 - Configure PIN length (optional)**

.. code:: bash

   ykman fido access set-min-length 8

Both keys should match.

**Step 3 - Register with services**

For every service (Google, GitHub, Microsoft, etc.):

1. Go to account security settings → Add security key.
2. Insert YubiKey A, tap when prompted.
3. Complete registration.
4. Repeat with YubiKey B on the **same service page** — register it as a
   second key.

This is the most important step: you must register **both** keys with
every service.

**Step 4 - Set up passwordless login (optional)**

Services supporting passkeys/resident keys: - Follow the service's
passkey registration flow. - The YubiKey will prompt for PIN + touch. -
Register both keys.

**Step 5 - Set up local sudo/login (Linux) with pam-u2f**

.. code:: bash

   # Install pam-u2f + pamu2fcfg (package name varies: libpam-u2f, pam-u2f, etc.)

   # Add both keys to the authfile
   pamu2fcfg -n > ~/.config/Yubico/u2f_keys
   # Tap YubiKey A
   pamu2fcfg -n >> ~/.config/Yubico/u2f_keys
   # Insert YubiKey B, tap it

   # Enable PAM module (add to /etc/pam.d/sudo or similar)
   # auth required pam_u2f.so authfile=/home/USER/.config/Yubico/u2f_keys

3b. Recovery After Losing Device A
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

FIDO recovery is the hardest — this is a known pain point of hardware
security keys.

**If you registered both keys:**

1. Use YubiKey B to log into every service.
2. Remove YubiKey A from each service's security settings.
3. Register a replacement key as a new second factor.

**If you only registered A:**

1. Use fallback methods (TOTP, recovery codes, SMS) to log into each
   service.
2. Register B (and a new replacement) once logged in.
3. **This is why you always register both keys.**

**IMPORTANT**: FIDO credentials are **not exportable**. There is no
“backup” for a FIDO private key. The only backup is having a second
physical key registered.

3c. Revocation
~~~~~~~~~~~~~~

- **Per-service**: Remove the lost key from each service's security
  settings.
- **FIDO reset** (nuclear option): If you have the physical key, you can
  wipe it:

.. code:: bash

   # NOTE: Must be triggered immediately after inserting the YubiKey
   #       and requires a touch confirmation
   ykman fido reset

This deletes ALL FIDO credentials on that key and clears the PIN.

- **Credential management**: You can list and delete individual resident
  credentials on a key you still possess:

  .. code:: bash

     ykman fido credentials list
     ykman fido credentials delete <CREDENTIAL_ID>

Reserve Notes
-------------

- **PAM U2F** supports **multiple keys** in a single authfile. Run
  ``pamu2fcfg -n`` for each key and append to the same file.
- **Always Register Both**: Every time you add a new FIDO-enabled
  service, register both YubiKeys before walking away.
- **Minimum PIN length**: Use ``ykman fido access set-min-length 8`` on
  both keys for consistency.
- If a service shows “Security key already registered” when adding B,
  you likely need to use a different slot/credential or the service may
  not support multiple keys — use TOTP/backup codes as fallback.
