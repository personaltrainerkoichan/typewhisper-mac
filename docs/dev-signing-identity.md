# Local dev signing identity

`scripts/build-dev-local.sh` signs the dev build with a **self-signed Code Signing
certificate** named `TypeWhisper Dev Self-Signed`, stored in the login keychain.

## Why

macOS keys TCC grants (microphone, accessibility) to the app's *designated
requirement*. The two options produce very different requirements:

| Signing        | Designated requirement                                          | Survives a rebuild? |
| -------------- | --------------------------------------------------------------- | ------------------- |
| ad-hoc (`-`)   | `cdhash H"..."`                                                  | No — the hash is of the binary, so every build is a new app to macOS |
| certificate    | `identifier "com.typewhisper.mac.dev" and certificate leaf H"..."` | Yes — independent of the binary |

With ad-hoc signing the microphone permission has to be granted again after every
single build. With the certificate it is granted once.

## Recreating the certificate

Needed on a new machine, or if the certificate is deleted from the keychain.
The build script falls back to ad-hoc signing (with a warning) when it is missing.

```bash
work="$(mktemp -d)" && chmod 700 "$work"

cat > "$work/openssl.cnf" <<'EOF'
[ req ]
distinguished_name = dn
x509_extensions    = v3_codesign
prompt             = no

[ dn ]
CN = TypeWhisper Dev Self-Signed

[ v3_codesign ]
basicConstraints     = critical,CA:false
keyUsage             = critical,digitalSignature
extendedKeyUsage     = critical,codeSigning
subjectKeyIdentifier = hash
EOF

openssl req -x509 -newkey rsa:2048 -sha256 -days 3650 -nodes \
  -config "$work/openssl.cnf" \
  -keyout "$work/key.pem" -out "$work/cert.pem"
chmod 600 "$work"/*.pem

p12pass="$(openssl rand -hex 16)"
openssl pkcs12 -export -out "$work/cert.p12" \
  -inkey "$work/key.pem" -in "$work/cert.pem" \
  -name "TypeWhisper Dev Self-Signed" -passout "pass:$p12pass"
chmod 600 "$work/cert.p12"

security import "$work/cert.p12" -k "$HOME/Library/Keychains/login.keychain-db" \
  -P "$p12pass" -T /usr/bin/codesign -T /usr/bin/security

rm -rf "$work"
```

Verify:

```bash
security find-identity -p codesigning | grep "TypeWhisper Dev Self-Signed"
```

The certificate shows as `CSSMERR_TP_NOT_TRUSTED` and does **not** appear under
`find-identity -v` (valid identities only). That is expected and fine — `codesign`
can use an untrusted self-signed identity, and only the designated requirement
matters for TCC.

## After recreating

The certificate's SHA-1 is part of the designated requirement, so a **new**
certificate is a new identity: microphone and accessibility permission must be
granted once more. Subsequent rebuilds keep the grant.

Alternatively, export the existing certificate (Keychain Access > right-click >
Export, as `.p12`) and import it on the other machine to keep the same identity.

## Overriding

```bash
export TYPEWHISPER_DEV_SIGN_IDENTITY="Some Other Identity"   # or "-" for ad-hoc
./scripts/build-dev-local.sh
```
