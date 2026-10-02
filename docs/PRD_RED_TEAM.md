# Fyminal PRD — Red-team validation

Date: 2 October 2026

Reviewed artifact: [PRD.md](PRD.md), initial version 1.0, revised version 1.1.

## Method and scope

An independent reviewer received the PRD and the conversation's product intent, inspected the document, and challenged scope, security assumptions, lifecycle behavior, session identity, failure recovery, and testability. The author then revised the PRD and requested a second pass against the concrete findings.

This is a document review. The repository was empty before these documents; there was no executable application, on-device test, dependency audit, or penetration test. Acceptance targets in the PRD remain proposed targets, not observed results.

## Initial verdict

Revise before adopting the implementation baseline. The reviewer found **one high-severity and three medium-severity blocking document issues**, plus four low-severity recommendations. The core scope remained appropriate: one phone SSH client, one Mac, existing Tailscale, shared tmux work, no AI-specific backend.

## Findings and dispositions

| ID | Severity | Failure case | Resolution in PRD v1.1 | Evidence to obtain during implementation |
|---|---|---|---|---|
| RT-01 | High; document blocker | A new tmux session with the old name is mistaken for surviving work; phone may present an unexpected program as resumed. | FR-08/09 and section 6 require a remembered session incarnation that changes across recreation/server restart. Changed/missing/unknown identity requires acknowledgement before attach/create. No command replay. Session identity is not a claim that every process remains alive. | AT-09: delete/recreate the same name, including after server restart; automatic resume must stop. AT-06/07: surviving process identity and no replay. |
| RT-02 | Medium; document blocker | A 15-second retry delay cannot consistently meet a 10-second recovery target; unlock time and retry-window expiry were ambiguous. | Section 6 caps delays at 5 seconds and network/handshake attempts at 10 seconds, within a 60-second wall-clock window. NFR-02 measures 20-second recovery after independently observed reachability restoration during the first 30 seconds, active/unlocked. Manual/foreground resume uses the healthy 5-second target after user interaction/unlock. | AT-07/14: restore mid-attempt and mid-delay, record timings and window expiry. A timeout after the window requires Resume, not a false automatic-recovery promise. |
| RT-03 | Medium; document blocker | Disabling Tailscale does not prove a CGNAT address is unreachable; another network may route it elsewhere. | FR-02 explicitly limits the claim to destination restriction, without route attestation. AT-02 permits safe failure or verified-host outcome and adds a different-server/same-IP host-key mismatch test before user authentication. | Demonstrate no endpoint fallback, no false VPN claim, and strict host identity verification. Literal transport attestation would require a separately agreed extension. |
| RT-07 | Medium; document blocker | tmux custom settings destroy work when the last client detaches; leaving WezTerm attached can conceal this defect. | FR-08 and section 6 establish zero-client survival prerequisites, incompatible-option handling, and custom-hook limitations. No silent global configuration change. | AT-06: detach the phone as last client; test incompatible destroy-unattached configuration. |
| RT-04 | Low | Shared phone/desktop sizing can hide a prompt or disrupt handoff. | G0 chooses a sizing policy; G2/AT-05 require usable phone prompts with keyboard and WezTerm still attached. Global settings are not silently changed. | Test keyboard, orientation, editor, and approval UI on the actual devices. |
| RT-05 | Low | Public-key setup or host fingerprint comparison could be unusable despite a sound abstract flow. | G1 explicitly requires a manual setup guide with SSH file permissions and the fingerprint corresponding to the negotiated host-key algorithm. | AT-01 executed from a fresh installation using only the delivered guide. |
| RT-06 | Low | A canceled local unlock or target change might reveal retained terminal output from the previous connection. | FR-01 and section 7 require clearing output on target/user change/reset and keeping retained output obscured until successful local authentication. | AT-12: cancel unlock, reset, and change connection using synthetic secret markers. |
| RT-08 | Low | Treating every inactive-to-active transition as background/resume can cause biometric prompt or reconnect loops. | FR-12 and section 6 separate local Locked/Unlocking/Unlocked from transport states; transient system UI only obscures the view, while true background/lock triggers teardown. | AT-08: complete/cancel authentication and dismiss system overlays; no recursion, duplicate connection, or exposed terminal. |

## Additional author verification

- Checked primary Apple, Tailscale, tmux, and WezTerm references for relevant platform constraints.
- Added the seven-day lifetime of Apple's free Personal Team provisioning as a G0 usability/distribution decision; no paid membership is assumed approved.
- Kept the product a general terminal: no AI API, automatic approvals, remote desktop, hosted service, session-manager UI, or account system was introduced to resolve findings.
- Identified iPhone, key-only authentication, numeric IPv4, one named session, and personal distribution as proposed defaults rather than falsely labeling them user-confirmed details.

## Remaining release gates

Even after document issues are resolved, implementation must prove:

1. An installable signed build on the owner's actual phone, with an acceptable provisioning/reinstallation method.
2. Compatible SSH and terminal libraries, strict host checking, protected key storage, Unicode/mobile input, resize behavior, and the owner's AI CLI.
3. Correct lifecycle cancellation, no automatic replay or duplicate connection, robust session identity, and zero-client tmux persistence.
4. Passing security negative tests, terminal resource limits, secret-handling checks, and measured connection/recovery targets.

No runtime tests have been executed by this document task. “Resolved” means the requirement or test specification was corrected; it does not mean the eventual code has passed that test.

## Final revalidation

**Pass at document level. No blocking document defects remain from this review.** The independent reviewer re-read version 1.1 and confirmed all eight findings were addressed. The proposed scope remains aligned with the personal iPhone-to-Mac SSH workflow.

The reviewer specifically retained session-identity check/attach races, actual iOS lifecycle behavior, credential protection, terminal compatibility, and measured recovery as implementation validation work. G0–G3 and the acceptance matrix remain unexecuted. Owner review of the proposed platform/authentication/distribution defaults remains appropriate; no additional product features are required by this review.
