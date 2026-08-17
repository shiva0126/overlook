# Collector Specs — Identity and IAM

**Series:** [Collector documentation](00-anatomy.md)

The collectors that feed E6 Entity Resolution, E7 Permission Closure and E8 Escalation Matcher. Without these, the three differentiating engines have no input.

`aws.iam.roles` is specified as the worked example in [01 §2](01-specification-format.md#2-the-template) and is not repeated here.

---

> **⚠ ALIGNED TO THE ENGINEERING HANDOFF.**
> `Overlook_Edge_Collector_Engineering_Handoff_v1.1` is the implementation
> boundary and takes precedence over this document. Content here that
> extends the handoff is a **PROPOSED EXTENSION** requiring review under
> handoff §25.3 / §35.1. Open escalations: `01-system-design.md` §41.
> Hard ceiling: **12 vCPU / 64 GB / 1 TB per collector — scale out, not up.**

---

## 1 · `aws.organizations`

The collector everyone skips, and the one whose absence makes the closure **confidently wrong** rather than incomplete.

```yaml
collector:
  id: aws.organizations
  connector: aws
  version: 1.0.0
  load_bearing: true

  purpose: >
    Provides the account tree and, critically, the Service Control
    Policies. Without SCPs the closure evaluates identity policies
    alone and reports permissions the org has explicitly denied —
    an over-permissive graph in which almost every principal
    reaches almost everything.

  source:
    api_surface: configuration
    calls:
      - ListRoots
      - ListOrganizationalUnitsForParent
      - ListAccountsForParent
      - ListPoliciesForTarget
      - DescribePolicy
      - ListDelegatedAdministrators
    object_type: Organization structure and policies
    scope_unit: organization
    scope_dimensions: [organization]

  auth:
    requires_scope:
      - organizations:ListRoots
      - organizations:ListOrganizationalUnitsForParent
      - organizations:ListAccountsForParent
      - organizations:ListPoliciesForTarget
      - organizations:DescribePolicy
    optional_scope: [organizations:ListDelegatedAdministrators]
    degrades_gracefully: false
    failure_remediation: >
      Must run from the management account or a delegated
      administrator. From a member account these calls return
      AccessDeniedException and NO SCP data is available —
      closure must then be marked reduced-confidence for every
      account in the org.

  fetch:
    pagination: marker
    page_size: 20
    max_pages: 5000
    stream_downstream: true
    estimated_calls: "1 per OU + 1 per account + 2 per policy attachment"
    rate_limit_domain: [aws, organization, organizations, list]

  cursor: { supported: false, field: null,
            advance_when: after_durable_write }

  coverage:
    emits_window: true
    scope: [organization]
    completeness_signal: "NextToken absent on every traversal branch"

  schedule: { interval: 24h, full_enumeration_interval: 24h,
              jitter_pct: 10, blackout_aware: true }

  produces:
    entities:
      - type: ACCOUNT
        canonical_key_source: [account]
        properties: [name, email, status, joined_at, arn]
      - type: ORG_UNIT
        canonical_key_source: [ou_id]
        properties: [name, parent]
    relationships:
      - predicate: CONTAINS
        from: ORG_UNIT
        to: ACCOUNT | ORG_UNIT
        confidence: 1.0
        significant_attributes: [containment_type]
    properties:
      - subject: ACCOUNT | ORG_UNIT
        key: scp_constraints
        value: "resolved SCP statements applying at this scope"

  mapping:
    Account.Id:      canonical_key via account:aws:<id>
    Account.Name:    properties.name
    Account.Status:  properties.status
    OrganizationalUnit.Id:   canonical_key via ou_id
    Policy.Content:  → constraint set, attached to every scope
                       BELOW the attachment point, inherited
    Policy.Type == SERVICE_CONTROL_POLICY: → SCP constraint
    Policy.Type == RESOURCE_CONTROL_POLICY: → RCP constraint

  health:
    success_criteria: "accounts_enumerated >= 1 AND scps_resolved >= 1"
    baseline_metric: accounts_enumerated
    silent_threshold_pct: 0        # an org never legitimately has 0
    staleness_threshold: 48h
    field_presence_monitored: [Policy.Content]

  depends_on:
    hard: []
    soft: []
    consumed_by: [E7]

  failure_modes:
    - condition: running from a member account
      behaviour: >
        Circuit break. Mark the ENTIRE org's closure
        reduced-confidence with reason "SCPs uncollected."
        Do NOT proceed as if no SCPs exist.
    - condition: SCP attached to a root the caller cannot read
      behaviour: >
        No coverage window. Partial constraint data is worse than
        none, because it produces confident wrong answers.
    - condition: an account is SUSPENDED
      behaviour: >
        Emit the ACCOUNT with status, do not treat as removed.

  fixtures:
    required:
      - 3-level OU tree, 42 accounts
      - an SCP attached at root, an SCP attached at OU
      - an account with a deny-list SCP and an allow-list SCP
      - AccessDeniedException from a member account
      - a suspended account
```

**The lesson in this spec:** `silent_threshold_pct: 0` and the member-account failure mode. Most collectors degrade; this one must **fail loudly**, because a missing constraint set silently inflates every permission in the org.

---

## 2 · `aws.iam.policies`

The most complex mapping in the catalog. Everything E7 evaluates comes from here.

```yaml
collector:
  id: aws.iam.policies
  connector: aws
  version: 1.0.0
  load_bearing: true

  purpose: >
    Provides the grant statements themselves — managed policies,
    inline policies, and every version of them. This is the raw
    material of permission closure; without it there are principals
    and resources and no relationship between them.

  source:
    api_surface: configuration
    calls:
      - ListPolicies
      - GetPolicy
      - GetPolicyVersion
      - ListPolicyVersions
      - ListAttachedRolePolicies
      - ListAttachedUserPolicies
      - ListAttachedGroupPolicies
      - ListRolePolicies
      - GetRolePolicy
      - ListUserPolicies
      - GetUserPolicy
      - ListGroupPolicies
      - GetGroupPolicy
    object_type: IAM policy documents
    scope_unit: account
    scope_dimensions: [account]

  auth:
    requires_scope:
      - iam:ListPolicies
      - iam:GetPolicy
      - iam:GetPolicyVersion
      - iam:ListAttached*Policies
      - iam:List*Policies
      - iam:Get*Policy
    optional_scope: []
    degrades_gracefully: false

  fetch:
    pagination: marker
    page_size: 100
    max_pages: 20000
    stream_downstream: true
    estimated_calls: "~2 per managed policy + 1 per inline policy"
    rate_limit_domain: [aws, account, iam, get]

  cursor: { supported: false, field: null,
            advance_when: after_durable_write }

  coverage:
    emits_window: true
    scope: [account]
    completeness_signal: >
      IsTruncated false on ListPolicies AND every attachment list
      completed for every principal enumerated by iam.users,
      iam.roles and iam.groups

  schedule: { interval: 4h, full_enumeration_interval: 24h,
              jitter_pct: 10, blackout_aware: true }

  produces:
    entities: []          # policies are not nodes — they are grants
    relationships: []     # E7 produces the edges; this produces input
    properties:
      - subject: IDENTITY | ROLE | GROUP
        key: grant_statements
        value: "normalised policy statements with their source"

  mapping:
    PolicyVersion.Document:
      Statement[]:
        Effect:      → allow | deny
        Action / NotAction:
                     → action set, NotAction expanded and FLAGGED
        Resource / NotResource:
                     → resource pattern, PRESERVED not expanded
        Condition:   → condition block, satisfiability classified
                       downstream by E7
        Principal:   → present only in resource policies
        Sid:         → provenance
    attachment:
      managed → granted_via: managed_policy:<arn>
      inline  → granted_via: inline_policy:<principal>:<name>
    default_version_only: true
      ⚠ collect ALL versions but evaluate only the DEFAULT.
        Non-default versions matter for the
        iam:SetDefaultPolicyVersion escalation primitive.

  health:
    success_criteria: "policies_enumerated >= 1 AND parse_failures == 0"
    baseline_metric: statements_parsed
    silent_threshold_pct: 5
    staleness_threshold: 18h
    field_presence_monitored: [Condition, NotAction, NotResource]

  depends_on:
    hard: [aws.iam.users, aws.iam.roles, aws.iam.groups]
    soft: [aws.organizations, aws.iam.boundaries]
    consumed_by: [E7, E8]

  failure_modes:
    - condition: a policy document fails to parse
      behaviour: >
        Quarantine that document with a sample. Continue.
        Mark the affected principal reduced-confidence.
        NO coverage window.
    - condition: NotAction or NotResource present
      behaviour: >
        Expand explicitly and FLAG. These invert set semantics and
        are a common source of policy-analysis bugs — never treat
        as syntactic sugar.
    - condition: a policy is attached to a principal not yet collected
      behaviour: >
        Emit the grant keyed on the principal ARN. E6 resolves it
        when the principal arrives. Do not drop.

  fixtures:
    required:
      - a managed policy with 3 versions, default not latest
      - an inline policy with a Condition block
      - a policy using NotAction
      - a policy using NotResource
      - a policy with an explicit Deny
      - a wildcard resource policy
      - a malformed policy document
      - a policy attached to a deleted principal
```

**The lesson:** `default_version_only: true` with the caveat. Evaluating a non-default version would be wrong; *not collecting* the other versions would make `iam:SetDefaultPolicyVersion` undetectable. Collect all, evaluate one.

---

## 3 · `aws.iam.boundaries`

Small, easily forgotten, and its absence produces the same failure as missing SCPs.

```yaml
collector:
  id: aws.iam.boundaries
  connector: aws
  version: 1.0.0
  load_bearing: true

  purpose: >
    Provides permission boundaries per principal. A boundary CAPS
    what identity policies can grant; omitting it inflates every
    bounded principal's effective permissions to their unbounded
    maximum.

  source:
    api_surface: configuration
    calls: [GetRole, GetUser, GetPolicyVersion]
    object_type: PermissionsBoundary attachment
    scope_unit: account
    scope_dimensions: [account]

  auth:
    requires_scope: [iam:GetRole, iam:GetUser, iam:GetPolicyVersion]
    degrades_gracefully: false

  fetch:
    pagination: none          # rides on role/user enumeration
    max_pages: 1
    stream_downstream: true
    estimated_calls: "0 additional — boundary arrives in GetRole/GetUser"
    rate_limit_domain: [aws, account, iam, get]

  coverage:
    emits_window: true
    scope: [account]
    completeness_signal: >
      every principal from iam.users and iam.roles was inspected

  schedule: { interval: 4h, full_enumeration_interval: 24h,
              jitter_pct: 10, blackout_aware: true }

  produces:
    properties:
      - subject: IDENTITY | ROLE
        key: permission_boundary
        value: "boundary policy ARN and its resolved statement set"

  mapping:
    Role.PermissionsBoundary.PermissionsBoundaryArn:
      → properties.permission_boundary.arn
    → fetch that policy's default version → resolved statements
    absent → properties.permission_boundary = null
             ⚠ NULL IS SIGNIFICANT. "No boundary" is a finding when
               the principal holds iam:PassRole.

  health:
    success_criteria: "principals_inspected == principals_enumerated"
    baseline_metric: principals_inspected
    silent_threshold_pct: 0
    staleness_threshold: 18h

  depends_on:
    hard: [aws.iam.users, aws.iam.roles]
    soft: []
    consumed_by: [E7, E8]

  failure_modes:
    - condition: boundary policy ARN references a deleted policy
      behaviour: >
        Treat as NO boundary and raise a finding. A dangling
        boundary reference caps nothing.

  fixtures:
    required:
      - a role with a boundary
      - a role without a boundary
      - a role with a boundary pointing at a deleted policy
      - a boundary that permits less than the identity policy
      - a boundary that permits more (and therefore caps nothing)
```

**The lesson:** *null is significant.* Most collectors treat an absent field as nothing to record. Here, the absence of a boundary on a principal holding `iam:PassRole` is precisely the precondition E8 checks.

---

## 4 · `aws.iam.providers`

Small, and the only source of cross-boundary trust edges into AWS.

```yaml
collector:
  id: aws.iam.providers
  connector: aws
  version: 1.0.0
  load_bearing: true

  purpose: >
    Provides SAML and OIDC identity providers. Every federated
    CAN_ASSUME edge — from Entra, Okta, GitHub Actions, GitLab,
    Terraform Cloud — depends on the provider existing here and
    being matched to a trust policy condition.

  source:
    api_surface: configuration
    calls:
      - ListSAMLProviders
      - GetSAMLProvider
      - ListOpenIDConnectProviders
      - GetOpenIDConnectProvider
    scope_unit: account
    scope_dimensions: [account]

  auth:
    requires_scope: [iam:ListSAMLProviders, iam:GetSAMLProvider,
                     iam:ListOpenIDConnectProviders,
                     iam:GetOpenIDConnectProvider]
    degrades_gracefully: false

  fetch:
    pagination: none
    max_pages: 1
    estimated_calls: "2 + 1 per provider"
    rate_limit_domain: [aws, account, iam, list]

  coverage:
    emits_window: true
    scope: [account]
    completeness_signal: "both list calls returned"

  schedule: { interval: 12h, full_enumeration_interval: 24h,
              jitter_pct: 10, blackout_aware: true }

  produces:
    entities:
      - type: IDENTITY
        subtype: application_sp
        canonical_key_source: [arn]
        properties: [provider_type, url, thumbprints, client_ids,
                     created_at]
    relationships:
      - predicate: TRUSTS
        from: ACCOUNT
        to: IDENTITY
        confidence: 1.0
        significant_attributes: [mechanism, conditions, direction]

  mapping:
    OpenIDConnectProviderArn: canonical_key
    Url:                      properties.url
                              → matched against trust policy
                                Condition keys to resolve which
                                external system this is
    ClientIDList:             properties.client_ids
    ThumbprintList:           properties.thumbprints
    SAMLMetadataDocument:     → entityID extracted → properties.url

  health:
    success_criteria: "auth_errors == 0"
    baseline_metric: providers_enumerated
    silent_threshold_pct: 100     # zero providers is legitimate
    staleness_threshold: 36h

  depends_on:
    hard: []
    soft: [aws.iam.roles]
    consumed_by: [E7, E8]

  failure_modes:
    - condition: an OIDC provider exists with no role trusting it
      behaviour: >
        Emit the provider. Raise a low-severity finding —
        an unused federation endpoint is attack surface.
    - condition: provider URL matches no known external system
      behaviour: >
        Emit with provider_type unknown. Do not guess. E8's
        subject-condition primitives require a known issuer.

  fixtures:
    required:
      - a GitHub Actions OIDC provider
      - an Entra SAML provider
      - an OIDC provider with no trusting role
      - an account with no providers
```

**The lesson:** `silent_threshold_pct: 100`. Zero results here is completely normal. A collector's silence threshold must reflect what is legitimate for *that* collector, not a global default.

---

## 5 · `entra.users`

The canonical key authority for most enterprises.

```yaml
collector:
  id: entra.users
  connector: entra
  version: 1.0.0
  load_bearing: true

  purpose: >
    Provides IDENTITY nodes with priority-1 canonical keys for the
    workforce. Nearly every other connector's identity records
    resolve against these; collect anything else first and those
    identities resolve on weak keys and may never merge.

  source:
    api_surface: configuration
    calls:
      - GET /users/delta
      - GET /users/{id}/authentication/methods
    object_type: Entra user
    scope_unit: tenant
    scope_dimensions: [tenant]

  auth:
    requires_scope: [User.Read.All, Directory.Read.All]
    optional_scope: [UserAuthenticationMethod.Read.All]
    degrades_gracefully: true
    failure_remediation: >
      Grant User.Read.All application permission and admin-consent it.
      Without UserAuthenticationMethod.Read.All, MFA state is absent
      and the "privileged identity without MFA" finding is disabled.

  fetch:
    pagination: link
    page_size: 999
    max_pages: 20000
    stream_downstream: true
    estimated_calls: "1 per 999 users + 1 per user for auth methods"
    rate_limit_domain: [entra, tenant, graph, read]

  cursor:
    supported: true
    field: deltaLink
    advance_when: after_durable_write

  coverage:
    emits_window: true
    scope: [tenant]
    completeness_signal: >
      deltaLink returned in place of nextLink — Graph's explicit
      end-of-collection signal. A delta run does NOT emit a window;
      only a full enumeration does.

  schedule: { interval: 1h, full_enumeration_interval: 24h,
              jitter_pct: 10, blackout_aware: true }

  produces:
    entities:
      - type: IDENTITY
        subtype: human_user | guest_user
        canonical_key_source: [email, idp]
        properties: [display_name, upn, department, job_title,
                     manager, account_enabled, created_at,
                     last_sign_in, mfa_registered, user_type,
                     on_premises_sync_enabled, on_premises_sid]

  mapping:
    mail:              canonical_key via email:  ← PRIORITY 1
                       ⚠ LOWERCASED. AD returns Priya.S@Meridian.com;
                         Entra returns priya.s@meridian.com. Without
                         normalisation these are two people.
    id:                canonical_key via idp:entra:<id>  ← PRIORITY 2
    userPrincipalName: alias via upn:
    onPremisesSecurityIdentifier: alias via adsid:
                       ⚠ THE AD JOIN. This is how a synced user
                         merges with its AD counterpart.
    onPremisesSamAccountName: alias via sam:
    userType == "Guest": → subtype guest_user
    accountEnabled:    properties.account_enabled
    signInActivity.lastSignInDateTime: properties.last_sign_in
                       → dormancy findings
    authentication methods: → properties.mfa_registered

  health:
    success_criteria: "users_enumerated >= 1 AND auth_errors == 0"
    baseline_metric: users_enumerated
    silent_threshold_pct: 2
    staleness_threshold: 6h
    field_presence_monitored: [mail, onPremisesSecurityIdentifier]

  depends_on:
    hard: []
    soft: []
    consumed_by: [E6, E7, E9]

  failure_modes:
    - condition: user has no mail attribute
      behaviour: >
        Fall back to idp:entra:<id> (priority 2, confidence 1.0).
        Still deterministic. Not a degradation.
    - condition: onPremisesSecurityIdentifier absent on a synced user
      behaviour: >
        The AD join is unavailable for that identity. Flag it —
        this is how cloud and on-prem views of one person silently
        fail to merge.
    - condition: deltaLink expired
      behaviour: >
        Fall back to full enumeration. Emit a coverage window.
        Do not attempt to resume from a stale token.

  fixtures:
    required:
      - 3-page enumeration with a deltaLink terminator
      - a user with mail and a user without
      - a synced user with onPremisesSecurityIdentifier
      - a cloud-only user
      - a guest user
      - a disabled user
      - an expired deltaLink
      - a user with no registered auth methods
```

**The lesson:** the two `field_presence_monitored` entries. If `mail` population drops, canonical keys silently degrade to priority 2 and cross-source merges start failing. If `onPremisesSecurityIdentifier` drops, the AD↔Entra join breaks. Neither raises an error; only field-presence monitoring catches them.

---

## 6 · `entra.app_role_assignments`

Small collector, largest escalation surface in Entra.

```yaml
collector:
  id: entra.app_role_assignments
  connector: entra
  version: 1.0.0
  load_bearing: true

  purpose: >
    Provides the Graph API application permissions granted to service
    principals. An app holding RoleManagement.ReadWrite.Directory is
    Global Administrator by another name, and nobody reviews app
    permissions with the rigour they review admin groups.

  source:
    api_surface: configuration
    calls:
      - GET /servicePrincipals/{id}/appRoleAssignments
      - GET /servicePrincipals(appId='00000003-0000-0000-c000-000000000000')
        (Microsoft Graph SP, to resolve appRole GUIDs to names)
    scope_unit: tenant
    scope_dimensions: [tenant]

  auth:
    requires_scope: [Application.Read.All, Directory.Read.All]
    degrades_gracefully: false

  fetch:
    pagination: link
    page_size: 999
    max_pages: 10000
    estimated_calls: "1 per service principal + 1 for the Graph SP"
    rate_limit_domain: [entra, tenant, graph, read]

  coverage:
    emits_window: true
    scope: [tenant]
    completeness_signal: >
      every service principal from entra.service_principals inspected

  schedule: { interval: 4h, full_enumeration_interval: 24h,
              jitter_pct: 10, blackout_aware: true }

  produces:
    relationships:
      - predicate: CAN_MODIFY
        from: IDENTITY (application_sp)
        to: IDENTITY | ROLE | GROUP
        confidence: 0.99
        significant_attributes: [mechanism, privilege_level]
        precondition: >
          the appRole GUID must resolve to a known Graph permission.
          An unresolved GUID produces no edge.

  mapping:
    appRoleId:      → resolved against the Graph SP's appRoles
                      → permission name, e.g.
                        RoleManagement.ReadWrite.Directory
    principalId:    → subject service principal
    resourceId:     → the API being granted against
    permission → capability:
      RoleManagement.ReadWrite.Directory → CAN_MODIFY directory_role
                                           privilege_level ADMIN
      AppRoleAssignment.ReadWrite.All    → CAN_MODIFY self (grant any)
                                           privilege_level ADMIN
      Application.ReadWrite.All          → CAN_MODIFY all SPs
                                           → CAN_ASSUME via credential add
      Directory.ReadWrite.All            → CAN_MODIFY directory
      Group.ReadWrite.All                → CAN_MODIFY groups
      Policy.ReadWrite.ConditionalAccess → CAN_MODIFY security_control

  health:
    success_criteria: "sps_inspected == sps_enumerated"
    baseline_metric: assignments_enumerated
    silent_threshold_pct: 10
    staleness_threshold: 18h
    field_presence_monitored: [appRoleId]

  depends_on:
    hard: [entra.service_principals]
    soft: [entra.app_registrations]
    consumed_by: [E7, E8]

  failure_modes:
    - condition: appRoleId does not resolve to a known permission
      behaviour: >
        Emit as a property with the raw GUID, produce NO edge, and
        surface for content update. Guessing a permission's meaning
        is how false escalation paths are created.
    - condition: a first-party Microsoft SP holds broad permissions
      behaviour: >
        Emit normally but flag as first-party. These are expected
        and would otherwise dominate the findings list.

  fixtures:
    required:
      - an SP with RoleManagement.ReadWrite.Directory
      - an SP with Application.ReadWrite.All
      - an SP with only read permissions
      - an unresolvable appRoleId
      - a first-party Microsoft SP
      - a tenant with no app role assignments
```

**The lesson:** the unresolvable-GUID failure mode. The safe behaviour is to emit **no edge** and ask for a content update. A collector that guesses at an unknown permission's meaning manufactures escalation paths that do not exist.

---

## 7 · What these six have in common

```
  1  FOUR OF THEM PRODUCE NO EDGES AT ALL
     organizations, policies, boundaries → they produce CONSTRAINTS
     and GRANTS. E7 produces the edges. A collector that emitted
     CAN_READ directly from a policy statement would be evaluating
     precedence in the wrong place, with none of the context.

  2  ABSENCE IS DATA
     no boundary · no SCP · no MFA method · no
     onPremisesSecurityIdentifier — each is a finding or a
     precondition, not a field to skip.

  3  SILENCE THRESHOLDS ARE PER-COLLECTOR
     0% for organizations (an org always has accounts)
     100% for iam.providers (zero federation is normal)
     2% for entra.users (a workforce does not shrink 2% overnight)

  4  THE FAILURE MODE THAT REPEATS
     "partial data is worse than no data." organizations without
     SCPs, policies with a parse failure, boundaries not inspected
     — every one refuses the coverage window rather than shipping
     a confident wrong answer.
```

---

*Next: [Deployment set specs](03-spec-deployment-set.md)*
