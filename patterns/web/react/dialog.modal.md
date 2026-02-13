---
id: "dialog.modal"
stack: "web/react"
status: "beta"
aliases: [dialog, modal]
tags: [dialog, modal, overlay, focus-trap, blocking]
summary: User-initiated blocking dialog that traps focus, inerts background content, and restores focus on close.
---

# <Pattern Title>

## Use When
- When interaction outside the container must be blocked until dismissal.
- When the user must complete or explicitly dismiss a task before continuing.
- When returning to the same page context after dismissal matters.

## Do Not Use When
- When the content requires long-form reading or editing (prefer a dedicated page).
- When side-by-side comparison with page content is required.
- When critical alerts require immediate announcement and stricter semantics (use `dialog.alert`).

## Must Haves
- The dialog container has `role="dialog"` and `aria-modal="true"`.
- The dialog has an accessible name via `aria-labelledby` or `aria-label`.
- Focus moves to the dialog when it opens.
- Focus is trapped within the dialog while open.
- Focus returns to the invoking element when the dialog closes.
- Escape closes the dialog.
- Background application content is not focusable or interactive while the dialog is open.

## Don’ts
- Do not rely on the native `<dialog>` element for consistent cross-browser modal behavior in portal-based applications.
- Don’t rely on `aria-modal="true"` to block background interaction; it does not prevent focus/pointer access on its own.
- Don’t render the modal inside containers that create clipping or stacking contexts (e.g., `overflow: hidden/auto`, `transform`), and don’t rely on `z-index` alone to “make it work.”
- Don’t inert `document.body` or `document.documentElement`; inert only the application content root.
- Don’t omit focus restoration; closing a modal must return the user to where they were.

## Golden Pattern

```js
"use client";

import * as React from "react";
import { createPortal } from "react-dom";

export function ModalDialog({
  open,
  title,
  description,
  onClose,
  children,
  inertRoot,
}) {
  const titleId = React.useId();

  const dialogRef = React.useRef(null);
  const openerRef = React.useRef(null);

  // Capture the opener at the moment we open (so focus can be restored on close).
  React.useLayoutEffect(() => {
    if (!open) return;
    openerRef.current = document.activeElement;
  }, [open]);

  // Background inert while open (optional; depends on target root existing).
  React.useEffect(() => {
    const target =
      inertRoot || document.getElementById("app-root");

    if (!target) return;

    if (open) {
      target.setAttribute("inert", "");
    } else {
      target.removeAttribute("inert");
    }

    return () => {
      target.removeAttribute("inert");
    };
  }, [open, inertRoot]);

  // Focus entry + restore.
  React.useEffect(() => {
    if (!open) {
      if (openerRef.current && typeof openerRef.current.focus === "function") {
        openerRef.current.focus();
      }
      return;
    }

    if (dialogRef.current && typeof dialogRef.current.focus === "function") {
      dialogRef.current.focus();
    }
  }, [open]);

  // Close on Escape.
  React.useEffect(() => {
    if (!open) return;

    function onKeyDown(e) {
      if (e.key === "Escape") {
        e.preventDefault();
        onClose();
      }
    }

    document.addEventListener("keydown", onKeyDown);
    return () => document.removeEventListener("keydown", onKeyDown);
  }, [open, onClose]);

  if (!open) return null;

  const modal = (
    <div
      role="presentation"
      onMouseDown={(e) => {
        if (e.target === e.currentTarget) {
          onClose();
        }
      }}
      style={{
        position: "fixed",
        inset: 0,
        display: "grid",
        placeItems: "center",
        padding: 16,
        background: "rgba(0,0,0,0.55)",
      }}
    >
      <div
        ref={dialogRef}
        role="dialog"
        aria-modal="true"
        aria-labelledby={titleId}
        tabIndex={-1}
        style={{
          width: "min(560px, 100%)",
          background: "white",
          color: "black",
          borderRadius: 12,
          padding: 16,
          boxShadow: "0 24px 60px rgba(0,0,0,0.35)",
        }}
      >
        <div
          style={{
            display: "flex",
            justifyContent: "space-between",
            alignItems: "flex-start",
            gap: 12,
          }}
        >
          <div style={{ minWidth: 0 }}>
            <h2 id={titleId} style={{ margin: 0 }}>
              {title}
            </h2>
            {description ? (
              <p style={{ marginTop: 8, marginBottom: 0 }}>
                {description}
              </p>
            ) : null}
          </div>

          <button
            type="button"
            onClick={onClose}
            aria-label="Close dialog"
            style={{
              width: 36,
              height: 36,
              borderRadius: 8,
              border: "1px solid rgba(0,0,0,0.2)",
              background: "transparent",
              display: "inline-grid",
              placeItems: "center",
              lineHeight: 1,
              cursor: "pointer",
              flex: "0 0 auto",
            }}
          >
            <span aria-hidden="true" style={{ fontSize: 18 }}>
              ×
            </span>
          </button>
        </div>

        <div style={{ marginTop: 16 }}>{children}</div>
      </div>
    </div>
  );

  return createPortal(modal, document.body);
}
```

## Acceptance Checks
- Open dialog via keyboard → focus moves inside dialog.
- Press Tab repeatedly → focus does not leave the dialog.
- Press Shift+Tab on first focusable → focus moves to last.
- Press Escape → dialog closes.
- After close → focus returns to the triggering element.
- With screen reader enabled → dialog role and title are announced.
