# J1 protocol vectors — test material only

This set belongs to the [J1 contract](../../j1_production_contracts.md). It is
not an implemented Backend/Runtime integration or a distributable product.

`manifest.json` pins the exact SHA-256 of `protocol-vectors.json` and names
portable positive, negative, replay and boundary cases. `input` is a dot path
inside the vector bundle. `setup` describes consumer test state/field overrides;
it is not an API request or permission to alter signed claims without resigning.
Package-location examples use `http.packageDirectoryHeaders.linux` by default;
Windows cases explicitly select `.windows` and the matching native host. These
headers accompany schema/prepare/start requests; they are not JSON body fields.
A setup `packageDirectoryHeader` replaces that raw header value; null means
missing and an array means duplicate field lines. `existingDirectoryHeader`
sets the earlier normalized root for a replay-conflict case. Consumer fixtures
create the listed package beneath a suitable test root and substitute its encoded
absolute directory without changing ZIP bytes or signed claims. A literal dino
home directory is never required for automated tests.

The 2026-11-25 pre-consumer contract revision updates only unsigned path/HTTP
examples and their digest to the approved directory-header/server-filename policy.
Existing ZIP, encryption, keys and signed vectors remain byte-identical; this
is not a new package format or signing version.

The expected classifications are contract assertions; later consuming phases
must execute them against their implementations.

The bundle contains a valid A8 reference IR, canonical schema/manifest, encrypted
three-entry ZIP (Base64), session RSA key pair, Ed25519 signing/verification pair,
lease/renewal/envelope and complete public JSON examples. All private keys and
the content key are **intentionally public test-only material**. Never install
them as deployment secrets/trust, ship them in binaries, or use their fixed nonce
for another encryption. Copying this fixture is not a key-rotation strategy.

Use `fixedClock`, not the current date, for the initial positive case. Signed
times are Unix seconds; HTTP times are UTC millisecond strings. Renewal uses
its listed later clock and an already accepted initial grant. A tampered lease
changes a signature byte without changing its signed header/payload.

J1 generation uses the existing Backend `canonicalize` and `yazl` dependencies
and Node standard crypto. It verifies both signatures, rejects the changed
signature, opens the OAEP envelope and authenticates/decrypts GCM. It follows the
package byte contract but **does not invoke the Backend package service**.
Later Backend tasks must add actual-service vectors; Runtime J2/J6–J8 and J15
must prove consumption. J1 does not claim Go/TypeScript parity or manual success.

Consumers vendor the pinned test bytes locally, check the digest, and do not
depend on a sibling Plan checkout. After acceptance this set is immutable;
change vectors/expectations by adding a new set version. Existing A8 v1–v3
fixtures and wire format versions are unchanged.

The implemented schema-inspection HTTP contract now accepts only `packageRef` and
`launcherVersion`; IDs are derived, and the actual ZIP hash is returned rather
than supplied. Some unsigned HTTP selection examples in this immutable bundle
retain the older redundant ID/hash shape. They are not current schema-request
examples. Use the [J4 specification](../../j4_package_input_schema_api.md) and
Launcher guide for the current transport. ZIP, manifest, encryption and signed
vectors remain unchanged; their expected cryptographic digests still apply.

## Current lease-claim compatibility

The 2026-11-26 authorization design uses `userId` in the closed signed claim set.
This accepted v1 bundle signs `customerId`. Its bytes/digest remain unchanged,
but those signed leases are **not positive acceptance inputs for current J6**.
No compatibility alias is introduced. The Backend issuer delivery before J6 must
produce a newly signed v2 set with current claims/requests, kid/public key and
fixed-clock positive, negative and replay cases. Include rejection of the old
customerId field. Changing decoded JSON without re-signing invalidates a lease.
The [v2 set](../v2/README.md) is now delivered through the Backend signing provider
with synthetic identities and passing Backend digest/signature checks. Runtime
J6 consumption and full Backend package/customer exchange remain pending.
Archive/AES/OAEP vectors remain useful for their unchanged byte contracts.

### Historical package location examples

The immutable v1 JSON contains the earlier three-segment Product/Release/filename
references and default directory spelling. They are historical transport examples,
not current accepted requests. Current J1/J2/J4 use the OS Documents
`a2matex/products` root and `<productId>/<local filename>.a2m`, normally
`<productId>/package.a2m`; Release identity comes from the validated manifest.
Consumer tests install the unchanged ZIP bytes at that new local location. The
ZIP/manifest/encryption fixture bytes and their digest are not changed by this
filesystem-layout update.
