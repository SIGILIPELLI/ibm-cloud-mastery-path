# 06 · Security Deep Dive (Hyper Protect, Key Management)

Level 3's Key Protect module covered bring-your-own-key (BYOK) encryption
— IBM still has technical access to unwrapped key material during
operations. This module covers **Hyper Protect Crypto Services (HPCS)**
for keep-your-own-key (KYOK), where IBM has no access at all, plus
confidential computing for protecting data in use.

## BYOK vs. KYOK: the actual difference

| | Key Protect (BYOK) | Hyper Protect Crypto Services (KYOK) |
|---|---|---|
| Hardware | Shared, IBM-managed HSM | Dedicated, single-tenant HSM |
| IBM operational access | Possible under specific conditions | Cryptographically prevented |
| Master key custodians | IBM | You, via smart cards you control |
| Cost | Per-key, low | Dedicated HSM instance, high fixed cost |
| Compliance fit | Most regulated workloads | Highest-sensitivity data (e.g. certain financial/health regulations) |

KYOK matters specifically when a compliance requirement states that *no
one at the cloud provider*, under any circumstance, can access key
material — a bar Key Protect's shared-HSM model cannot clear regardless
of policy, because the hardware itself is shared.

## Provision Hyper Protect Crypto Services

```bash
ibmcloud resource service-instance-create hpcs-mastery \
  hs-crypto standard us-south \
  --parameters '{"units": 2}'
```

```text
Service instance hpcs-mastery is being created.
This provisions dedicated HSM hardware and can take up to several hours.
```

Unlike every other service instance in this curriculum, HPCS provisioning
is not fast — it's dedicated hardware being allocated, not a shared
control-plane record being written.

## Initialize with smart cards (the KYOK ceremony)

HPCS master key initialization requires a physical **crypto officer
ceremony** using smart cards shipped to designated custodians — this is
not something the CLI alone can complete:

```bash
ibmcloud hpcs crypto-unit-info --instance hpcs-mastery
```

```text
Crypto Unit: hpcs-mastery-cu-1
Status: uninitialized
Signature threshold: 2 of 3 (configured by custodians)
```

A 2-of-3 threshold means initializing or changing the master key needs
two of three designated custodians present with their smart cards — no
single IBM employee or your own single administrator can do it alone.
Document who holds each card and where, as part of the actual DR plan;
losing quorum of smart cards means losing the ability to manage the
master key, by design.

## Once initialized, use it like Key Protect

```bash
ibmcloud hpcs key create root-key-kyok \
  --instance hpcs-mastery \
  --key-ring-name orders-app-kyok \
  --standard-key false

ibmcloud cos bucket-create --bucket orders-archive-kyok \
  --ibm-service-instance-id <cos-guid> \
  --region us-south \
  --kms-key-crn $(ibmcloud hpcs key root-key-kyok --instance hpcs-mastery --output json | jq -r .crn)
```

The CRN-referencing pattern from Level 3's Key Protect module is
identical — HPCS is a drop-in replacement for "the service that owns root
keys," not a different integration pattern.

## Confidential computing: protecting data in use

Encryption at rest (Key Protect/HPCS) and in transit (TLS everywhere)
still leaves data unencrypted in memory while a workload processes it.
**IBM Cloud confidential computing** (secure execution environments on
select bare metal / VSI profiles) protects that gap:

```bash
ibmcloud is instance-create secure-orders-proc \
  --vpc platform-vpc \
  --zone us-south-1 \
  --profile bx3d-2x10 \
  --confidential-compute-mode sgx \
  --image ibm-redhat-9-2-minimal-amd64-2
```

```text
Instance secure-orders-proc is being created.
Confidential compute mode: sgx
```

SGX enclaves let a process attest that it's running unmodified code
inside a protected memory region — even an operator with root on the
host cannot inspect the enclave's memory. Realistic use: processing
payment card data or health records where "encrypted everywhere except
the brief moment it's being computed on" is not an acceptable gap.

## Certificate management across all of this

```bash
ibmcloud resource service-instance-create certmgr-mastery \
  cloudcerts free us-south

ibmcloud certificate-manager certificate-order \
  --instance-id <certmgr-guid> \
  --name app-example-com \
  --domains "app.example.com" \
  --domain-validation-method dns-01
```

```text
Certificate order submitted. Awaiting DNS validation.
```

Automate renewal rather than tracking expiry manually — an expired
certificate causing an outage is one of the most common, most avoidable
production incidents, and Certificate Manager supports auto-renewal
hooks that redeploy to services referencing the certificate.

## Terraform for HPCS

```hcl
resource "ibm_resource_instance" "hpcs" {
  name              = "hpcs-mastery"
  service           = "hs-crypto"
  plan              = "standard"
  location          = "us-south"
  resource_group_id = data.ibm_resource_group.mastery_path.id
  parameters = {
    units = 2
  }
}
```

```bash
terraform validate
# Success! The configuration is valid.
```

## Gotchas

- **HPCS master key loss is unrecoverable by design** — losing quorum of
  custodian smart cards (below the signature threshold) permanently loses
  access to every key wrapped by that master key; the DR plan for HPCS
  is fundamentally a physical smart-card custody plan, not a technical
  backup.
- **Confidential compute profiles are a specific subset of instance
  profiles** — not every flavor supports `--confidential-compute-mode`;
  check `ibmcloud is instance-profiles --output json | jq '.[] |
  select(.confidential_compute_modes)'` before assuming a chosen profile
  supports it.
- **KYOK's cost is fixed regardless of usage** — a dedicated HSM instance
  costs the same whether it wraps one key or a thousand; it's the wrong
  tool for a workload that doesn't have a genuine "no cloud-provider
  access, ever" requirement.
- **Certificate auto-renewal still needs the redeploy step wired** — a
  renewed certificate that never gets pushed to the load balancer or
  Route referencing it still expires functionally, even though
  Certificate Manager itself shows it as valid.

## How It Actually Works

- **The BYOK-vs-KYOK difference is a hardware-tenancy fact, not a policy
  setting — Key Protect's HSMs are shared across many customers' key
  material inside one appliance, partitioned by software access controls,
  while HPCS provisions a physically dedicated HSM partition that only
  your organization's custodians can operate.** IBM's operational access
  to a Key Protect instance is a permissions boundary enforced in
  software running on shared hardware; on HPCS, the master key never
  exists in a form any IBM process can reconstruct, because the crypto
  officer ceremony described below establishes cryptographic material
  split across custodian smart cards that IBM's own systems never hold a
  complete copy of.
- **The 2-of-3 signature threshold works via a secret-sharing scheme,
  not a simple "two people must click approve" workflow** — the master
  key material (or the authority to regenerate it) is mathematically
  split so that any two of the three shares can reconstruct it but any
  one alone reveals nothing. That's the literal, unforgeable mechanism
  behind "no single administrator can do it alone," and it's also why
  losing quorum of cards is unrecoverable: the math that lets two shares
  reconstruct the secret provides no path with fewer than two.
- **An SGX enclave protects data in use by encrypting a region of memory
  with a processor-internal key that never leaves the CPU package,
  decrypting data only inside the CPU's execution pipeline and re-
  encrypting it before it's ever written back out to RAM.** Attestation
  lets a remote party cryptographically verify the exact code measurement
  running inside that protected region before trusting it with secrets —
  which is the actual capability TLS and at-rest key management don't
  provide: both protect data before it's decrypted for processing and
  after it's re-encrypted for storage, leaving the CPU's plaintext working
  set as the one place they can't reach, and that's exactly the gap an
  enclave closes.
- **Certificate Manager's DNS-01 validation proves domain ownership by
  requiring you to publish a specific TXT record the certificate
  authority queries for before issuing** — it's the same domain-validation
  protocol (ACME-based) any Let's-Encrypt-style CA uses, not an
  IBM-specific check. Auto-renewal automates re-running that same
  challenge-response exchange before expiry, but the certificate's actual
  presentation to clients still depends on whichever load balancer or
  Route references its CRN re-reading the updated certificate — renewal
  updates the stored object, not every consumer of it, automatically.

## Cheat sheet

| Task | Command |
|---|---|
| Create HPCS instance | `ibmcloud resource service-instance-create <n> hs-crypto standard <region> --parameters '{"units": <n>}'` |
| Check crypto unit status | `ibmcloud hpcs crypto-unit-info --instance <n>` |
| Create a KYOK root key | `ibmcloud hpcs key create <n> --instance <inst> --standard-key false` |
| Create a confidential-compute VSI | `ibmcloud is instance-create <n> --confidential-compute-mode sgx --profile <profile>` |
| Order a TLS certificate | `ibmcloud certificate-manager certificate-order --instance-id <id> --domains <fqdn>` |

## Exercise

1. Explain, in your own words, the difference between BYOK and KYOK, and
   describe one workload from an earlier module where KYOK's cost would
   be justified and one where it would not.
2. Write out the HPCS Terraform resource and validate it.
3. Identify a confidential-compute-capable instance profile and describe,
   in prose, what an SGX enclave protects that TLS and at-rest encryption
   do not.
4. Design a certificate renewal automation flow (prose or a small script)
   that redeploys a renewed certificate to a load balancer, not just
   renews it in Certificate Manager.
