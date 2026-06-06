# Requirements & Use Case Templates

> Part of the **signalXautomotive** Gateway Infrastructure.

This folder contains reusable templates for specifying an **embedded application that receives and
sends data over Ethernet from an Infineon AURIX TC4 Data Routing Engine (DRE)** — i.e., a software
endpoint that consumes and produces **IEEE 1722 ACF_CAN_BRIEF** frames (CAN-over-Ethernet) on the
automotive Ethernet backbone.

For background on the protocol and the DRE hardware path, see:
- [`../CAN_to_Ethernet_Protocol_Overview.md`](../CAN_to_Ethernet_Protocol_Overview.md)
- [`../IEEE_1722_ACF_CAN_BRIEF_Overview.md`](../IEEE_1722_ACF_CAN_BRIEF_Overview.md)

## Templates

| Template | Purpose |
|---|---|
| [`Requirements_Template.md`](./Requirements_Template.md) | Capture functional, platform (OS, programming languages, coding style guides, toolchain), interface, performance, safety, and security requirements. |
| [`Use_Case_Template.md`](./Use_Case_Template.md) | Describe a single interaction (actors, flows, pre/postconditions, acceptance criteria) and trace it back to requirements. |

## Intended workflow (manual + AI)

These templates are deliberately structured so they can be **filled in both manually and by an AI
assistant**, and then **consumed by AI** to derive further requirements and implementation:

1. **Copy** a template:
   - Requirements → `docs/requirements/<COMPONENT_NAME>_Requirements.md`
   - Use cases → `docs/use-cases/UC-<NNN>_<short-name>.md`
2. **Fill** the placeholders. Anything in `<…>` or marked `TBD` must be resolved. Sections marked
   _(AI-fillable)_ may be drafted/expanded by an AI assistant and then reviewed by a human.
3. **Review & approve** — set each requirement/use case `Status` (`Draft` → `Reviewed` → `Approved`).
4. **Derive** — use the approved documents as structured input for AI to generate detailed
   requirements, design, and implementation.

## Conventions

- **Stable IDs**: requirement IDs (`REQ-…`) and use case IDs (`UC-…`) must not change once published;
  they are the basis for traceability. Deprecate instead of deleting.
- **Testable statements**: use `shall` (mandatory), `should` (recommended), `may` (optional).
- **Traceability**: every use case links to the requirements it satisfies, and every requirement
  should map to at least one use case and one verification activity.
