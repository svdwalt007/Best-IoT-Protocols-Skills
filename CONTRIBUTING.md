# Contributing to Best IoT Protocols Skills

## How to Contribute

This repository welcomes contributions from IoT protocol experts, standards engineers, and implementation practitioners.

### Protocol Accuracy Standards
- All protocol descriptions must cite primary sources (IETF RFC, IEEE standard, 3GPP TS, OMA TS, GSMA SGP, BBF TR, IEC/ISO standard, ETSI TS)
- Version numbers must be explicit
- IP licensing restrictions must be flagged
- Draft/evolving standards must be flagged as such

### Adding New Protocols
1. Identify the correct section in `SKILL.md` Knowledge Domains
2. Add protocol to the appropriate `references/` file
3. Update the Cross-Reference Index in `SKILL.md`
4. Add trigger keywords to the YAML `description` in `SKILL.md`
5. Update `README.md` feature matrix if adding a new domain

### RFC/Standard Citation Format
- IETF: `RFC XXXX (short description)` — link to https://www.rfc-editor.org/rfc/rfcXXXX
- IEEE: `IEEE 802.15.4-2020 §X.Y` — link to IEEE Xplore
- 3GPP: `3GPP TS XX.XXX Release X §X.X` — link to 3GPP portal
- OMA: `OMA-TS-LightweightM2M_Core-VX_X §X.X` — link to OMA SpecWorks
- GSMA: `GSMA SGP.XX vX.X §X.X` — link to GSMA TS page
- BBF: `BBF TR-XXX Issue X §X.X` — link to broadband-forum.org

### Pull Request Process
1. Fork the repository
2. Create a feature branch (`git checkout -b feat/add-protocol-X`)
3. Ensure all changes cite primary sources
4. Update SKILL.md trigger keywords if adding new protocols
5. Submit PR with description of protocols added/modified
6. PRs reviewed by repository maintainers for technical accuracy

### What Not to Contribute
- Vendor-specific implementation details not backed by standards
- Outdated protocol information without clear deprecation notices
- Content duplicating the sibling LwM2M or 3GPP skills (cross-reference instead)

## Code of Conduct
See [CODE_OF_CONDUCT.md](CODE_OF_CONDUCT.md)

## License
Contributions are accepted under the Apache 2.0 License.
