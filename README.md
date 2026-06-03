# dsmr-email-admin

Health check and repair tool for the [DSMR mail stack](https://github.com/fade/dsmr-qmail)
(qmail + vpopmail + dovecot).

## Why this exists

Several DSMR control and configuration files are written by a postinst only
*if they don't already exist*. That is the right policy for files that hold
live credentials or admin choices — an upgrade must not clobber them — but it
has a cost: a host first provisioned **before** a packaging fix keeps the old,
broken value forever, because the upgrade never rewrites it.

In practice this produced a recurring set of faults on older-than-the-fix hosts:

| Fault | Symptom |
| --- | --- |
| `control/softlimit` too low (12 MB) | qmail-smtpd links OpenSSL 3 + krb5 and dies with `libgssapi_krb5.so.2: failed to map segment` — inbound :25 and submission :587 both broken |
| `vpopmail.pgsql` password/database fields swapped | vpopmail and dovecot can't authenticate to PostgreSQL |
| dovecot SQL conf out of sync with `vpopmail.pgsql` | dovecot auth returns `temp_fail` |
| vpopmail's PostgreSQL cluster stopped / not enabled | no virtual-user auth or delivery; lost after reboot |

`dsmr-email-admin` turns the manual diagnosis-and-repair for these into two
commands, plus an end-to-end self test.

## Usage

```sh
# read-only audit of the whole stack
sudo dsmr-email-admin check

# preview repairs, then apply them (each backs up the file it changes)
sudo dsmr-email-admin fix --dry-run
sudo dsmr-email-admin fix

# repair one thing
sudo dsmr-email-admin fix softlimit

# prove delivery + dovecot auth end to end (throwaway mailbox, auto-removed)
sudo dsmr-email-admin selftest
```

Each `fix` runs only when its detector fires, so a value you set deliberately
is left alone. See `dsmr-email-admin(8)`.

The companion tool [`dsmr-smtpauth(8)`](https://github.com/fade/dsmr-checkpassword-dovecot)
manages the SMTP-AUTH relay allow-list; `dsmr-email-admin` audits the stack
those credentials run on.
