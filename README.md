# Unmaintained Project

I am not using, even less maintaining this project anymore.

# yubikey-ca

## Overview

`yubikey-ca` is a simple command line tool used to build a PKI and SSH
CA, powered by a Yubikey (or other PKCS#11 tokens) for private key
management and git for history.

Every operation that change the state of the CA (such as emitting or
revoking certificates) creates a git commit ; you can for example `git
revert` a revoke operation.

## Dependencies

* Python >= 3.2
* OpenSSL
* [OpenSSL PKCS#11 engine from OpenSC](https://github.com/OpenSC/libp11/)
* `p11tool` from GnuTLS
* A PKCS#11 provider; for a Yubikey, you can use OpenSC
* SSH (for the SSH CA, not required for the SSL PKI)

## PIN

The default PIN used to access the Yubikey is 123456. You can change
it with the `PIN` environment variable ; the special value `ask` will
issue a prompt if the PIN is needed.

## Initialization

`yubikey-ca init-ca` will initialize a new CA in the current directory
(you probably want an empty one). It will create a new git repository,
files needed by the CA, and do a first commit.

`yubikey-ca init-key` will initialize the private key on the Yubikey. You
have three possibilities :

* On-device key generation: `yubikey-ca init-key -g`. This
is the recommended method, unless your device is vulnerable to
[this security advisory](https://www.yubico.com/support/security-advisories/ysa-2017-01/).
* Software key generation: `openssl genrsa 2048 | yubikey-ca init-key`. This
is the recommened method if your device is affected by the previous issue.
* Using an existing key, in PEM format: `yubikey-ca init-key -f key.pem`.

You should also give a name to your CA, the default being "Yubikey
CA". You can do so with the `-s` option :

* `yubikey-ca init-key -g -s "/O=Example Inc/CN=Certificate Authority"`

Now that your CA key is on the device, you can access to your CA
certificate:

* `yubikey-ca ca-cert`

or to your SSH CA certificate:

* `yubikey-ca ssh-ca`

## Using the PKI

### Issuing and revoking certificates

Generating a certificate:

* `openssl genrsa 2048 | yubikey-ca client-cert`

You must give a name to the certificate, for example:

* `yubikey-ca client-cert -s "/O=Example Inc/CN=John Doe"`

You can use other algorithms for your client certificates, for example
`openssl genrsa 4096` for a different key size or `openssl ecparam
-genkey -name secp256k1` for an ECC key.

The `-e` option can be used to export the certificate + key in the PKCS#12
format for importing it in the browser.

The certificates are stored in the `certs/` directory, and are listed
in the `index.txt` file. The first field describe the certificate status
(`V` for valid), the second the issuing date, the third the certificate
serial and the last one the certificate name.

To revoke certificates, you must provide their serial:

* `yubikey-ca client-revoke 01 03`

Generate a CRL containing the revoked certificates:

* `yubikey-ca crl`

## Using the SSH CA

### Signing SSH certificates

* `yubikey-ca ssh-cert id_rsa.pub`

It will generate `id_rsa-cert.pub`, that can be used by the owner of the
corresponding private key, either as-is, or with the `CertificateFile`
option.

Issued certificates are listed in `ssh-index.txt`. It contains the
following fields:

* First field: validity, V for valid and R for revoked
* `serial`: identifier for revokation
* `pub`: public key file, base64 encoded
* `principals`: list of principals for the certificate. Defaults to empty,
can be configured with `-n`.
* `id`: id of the certificate, used in logs. Defaults to the comment
field of the provided SSH public key, can be configured with `-I`.

Revoking certificates is done by providing serials:

* `yubikey-ca ssh-revoke 16388656242419284907 14560815252314548972`

You can the generate a KRL containing the revoked certificates :

* `yubikey-ca krl`

### Principals

SSH certificates grants access to *principals*. The server possess a list
of principals that can be used to grant access, the client certificate
possess a list of valid principals.  A client is allowed to authenticate
if the intersection of the server-side principals and the client-side
principals (from your certificates) is not empty.

For example, if the server-side principals for `website@staging-host`
are `env:staging` and `type:wordpress`, you need either `env:staging`
or `type:wordpress` in the principals list in your certificate for
successful authentication.

By default, the server-side principals list only contains the target user
(for `website@staging-host`, the list would contain a single principal,
`website`).

### Server-side configuration

In `/etc/ssh/sshd_config`:

* `TrustedUserCAKeys /etc/ssh/authorized_ca` where `authorized_ca`
contains the result of `yubikey-ca ssh-ca`. You can have multiple entries.
* `RevokedKeys /etc/ssh/revoked_keys` where `revoked_keys` contains the
result of `yubikey-ca krl`. **Warning**: Be sure to have an empty KRL
here initially, because if the file does not exists all keys will be
rejected after restarting `sshd`

Suggested configuration:

* `AuthorizedPrincipalsCommandUser nobody` and
* `AuthorizedPrincipalsCommand /bin/sh -c "printf 'u:%u\nu:%u@host\nh:host\n*\n'"`

(replace host by the name of your server)

This add some useful default principals:

* `*`, so a certificate containing the `*` principal will be allowed to
login anywhere.
* `u:%u`, to allow creation of "an user on all hosts" certificates.
* `u:%u@host` to allow creation of "specific user on specific host" certificates.
* `h:host`, to allow creation of "all users on a host" certificates.

You can add "thematic" server-wide principals here, like `env:staging`
or `OU:qa`, or the roles or your server from your Ansible/Puppet/Chef
setup.

* `AuthorizedPrincipalsFile .ssh/authorized_principals`

This allows you to configure a per-user list of principals in its home
directory, similarly to `.ssh/authorized_keys` (one principal per line,
not comma-separated). It does not override the server-wide list of
principals but adds to it.

## Known bugs

* The `ssh-cert` subcommand always ask for your PIN, regardless of the
PIN environment variable.

## Limitations

To keep things simple, a lot of things that are configurable in a
full-fledged OpenSSL-powered CA (like `easy-rsa`) are hardcoded. In
particular (this is not an exhaustive list):

* `init-key` will use RSA 2048
* The CA certificate is valid for 10 years
* The CA CRLs are valid for 10 years
* The SSL certificates are valid for 1 year
* The SSL certificates are created from a private key, not a CSR
* The SSH certificates are valid forever
* CA policy is extremely loose, just requiring a `CN` in the subject
* Certificates purpose is hardcoded (SSL client and S/MIME)

If you really need different values, patches are welcome, but as for
now I have no plan to implement options for alternative values for every
single thing "just in case someone needs them".

## Using another PKCS#11 token

Althought yubikey-ca is primarily intended to be used with a Yubikey,
it can be used with others PKCS#11 tokens:

* If you must use another provider than the default (`opensc-pkcs11.so`),
either use the `PKCS11_MODULE` environment variable or the `pkcs11-module`
command-line option.
* Specify the slot URL with `PKCS11_URL` environment
variable or `pkcs11-url` option. You can get the list with `p11tool
--provider=$PKCS11_MODULE --list-all` ; you can use only parts of the URL
(for example, only `manufacturer` and `id`), you must remove the `type=`
part of the URL, and you must keep the `id=` part of the URl.
* `init-key` won't work, you will have to manually initialize the token.

### Example with SoftHSM

Initialization (you may want to change the token name, `SoftHSM`, the
CA subject, `Test CA`, and the PINs):

```
export SOFTHSM2_CONF=/path/to/softhsm2.conf
TOKEN="SoftHSM"
CA="/CN=Test CA"
PIN="123456"
SO_PIN="$PIN"

softhsm2-util --init-token --slot 0 --label "$TOKEN" --pin "$PIN" --so-pin "$SO_PIN"
KEY=$(openssl genrsa 2048)
pkcs11-tool --module /usr/lib/softhsm/libsofthsm2.so --id 02 --type privkey --label "SIGN privkey" --pin "$PIN" --write <(openssl rsa -in <(echo "$KEY") -outform der)
pkcs11-tool --module /usr/lib/softhsm/libsofthsm2.so --id 02 --type pubkey --label "SIGN pubkey" --pin "$PIN" --write <(openssl rsa -in <(echo "$KEY") -outform der -pubout)
pkcs11-tool --module /usr/lib/softhsm/libsofthsm2.so --id 02 --type cert --label "Certificate for Digital Signature" --pin "$PIN" --write <(openssl req -x509 -key <(echo "$KEY") -sha256 -days 3560 -subj "$CA" -outform der)
```

Example usage:

```
export SOFTHSM2_CONF=/path/to/softhsm2.conf

yubikey-ca -m /usr/lib/softhsm/libsofthsm2.so -u "pkcs11:token=SoftHSM;id=%02" ca-cert
PKCS11_MODULE="/usr/lib/softhsm/libsofthsm2.so" PKCS11_URL="pkcs11:token=SoftHSM;id=%02" yubikey-ca client-cert
```
