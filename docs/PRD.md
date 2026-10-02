# Fyminal — Product Requirements Document

Version: 1.1 • 2 October 2026 • Status: proposed product baseline

Repository: https://github.com/lukehungngo/fyminal

## 1. Product brief

**Problem.** When away from the Mac, the owner needs to inspect and control work running in its terminal, including an AI coding agent, from a phone. Returning to the desk for an approval or a short command interrupts that workflow. Mobile network changes and app suspension make returning to the same work unreliable unless session persistence is handled on the Mac.

**Product.** Fyminal is a personal phone SSH terminal for one Mac, reached through the owner's existing Tailscale network. It displays a real interactive terminal and sends the user's input. Commands, files, AI tools, and their execution remain on the Mac. The app has no AI-specific orchestration or model API integration.

**Core journey.** Open Fyminal → connect to Mac → attach to the working terminal → type, inspect, approve, or interrupt → leave → return to the same work.

**MVP boundary.** One owner, one saved Mac connection, one active terminal attachment. A persistent tmux session is the default; an explicitly chosen plain SSH shell is available for general commands. Keep WezTerm as the Mac terminal interface.

**Success.** The owner can complete a real AI CLI interaction from the phone, leave for 30 minutes, and resume the same live process after reconnecting. This requires the Mac and the remote process to remain running. Reconnection must never silently replay commands or approval keystrokes.

**No separate hosting.** Use macOS Remote Login and the separately installed Tailscale apps. No Fyminal cloud, relay service, database server, Mac companion daemon, account, or subscription is required. Tailscale may itself use its infrastructure or encrypted relays; “no hosting” means no additional Fyminal service. [S1, S2]

## 2. Evidence, assumptions, and ownership

### Confirmed by the owner

- Wants a small alternative to Termius focused on SSH, not its wider feature set.
- Already uses Tailscale and WezTerm on the Mac.
- Wants a general terminal with the permissions of the SSH user, mainly to work with an AI tool on the Mac from a phone.
- Cares about security, connection stability, backgrounding, and resuming work.
- Provided this repository and requested a full PRD plus adversarial validation.

### Proposed defaults, not previously confirmed requirements

| Decision | Default for this PRD | Reason / validation |
|---|---|---|
| Phone platform | Native iPhone app first | Matches the conversation context; Android and web are excluded from this release. Confirm on PRD review. |
| Authentication | One app-generated Ed25519 key; no password login UI | Small credential surface; owner can install a public key on the Mac. Verify library support in the feasibility gate. |
| Target input | Mac's numeric Tailscale IPv4 address; fixed SSH port 22 | Avoid discovery, DNS, arbitrary hosts, and connection-manager scope. IPv6/MagicDNS can be added later. |
| Persistent session | Named tmux session `fyminal` under the SSH user | Standard SSH interoperability, including WezTerm and a custom phone client. |
| Distribution | Personal installation using Xcode initially | Validate signing, provisioning, supported OS, and reinstall workflow on the actual phone. TestFlight is optional later. |
| Delivery appetite | Propose a 1–2 week personal MVP after feasibility | Planning allowance, not a commitment. Earlier “few hours” estimates cover a connection demo, not this acceptance bar. |

The owner decides product trade-offs and accepts the MVP. The implementing engineer chooses compatible libraries and records device/OS versions. A reviewer checks security-sensitive behavior and the acceptance evidence. For this personal project one person may fill multiple roles; no enterprise governance process is implied.

Repository inspection on 2 October 2026 found an empty public repository with default branch name `master`. There was no code, architecture, README, or repository instruction file to inherit. This document specifies intended behavior; it is not evidence of an implemented or tested app.

## 3. Goals and exclusions

### Goals

1. Reach the Mac terminal with minimal repeated setup.
2. Reliably type prompts and commands, read terminal output, and interact with CLI menus and approvals from a phone.
3. Resume surviving remote work after phone backgrounding, termination, or a network interruption.
4. Protect the SSH credential and verify the Mac's identity without adding a Fyminal backend.

### Not in this release

Multi-host lists, multiple phone tabs, file transfer/SFTP, graphical file editing, port forwarding, SSH agent forwarding, jump hosts, public-internet SSH, password authentication UI, credential import/export, cloud sync, team sharing, built-in AI chat, AI subscriptions, remote desktop/screen sharing, notification of AI completion, background audio/VPN tricks, Mosh, WezTerm mux protocol integration, automatic Mac configuration, wake-on-LAN, or process restoration after a Mac reboot.

Existing tmux windows can be operated through terminal shortcuts; a separate session/window-management UI is excluded. The app does not take over arbitrary existing WezTerm tabs. Work must already be inside the selected tmux session to share it. [S3, S4]

## 4. User journeys and interface

### First connection

1. Show a short prerequisite checklist: Mac awake and online, Tailscale connected on both devices, macOS Remote Login enabled for the intended user, and tmux installed for persistent mode.
2. Generate a dedicated SSH key on the phone. Show the public key and instructions for adding it to the Mac user's `authorized_keys`. The private key never needs to leave the phone.
3. Save the Mac's Tailscale IP and short macOS username. Provide one Connect button.
4. On first SSH contact, show the actual host-key algorithm and SHA-256 fingerprint. The user compares it with a fingerprint obtained locally on the Mac before trusting it.
5. Authenticate, allocate an interactive PTY, then offer to create the `fyminal` session if it does not exist. If it exists, attach without disconnecting other clients.
6. Display the terminal with a compact key toolbar and connection status.

### Daily use and desktop handoff

The owner opens WezTerm on the Mac and starts or attaches to `fyminal`, then launches the AI CLI or other work there. Fyminal connects as the same Mac user and attaches to that session. Both displays can observe it and both can send input. This is shared control, not independent copies; avoid typing from both at the same time.

Phone viewport changes may resize a shared tmux pane according to tmux's configuration. Do not detach or kill the desktop client to improve phone layout or silently change global tmux settings. G0 must select a session/window sizing policy; G2 must demonstrate readable phone prompts and approvals with the keyboard visible while WezTerm remains attached. Desktop redraw or a smaller shared pane is acceptable; an inaccessible phone cursor or approval is not.

### Leave and return

The app obscures its terminal snapshot when inactive. On backgrounding, stop accepting input and close the phone SSH connection within the normal lifecycle allowance. Detaching must not terminate the tmux shell. Do not promise indefinite iOS background execution. After a true background/lock transition, unlock local credential access, reconnect if a connection was previously active, and reattach only after continuity verification. Transient inactivity caused by an OS authentication sheet or other system overlay only obscures the view; it must not by itself tear down the connection or trigger a new unlock cycle. Returning from successful authentication cannot recursively trigger another authentication prompt. [S3, S5]

After a full app termination, open disconnected with a Resume button rather than executing commands on launch. In plain-shell mode, backgrounding/disconnection may end the shell and attached work; explain this before entry. tmux persistence is an outcome on the Mac, not a guarantee that the phone connection stays alive.

### Minimal surfaces

| Surface | Necessary content |
|---|---|
| Connection setup | IP, username, public-key setup, Connect, short prerequisite help |
| Host trust prompt | Endpoint, key algorithm, fingerprint, Trust after verification, Cancel |
| Terminal | Output, keyboard, Ctrl/Esc/Tab/arrows toolbar, status, Disconnect |
| Small settings/help sheet | Edit connection, persistent/plain mode, key/public-key details, verified host identity, reset instructions |
| Recovery prompt | What is known to have failed and the next useful action; no raw secret-bearing debug output |

No dashboard, onboarding account, subscription screen, or AI configuration screen.

## 5. Functional requirements

All requirements below are MVP requirements. Acceptance is observable behavior; architecture choices remain open where they do not affect that behavior.

| ID | Requirement | Acceptance criteria |
|---|---|---|
| FR-01 | One saved connection | Save/edit the IP and username locally; restore after app relaunch; reject malformed input. Editing target or user closes any existing connection, clears the old terminal view and continuity identity, and invalidates stale connection callbacks. A changed endpoint requires its own host verification. |
| FR-02 | Tailscale destination restriction | Accept only numeric IPv4 within Tailscale's documented `100.64.0.0/10` range, fixed port 22. Connect only to that saved address. No discovery, fallback LAN/public host, proxy, or bypass path. This range check is not proof that Tailscale is active or that the endpoint is genuine; host verification remains mandatory. MVP restricts destinations and is intended for the separately managed Tailscale route; it does not attest the network path. [S8, S9] |
| FR-03 | Dedicated key authentication | Generate and retain one Ed25519 key, expose its public key for setup, and authenticate without asking for an AI credential. Unsupported server authentication produces an actionable failure, never password fallback. |
| FR-04 | Verify host identity | Before user authentication, explicitly verify and pin the first host key. On later connections the presented key must match. Mismatch blocks connection and automatic retry; reset is a separate explicit settings action requiring fresh verification. |
| FR-05 | Real interactive SSH terminal | Allocate a PTY, start the user's shell/session, handle standard terminal escape sequences, colors, cursor positioning, UTF-8, and alternate screens. A streamed command-output text box does not satisfy this requirement. |
| FR-06 | Mobile input | Provide Ctrl combinations including Ctrl-C and Ctrl-D, Esc, Tab, arrows, Enter, Backspace, text selection/copy, explicit paste, and hardware keyboard input. Disable autocorrection and smart punctuation for terminal entry; composed Unicode input must not duplicate characters. |
| FR-07 | Terminal resizing | Keyboard show/hide and device rotation update usable rows/columns and send PTY resize events. Cursor and active input remain visible; full-screen CLI redraws correctly. |
| FR-08 | Persistent mode | After authentication, check the named session and continuity identity; attach only under the rules below. Otherwise ask before creating or using a replacement. Validate that supported session settings retain work with zero clients; incompatible destroy-unattached behavior blocks persistent mode with setup guidance. If tmux is absent or cannot start, offer help or explicit plain-shell mode; never silently change modes or global configuration. |
| FR-09 | Resume the same work | Remember a session-incarnation identifier that survives detachment but changes on session recreation or tmux-server restart. Reconnect automatically only when it matches; do not restart the AI CLI or any command. Missing session, missing identity, or same-name replacement requires explicit acknowledgement before creation/attachment. Session continuity does not prove that a particular process inside it is still running. |
| FR-10 | General shell mode | Explicit plain SSH mode gives normal command access under the Mac user. Show a concise persistence limitation before entry. Reconnection to this mode requires confirmation that a new shell will be opened. |
| FR-11 | Automatic network recovery | While active and in persistent mode, transient transport loss triggers bounded retries and then reattachment. Trust/authentication failures require user action. Disconnect cancels all retries. Only one live SSH connection/PTY may be owned by the app. |
| FR-12 | Background/resume lifecycle | Stop input and detach on true background/device lock; revalidate a connection on foreground before enabling input. Auto-resume only a previously active persistent session, after device authentication. Cold launch needs a tap. Transient inactive/system-auth transitions must not cause reconnect/unlock loops. No background polling or infinite runtime claim. |
| FR-13 | Input uncertainty | Disable terminal input whenever the connection is not interactive. Never queue offline keystrokes or replay bytes from a failed connection. Show that the last input may have reached the Mac; let the user inspect remote state before repeating an operation. |
| FR-14 | Safe client disconnect | Disconnect closes the phone attachment, not the tmux session. Sending Ctrl-D or `exit` is user input and can terminate the remote shell; it must not be sent by a Disconnect button. |
| FR-15 | Output and scrollback | Show live output and allow scrolling/selecting. Retain a bounded in-memory buffer; never write terminal transcripts to disk. After reattachment, show tmux's current state; complete lost output history is not promised. |
| FR-16 | Error recovery | Distinguish observable timeout/unreachable, connection refused, host-key mismatch, authentication rejected, PTY failure, tmux missing, and session missing. Do not assert “Mac asleep” or “Tailscale off” from a timeout alone. |
| FR-17 | Reset and revoke guidance | Reset connection data, host pin, or local key only through explicit actions with impact explained. Key regeneration requires installing the new public key. Removing a local key does not revoke the old public key on the Mac; show the manual revocation step. |
| FR-18 | Paste and remote output safety | Preview and confirm multiline/control-character paste, do not append Enter automatically, and honor bracketed paste when supported. Remote output must not write/read the system clipboard, open URLs, or invoke local app actions without a user gesture. |

## 6. Connection and session behavior

Connection states: Unconfigured, Disconnected, Connecting, Verify host, Authenticating, Preparing session, Connected, Reconnecting, and Needs action. Local access separately has Locked, Unlocking, and Unlocked states. Terminal input/output is exposed only while Connected and Unlocked. Cancellation of OS authentication leaves the retained terminal view obscured.

| Event | Required behavior |
|---|---|
| Connect tapped | Start one attempt; repeated taps do not spawn extra clients. |
| Healthy connection | Keepalive may detect dead transport; it does not override iOS suspension. |
| Transient loss while active | Reconnect with increasing delays; proposed delays 1, 2, 4, then at most 5 seconds, bounded to a 60-second wall-clock recovery window including attempts and delays. Only one attempt is in flight. Once the window expires, require Resume. Show progress and Cancel. |
| Network becomes available / foreground resumes | Attempt promptly if persistent-session recovery was pending; never run two attempts concurrently. |
| Timeout | Stop the attempt after a finite deadline (proposed 10 seconds total for network/handshake); do not count time waiting for user verification as network time. |
| Wrong key or changed host key | Stop immediately; no retry storm and no automatic downgrade. |
| User disconnects | Cancel attempts and suppress automatic reconnection until another user Connect/Resume. |
| Mac asleep/offline | Explain that the Mac must become reachable; Fyminal cannot continue computation on a sleeping Mac. |
| Mac reboot / tmux exited | Session may be missing or replaced under the same name; check its incarnation. A missing/changed/unknown identity requires user confirmation, not a claim of restored work. |
| tmux detaches or remote shell exits cleanly | Return to Disconnected; do not respawn the shell or AI process automatically. |

Input delivery is not an exactly-once application protocol. If Enter reaches the Mac just before a disconnect, the command might already be running. The app must not attempt to deduplicate or automatically rerun arbitrary shell commands.

Continuity identity must distinguish same-name recreation across server restarts; a session name or a reusable numeric tmux ID alone is insufficient. G0 can choose a per-session random marker stored as tmux metadata and remembered in the phone's local connection state. It is a continuity aid, not authentication against a compromised Mac. An existing unmarked session requires explicit first-attachment acknowledgement before recording its identity. A confirmation authorizes attaching to the changed session, never replaying prior input.

Persistent mode requires zero-client survival. Validate relevant tmux options for the supported version, document that custom hooks/process exits can still terminate work, and never silently alter the owner's global config. A session can keep existing while its AI process has exited; the terminal must show the actual current state.

tmux discovery and attachment need to handle macOS SSH PATH differences. Validate the executable path rather than treating a missing PATH entry as proof that tmux is uninstalled. Generated remote commands must use fixed/validated arguments and proper escaping; do not interpolate free-form settings into a shell script.

## 7. Security and data boundaries

**Trust model.** Ordinary SSH connects to macOS Remote Login across Tailscale networking. This does not require the separate Tailscale SSH server feature. Network authorization and SSH user authentication are distinct. A compromised authorized device/account, compromised Mac, or compromised SSH client remains a risk. [S1, S2]

| Asset / boundary | Requirement |
|---|---|
| Private SSH key | Store with device-only Keychain protection available only while unlocked; no iCloud sync/export, application logs, crash reports, or repository copies. Never claim Secure Enclave-backed Ed25519 without verified support. Clear temporary key bytes where supported. [S6] |
| Phone access | Require OS device authentication before opening a new/returning interactive session; permit system passcode fallback to biometrics. No repeated prompts per keystroke. Cancellation leaves the terminal locked. |
| Host identity | Store the verified pin in protected local storage associated with the connection identity. Do not silently accept a changed key, even at the same Tailscale IP. |
| Visible terminal content | Obscure app-switcher snapshots and retained output until local authentication succeeds; clear app-owned output buffers on reset or target/user change, and exclude transcripts from logging/backup. The app cannot promise to prevent user screenshots or eliminate all OS-managed memory remnants. |
| Logs | Keep only bounded local lifecycle/error codes and timings in memory. Never capture terminal input/output, public/private keys, tokens, file contents, or commands. No analytics/crash SDK in MVP. |
| Terminal parser | Use an established parser, bounded buffers, and defensive limits for malformed/oversized sequences. Disable remote clipboard escape actions such as OSC 52 by default. |
| Mac privileges | Use the configured macOS user. No automatic root login, stored sudo password, agent forwarding, or automatic AI approval. User-entered `sudo` works according to macOS policy. |
| Tailscale policy | Setup guidance recommends allowing the phone to reach the Mac on TCP 22 and reviewing broader overlapping grants. The app does not modify tailnet policy or prove effective ACL restrictions. |
| SSH exposure | Tailscale does not automatically bind macOS SSH only to its interface. No router port-forwarding is needed; strict Tailscale-only ingress requires separately validated Mac firewall configuration. |
| Lost phone | Revoke/remove the device in Tailscale and remove its SSH public key from the Mac using another trusted device. Biometric gating alone is not revocation. |

AI tool credentials remain managed on the Mac, but AI commands can print secrets into terminal output; the phone must treat all output as sensitive. Data subsequently sent by the AI CLI to its provider is outside Fyminal's transport and privacy promises.

## 8. System responsibilities and options

| Component | Responsibility |
|---|---|
| Fyminal on iPhone | Connection settings, SSH client, terminal emulator, input, host trust, Keychain, lifecycle and reconnect |
| Tailscale on each device | Private network reachability and applicable network access policy; installed and managed separately |
| macOS Remote Login | SSH service and user authorization |
| tmux on Mac | Persistent shell/session and concurrent client attachment |
| WezTerm on Mac | Local terminal view into the same tmux session |
| AI CLI / shell tools | Actual commands, approvals, files, API calls, and task execution |

**Selected direction: native iPhone client + SSH + tmux.** Avoids a web-to-SSH bridge and uses the existing Mac services. Native UI technology and libraries are implementation decisions; SwiftUI/UIKit with a maintained SSH library and terminal component is a reasonable candidate, not a verified dependency selection.

**Alternative: browser terminal with a Mac bridge.** Adds a local service, HTTP/WebSocket authentication, and more lifecycle/deployment work. Excluded given the simple direct-SSH goal.

**Alternative: WezTerm's mux protocol.** Can provide session persistence but requires a compatible client/integration beyond ordinary SSH. Excluded from MVP. [S4]

There is no product API, hosted account system, application database, or AI inference layer to design. Local metadata and protected credentials are sufficient.

## 9. Measurable acceptance and nonfunctional targets

These are proposed release targets, not measured results or service guarantees. Measure on the owner's actual iPhone and Mac, with OS, app/library versions, Tailscale path (direct/relay when observable), and conditions recorded. No production telemetry service is needed.

| ID | Target | Measurement |
|---|---|---|
| NFR-01 | At least 19/20 healthy-network connection attempts become interactive within 5 seconds | Both devices awake, authenticated Tailscale online, trusted host and ready key; exclude human unlock/verification time. |
| NFR-02 | At least 19/20 interrupted persistent sessions reattach within 20 seconds after end-to-end reachability is restored | App active and locally unlocked; restore the route within the first 30 seconds of the 60-second recovery window. Time restoration using an independent connection probe/test fixture. Verify session-incarnation continuity and surviving test-process PID. Measure foreground/manual Resume separately against the 5-second healthy-connection target after unlock and Resume. An expired window needs a tap; report relay results separately. |
| NFR-03 | 10/10 background-and-resume trials preserve the remote test process | Include 1, 5, and 30-minute phone absences and at least two forced app terminations; Mac awake throughout. Cold launches require Resume. |
| NFR-04 | Zero automatic input replay, duplicate connections, or automatic AI restart | Instrument a harmless remote counter/log fixture during 20 interruption trials, including a break immediately after Enter. Manual input repeats are excluded and recorded. |
| NFR-05 | Responsive terminal under representative output | 30-minute session including 60 seconds of 100 KB/s generated output; keyboard and Disconnect remain responsive, memory plateaus under the chosen buffer cap, no crash. Measure UI response target under 100 ms separately from network echo. |
| NFR-06 | Supported terminal workflows work on-device | Shell, multiline prompt paste, full-screen editor, CLI approval menu, Ctrl-C, Unicode, rotation, keyboard changes, and one owner-selected AI CLI all pass AT-04/05. |
| NFR-07 | Every defined security negative test passes | Key mismatch, unauthorized key, locked credential store, malformed terminal output, unsafe paste, and reset/revocation scenarios. |
| NFR-08 | Basic mobile accessibility | Setup/status controls have accessible labels, standard touch targets, and support system text size without blocking Connect/Disconnect. Terminal font size is adjustable; limitations of screen-reader terminal output must be stated. |

Performance failure on a healthy direct route is an implementation issue. An unavailable Mac, denied policy, revoked key, stopped tmux server, or internet outage is an external condition, but its UI handling must still pass.

## 10. Acceptance test matrix

| Test | Scenario and expected result | Covers |
|---|---|---|
| AT-01 | Fresh install → generate key → manually authorize on Mac → compare fingerprint → connect. Canceling trust/auth produces no interactive channel. | FR-01–05 |
| AT-02 | Attempt malformed/LAN/public IP, then valid Tailscale IP. Invalid targets blocked; no fallback endpoint is contacted. Disable Tailscale and require safe failure or the explicitly verified host outcome, without a false VPN-status claim. Simulate a different SSH server answering at the same CGNAT IP: a pinned client blocks on mismatch before user authentication. Prefix validation must not be presented as route attestation. | FR-02, FR-16 |
| AT-03 | Wrong public key, changed host key, and credential-store lock. Each blocks interaction/retry as appropriate; legitimate key rotation needs explicit reset and verification. | FR-03–04, NFR-07 |
| AT-04 | Run shell, editor, `sudo` prompt with a permitted test command, Ctrl-C, Ctrl-D in a disposable shell, arrows/Tab/Esc, selection/copy, hardware keyboard, composed Unicode, and multiline paste. Correct terminal behavior; no unwanted Enter. | FR-05–07, FR-18 |
| AT-05 | Start the owner's chosen AI CLI in tmux from WezTerm; phone reads output, types a prompt, approves one harmless action, rejects another, and interrupts a test action. Nothing is auto-approved. Resize with both clients attached and the phone keyboard visible; prompts/approvals stay usable without detaching WezTerm or mutating global tmux settings. | FR-06–09, NFR-06 |
| AT-06 | Phone background/lock/force-quit and 30-minute absence while a benign counter runs; include trials where the phone is the last client and zero clients remain attached. Resume same incarnation/shell/PID and counter; no restarted AI process or automatic input replay. With destroy-unattached enabled, persistent mode must identify the incompatible prerequisite before promising continuity. | FR-09, FR-12–15, NFR-03 |
| AT-07 | Interrupt connection just before/after Enter. Reconnect and inspect remote counter; client never re-sends uncertain bytes or buffered offline input. | FR-11–13, NFR-02/04 |
| AT-08 | Tap Connect repeatedly, cancel during auth/retry, edit target during an attempt, disconnect during network change. No stale callback reconnects; one connection at most. Opening/dismissing system UI and completing/canceling OS authentication never creates a prompt loop, extra connection, or unauthorized terminal exposure. | FR-01, FR-11–12 |
| AT-09 | Missing tmux, PATH missing tmux, session deleted, Mac reboot, tmux terminated, and same-name session recreation both within a server and after restart. Missing or changed incarnation requires acknowledgement; absent session requires explicit creation; no silent plain-shell fallback. | FR-08–10, FR-16 |
| AT-10 | Disconnect button from persistent mode leaves work alive. Plain mode warns and creates a new shell only after confirmation. Remote clean exit does not trigger a reconnect loop. | FR-10–14 |
| AT-11 | Send oversized/malformed escape sequences and OSC clipboard requests; paste multiline/control-bearing text. No automatic clipboard access/local execution; bounded resource use and no crash. | FR-15, FR-18, NFR-05/07 |
| AT-12 | Background snapshot, app logs, app-owned files and backup inspection using synthetic secret markers. No transcript on disk or private key outside protected storage; no cloud/network telemetry. Canceled unlock hides retained output; reset or target/user change clears the prior terminal view. | FR-03, FR-15, NFR-07 |
| AT-13 | Reset/regenerate key and remove old public key on Mac. Old key fails; new key works only after authorization. App explains local deletion versus server revocation. | FR-17 |
| AT-14 | Repeat connection/recovery timing series and accessible-control checks on the signed personal build, on the actual phone. Capture denominators and failures, not just a successful demo. | NFR-01–08 |

Testing uses disposable directories and harmless fixtures. No automated test authorizes destructive commands in personal projects. All runtime rows are unexecuted until an implementation exists.

## 11. Delivery gates and feasibility questions

| Gate | Deliverable | Exit condition |
|---|---|---|
| G0 — Feasibility | On-device SSH/PTY and terminal proof; proposed timebox 1–2 focused days | Selected libraries support key authentication, strict host checking, modern macOS SSH, Unicode, resizing and required controls; app can be signed/installed on the owner's phone. Demonstrate session-incarnation detection and select a shared viewport policy. Record dependency licenses and versions. |
| G1 — Usable terminal | Setup, key/host trust, terminal controls, manual connection/disconnection | AT-01–04 pass; basic general shell useful without AI coupling. Include a usable manual setup guide for public-key installation/permissions and fingerprint verification matched to the negotiated host-key algorithm. |
| G2 — Continuity | tmux attachment, WezTerm handoff, reconnect and lifecycle handling | AT-05–10 pass, including input uncertainty and no session loss caused by client disconnect. |
| G3 — Personal MVP | Security checks, responsive rendering, setup/recovery guide, installable build | AT-11–14 and all prior tests pass; owner completes the real workflow; no open critical/high security or session-safety defect. |

If G0 fails, revise the proposed library/platform design before adding features. A thin successful connection demo does not waive later gates. Do not promise store distribution or unlimited validity of a personally provisioned build. Apple's free Personal Team provisioning currently expires after seven days and requires rebuilding/reinstalling. G0 must confirm whether the owner accepts that maintenance or chooses a suitable paid-program distribution method before treating the app as dependable away-from-home access. No Apple fee is assumed to have been accepted. [S7]

Implementation decisions to resolve in G0: exact minimum iOS/macOS versions, SSH/terminal libraries and licenses, personal signing method, safe reconnect cancellation mechanics, tmux executable discovery, session-incarnation tracking, shared viewport policy, terminal buffer cap, and the actual AI CLI compatibility case. These are explicit engineering gates, not undefined user-facing MVP behavior.

## 12. Product decision log and review status

| Decision | Rationale |
|---|---|
| General SSH terminal; no AI feature layer | Owner explicitly wants normal shell control. AI is a representative workflow. |
| Native phone client, no Fyminal host | Direct SSH fits the existing Tailscale/Mac setup. |
| Persistent mode uses tmux, with explicit plain mode | Resume live work while retaining basic SSH utility. |
| One Mac and one active terminal | Preserve the requested small scope. |
| Session disappearance requires confirmation | Avoid presenting a new shell as the old task. |
| No command/keystroke replay | Arbitrary terminal input can have irreversible effects. |
| No background connection guarantee | Respect iOS lifecycle and preserve work on the Mac instead. |

Version 1.1 incorporates the independent reviewer's four blocking document findings and four nonblocking recommendations. Adversarial document review and its resolutions are recorded in [PRD_RED_TEAM.md](PRD_RED_TEAM.md). Document validation is not a penetration test, code review, runtime certification, or proof of iOS feasibility. Product defaults remain reviewable by the owner.

## Appendix A. Mac-side setup contract

Fyminal's setup guide must explain these manual prerequisites; the app does not silently install or configure them:

1. Connect both devices to the same authorized Tailscale network and obtain the Mac's Tailscale IPv4 address.
2. Enable macOS Remote Login for the intended account. Install the phone's public key with correct SSH file ownership/permissions; keep the private key on the phone.
3. Obtain the actual SSH host public-key fingerprint locally on the Mac and compare it in the phone app.
4. Install tmux if persistent mode is wanted. Ensure the session survives with zero clients (including compatible destroy-unattached settings and no terminating detach hooks), then in WezTerm run `tmux new-session -A -s fyminal` and start the desired work there. This command is a manual setup convenience; app reconnection must follow the explicit missing-session rule.
5. Keep the Mac awake, powered, and online for ongoing work. Closing a laptop lid, sleeping, rebooting, or terminating the process changes availability. FileVault/pre-login state after reboot may require local intervention; do not promise unattended recovery.
6. Review effective Tailscale access rules and any separate Mac ingress restrictions. The app must not advise disabling the firewall or exposing SSH publicly to fix connectivity.

## Appendix B. Sources and authoring method

Requirements come from this conversation. External references support platform constraints, not claims that an implementation has passed testing. Checked 2 October 2026.

- **S1 — Apple Remote Login:** https://support.apple.com/guide/mac-help/allow-a-remote-computer-to-access-your-mac-mchlp1066/mac
- **S2 — Ordinary SSH over Tailscale:** https://tailscale.com/docs/reference/ssh-over-tailscale
- **S3 — tmux manual, session lifecycle and attachment:** https://man.openbsd.org/tmux.1
- **S4 — WezTerm multiplexing:** https://wezterm.org/multiplexing.html
- **S5 — Apple background execution:** https://developer.apple.com/documentation/uikit/extending-your-app-s-background-execution-time
- **S6 — Apple Keychain services:** https://developer.apple.com/documentation/security/keychain-services/
- **S7 — Apple developer account and distribution:** https://developer.apple.com/help/account/basics/about-your-developer-account
- **S8 — Tailscale address ranges:** https://tailscale.com/kb/1015/100.x-addresses
- **S9 — OpenSSH client identity checking:** https://man.openbsd.org/ssh.1

Used the publicly available `writing-prds` skill from RefoundAI/lenny-skills because no installed copy was available: https://github.com/RefoundAI/lenny-skills/blob/main/skills/writing-prds/SKILL.md. Applied its problem-first brief, measurable outcomes, explicit boundaries, and clarity review, with detailed acceptance criteria following the brief. This use did not install or modify the skill.
