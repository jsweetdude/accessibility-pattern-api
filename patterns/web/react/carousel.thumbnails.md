---
id: carousel.thumbnails
stack: web/react
status: beta
tags: [carousel, slider, thumbnails, autoplay, reduced-motion]
aliases: [hero-carousel-thumbnails, marquee-thumbnails, featured-gallery-thumbnails, hero-gallery-thumbnails]
summary: Horizontally-advancing carousel with prev/next buttons, thumbnail navigation, and pause behavior.
---

# Carousel with Thumbnail Navigation

## Use When
- Use when you need to highlight a small set of featured items (typically 3–10) in a horizontally-advancing hero region.
- Use when each slide contains a clear headline and a single primary CTA.
- Use when autoplay is desired but must respect reduced motion and user interaction.
- Use when you want to visually preview upcoming slides to encourage exploration beyond the first slide.

## Do Not Use When
- Do not use when users must compare multiple items at once (use a grid or list).
- Do not use when the content is long-form or requires extended reading (use a page/section).
- Do not use when you cannot provide a reliable pause mechanism for motion.

## Must Haves
- Render a carousel container with `aria-roledescription="carousel"` and an accessible name (`aria-label`).
- Each slide must have `aria-roledescription="slide"` and an `aria-label` like “1 of N”.
- Provide Previous/Next buttons as real `<button>` elements.
- Provide thumbnail navigation as real `<button>` elements in normal tab order (no roving tabindex).
- Provide a Pause/Play button.
- Default to paused when `prefers-reduced-motion: reduce`.
- Pause when focus enters the carousel region.
- Slides must include:
  - A title as an `<h2>`
  - A short description
  - One primary CTA (e.g., “View details”)

## Don’ts
- Don’t implement thumbnail navigation using `role="tablist"` / `role="tab"` for this variant.
- Don’t require arrow-key navigation between thumbnails.
- Don’t autoplay without a visible Pause/Play control.
- Don’t ignore `prefers-reduced-motion`.
- Don’t keep moving while the user is interacting (focus inside carousel must pause autoplay).

## Golden Pattern
```js
"use client";

import * as React from "react";
import { Pause, Play } from "lucide-react";

export function CarouselThumbnailsDemo({
  ariaLabel = "Featured content",
  items = DEFAULT_ITEMS,
  autoplay = true,
  intervalMs = 5000,
}) {
  const [index, setIndex] = React.useState(0);
  const [isPaused, setIsPaused] = React.useState(false);

  const hasUserToggledPauseRef = React.useRef(false);
  const reducedMotionRef = React.useRef(false);
  const timerRef = React.useRef(null);
  const skipFocusPauseRef = React.useRef(false);

  const count = items.length;

  // Reduced motion: paused by default.
  React.useEffect(() => {
    const mq = window.matchMedia("(prefers-reduced-motion: reduce)");
    const apply = () => {
      reducedMotionRef.current = !!mq.matches;
      if (mq.matches) {
        setIsPaused(true);
      }
    };

    apply();

    // Safari still supports addListener in some versions
    if (mq.addEventListener) mq.addEventListener("change", apply);
    else mq.addListener(apply);

    return () => {
      if (mq.removeEventListener) mq.removeEventListener("change", apply);
      else mq.removeListener(apply);
    };
  }, []);

  function goTo(nextIndex) {
    const clamped = ((nextIndex % count) + count) % count;
    setIndex(clamped);
  }

  function goPrev() {
    goTo(index - 1);
  }

  function goNext() {
    goTo(index + 1);
  }

  function pause() {
    setIsPaused(true);
  }

  function togglePause() {
    skipFocusPauseRef.current = false;
    hasUserToggledPauseRef.current = true;
    setIsPaused((p) => !p);
  }

  function onPauseButtonPointerDown() {
    skipFocusPauseRef.current = true;
  }

  // Pause autoplay when focus enters the carousel.
  function onFocusCapture() {
    if (skipFocusPauseRef.current) {
      skipFocusPauseRef.current = false;
      return;
    }
    pause();
  }

  // Autoplay (respects reduced motion, focus-paused state, and user pause).
  React.useEffect(() => {
    // Clear any existing timer.
    if (timerRef.current) {
      window.clearInterval(timerRef.current);
      timerRef.current = null;
    }

    if (!autoplay) return;
    if (isPaused) return;
    if (reducedMotionRef.current) return;

    timerRef.current = window.setInterval(() => {
      // Do not rotate if user paused manually.
      // (If you want “Play” to resume, that’s handled by togglePause.)
      goTo((prev) => {
        return prev;
      });
    }, intervalMs);

    window.clearInterval(timerRef.current);
    timerRef.current = window.setInterval(() => {
      setIndex((i) => ((i + 1) % count));
    }, intervalMs);

    return () => {
      if (timerRef.current) {
        window.clearInterval(timerRef.current);
        timerRef.current = null;
      }
    };
  }, [autoplay, isPaused, intervalMs, count]);

  const active = items[index];

  // aria-live: off while moving, polite when paused so changes can be announced if user moves slides.
  const ariaLive = isPaused ? "polite" : "off";

  return (
    <section
      aria-roledescription="carousel"
      aria-label={ariaLabel}
      onFocusCapture={onFocusCapture}
      style={{
        position: "relative",
        maxWidth: 960,
        margin: "0 auto",
        padding: 16,
        background: "#fff",
        color: "#111",
        borderRadius: 12,
      }}
    >
      {/* Slides viewport */}
      <div
        aria-live={ariaLive}
        style={{
          position: "relative",
          overflow: "hidden",
          borderRadius: 12,
          minHeight: 400,
          background: "#000",
        }}
      >
        <button
          type="button"
          onPointerDown={onPauseButtonPointerDown}
          onClick={togglePause}
          aria-label={isPaused ? "Play automatic rotation" : "Pause automatic rotation"}
          aria-pressed={isPaused}
          style={pauseButtonStyle}
        >
          {isPaused ? (
            <Play size={18} aria-hidden="true" focusable="false" />
          ) : (
            <Pause size={18} aria-hidden="true" focusable="false" />
          )}
        </button>

        {/* Slide */}
        <div
          aria-roledescription="slide"
          aria-label={`${index + 1} of ${count}`}
          style={{
            display: "grid",
            gridTemplateColumns: "1fr",
            alignItems: "end",
            minHeight: 400,
            padding: "16px 72px 72px",
            backgroundImage: active.image ? `url(${active.image})` : undefined,
            backgroundSize: "cover",
            backgroundPosition: "center",
          }}
        >
          <div
            style={{
              maxWidth: 520,
              background: "#000",
              color: "#fff",
              padding: 12,
              borderRadius: 10,
            }}
          >
            <h2 style={{ margin: 0, fontSize: 24 }}>{active.title}</h2>
            <p style={{ marginTop: 8, marginBottom: 12, lineHeight: 1.4 }}>
              {active.description}
            </p>
            <a
              href={active.href}
              style={{
                display: "inline-block",
                padding: "10px 12px",
                borderRadius: 10,
                background: "#fff",
                color: "#000",
                textDecoration: "none",
                fontWeight: 600,
              }}
            >
              View details
            </a>
          </div>
        </div>

        {/* Prev / Next (inside visible slide area) */}
        <button
          type="button"
          onClick={() => {
            pause();
            goPrev();
          }}
          aria-label="Previous slide"
          style={navButtonStyle("left")}
        >
          ‹
        </button>

        <button
          type="button"
          onClick={() => {
            pause();
            goNext();
          }}
          aria-label="Next slide"
          style={navButtonStyle("right")}
        >
          ›
        </button>

        {/* Thumbnail navigation overlay (inside viewport, outside slide node) */}
        <div
          style={{
            position: "absolute",
            right: 12,
            bottom: 12,
            zIndex: 2,
          }}
        >
          <div style={{ display: "flex", gap: 10 }} aria-label="Choose a slide">
            {items.map((item, i) => {
              const isActive = i === index;
              return (
                <button
                  key={i}
                  type="button"
                  onClick={() => {
                    pause();
                    goTo(i);
                  }}
                  aria-label={`Go to slide ${i + 1}: ${item.title}`}
                  aria-current={isActive ? "true" : undefined}
                  style={thumbnailButtonStyle(isActive)}
                >
                  <span
                    aria-hidden="true"
                    style={{
                      display: "block",
                      width: "100%",
                      height: "100%",
                      borderRadius: 10,
                      backgroundImage: item.thumbnail ? `url(${item.thumbnail})` : undefined,
                      backgroundSize: "cover",
                      backgroundPosition: "center",
                    }}
                  />
                </button>
              );
            })}
          </div>
        </div>
      </div>
    </section>
  );
}

function navButtonStyle(side) {
  return {
    position: "absolute",
    top: "50%",
    transform: "translateY(-50%)",
    [side]: 12,
    width: 44,
    height: 44,
    borderRadius: 12,
    border: "1px solid rgba(255,255,255,0.25)",
    background: "rgba(0,0,0,0.45)",
    color: "#fff",
    display: "grid",
    placeItems: "center",
    cursor: "pointer",
    fontSize: 28,
    lineHeight: 1,
  };
}

const pauseButtonStyle = {
  position: "absolute",
  top: 12,
  left: 12,
  zIndex: 2,
  width: 40,
  height: 40,
  borderRadius: 10,
  border: "1px solid rgba(255,255,255,0.25)",
  background: "rgba(0,0,0,0.6)",
  color: "#fff",
  display: "grid",
  placeItems: "center",
  cursor: "pointer",
};

function thumbnailButtonStyle(active) {
  return {
    width: 72,
    height: 44,
    padding: 0,
    borderRadius: 10,
    border: active ? "2px solid #111" : "1px solid rgba(0,0,0,0.35)",
    background: active ? "rgba(0,0,0,0.06)" : "transparent",
    cursor: "pointer",
  };
}

const DEFAULT_ITEMS = [
  {
    title: "Neighbors",
    description: "A chaotic dispute spirals. Watch the latest episode now.",
    href: "#",
    image: "https://picsum.photos/seed/hero-1/1200/600",
    thumbnail: "https://picsum.photos/seed/thumb-1/240/140",
  },
  {
    title: "UCLA at Michigan",
    description: "Tip-off at 12:45 PM ET. Catch it live.",
    href: "#",
    image: "https://picsum.photos/seed/hero-2/1200/600",
    thumbnail: "https://picsum.photos/seed/thumb-2/240/140",
  },
  {
    title: "Fire Country",
    description: "A risky mission tests loyalties and nerves.",
    href: "#",
    image: "https://picsum.photos/seed/hero-3/1200/600",
    thumbnail: "https://picsum.photos/seed/thumb-3/240/140",
  },
];
```

## Acceptance Checks
- Tab into the carousel: autoplay pauses immediately.
- With prefers-reduced-motion: reduce, autoplay is paused by default.
- Previous/Next buttons are reachable by Tab and change slides on activation.
- Each thumbnail is reachable by Tab and activates its slide on activation.
- Each slide has an <h2>, description text, and one “View details” link.
- Screen reader announces the carousel label and slide position (e.g., “2 of 3”) when slides change while paused.
