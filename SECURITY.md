# Security Policy

## About This Repository
This is a documentation/knowledge repository containing IoT protocol reference material for AI agent use. It does not contain executable code, credentials, or sensitive infrastructure details.

## Reporting Security Concerns

### Inaccurate Security Information
If you identify security-critical inaccuracies in the protocol documentation (e.g., incorrect DTLS CID extension type, incorrect eSIM object IDs, outdated cipher suite recommendations):

1. Open a GitHub Issue with label `security-accuracy`
2. Describe the inaccuracy and correct information with source citation
3. We will review and correct within 5 business days

### Known Critical Notes (Already Documented)
- DTLS CID: Extension type **54** (RFC 9146) is correct. Type 53 = obsolete draft. See `references/security-provisioning.md`
- SGP.32 Object IDs: Objects 504 (RSP) and 3443 (eSIM IoT) are correct. Object 146 is NOT an eSIM object.
- EU CRA/RED compliance: See `references/security-provisioning.md` for current regulatory requirements

## Supported Versions
We maintain accuracy for the current main branch only.

## Contact
Security concerns: Open a GitHub Issue with label `security-accuracy`
General questions: Open a GitHub Discussion
