---
name: frontend-media-player
description: Build or modify a frontend video or audio player, including playback overlays, seek and volume controls, mobile/touch behavior, keyboard shortcuts, and full-screen interfaces. Use when a user asks to add, fix, redesign, embed, or improve browser media playback.
---

# Frontend Media Player

Build media players as a complete interaction surface, not a link to another page. Preserve the browsing context, make touch behavior deliberate, and avoid requesting media while the user is still choosing a seek position.

## Player Launch

Default to a full-viewport in-page overlay when playback starts from a media browser, catalogue, feed, or search result.

- Keep the underlying application mounted. This preserves loaded items, filters, scroll position, and UI state without session-storage restoration code.
- Render the player in an iframe only when it is already a standalone player document. Use `v-if`/conditional rendering so closing the overlay destroys the iframe and stops playback.
- Set iframe permissions explicitly: `allow="autoplay; fullscreen"`.
- Have the child player send a same-origin `postMessage` close event. Validate `event.origin` in the parent before closing the overlay.
- Support standalone use as a fallback: a close action should use browser history if there is no parent frame.
- Do not navigate with `window.location.href` solely to open a player when an overlay can preserve the existing page.

## Viewport And Controls

- Cover the viewport with a fixed, opaque overlay and place the video at `width: 100%; height: 100%; object-fit: contain`.
- Provide an explicit back button in the top-left. It must use the same close operation as the overlay close message and keyboard shortcut.
- Place playback controls over the bottom of the video with a readable gradient or solid backdrop.
- Show the back button and control panel on pointer movement or touch. Fade both after a short shared inactivity delay, including while paused. Do not keep controls permanently visible through a `:hover` rule.
- Use `100dvh` for mobile-aware panels where browser chrome changes viewport height. Keep the player itself non-scrollable.
- Reserve focus and `z-index` for the player overlay so underlying controls cannot be reached while open.

## Pointer And Touch Input

- Use Pointer Events and `setPointerCapture()` for seek and volume controls. This gives identical mouse, pen, and touch behavior and keeps a drag alive after leaving the slider.
- Set `touch-action: none` on draggable controls so a drag does not become page scrolling.
- Clamp all pointer-derived percentages to `[0, 1]` and guard against zero-width controls and unknown media duration.
- Do not implement separate click and touch code paths when pointer events cover both.

### Seek Behavior

- On pointer down and move, update only local preview state: progress width, thumb position, and preview time.
- Do not assign `video.currentTime` during a seek drag. For range-backed streaming, each assignment can trigger a server request for a new segment.
- On pointer up, assign `video.currentTime` once using the final preview value. On pointer cancel, discard the preview and restore the played position.
- While dragging, show a current-time popup above the thumb and a total-time popup at the far end of the seek bar. Hide both on release.
- Keep the progress thumb visible during the drag and keep controls visible until the interaction finishes.

### Volume Behavior

- Update `video.volume` continuously on pointer down and move. Volume should respond in real time.
- Unmute when the user directly adjusts the volume slider.
- Persist the final volume after pointer up or cancellation, not on every move.
- Keep the volume icon and slider fill synchronized through `volumechange`.

## Keyboard And Accessibility

- Make buttons actual `<button>` elements with useful titles or accessible names.
- Treat Backspace as a close-player command and call `preventDefault()` through the player keyboard handler to prevent browser navigation.
- Preserve existing expected shortcuts for play/pause, seek, mute, volume, fullscreen, and help when they exist.
- Never capture browser shortcuts outside the active player overlay.
- Keep seek popup text readable and use tabular numerals for time values.

## Mobile Rules

- Test portrait, landscape, and wide foldable viewports. Do not treat `max-width: 768px` as sufficient phone detection.
- Use coarse-pointer and orientation media queries where a folded device can become wider than a common mobile breakpoint.
- Ensure draggable controls work without hover. A touch user must be able to discover and manipulate the seek thumb and volume bar.
- Avoid fixed desktop-only width assumptions for controls; keep the progress bar flexible and allow compact volume controls.

## Verification

- Test mouse drag, touch drag, pointer cancellation, and release behavior for progress and volume.
- Confirm a seek causes one `currentTime` update on release, not continuous updates while dragging.
- Confirm the player overlay closes without resetting the parent page state or navigating away.
- Check play, pause, close, and error states all expose the controls correctly.
- Test controls in portrait and landscape mobile viewports, plus desktop.
- Run the project's frontend checks and inspect the final diff. If static assets use cache-busting query versions, increment the changed asset's version.
