# L5 Narrow / L2 General Classification — PAX_ROUTER
**Platform:** Anticloud | **PAX:** 27B | **IP:** USPTO pending 2026, Anticloud FZ LLE
**Domain:** Request router: domain classification and module dispatch for PAX

## L5 Narrow
PAX_ROUTER operates at L5 Narrow within its specialized scope: request router: domain classification and module dispatch for pax.
It does not generalize outside this function. PAX 27B inference is scoped to this module's
specific input/output contract. All outputs are deterministically validated before AIOSS append.

## L2 General
PAX_ROUTER is available to all 9 Anticloud deployment tiers. Any tier project that needs
request router: domain classification and module dispatch for pax capability calls PAX_ROUTER without reconfiguration. Same API across all domains.

## PAX Integration
PAX 27B interfaces with PAX_ROUTER as a specialized inference module. Inputs are preprocessed
to PAX_ROUTER's schema, PAX generates outputs within that schema, and results are AIOSS-chained
before being returned to the calling module.

## AIOSS Audit Relevance
Every routing decision (input hash + domain label + confidence + target module) is appended to the AIOSS chain.
H_n = SHA3-256(H_{n-1} || entry_hash_n || timestamp_n)
Full reproducible audit trail, verifiable offline without cloud.

## Regulatory
NIST SP 800-53 SC-8 (transmission integrity)
