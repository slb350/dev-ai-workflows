# Style Guide - Design System Best Practices

**Version:** 2.0
**Last Updated:** 2026-02-09
**Purpose:** Comprehensive guide for creating consistent, accessible, and beautiful design systems

---

## Overview

This guide prioritizes **consistency**, **accessibility**, and **visual coherence** through systematic design decisions. Every design choice is documented, reusable, and intentional, creating a cohesive user experience across all touchpoints.

---

## Core Principles

1. **Design Tokens First** - Centralize colors, typography, spacing before components
2. **Accessibility Always** - WCAG AA compliance, semantic HTML, keyboard navigation
3. **Component Reusability** - Build once, use everywhere with clear variations
4. **Consistent Naming** - Clear, predictable naming conventions across all assets
5. **Progressive Enhancement** - Core functionality works without JavaScript
6. **Performance Conscious** - Optimize assets, minimize dependencies, lazy load
7. **Documentation Driven** - Every design decision has documented rationale
8. **User Feedback** - Visual responses for every interaction

---

## The Workflow (Design-Build-Test-Document)

### Phase 1: Design Tokens & Foundation

#### Step 1: Define Color Palette

```css
/* Design Tokens: Colors */
:root {
  /* Primary Brand Colors */
  --color-primary: #E85D4A;           /* Main accent, CTAs, highlights */
  --color-primary-dark: #d14935;      /* Hover states, emphasis */
  --color-primary-light: #FEE9E7;     /* Subtle backgrounds, light emphasis */

  /* Background Colors */
  --color-bg-base: #F5F1EA;           /* Page background (warm cream) */
  --color-bg-surface: #ffffff;        /* Cards, overlays, content areas */

  /* Neutral Colors */
  --color-text-primary: #111827;      /* Headings, primary text (gray-900) */
  --color-text-secondary: #6b7280;    /* Labels, secondary text (gray-500) */
  --color-border: #e5e7eb;            /* Borders, dividers (gray-200) */

  /* Status Colors */
  --color-success: #10B981;           /* Success, completed states */
  --color-error: #EF4444;             /* Error, danger states */
  --color-warning: #EF6B19;           /* Warning, medium priority */
  --color-info: #3B82F6;              /* Info, pending states */
}
```

**Color Selection Checklist:**

- [ ] Brand colors chosen with purpose (warm vs cool, energetic vs calm)
- [ ] Sufficient contrast for WCAG AA (4.5:1 for text, 3:1 for UI)
- [ ] Color-blind friendly palette (test with simulator)
- [ ] Semantic colors defined (success, error, warning, info)
- [ ] Dark/light variations for each primary color

#### Step 2: Typography System

```css
/* Design Tokens: Typography */
:root {
  /* Font Families */
  --font-primary: 'JetBrains Mono', 'SF Mono', 'Monaco', 'Cascadia Code', monospace;
  --font-secondary: -apple-system, BlinkMacSystemFont, 'Segoe UI', 'Roboto', sans-serif;

  /* Font Sizes */
  --font-size-xs: 0.75rem;    /* 12px - Small labels, captions */
  --font-size-sm: 0.875rem;   /* 14px - Labels, secondary text */
  --font-size-base: 1rem;     /* 16px - Body text */
  --font-size-lg: 1.125rem;   /* 18px - Large body text */
  --font-size-xl: 1.25rem;    /* 20px - H3, subheadings */
  --font-size-2xl: 1.5rem;    /* 24px - H2 */
  --font-size-3xl: 1.875rem;  /* 30px - H1 */
  --font-size-4xl: 2.25rem;   /* 36px - Large H1 */
  --font-size-5xl: 3rem;      /* 48px - Hero titles */

  /* Font Weights */
  --font-weight-light: 300;
  --font-weight-normal: 400;
  --font-weight-medium: 500;
  --font-weight-semibold: 600;
  --font-weight-bold: 700;

  /* Line Heights */
  --line-height-tight: 1.25;
  --line-height-normal: 1.5;
  --line-height-relaxed: 1.75;

  /* Letter Spacing */
  --letter-spacing-tight: -0.025em;
  --letter-spacing-normal: 0;
  --letter-spacing-wide: 0.025em;
}

/* Typography Hierarchy */
h1 {
  font-size: var(--font-size-3xl);
  font-weight: var(--font-weight-bold);
  line-height: var(--line-height-tight);
  color: var(--color-text-primary);
  margin-bottom: 1.5rem;
}

h2 {
  font-size: var(--font-size-2xl);
  font-weight: var(--font-weight-bold);
  line-height: var(--line-height-tight);
  color: var(--color-text-primary);
  margin-bottom: 1rem;
}

h3 {
  font-size: var(--font-size-xl);
  font-weight: var(--font-weight-semibold);
  line-height: var(--line-height-normal);
  color: var(--color-text-primary);
  margin-bottom: 0.75rem;
}

body {
  font-size: var(--font-size-base);
  font-weight: var(--font-weight-normal);
  line-height: var(--line-height-normal);
  color: var(--color-text-primary);
}

label {
  font-size: var(--font-size-sm);
  font-weight: var(--font-weight-semibold);
  color: var(--color-text-primary);
}

small {
  font-size: var(--font-size-xs);
  color: var(--color-text-secondary);
}
```

**Typography Checklist:**

- [ ] Font stack chosen (web-safe fallbacks included)
- [ ] Font loaded with `font-display: swap` to prevent FOIT
- [ ] Size scale follows consistent ratio (1.2x or 1.25x recommended)
- [ ] Line heights adjusted per font size (tighter for headings, relaxed for body)
- [ ] Font weights available for hierarchy (minimum: 400, 600, 700)

#### Step 3: Spacing System

```css
/* Design Tokens: Spacing */
:root {
  /* Base spacing unit (0.25rem = 4px) */
  --space-1: 0.25rem;    /* 4px */
  --space-2: 0.5rem;     /* 8px */
  --space-3: 0.75rem;    /* 12px */
  --space-4: 1rem;       /* 16px */
  --space-5: 1.25rem;    /* 20px */
  --space-6: 1.5rem;     /* 24px */
  --space-8: 2rem;       /* 32px */
  --space-10: 2.5rem;    /* 40px */
  --space-12: 3rem;      /* 48px */
  --space-16: 4rem;      /* 64px */
  --space-20: 5rem;      /* 80px */

  /* Semantic Spacing */
  --spacing-section: var(--space-8);      /* Between major sections */
  --spacing-component: var(--space-6);    /* Between components */
  --spacing-element: var(--space-4);      /* Between elements */
  --spacing-inline: var(--space-2);       /* Inline element gaps */
}
```

**Spacing Checklist:**

- [ ] Base unit defined (4px or 8px recommended)
- [ ] Consistent scale (multiples of base unit)
- [ ] Semantic names for common patterns (section, component, element)
- [ ] Applied consistently to padding, margin, gap

#### Step 4: Layout & Grid System

```css
/* Design Tokens: Layout */
:root {
  /* Container Widths */
  --container-sm: 640px;
  --container-md: 768px;
  --container-lg: 1024px;
  --container-xl: 1280px;
  --container-2xl: 1536px;

  /* Grid Columns */
  --grid-columns-mobile: 1;
  --grid-columns-tablet: 2;
  --grid-columns-desktop: 3;
  --grid-columns-wide: 4;

  /* Grid Gaps */
  --grid-gap-sm: var(--space-4);
  --grid-gap-md: var(--space-6);
  --grid-gap-lg: var(--space-8);
}

/* Responsive Container */
.container {
  width: 100%;
  max-width: var(--container-lg);
  margin-left: auto;
  margin-right: auto;
  padding-left: var(--space-4);
  padding-right: var(--space-4);
}

/* Responsive Grid */
.grid {
  display: grid;
  gap: var(--grid-gap-md);
  grid-template-columns: repeat(var(--grid-columns-mobile), minmax(0, 1fr));
}

@media (min-width: 768px) {
  .grid {
    grid-template-columns: repeat(var(--grid-columns-tablet), minmax(0, 1fr));
  }
}

@media (min-width: 1024px) {
  .grid {
    grid-template-columns: repeat(var(--grid-columns-desktop), minmax(0, 1fr));
  }
}
```

#### Step 5: Animation & Transitions

```css
/* Design Tokens: Animation */
:root {
  /* Duration */
  --duration-fast: 0.15s;
  --duration-base: 0.2s;
  --duration-slow: 0.3s;
  --duration-slower: 0.5s;

  /* Easing Functions */
  --ease-linear: linear;
  --ease-in: cubic-bezier(0.4, 0, 1, 1);
  --ease-out: cubic-bezier(0, 0, 0.2, 1);
  --ease-in-out: cubic-bezier(0.4, 0, 0.2, 1);
  --ease-spring: cubic-bezier(0.68, -0.55, 0.265, 1.55);

  /* Common Transitions */
  --transition-base: all var(--duration-base) var(--ease-in-out);
  --transition-colors: color var(--duration-base) var(--ease-in-out),
                       background-color var(--duration-base) var(--ease-in-out),
                       border-color var(--duration-base) var(--ease-in-out);
  --transition-transform: transform var(--duration-base) var(--ease-in-out);
}

/* Animation Keyframes */
@keyframes fade-in {
  from {
    opacity: 0;
    transform: translateY(10px);
  }
  to {
    opacity: 1;
    transform: translateY(0);
  }
}

@keyframes slide-in {
  from {
    transform: translateX(400px);
  }
  to {
    transform: translateX(0);
  }
}

@keyframes spin {
  from {
    transform: rotate(0deg);
  }
  to {
    transform: rotate(360deg);
  }
}
```

**Animation Checklist:**

- [ ] Duration values defined (fast for micro-interactions, slow for transitions)
- [ ] Easing functions chosen appropriately (ease-out for entrances, ease-in for exits)
- [ ] Respects `prefers-reduced-motion` media query
- [ ] Keyframe animations defined for common patterns
- [ ] Smooth 60fps performance (use transform/opacity, not layout properties)

---

### Phase 2: Component Library

#### Step 1: Button Components

```css
/* Base Button Styles */
.btn {
  /* Layout */
  display: inline-flex;
  align-items: center;
  justify-content: center;
  gap: var(--space-2);

  /* Spacing */
  padding: var(--space-3) var(--space-6);

  /* Typography */
  font-size: var(--font-size-base);
  font-weight: var(--font-weight-semibold);
  text-decoration: none;

  /* Visual */
  border-radius: 8px;
  border: 2px solid transparent;

  /* Interaction */
  cursor: pointer;
  transition: var(--transition-base);
  user-select: none;
}

.btn:focus-visible {
  outline: 2px solid var(--color-primary);
  outline-offset: 2px;
}

.btn:disabled {
  opacity: 0.5;
  cursor: not-allowed;
}

/* Button Variants */
.btn-primary {
  background-color: var(--color-primary);
  color: white;
}

.btn-primary:hover:not(:disabled) {
  background-color: var(--color-primary-dark);
  transform: translateY(-2px);
}

.btn-secondary {
  background-color: white;
  color: var(--color-primary);
  border-color: var(--color-primary);
}

.btn-secondary:hover:not(:disabled) {
  background-color: var(--color-primary-light);
}

/* Button Sizes */
.btn-sm {
  padding: var(--space-2) var(--space-4);
  font-size: var(--font-size-sm);
}

.btn-lg {
  padding: var(--space-4) var(--space-8);
  font-size: var(--font-size-lg);
}
```

**Button Component Checklist:**

- [ ] Base button with clear focus states
- [ ] Primary, secondary, tertiary variants
- [ ] Disabled state with visual feedback
- [ ] Size variations (small, medium, large)
- [ ] Icon support (left, right, icon-only)
- [ ] Loading state with spinner
- [ ] Keyboard accessible (Enter/Space to activate)

#### Step 2: Card Components

```css
.card {
  /* Layout */
  display: flex;
  flex-direction: column;

  /* Spacing */
  padding: var(--space-6);

  /* Visual */
  background-color: var(--color-bg-surface);
  border-radius: 12px;
  box-shadow: 0 1px 3px rgba(0, 0, 0, 0.1),
              0 1px 2px rgba(0, 0, 0, 0.06);

  /* Interaction */
  transition: transform var(--duration-base) var(--ease-in-out),
              box-shadow var(--duration-base) var(--ease-in-out);
}

.card-interactive {
  cursor: pointer;
}

.card-interactive:hover {
  transform: translateY(-4px);
  box-shadow: 0 10px 15px rgba(0, 0, 0, 0.1),
              0 4px 6px rgba(0, 0, 0, 0.05);
}

.card-header {
  margin-bottom: var(--space-4);
}

.card-title {
  font-size: var(--font-size-xl);
  font-weight: var(--font-weight-semibold);
  color: var(--color-text-primary);
  margin-bottom: var(--space-2);
}

.card-description {
  font-size: var(--font-size-sm);
  color: var(--color-text-secondary);
}

.card-body {
  flex: 1;
  margin-bottom: var(--space-4);
}

.card-footer {
  border-top: 1px solid var(--color-border);
  padding-top: var(--space-4);
  display: flex;
  gap: var(--space-2);
}
```

#### Step 3: Form Components

```css
/* Input Fields */
.form-group {
  margin-bottom: var(--space-6);
}

.form-label {
  display: block;
  font-size: var(--font-size-sm);
  font-weight: var(--font-weight-semibold);
  color: var(--color-text-primary);
  margin-bottom: var(--space-2);
}

.form-label-required::after {
  content: " *";
  color: var(--color-error);
}

.form-input {
  /* Layout */
  width: 100%;

  /* Spacing */
  padding: var(--space-3) var(--space-4);

  /* Typography */
  font-size: var(--font-size-base);
  font-family: inherit;

  /* Visual */
  background-color: white;
  border: 2px solid var(--color-border);
  border-radius: 8px;

  /* Interaction */
  transition: var(--transition-colors);
}

.form-input:focus {
  outline: none;
  border-color: var(--color-primary);
  box-shadow: 0 0 0 3px var(--color-primary-light);
}

.form-input:invalid:not(:placeholder-shown) {
  border-color: var(--color-error);
}

.form-input:disabled {
  background-color: var(--color-bg-base);
  cursor: not-allowed;
  opacity: 0.6;
}

.form-error {
  display: block;
  margin-top: var(--space-2);
  font-size: var(--font-size-sm);
  color: var(--color-error);
}

/* Checkbox & Radio */
.form-checkbox,
.form-radio {
  /* Custom styling with accent color */
  accent-color: var(--color-primary);
  width: 1.25rem;
  height: 1.25rem;
  cursor: pointer;
}

.form-check-label {
  display: flex;
  align-items: center;
  gap: var(--space-3);
  cursor: pointer;
}
```

**Form Component Checklist:**

- [ ] Text input, textarea, select dropdown
- [ ] Checkbox, radio buttons, toggle switches
- [ ] Required field indicators
- [ ] Focus states with visible outline
- [ ] Error states with clear messaging
- [ ] Disabled states
- [ ] Label association (for attribute matching id)
- [ ] Keyboard navigation (Tab order, Enter to submit)

#### Step 4: Modal/Dialog Components

```css
.modal-overlay {
  /* Positioning */
  position: fixed;
  top: 0;
  left: 0;
  right: 0;
  bottom: 0;
  z-index: 1000;

  /* Visual */
  background-color: rgba(0, 0, 0, 0.8);

  /* Layout */
  display: flex;
  align-items: center;
  justify-content: center;
  padding: var(--space-4);

  /* Animation */
  animation: fade-in var(--duration-base) var(--ease-out);
}

.modal-content {
  /* Layout */
  width: 100%;
  max-width: 600px;
  max-height: 90vh;
  overflow-y: auto;

  /* Visual */
  background-color: var(--color-bg-surface);
  border-radius: 12px;
  box-shadow: 0 25px 50px rgba(0, 0, 0, 0.25);
}

.modal-header {
  display: flex;
  align-items: center;
  justify-content: space-between;
  padding: var(--space-6);
  border-bottom: 1px solid var(--color-border);
}

.modal-title {
  font-size: var(--font-size-2xl);
  font-weight: var(--font-weight-bold);
  color: var(--color-text-primary);
  margin: 0;
}

.modal-close {
  /* Reset button styles */
  background: none;
  border: none;
  padding: var(--space-2);
  cursor: pointer;

  /* Visual */
  color: var(--color-text-secondary);
  border-radius: 4px;

  /* Interaction */
  transition: var(--transition-colors);
}

.modal-close:hover {
  background-color: var(--color-bg-base);
  color: var(--color-text-primary);
}

.modal-body {
  padding: var(--space-6);
}

.modal-footer {
  display: flex;
  gap: var(--space-3);
  justify-content: flex-end;
  padding: var(--space-6);
  border-top: 1px solid var(--color-border);
}
```

**Modal Component Checklist:**

- [ ] Overlay with semi-transparent background
- [ ] Centered content with max-width
- [ ] Close button (X) and Escape key handler
- [ ] Focus trap (Tab cycles within modal)
- [ ] Scroll lock on body when modal open
- [ ] ARIA attributes (role="dialog", aria-labelledby, aria-modal)
- [ ] Animation on open/close

#### Step 5: Toast/Notification Components

```css
.toast-container {
  /* Positioning */
  position: fixed;
  top: var(--space-6);
  right: var(--space-6);
  z-index: 2000;

  /* Layout */
  display: flex;
  flex-direction: column;
  gap: var(--space-3);

  /* Maximum width */
  max-width: 420px;
}

.toast {
  /* Layout */
  display: flex;
  align-items: center;
  gap: var(--space-4);

  /* Spacing */
  padding: var(--space-4);

  /* Visual */
  background-color: white;
  border-radius: 8px;
  border-left: 4px solid;
  box-shadow: 0 10px 15px rgba(0, 0, 0, 0.1),
              0 4px 6px rgba(0, 0, 0, 0.05);

  /* Animation */
  animation: slide-in var(--duration-base) var(--ease-out);
}

.toast-success {
  border-left-color: var(--color-success);
}

.toast-error {
  border-left-color: var(--color-error);
}

.toast-warning {
  border-left-color: var(--color-warning);
}

.toast-info {
  border-left-color: var(--color-info);
}

.toast-icon {
  flex-shrink: 0;
  width: 24px;
  height: 24px;
}

.toast-content {
  flex: 1;
}

.toast-title {
  font-weight: var(--font-weight-semibold);
  color: var(--color-text-primary);
  margin-bottom: var(--space-1);
}

.toast-message {
  font-size: var(--font-size-sm);
  color: var(--color-text-secondary);
}

.toast-close {
  flex-shrink: 0;
  background: none;
  border: none;
  padding: var(--space-1);
  cursor: pointer;
  color: var(--color-text-secondary);
  transition: var(--transition-colors);
}

.toast-close:hover {
  color: var(--color-text-primary);
}
```

**Toast Component Checklist:**

- [ ] Success, error, warning, info variants
- [ ] Icon indicating status type
- [ ] Auto-dismiss after timeout (5 seconds default)
- [ ] Manual dismiss button
- [ ] Stackable notifications
- [ ] ARIA live region (aria-live="polite" or "assertive")
- [ ] Smooth entrance/exit animations

---

### Phase 3: Accessibility Implementation

#### ARIA Attributes

```html
<!-- Button with ARIA -->
<button
  type="button"
  aria-label="Close dialog"
  aria-pressed="false">
  <span aria-hidden="true">×</span>
</button>

<!-- Form with ARIA -->
<div class="form-group">
  <label for="email" class="form-label">
    Email Address
    <span class="form-label-required" aria-hidden="true">*</span>
  </label>
  <input
    type="email"
    id="email"
    class="form-input"
    aria-required="true"
    aria-invalid="false"
    aria-describedby="email-error">
  <span id="email-error" class="form-error" role="alert">
    Please enter a valid email address
  </span>
</div>

<!-- Modal with ARIA -->
<div
  class="modal-overlay"
  role="dialog"
  aria-modal="true"
  aria-labelledby="modal-title"
  aria-describedby="modal-description">
  <div class="modal-content">
    <h2 id="modal-title" class="modal-title">Confirmation</h2>
    <p id="modal-description" class="modal-body">Are you sure?</p>
  </div>
</div>

<!-- Toast with ARIA -->
<div
  class="toast toast-success"
  role="alert"
  aria-live="polite"
  aria-atomic="true">
  <div class="toast-content">
    <div class="toast-title">Success!</div>
    <div class="toast-message">Profile saved successfully</div>
  </div>
</div>
```

#### Keyboard Navigation

```javascript
// Focus trap for modal
function trapFocus(element) {
  const focusableElements = element.querySelectorAll(
    'button, [href], input, select, textarea, [tabindex]:not([tabindex="-1"])'
  );
  const firstFocusable = focusableElements[0];
  const lastFocusable = focusableElements[focusableElements.length - 1];

  element.addEventListener('keydown', (e) => {
    if (e.key === 'Tab') {
      if (e.shiftKey && document.activeElement === firstFocusable) {
        e.preventDefault();
        lastFocusable.focus();
      } else if (!e.shiftKey && document.activeElement === lastFocusable) {
        e.preventDefault();
        firstFocusable.focus();
      }
    }

    if (e.key === 'Escape') {
      closeModal();
    }
  });

  // Focus first element
  firstFocusable.focus();
}

// Keyboard navigation for custom dropdown
function handleDropdownKeys(e) {
  const items = dropdown.querySelectorAll('[role="option"]');
  const currentIndex = Array.from(items).indexOf(document.activeElement);

  switch(e.key) {
    case 'ArrowDown':
      e.preventDefault();
      items[(currentIndex + 1) % items.length].focus();
      break;
    case 'ArrowUp':
      e.preventDefault();
      items[(currentIndex - 1 + items.length) % items.length].focus();
      break;
    case 'Enter':
    case ' ':
      e.preventDefault();
      document.activeElement.click();
      break;
    case 'Escape':
      closeDropdown();
      break;
  }
}
```

**Accessibility Checklist:**

- [ ] Semantic HTML (header, nav, main, article, aside, footer)
- [ ] Heading hierarchy (h1 → h2 → h3, no skipping levels)
- [ ] ARIA labels for icon-only buttons
- [ ] ARIA live regions for dynamic content
- [ ] Focus visible on all interactive elements
- [ ] Focus trap in modals
- [ ] Keyboard navigation (Tab, Enter, Space, Escape, Arrow keys)
- [ ] Skip to main content link
- [ ] Color contrast WCAG AA (4.5:1 for text, 3:1 for UI)
- [ ] Screen reader tested (VoiceOver, NVDA, JAWS)

---

### Phase 4: Responsive Design

#### Breakpoints

```css
:root {
  /* Breakpoint values */
  --breakpoint-sm: 640px;
  --breakpoint-md: 768px;
  --breakpoint-lg: 1024px;
  --breakpoint-xl: 1280px;
  --breakpoint-2xl: 1536px;
}

/* Mobile First Media Queries */
/* Base styles are mobile */

/* Tablet and up */
@media (min-width: 768px) {
  .grid {
    grid-template-columns: repeat(2, minmax(0, 1fr));
  }

  .container {
    padding-left: var(--space-6);
    padding-right: var(--space-6);
  }
}

/* Desktop and up */
@media (min-width: 1024px) {
  .grid {
    grid-template-columns: repeat(3, minmax(0, 1fr));
  }

  h1 {
    font-size: var(--font-size-5xl);
  }
}

/* Wide desktop */
@media (min-width: 1280px) {
  .grid-wide {
    grid-template-columns: repeat(4, minmax(0, 1fr));
  }
}
```

**Responsive Design Checklist:**

- [ ] Mobile-first approach (base styles = mobile)
- [ ] Breakpoints chosen based on content, not devices
- [ ] Typography scales at different breakpoints
- [ ] Spacing adjusts for larger screens
- [ ] Grid columns increase at breakpoints
- [ ] Images responsive (max-width: 100%, height: auto)
- [ ] Touch targets minimum 44×44px on mobile
- [ ] Tested on actual devices (not just browser resize)

---

### Phase 5: Performance Optimization

#### CSS Optimization

```css
/* Critical CSS (inline in <head>) */
/* Only include above-the-fold styles */
body {
  font-family: var(--font-primary);
  background-color: var(--color-bg-base);
  color: var(--color-text-primary);
}

.container {
  max-width: var(--container-lg);
  margin: 0 auto;
  padding: var(--space-4);
}

/* Non-critical CSS (external stylesheet) */
/* Load asynchronously or defer */
```

#### Image Optimization

```html
<!-- Responsive images with srcset -->
<img
  src="image-800w.jpg"
  srcset="image-400w.jpg 400w,
          image-800w.jpg 800w,
          image-1200w.jpg 1200w"
  sizes="(max-width: 640px) 100vw,
         (max-width: 1024px) 50vw,
         33vw"
  alt="Descriptive alt text"
  loading="lazy"
  decoding="async">

<!-- Modern image formats with fallback -->
<picture>
  <source srcset="image.avif" type="image/avif">
  <source srcset="image.webp" type="image/webp">
  <img src="image.jpg" alt="Descriptive alt text">
</picture>
```

#### Font Loading

```css
/* Font loading with font-display */
@font-face {
  font-family: 'JetBrains Mono';
  src: url('/fonts/JetBrainsMono-Regular.woff2') format('woff2'),
       url('/fonts/JetBrainsMono-Regular.woff') format('woff');
  font-weight: 400;
  font-style: normal;
  font-display: swap; /* Prevent FOIT (Flash of Invisible Text) */
}

/* Preload critical fonts */
/* Add to <head> */
/*
<link rel="preload"
      href="/fonts/JetBrainsMono-Regular.woff2"
      as="font"
      type="font/woff2"
      crossorigin>
*/
```

**Performance Checklist:**

- [ ] CSS minified and concatenated
- [ ] Critical CSS inlined in head
- [ ] Unused CSS removed (PurgeCSS, uncss)
- [ ] Images optimized (compressed, correct format)
- [ ] Images lazy loaded (loading="lazy")
- [ ] Fonts subset to required characters
- [ ] Font preloaded for critical text
- [ ] JavaScript deferred or async loaded
- [ ] Assets served with compression (gzip/brotli)
- [ ] CDN used for static assets

---

## Documentation Standards

### Component Documentation Template

````markdown
# Component Name

## Purpose

Brief description of what this component does and when to use it.

## Variants

- **Primary**: Main call-to-action button
- **Secondary**: Alternative actions
- **Tertiary**: Low-priority actions

## States

- Default
- Hover
- Focus
- Active/Pressed
- Disabled
- Loading

## Accessibility

- ARIA attributes used: `aria-label`, `aria-pressed`
- Keyboard support: Enter, Space to activate
- Focus visible: 2px outline with offset

## Usage Example

```html
<button class="btn btn-primary" type="button">
  Click me
</button>
```

## Props/Attributes

| Attribute | Type | Default | Description |
|-----------|------|---------|-------------|
| type | string | "button" | Button type (button, submit, reset) |
| disabled | boolean | false | Disables the button |
| aria-label | string | - | Accessible label for screen readers |

## Design Tokens Used

- `--color-primary`
- `--color-primary-dark`
- `--font-weight-semibold`
- `--space-3`, `--space-6`
- `--transition-base`

## Browser Support

- Chrome 90+
- Firefox 88+
- Safari 14+
- Edge 90+

## Related Components

- Link Button
- Icon Button
- Button Group

````

---

## Design System Maintenance

### Version Control for Design

```bash
# Semantic versioning for design system
MAJOR.MINOR.PATCH

# MAJOR: Breaking changes (color palette overhaul, component removal)
# MINOR: New features (new components, new variants)
# PATCH: Bug fixes, small improvements

# Example changelog
v2.1.3:
- PATCH: Fixed button focus outline in Safari
- PATCH: Improved toast close button contrast

v2.1.0:
- MINOR: Added new Toggle component
- MINOR: Added dark mode variants for all components

v2.0.0:
- MAJOR: Updated primary color from blue to coral
- MAJOR: Removed deprecated .btn-outline class
```

### Design Token Management

```json
// tokens.json - Single source of truth
{
  "color": {
    "primary": {
      "value": "#E85D4A",
      "type": "color",
      "description": "Primary brand color for CTAs and highlights"
    },
    "primary-dark": {
      "value": "#d14935",
      "type": "color",
      "description": "Darker variant for hover states"
    }
  },
  "spacing": {
    "4": {
      "value": "1rem",
      "type": "spacing",
      "description": "Base spacing unit (16px)"
    }
  },
  "typography": {
    "font-size": {
      "base": {
        "value": "1rem",
        "type": "fontSize",
        "description": "Body text size"
      }
    }
  }
}
```

---

## Anti-Patterns to Avoid

❌ **DON'T:**

- Use inline styles (violates separation of concerns)
- Use `!important` (indicates specificity issues)
- Use magic numbers (hardcode values without semantic meaning)
- Create one-off components (violates DRY principle)
- Skip accessibility (excludes users, legal risk)
- Use low-contrast colors (fails WCAG standards)
- Ignore focus states (breaks keyboard navigation)
- Use divs for buttons (semantic HTML matters)
- Hardcode colors in components (breaks theming)
- Use animations without considering reduced-motion
- Skip responsive testing on real devices
- Create components without documentation

✅ **DO:**

- Use CSS classes and design tokens
- Increase specificity or restructure CSS
- Use semantic design tokens (var(--space-4))
- Create reusable, composable components
- Test with screen readers and keyboard
- Ensure 4.5:1 contrast minimum
- Style :focus and :focus-visible states
- Use `<button>` for buttons, `<a>` for links
- Reference design tokens (var(--color-primary))
- Respect prefers-reduced-motion media query
- Test on iOS, Android, various screen sizes
- Document every component thoroughly

---

## Tools & Commands Reference

### Design Tools

- **Figma**: Design mockups, component library, prototyping
- **Sketch**: Vector design, symbols, design systems
- **Adobe XD**: Design, prototyping, collaboration

### Development Tools

```bash
# CSS preprocessing
sass styles.scss styles.css           # Compile Sass
postcss styles.css -o dist/styles.css # PostCSS processing

# CSS optimization
purgecss --css styles.css --content index.html --output dist/  # Remove unused CSS
cssnano styles.css dist/styles.min.css                         # Minify CSS

# Accessibility testing
axe-core index.html                   # Automated accessibility testing
pa11y http://localhost:3000           # Accessibility audit

# Performance testing
lighthouse http://localhost:3000      # Performance, accessibility, SEO audit
webpack-bundle-analyzer              # Analyze bundle size

# Visual regression testing
backstop test                         # Visual regression testing
percy snapshot                        # Visual diff across browsers
```

### Browser DevTools

- **Chrome DevTools**: Inspect, Lighthouse, Coverage
- **Firefox Developer Edition**: Accessibility Inspector, Grid Inspector
- **Safari Web Inspector**: Responsive Design Mode, Timeline

---

## Success Metrics

After following this guide, you should have:

✅ **Design System** - Comprehensive token library (colors, typography, spacing)
✅ **Component Library** - Reusable, documented components
✅ **Accessibility** - WCAG AA compliant, keyboard navigable, screen reader tested
✅ **Responsiveness** - Mobile-first, tested on real devices
✅ **Performance** - Optimized assets, lazy loading, critical CSS
✅ **Documentation** - Every component documented with examples
✅ **Consistency** - Predictable naming, unified visual language
✅ **Maintainability** - Version controlled, single source of truth

---

## Troubleshooting

### CSS Specificity Issues

```css
/* Problem: Styles not applying due to specificity */
/* ❌ Bad: Using !important */
.button {
  color: blue !important;
}

/* ✅ Good: Increase specificity correctly */
.card .button {
  color: blue;
}

/* ✅ Better: Use BEM naming to avoid conflicts */
.card__button {
  color: blue;
}
```

### Color Contrast Failures

```bash
# Check contrast ratio
# Use online tools:
# - WebAIM Contrast Checker
# - Contrast Ratio by Lea Verou
# - Chrome DevTools (Inspect element → Color picker shows contrast ratio)

# Formula: (L1 + 0.05) / (L2 + 0.05)
# L1 = lighter color luminance, L2 = darker color luminance
# WCAG AA: 4.5:1 for normal text, 3:1 for large text (18pt+)
```

### Font Loading Issues (FOUT/FOIT)

```css
/* FOUT: Flash of Unstyled Text (system font appears first) */
/* FOIT: Flash of Invisible Text (text hidden until font loads) */

/* Solution: Use font-display */
@font-face {
  font-family: 'CustomFont';
  src: url('font.woff2') format('woff2');
  font-display: swap; /* Shows fallback immediately, swaps when loaded */
}

/* Alternative: Preload critical fonts */
/* <link rel="preload" href="font.woff2" as="font" type="font/woff2" crossorigin> */
```

---

**Remember:** Good design is invisible. Users should feel the thoughtfulness in every interaction without noticing the individual design decisions. Consistency, accessibility, and performance create trust and delight.

**When in doubt, test with real users!**
