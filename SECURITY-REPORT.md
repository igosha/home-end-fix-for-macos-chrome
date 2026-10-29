# Security Review: home-end-fix-for-macos-chrome

## Scope

Standard scan of the entire current repository: 6 of 6 files fully reviewed, 479 lines total, at clean Git revision d24585d44cf4e8de183af6ef6d4c160de2a5c8bc.

- Scan mode: repository
- Target kind: git_revision
- Target ID: target_sha256_f63930b47e60875ddc55336d833b6ce24c1ee52e8b258247be91490514da1d61
- Revision: d24585d44cf4e8de183af6ef6d4c160de2a5c8bc
- Inventory strategy: repository
- Included paths: .
- Excluded paths: none
- Artifacts reviewed: LICENCE, content.js, manifest.json, README.md, background.js, CHANGELOG.md
- Scan context: User requested leaks/threats review and a Standard scan of the entire repository. A fresh-context baseline audit, independent architecture review, focused privacy/capability investigation and parent full-source audit were completed. The generated architecture model was checked against actual source consumers and retained. No source-supported vulnerabilities survived review. Findings were assessed against actual attacker capability gain, not broad permissions or temporary DOM copies alone. The documented disabled-state startup timing discrepancy is retained as a non-security observation in coverage. No SECURITY.md policy or user-supplied threat model was present.

Limitations and exclusions:
- Git metadata, past revisions and historical secrets were not scanned.
- Live Chrome/Chromium behavior, browser-host internals, platform compatibility and outbound traffic were not dynamically tested.
- This result applies to the six current repository files, not any separately packaged, installed or future version.
- Excluded .git/\*\*: Git metadata and past revisions are outside this current-state source audit.

### Scan Summary

| Field | Value |
| --- | --- |
| Scan outcome | completed |
| Reportable findings | 0 |
| Severity mix | none |
| Confidence mix | none |
| Coverage | complete |
| Validation mode | Offline full-source analysis with independent baseline and focused privacy review; parent-verified dataflow, control and attacker-capability reasoning. |

Canonical artifacts: `scan-manifest.json`, `findings.json`, and `coverage.json`. This report is a deterministic projection of those files.

## Threat Model

Manifest V3 Chrome extension intended to provide Windows-style Home/End navigation in editable fields on macOS. Chrome loads content.js at document_start in matching frames; it reads extension-local enabled state, inspects Home/End events and editable DOM, and updates selections. Textarea navigation temporarily copies the current paragraph into a hidden div attached to the page body for layout measurement. background.js initializes state, handles toolbar toggles, and updates the badge. Installation is documented as loading the repository unpacked; other Manifest V3 Chromium browsers are claimed compatible. Sources: manifest.json:2-18; content.js:1-30,96-195,223-318; background.js:1-38; README.md:36-56.

### Assets

- Editable-field confidentiality: content.js reads input values, including supported password fields, into local variables for selection calculation; textarea paragraph text is temporarily written to a page-body div. Input values are not written to extension storage by the inspected code. Sources: README.md:16; content.js:96-122,223-275,315-317.
- Editing integrity and user intent: the extension can prevent Home/End defaults and change selection ranges in inputs, textareas, and contenteditable roots, including controls reached through Shadow DOM traversal. Sources: content.js:11-45,48-91,125-144,149-208.
- Persistent enabled preference and toolbar status: extension-local key enabled controls content handling and the badge; the background worker writes booleans on initial installation and toolbar clicks. Sources: content.js:1-12; background.js:1-38.
- Installed extension package integrity and broad page reach: the manifest registers one local service worker and one local content script using \<all_urls\>, document_start, and all_frames. The only declared permission is storage. Sources: manifest.json:7-18.

### Trust Boundaries

- Browser host to extension execution: manifest registration requests content.js in matching frames and background.js as a service worker. Chrome owns actual injection eligibility, frame/origin restrictions, extension API authorization, and storage isolation; this repository supplies configuration but not those host implementations. No explicit execution world, external messaging, web-accessible resources, native messaging, or additional permissions are declared. Source: manifest.json:1-20.
- Page-controlled DOM and events to content-script behavior: keydown events are captured on document; Home/End, modifier, enabled-state, editable-context, and selection-support checks precede preventDefault and selection changes. The script does not require e.isTrusted. A hostile page can supply DOM and synthetic events, but normally already controls its own document; no extension-message or storage-write operation is invoked from this event path. Sources: content.js:11-45,48-91,96-195.
- Sensitive textarea value to shared page DOM: getVisualLineBounds extracts the current paragraph, createMirror appends a hidden div to document.body, and mirror.textContent receives the paragraph. The div is removed in finally. Hidden/offscreen styling is a visibility measure, not a confidentiality control against page scripts that can inspect DOM. Sources: content.js:223-275,315-317.
- Text to rendering: textarea paragraph content is assigned through textContent, while individual characters are measured using DOM Range geometry; source text is not used as HTML in this flow. Sources: content.js:269-282.
- Background worker to extension-local storage and badge: initial installation writes enabled=true only when the key is absent; updates, browser startup, and worker evaluation read state and set badge; toolbar clicks read the prior state, write its boolean inverse, and update the badge. Browser-owned action events provide the toolbar entry point. Sources: background.js:1-38.
- Stored preference to content-script enforcement: content.js starts with enabled=true, then asynchronously loads the stored value and listens for subsequent local changes. A persisted disabled preference is therefore not enforced before the initial storage callback. Sources: content.js:1-12. This differs from a literal immediate interpretation of the documented off toggle and persisted disable behavior: README.md:47; CHANGELOG.md:15.

### Attacker Capabilities

- An arbitrary matched webpage can control its own DOM, editable contents, styles, frame contents, and script-generated events. These inputs reach content.js through document keydown handling and DOM/layout traversal. Sources: manifest.json:12-18; content.js:11-91,223-282.
- A page can observe the temporary textarea mirror within its own document, but same-document page script ordinarily can already read that textarea value; mirror visibility alone does not establish a new attacker capability or a confirmed leak. Sources: content.js:254,259-271,315-317. Browser DOM access semantics are a host assumption.
- The inspected source does not grant webpage callers an extension message API, external command API, credential API, native process API, or background storage-write request path. The sole persistent writes originate in background installation and action handlers. Sources: manifest.json:1-20; background.js:10-38; content.js:1-318.
- A local actor able to alter the unpacked extension directory could change the installed extension code, but that assumes package-write authority and is distinct from a hostile webpage attacker. The supported installation procedure selects this directory directly. Source: README.md:36-41.

### Security Objectives

- Keep editable text and password values within the browser editing workflow; do not transmit or persist them. The complete inspected runtime sources contain no network client, telemetry, logging, remote script loader, or text-persistence operation; storage use is limited to enabled state. Sources: content.js:1-318; background.js:1-38.
- Apply selection changes only for enabled Home/End handling in supported editable contexts, preserve Alt/Meta combinations, and avoid interpreting field text as executable markup. Sources: content.js:11-30,90-91,135-144,269-282.
- Preserve the user’s enabled preference across updates and keep the badge consistent with the effective preference. Background initialization distinguishes initial installation from updates; content startup has an asynchronous disabled-state enforcement interval requiring separate consideration. Sources: background.js:10-29; content.js:1-12; CHANGELOG.md:15.
- Keep page-controlled data from acquiring background or extension-storage mutation authority. Current page event handling performs selection/layout operations, while persistent writes are confined to background install/action listeners. Sources: content.js:11-318; background.js:21-38.

### Assumptions

- User-provided scope is the entire current six-file repository, excluding Git metadata/history. Parent-supplied resolver result reports no root SECURITY.md. No supplied deployment threat model or knowledge base was available.
- Chrome/Chromium host controls are external dependencies: actual \<all_urls\> eligibility, restricted pages, file URL access, frame injection, default content-script world, action-event provenance, and extension storage access were not verified from browser implementation. README.md:47,51 describes all pages/frames; manifest.json:14-17 requests \<all_urls\> and all_frames, which does not itself prove unrestricted browser coverage.
- Documentation says the toolbar toggles the extension off and disable state survives updates. Background code preserves stored false, but content.js initializes enabled=true before its asynchronous storage read; immediate disabled enforcement is therefore not guaranteed by the source. Sources: README.md:47; CHANGELOG.md:15; background.js:10-29; content.js:1-12.
- Normal extension-owned writes store booleans. Background treats every value except false as enabled, whereas content.js uses the stored value’s truthiness after loading; inconsistent behavior requires a non-boolean stored value from outside the normal writer paths. Sources: background.js:12-17,32-35; content.js:2-8,12.
- The README claims compatibility with other Manifest V3 Chromium browsers, but the repository contains no browser-specific deployment paths or compatibility tests; browser-version and platform enforcement remain outside this source-only model. Source: README.md:43.
- The repository supplies no build system, release automation, update infrastructure, dependency graph, native component, or hosted service. Only the documented unpacked installation and manifest runtime paths are evidenced. Sources: README.md:36-43; manifest.json:1-20.
- This architecture mapping is not completed vulnerability-audit coverage. No application execution, external service access, exploit inputs, or runtime verification was performed.

## Findings

### No findings

No reportable findings survived the canonical discovery, validation, and reportability gates.

## Reviewed Surfaces

| Surface | Risk Area | Outcome | Notes |
| --- | --- | --- | --- |
| Independent baseline audit of all six repository files | Repository-wide vulnerabilities and embedded secrets | No issue found | The independent baseline auditor fully reviewed LICENCE, content.js, manifest.json, README.md, background.js and CHANGELOG.md and returned no findings. Returned control dispositions: no page-to-background messaging bridge or privileged dataflow (manifest.json:7-17; background.js:10-38; content.js:1-30); only enabled is stored (background.js:11-35); field values are used for selection or temporary same-document mirror (content.js:96-145,223-318); textContent avoids HTML parsing (content.js:271). Synthetic events and large paragraphs were considered without evidence of new attacker capability. Password inputs are not mirrored. No network code, secrets, logging, remote code or dependency execution was found. Static only; no browser execution, network access or history review. Parent reconciliation and focused privacy review completed with no source-supported vulnerability. |
| Focused privacy and extension-capability investigation | Form-text confidentiality, Shadow DOM, frames, synthetic events and privilege boundaries | No issue found | The focused investigator fully reviewed all six files and returned no findings. A MutationObserver can retain the temporary mirror, so hidden styling/removal is not confidentiality protection; however, ordinary textarea values and open shadow roots are already readable by same-document scripts. getEventTarget uses the native composed path, and deep focus traversal descends accessible shadowRoot only (content.js:48-76); no closed-root or cross-frame access path is established. Passwords take the input branch and are not mirrored (content.js:96-143). Synthetic events reach selection/layout only (content.js:11-30,135-195), not background storage writes (background.js:21-38). textContent is a text sink (content.js:271). No secrets or network paths were found. Parent reconciliation confirmed this static-only result; no demonstrated origin, closed-shadow-root or extension-authority crossing. |
| Field confidentiality, storage and outbound dataflow | Data exfiltration and sensitive information retention | No issue found | Parent reviewed content.js:1-318 and background.js:1-38 in full. Only the enabled preference is persisted (background.js:11-35). Input/password values are used for local selection arithmetic (content.js:96-145); only TEXTAREA reaches the temporary mirror (content.js:116-118,258-271). No network clients, telemetry, field logging, clipboard access, cross-frame data aggregation or remote script loading were present. README.md:3-4 badge images and README.md:12 links are documentation resources, not extension runtime dataflow. |
| Plain-text rendering and code interpretation | DOM injection and code execution | No issue found | Textarea text reaches mirror.textContent at content.js:271, then Range geometry at content.js:279-282. No HTML parsing sink, eval, Function constructor, executable dependency, remote script, native bridge, command execution or string-to-code transformation exists in either complete runtime file. Untrusted field text cannot execute as HTML through the reviewed path. |
| Permissions, event provenance, frames and package integrity | Privilege escalation and cross-origin access | No issue found | manifest.json:7-17 declares storage, local background/content scripts, \<all_urls\>, document_start and all_frames. Broad injection expands exposure but is consistent with the stated all-sites editing feature (README.md:47-58); it does not demonstrate leakage. Every script instance uses its current document; no parent/top/frame document traversal or messaging receiver exists. Synthetic Home/End events reach DOM selection/layout (content.js:11-30,135-195) already available to same-page scripts, not extension storage writes or badge operations. Local source tampering presupposes write access to the unpacked package (README.md:36-41); no weaker attacker gaining that access is established. Browser host eligibility/isolation semantics were assumed rather than tested. |
| Enabled preference startup and value consistency | User intent and configuration enforcement | Rejected | A concrete non-security behavior discrepancy is retained: content.js:1-4 starts enabled until its asynchronous read applies stored false, despite off/preservation documentation (README.md:47; CHANGELOG.md:15). background.js:10-29 preserves false across updates. A relevant early event could cause temporary selection/layout handling; no new attacker authority, transmission or sensitive persistence follows, so this is not a security finding. Background uses !== false while content uses truthiness (background.js:12-17,32-35; content.js:2-8,12); disagreement requires a non-boolean stored value outside the normal boolean writer paths. No untrusted-page storage-write path exists. |
| Secrets, source completeness and repository-only surfaces | Embedded credentials, dependencies and unreviewed source | No issue found | All six current files (479 lines total) were read in full by the parent and independent baseline/privacy workers: content.js, background.js, manifest.json, README.md, CHANGELOG.md and LICENCE. Current inventory matched git ls-files --cached --others --exclude-standard. No embedded credential or private key was identified. No build/release system, dependency manifest, bundled library, backend, database, upload handler, filesystem/native component or server parser is present. No services were contacted or credentials exercised. |
| Large textarea paragraphs and renderer resource use | Availability and resource exhaustion | Rejected | content.js:258-271 synchronously copies the current paragraph, and content.js:289-312 uses binary searches plus layout measurements; the mirror is removed in finally (content.js:315-317). Large paragraphs can impose work, but a malicious page already controls its renderer and can directly allocate/layout DOM. No meaningful cross-origin, extension-privileged or shared-service amplification path is supported, so no security vulnerability is established. |
