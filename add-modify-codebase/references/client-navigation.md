# Client navigation and editing

This contract applies to Web GUI and mobile application tabs and editable records, including Android and iOS.
It also applies to nested tabs, receipt editors, account editors, and other editors.

## Preserve the original view

- Keep each visited tab mounted or retain equivalent view state. Load a tab when the user first opens it.
- Preserve loaded data, scroll positions, filters, sorting, searches, selections, and unfinished input across tab switches.
- Give each tab and nested tab an independent scroll position or scroll controller.
- Open editors as overlays, dialogs, or sheets above the original content. A full-screen overlay is valid on small screens.
- Keep the original view behind the editor. Do not replace or recreate it as a separate navigation page.
- After the editor closes, restore the original content, position, and focus. Preserve underlying detail panels during nested editing.
- Hide inactive content from focus and accessibility navigation. Preserve keyboard dismissal, system Back, and unsaved-change protection.

## Writes and requests

- Keep unfinished changes in the client until an explicit save or submission. Do not save automatically after a delay.
- On discard, abandon local changes and return directly. Do not save them or reload the underlying view.
- After a successful save, patch the affected records from the authoritative server response. Keep unrelated view state intact.
- After a successful deletion, remove the corresponding record locally. Remove filtered entries when their returned associations no longer match.
- Do not request backend data merely because a tab becomes active or an editor closes.
- Fetch missing editor details when needed. Retain necessary requests for explicit writes, first loads, manual refresh, and genuine data changes.
- Preserve updates for queued jobs, uploads, and reconnection. Suspend unnecessary polling in hidden or covered Web views.
- For Receipt Master, keep statistics and business calculations on the backend. Do not calculate replacement reports to avoid requests.
- Preserve authentication, account isolation, concurrency versions, and offline behavior. Do not reuse another account's retained view data.

## Acceptance

Exercise tab switching and editor save, discard, cancel, Back, and nested editing.
Make sure that each action preserves the intended content and position.
Measure backend request counts. Make sure that normal return paths do not reload the original data.
Make sure that saves and deletions update affected rows from server results and that required background updates still work.
For concepts, prototypes, and audits, describe this behavior within the skill's scope. Do not claim runtime tests passed for generated images.
Apply the same behavior to Web and each supported mobile client. Report any platform build or device tests not performed.
