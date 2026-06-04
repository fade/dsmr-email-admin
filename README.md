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

## DKIM and SPF for virtual domains

Gmail and Yahoo now reject mail whose sending domain authenticates with
neither SPF nor DKIM (`550-5.7.26 ... the sender is unauthenticated`). On a
multi-domain vpopmail host only the primary domain is usually signed, so every
other hosted domain is undeliverable to the big providers until it gets an SPF
record and a DKIM key.

`dsmr-email-admin` provisions the signing side and hands you ready-to-publish
DNS. It **never writes to a nameserver** — the hosted domains use a mix of
in-house and external-registrar DNS — so it writes a per-domain file of
copy-pasteable records to `/var/lib/qmail/control/authdns/<domain>` for you to
enter by hand.

```sh
# which virtual domains are signed; full DNS coverage table
sudo dsmr-email-admin dkim list
sudo dsmr-email-admin authdns status

# narrative findings report — what's authenticated, what isn't, and why it matters
sudo dsmr-email-admin authdns report
sudo dsmr-email-admin authdns report -o /root/sender-auth-report.txt

# provision one domain: install a 2048-bit RSA key (signs immediately via
# qmail-dkim) and write its SPF + DKIM (+ optional DMARC) records
sudo dsmr-email-admin authdns provision example.com
sudo cat /var/lib/qmail/control/authdns/example.com   # paste these at your DNS host

# after publishing, confirm DNS matches the installed key
sudo dsmr-email-admin authdns verify example.com

# provision every domain that has no SPF/DKIM yet
# (skips domains that already publish either — those send via another path)
sudo dsmr-email-admin authdns provision-all
```

The record file gives each TXT in both registrar-form (host + value) and
zone-file form. SPF defaults to `~all`; use `spf suggest --hard` for `-all`
once you are sure a domain sends only through this host.

The companion tool [`dsmr-smtpauth(8)`](https://github.com/fade/dsmr-checkpassword-dovecot)
manages the SMTP-AUTH relay allow-list; `dsmr-email-admin` audits the stack
those credentials run on.
