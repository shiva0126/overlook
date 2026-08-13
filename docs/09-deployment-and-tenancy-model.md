# Overlook — Deployment and Tenancy Model

**Version:** 0.1
**Date:** 2026-08-13
**Companion to:** `01-system-design.md`, `03-connectors.md`, `05-controller-ui.md`, `08-connector-benchmark-and-alignment.md`
**Status:** Records a decision taken 2026-08-13. Resolves the open question asked in `06 C3`, `07 Q3` and `08 Q2`.

---

## 1. The model, stated precisely

```
   Overlook operates as an MSSP.

   Each CUSTOMER receives their own DEDICATED, SINGLE-TENANT
   deployment, on their own premises or in their own cloud.

   Deployment is SHAPED BY THE CUSTOMER'S INFRASTRUCTURE.
   We do not ship every component to every customer.

   The CATALOG is complete and visible — like Stellar Cyber's —
   but only what the customer actually runs is deployed and enabled.
```

Three commitments follow, and each one simplifies the architecture:

1. **One appliance serves one customer.** No shared appliance, ever.
2. **The customer's environment determines what gets installed.** A cloud-only customer never receives on-prem collectors. An organisation without Kubernetes never sees the Kubernetes connector deployed.
3. **The catalog is the map, the deployment is the territory.** Every connector is browsable; a small subset is live.

---

## 2. Multi-tenancy is not what we need

The distinction that was buried in the earlier documents, and the reason the question kept coming back unresolved.

```
   MULTI-TENANCY  (Stellar Cyber's MSSP model)
     ONE deployment serves N customers.
     Requires: data isolation, per-tenant keys, per-tenant retention,
     per-tenant ML, tenant field on every connector, tenant-scoped
     RBAC, noisy-neighbour protection, and a permanent risk that one
     missing WHERE clause is a cross-customer breach.

   MULTI-INSTANCE  (Overlook's model)
     N deployments, each serving ONE customer.
     Requires: fleet management across deployments, version discipline,
     content distribution, and remote operability without data access.
     Isolation is PHYSICAL, not logical.
```

**We are multi-instance.** That has consequences worth stating plainly:

```
   WHAT WE DO NOT NEED TO BUILD
     ✕ tenant_scoped field on connector manifests   (retract 08 §6.2)
     ✕ per-tenant data isolation inside an appliance
     ✕ tenant-aware query paths and RBAC in the pipeline
     ✕ noisy-neighbour resource governance between customers
     ✕ the cross-customer breach risk class, entirely

   WHAT WE MUST BUILD INSTEAD
     + a fleet management plane across customer deployments
     + version and content distribution across N appliances
     + remote diagnostics that work WITHOUT access to customer data
     + per-customer onboarding that scales to the next customer
```

The trade is favourable. Logical isolation is a permanent correctness burden carried in every query; physical isolation is a one-time operational burden carried at deployment. For a small team, the second is far cheaper and far safer.

### 2.1 So where does `tenant_id` go? Nowhere.

Asked directly on 2026-08-13: *if we deploy one appliance per customer, why carry a tenant identifier at all?* The answer is that we should not, and the schemas have been corrected.

```
   REMOVED
     tenant_id from the Security Fact schema        (01 §5.2)
     tenant_id from the fact merge key              (01 §5.5, 04 §20)
     tenant_id from the TrustGraph node schema      (01 §9.1)
     tenant_scoped from the connector manifest      (08 §6.2)

   WHY IT WAS NEVER NEEDED
     Inside the appliance, every entity, edge and fact belongs to one
     customer by construction. A discriminator that always holds the
     same value is not a discriminator — it is redundant bytes on every
     record and, worse, an invitation to build tenant-aware query paths
     we have deliberately designed out.

   WHAT REPLACES IT AT THE BOUNDARY
     The mTLS client certificate on the sync channel. The authenticated
     connection already establishes which deployment is talking. A fact
     does not need to assert what the channel has already proven.
     deployment_id therefore appears in the BATCH HEADER and the
     enrollment certificate — once per batch, not once per fact.

   AND ON THE SaaS SIDE
     Each customer's graph is a SEPARATE STORE. Physical isolation
     continues into the console: there is no shared table needing a
     discriminator column, and therefore no missing-WHERE-clause class
     of bug. The console routes to the right store based on which
     customer the analyst has selected.
```

### 2.2 But the per-customer KEY stays — for a different reason

One thing that looks like tenancy is not, and must survive: the tokenization key.

```
   token = HMAC-SHA256( deployment_key, canonical_key )

   The key is per-customer, but NOT because of multi-tenancy.
   It exists because:

     1. THE CUSTOMER HOLDS IT.  It is generated at enrollment, wrapped
        by the customer's KMS/HSM, and never transmitted to us. That is
        the entire mechanism behind "residency is not blindness"
        (01 §2.3). Without a customer-held key there is no claim.

     2. ONE CUSTOMER MAY HAVE SEVERAL EDGE NODES.  The hybrid archetype
        (§4, Archetype 3) deploys one on-prem and one in cloud. Both
        must produce the SAME token for the same entity or the graph
        fragments and the hybrid attack path — the reason the customer
        bought the product — becomes invisible.

   So it was renamed: tenant_key -> deployment_key.
   Same mechanism, honest name. It is a customer-held secret, not a
   tenancy discriminator.
```

### 2.3 It also strengthens the one claim that survived

`01 §2.3` Claim 1 is *"residency is not blindness."* Single-tenant physical deployment makes that claim stronger, not weaker:

```
   "Your data is processed on hardware you own, in your building,
    by an appliance no other customer touches, and what reaches our
    console is tokenized with a key we never receive."

   Versus a multi-tenant competitor:
   "Your data is logically separated from other customers'
    data on our shared infrastructure."
```

For the regulated buyers this product targets, that is not a marginal difference.

---

## 3. The MSSP console — and an elegant property

We still need one console across all customers. An MSSP analyst works across accounts; a per-customer login that must be switched manually does not scale past a handful.

The privacy architecture handles this better than expected:

```
   OVERLOOK MSSP CONSOLE
     shows all customers' graphs — as TOKENS
     cross-customer health, coverage, findings counts, trends
     analyst switches customer context freely
          │
          │ analyst opens a finding for Customer A
          ▼
   Browser calls CUSTOMER A's Edge Node directly to de-tokenize
     requires: network reach to A's Edge
             + a SaaS-issued JWT scoped to that analyst
             + Customer A's LOCAL RBAC permitting it
          │
          ▼
   Real names render for Customer A only.
   Customer B's tokens on the same screen stay opaque.
```

**Consequences:**

- An MSSP analyst can only see plaintext for customers they are authorised for **at that customer's own Edge**. The customer controls this, not the MSSP. That is a genuine assurance we can offer a customer about their own service provider.
- The MSSP console can compute cross-customer analytics — "this misconfiguration class appears in 7 of 12 customers" — on tokens alone, because the analysis is structural.
- A compromise of the MSSP console does not expose any customer's names, hosts or data. It exposes token graphs.

This is worth designing deliberately rather than discovering later. It is also a sales asset when a customer asks the obvious question: *"what can your analysts see?"*

### 3.1 Network reach becomes the practical constraint

The awkward part. De-tokenization requires the analyst's browser to reach the customer's Edge Node.

```
   OPTIONS, in order of preference

   1. MSSP-to-customer site-to-site VPN or private link
      Standard for managed services. Most customers already have one.

   2. Customer-hosted reverse proxy / DMZ Edge with mTLS
      Customer publishes a restricted resolve endpoint to the MSSP.

   3. Customer-operated relay
      Customer controls the path entirely.

   ✕ NEVER: relay through Overlook SaaS.
      That would put plaintext on our servers and destroy the premise.

   4. DEGRADED MODE (01 §7.4)
      Analyst triages on token shapes and non-identifying attributes.
      Genuinely usable for prioritisation. Useless for action.
```

Option 1 is the realistic default for an MSSP engagement, and it should be part of the standard onboarding runbook rather than an exception.

---

## 4. Deployment shaped by customer infrastructure

"Not shipping everything at once" becomes a set of archetypes. The onboarding survey determines which one applies.

```
  ARCHETYPE 1 — CLOUD-ONLY
    customer runs AWS/Azure/GCP, IdP is Entra or Okta, no data centre
    DEPLOY   Edge Node in their VPC/VNet, private subnet, no public IP
             workload identity, no stored credentials
    ENABLE   cloud + IdP + code connectors
    OMIT     AD, AD CS, network device connectors, satellite collectors
    SIZE     profile S or M, all-in-one

  ARCHETYPE 2 — ON-PREM HEAVY
    customer runs AD, VMware, on-prem databases, physical network
    DEPLOY   Edge Node as a VM in the data centre
    ENABLE   AD, AD CS, VMware, database, firewall, syslog, flow
    OMIT     cloud connectors beyond whatever little cloud exists
    SIZE     profile M or L, scanner role split if DSPM is in scope
    NOTE     AD collection profile must be handed to their SOC to
             allowlist — LDAP sweeps look like reconnaissance

  ARCHETYPE 3 — HYBRID (the common case)
    both, plus federation between them
    DEPLOY   Edge Node on-prem + Edge Node in cloud
             resolution primary elected on-prem (identity authority)
    ENABLE   everything relevant, including federation trust collection
    SIZE     profile L
    NOTE     this is where the hybrid attack path lives — the reason
             the customer bought the product

  ARCHETYPE 4 — SEGMENTED / REGULATED
    OT, PCI, air-gapped or otherwise isolated zones
    DEPLOY   Edge Node in the main zone
             lightweight satellite collectors in each segment
             (collection only — no analytics, no vault, no state)
    ENABLE   per-zone, with the customer's explicit approval per source
    NOTE     air-gapped zones need offline content bundles (01 §17)

  ARCHETYPE 5 — SMALL
    under 500 hosts, one cloud, one IdP
    DEPLOY   single all-in-one VM
    ENABLE   6-10 connectors
    SIZE     profile S
```

### 4.1 The deployment survey

Onboarding starts with a short questionnaire that resolves the archetype, then discovery confirms it:

```
   1. Which clouds, and how many accounts/subscriptions/projects?
   2. Is there an on-premises Active Directory? Forest count?
   3. Which IdP is authoritative for workforce identity?
   4. Kubernetes? Managed or self-hosted?
   5. Source control and CI/CD?
   6. Which segmented zones exist (OT, PCI, air-gapped)?
   7. Where may the Edge Node run, and what egress is permitted?
   8. Who operates it — customer staff, or us?
   9. Is response ever in scope, or read-only permanently?
  10. What must never be collected? (data sensitivity boundaries)

   → produces the archetype, the profile, and the initial connector set
```

Question 10 deserves emphasis. Asking a customer up front what we must *not* touch is a trust-building act, and it feeds directly into the Privacy Policy allow-list (`05 §19`).

---

## 5. Catalog like Stellar Cyber, deployment unlike it

This was already the right design in `05-controller-ui.md`; the MSSP model confirms it.

```
   CATALOG          all 118 connectors, browsable, searchable,
                    filterable by what they produce
                    each shows: what it adds to the graph, what it
                    reads, what it needs, estimated cost
                    ✦ DETECTED IN YOUR ENVIRONMENT — discovery-first

   DEPLOYED         only what the customer runs and has approved
                    typically 6-15 connectors per customer

   The gap between them is not a deficiency to hide. It is the
   coverage view (05 §14) and the growth mechanism:
     "Network coverage 62% — connecting your 3 Palo Alto firewalls
      would reveal an estimated 14 additional attack paths."
```

**The difference from Stellar Cyber:** their catalog entries are built per customer request and shipped on the release train (`08 §2.3`). Ours must be manifest-driven and SDK-built so a new source is a configuration exercise, not an engineering ticket. As an MSSP with N customers, each with one unusual source, the per-request model would consume the entire engineering capacity.

---

## 6. What the MSSP model adds: the fleet plane

The one genuinely new requirement. It sits *above* the per-customer Controller.

```
  OVERLOOK FLEET  ·  MSSP operations across all customer deployments

  ┌─────────────────────────────────────────────────────────────┐
  │ CUSTOMERS                                    12 deployments │
  │                                                             │
  │ ● Acme Bank        L · hybrid   · 41 conn · healthy   4m    │
  │ ● Northwind Corp   M · cloud    · 12 conn · healthy   2m    │
  │ ◐ Contoso Health   L · on-prem  · 28 conn · DEGRADED  11m   │
  │     AD connector credential expired 2h                      │
  │     → 1,204 entities going stale                            │
  │ ⏸ Fabrikam         S · small    ·  8 conn · paused          │
  │     customer change freeze until 2026-08-20                 │
  │ ...                                                         │
  ├─────────────────────────────────────────────────────────────┤
  │ FLEET HEALTH                                                │
  │   appliance versions   2.1.0 (9) · 2.0.4 (3)  [ stage ]     │
  │   content versions     current (11) · 1 behind (1)          │
  │   certs expiring <30d  2   [ renew ]                        │
  │   offline >1h          0                                    │
  ├─────────────────────────────────────────────────────────────┤
  │ CROSS-CUSTOMER INSIGHT        (computed on tokens only)     │
  │   IAM-001 escalation-to-admin present in 9 of 12            │
  │   median time-to-remediate  14 days                         │
  │   most common missing connector: Kubernetes (8 customers)   │
  └─────────────────────────────────────────────────────────────┘
```

### 6.1 What the fleet plane must do

```
  DEPLOY     provision a new customer appliance from an archetype
             template; enrollment token; connector set preselected
  MONITOR    health, coverage, freshness, queue depth across all
  UPDATE     stage appliance and content versions across the fleet,
             canary → subset → all, with per-customer pinning
  DIAGNOSE   remote troubleshooting WITHOUT access to customer data
             (redacted bundles, metrics, journal replay initiated
             remotely but executed and inspected locally)
  BENCHMARK  cross-customer structural comparison, tokens only
  BILL       per-customer entitlement, connector counts, asset counts
```

### 6.2 The hard constraint the fleet plane must respect

```
   The fleet plane may hold: health, versions, coverage percentages,
   finding COUNTS by type, token graphs, structural comparisons.

   The fleet plane may NEVER hold: customer names for entities,
   hostnames, credentials, raw evidence, prompt content, or any
   customer's tokenization key.

   Remote diagnostics is the sharp edge. An engineer debugging a
   parser must be able to work WITHOUT receiving customer data.
   This is why journal replay (04 §8.2) is executed locally and
   its output inspected locally — the fleet plane triggers it and
   receives only redacted diagnostics.
```

---

## 7. Operational economics

The multi-instance model moves cost from engineering to operations, and that must be planned rather than discovered.

```
   PER CUSTOMER, ONGOING
     1 appliance to keep patched, upgraded, and healthy
     N connector credentials to rotate
     1 content update stream to keep current
     1 set of coverage gaps to chase
     1 VPN/link to maintain for de-tokenization

   AT 10 CUSTOMERS   manageable manually
   AT 30 CUSTOMERS   the fleet plane is mandatory, not optional
   AT 100 CUSTOMERS  automation of deployment and upgrade is the
                     entire operational model

   DESIGN CONSEQUENCES
     - the appliance must self-heal aggressively; a customer visit
       to fix a stuck connector is unaffordable
     - upgrades must be staged, reversible, and never require
       customer downtime
     - health must be honest and pushed, not polled by a human
     - onboarding must be repeatable to the point of being boring
```

For a solo or small build, this argues strongly for the feasible options in `07 §5`: **fewer connectors, deeper, on fewer customers** — because every additional connector on every additional appliance is recurring operational surface.

---

## 8. What changes in the other documents

```
  01 §14.2  Multi-tenancy section — REWRITE.
            Currently specifies per-tenant schemas and compute
            isolation for a shared SaaS. Replace with the
            multi-instance model and the MSSP console described
            in §3 above.

  03        Connector framework unchanged. The catalog stays at 118
            as the map. Add the archetype-driven initial connector
            sets from §4.

  05        Controller unchanged and confirmed — catalog vs
            connections split, discovery-first onboarding, coverage
            as growth mechanism all validated by this model.
            ADD: the fleet plane as a layer above it.

  08 §6.2   RETRACT the `tenant_scoped` manifest field. Not needed.
            KEEP api_surface, functions, and execution placement —
            execution placement is now MORE relevant, since
            Archetype 3 deploys two Edge Nodes per customer.

  06, 07    The MSSP question these documents raised is now answered.
```

---

## 9. New open questions

```
  Q1  Who operates the appliance day to day — our staff, or the
      customer's? Changes the Controller's persona (05 §1) and the
      RBAC split entirely. Likely varies per customer, which means
      both must be supported.

  Q2  Does each customer get their own Overlook SaaS tenant, or do
      all customers live in one MSSP-operated SaaS with token
      separation? §3 assumes the latter. Confirm.

  Q3  Do we host the SaaS side ourselves, or does it run in our
      cloud per region? Data residency for the TOKEN graph still
      matters to some regulators even though it holds no plaintext.

  Q4  Is the fleet plane a separate product surface, or a view
      inside the same SaaS console with an MSSP role?

  Q5  At what customer count does the fleet plane become required?
      Recommend building the minimum version at customer 3, not
      customer 30 — retrofitting fleet operations onto manual
      processes is painful.

  Q6  Cross-customer benchmarking: opt-in per customer, or a term
      of the service agreement? Recommend opt-in, published
      methodology — it is a trust asset if handled openly and a
      liability if assumed.
```

---

*End of document.*
