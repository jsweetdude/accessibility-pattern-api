---
id: "dialog.modal"
stack: "web/react"
status: "beta"
aliases: [dialog, modal]
tags: [dialog, modal, overlay, focus-trap]
summary: <one sentence>
---

# <Pattern Title>

## Use When
- Use when you must block interaction with the page until the user completes or dismisses a task.
- Use when the user needs to make a decision that must be acknowledged before continuing (confirmations, required choices).
- Use when you need to collect a small amount of input without navigating away (short forms like name/email, a single setting, a quick filter).
- Use when the content is contextual to the current page and returning to the same point matters.

## Do Not Use When
- Do not use when the content is long-form or requires sustained reading/editing (prefer a dedicated page or full-screen route).
- Do not use when the user may need to compare modal content with page content side-by-side (modal blocks context).
- Do not use for critical alerts that require specialized semantics (use an alert dialog pattern with role="alertdialog" and stricter focus rules).

## Must Haves
- Render the modal in a portal outside the application content tree (e.g., document.body) to avoid clipping and stacking-context issues.
- Use semantic dialog markup: the dialog container must have role="dialog" and aria-modal="true".
- Provide an accessible name: the dialog must reference a visible title via aria-labelledby.
- If a description is present, reference it via aria-describedby.
- While open, make the background application container inert (e.g., #app-root or a provided inertRoot) so background content cannot receive focus or pointer input.
- Move focus into the dialog when it opens (at minimum, focus the dialog container).
- Restore focus to the opener element when the dialog closes.
- Support closing via Escape.
- Support closing via backdrop click (unless the product explicitly requires otherwise).

## Don’ts
- Don’t use <dialog> for this pattern; implement the modal using a normal element with role="dialog" for predictable portal-based behavior.
- Don’t rely on aria-modal="true" to block background interaction; it does not prevent focus/pointer access on its own.
- Don’t render the modal inside containers that create clipping or stacking contexts (e.g., overflow: hidden/auto, transform), and don’t rely on z-index alone to “make it work.”
- Don’t inert document.body or document.documentElement; inert only the application content root.
- Don’t omit focus restoration; closing a modal must return the user to where they were.

## Golden Pattern

```tsx
type ModalDialogProps = {
  open: boolean;
  title: string;
  description?: string;
  onClose: () => void;
  children: React.ReactNode;

  /**
   * Optional. The element to mark inert while the dialog is open.
   * Recommended: pass a stable app root element (e.g., document.getElementById("app-root")).
   * If omitted and #app-root is not found, background inert will be skipped.
   */
  inertRoot?: HTMLElement | null;
};

export function ModalDialog({
  open,
  title,
  description,
  onClose,
  children,
  inertRoot,
}: ModalDialogProps) {
  const titleId = React.useId();
  const descId = React.useId();

  const dialogRef = React.useRef<HTMLDivElement | null>(null);
  const openerRef = React.useRef<HTMLElement | null>(null);

  // Capture the opener at the moment we open (so focus can be restored on close).
  React.useLayoutEffect(() => {
    if (!open) return;
    openerRef.current = document.activeElement as HTMLElement | null;
  }, [open]);

  // Background inert while open (optional; depends on target root existing).
  React.useEffect(() => {
    const target = inertRoot ?? document.getElementById("app-root");
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

  // Focus entry (simple) + focus restore (important).
  React.useEffect(() => {
    if (!open) {
      openerRef.current?.focus?.();
      return;
    }

    // Put focus into the dialog container. No focus trap; Tab may leave page (accepted).
    dialogRef.current?.focus();
  }, [open]);

  // Close on Escape.
  React.useEffect(() => {
    if (!open) return;

    function onKeyDown(e: KeyboardEvent) {
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
        // Click outside closes.
        if (e.target === e.currentTarget) onClose();
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
        aria-describedby={description ? descId : undefined}
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
              <p id={descId} style={{ marginTop: 8, marginBottom: 0 }}>
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
- 
