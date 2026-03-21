---
title: Web Components in EDS
impact: MEDIUM
impactDescription: Web Components provide encapsulation for reusable widgets — but overuse adds complexity and breaks the simplicity that makes EDS fast
tags: eds, web-components, custom-elements, shadow-dom, slots, templates, encapsulation, vanilla-js
---

## Web Components in EDS

Web Components (Custom Elements + Shadow DOM) provide true encapsulation for reusable UI widgets in EDS. Use them when you need style isolation or cross-site portability — but prefer plain vanilla JS blocks for most EDS development.

---

### 1. When to Use Web Components vs Vanilla JS Blocks

| Scenario | Vanilla JS Block | Web Component |
|----------|-----------------|---------------|
| Standard page blocks (hero, cards, etc.) | **Preferred** | Overkill |
| Reusable widget shared across sites | Good | **Preferred** |
| Third-party embeddable widget | Not possible | **Required** |
| Style isolation needed | Difficult | **Built-in** |
| Complex interactive component (date picker, color picker) | Possible | **Preferred** |
| Simple content display | **Preferred** | Overkill |

**Rule of thumb:** Use vanilla JS blocks for 90% of EDS development. Reach for Web Components only when you need encapsulation or cross-site reuse.

---

### 2. Basic Custom Element

```javascript
// blocks/video-player/video-player.js
class VideoPlayer extends HTMLElement {
  constructor() {
    super();
    this.attachShadow({ mode: 'open' });
  }

  static get observedAttributes() {
    return ['src', 'poster', 'autoplay'];
  }

  connectedCallback() {
    this.render();
    this.setupEvents();
  }

  disconnectedCallback() {
    // Clean up event listeners, timers, observers
    this.cleanup();
  }

  attributeChangedCallback(name, oldValue, newValue) {
    if (oldValue !== newValue) {
      this.render();
    }
  }

  render() {
    const src = this.getAttribute('src');
    const poster = this.getAttribute('poster');

    this.shadowRoot.innerHTML = `
      <style>
        :host {
          display: block;
          position: relative;
          aspect-ratio: 16 / 9;
        }
        video {
          width: 100%;
          height: 100%;
          object-fit: cover;
          border-radius: 8px;
        }
        .controls {
          position: absolute;
          bottom: 0;
          width: 100%;
          padding: 12px;
          background: linear-gradient(transparent, rgba(0,0,0,0.7));
          display: flex;
          gap: 8px;
          align-items: center;
        }
        button {
          background: none;
          border: none;
          color: white;
          cursor: pointer;
          font-size: 1.2rem;
        }
        .progress {
          flex: 1;
          height: 4px;
          background: rgba(255,255,255,0.3);
          border-radius: 2px;
          cursor: pointer;
        }
        .progress-bar {
          height: 100%;
          background: white;
          border-radius: 2px;
          width: 0%;
          transition: width 0.1s;
        }
      </style>
      <video src="${src}" poster="${poster || ''}" playsinline preload="metadata"></video>
      <div class="controls">
        <button class="play-btn" aria-label="Play">&#9654;</button>
        <div class="progress"><div class="progress-bar"></div></div>
        <button class="mute-btn" aria-label="Mute">&#128266;</button>
      </div>
    `;
  }

  setupEvents() {
    const video = this.shadowRoot.querySelector('video');
    const playBtn = this.shadowRoot.querySelector('.play-btn');
    const progressBar = this.shadowRoot.querySelector('.progress-bar');

    playBtn.addEventListener('click', () => {
      if (video.paused) {
        video.play();
        playBtn.textContent = '⏸';
      } else {
        video.pause();
        playBtn.textContent = '▶';
      }
    });

    video.addEventListener('timeupdate', () => {
      const pct = (video.currentTime / video.duration) * 100;
      progressBar.style.width = `${pct}%`;
    });
  }

  cleanup() {
    // Remove observers, abort controllers, etc.
  }
}

customElements.define('video-player', VideoPlayer);
```

---

### 3. Using Web Components in EDS Blocks

```javascript
// blocks/video/video.js — EDS block that creates a Web Component
export default function decorate(block) {
  const rows = [...block.children];
  const videoSrc = rows[0]?.querySelector('a')?.href;
  const posterSrc = rows[1]?.querySelector('img')?.src;

  // Clear block and replace with Web Component
  block.textContent = '';

  const player = document.createElement('video-player');
  player.setAttribute('src', videoSrc);
  if (posterSrc) player.setAttribute('poster', posterSrc);

  block.append(player);
}
```

Lazy-load the Web Component definition:

```javascript
// blocks/video/video.js
export default async function decorate(block) {
  // Dynamic import — only loads when block is on the page
  await import('./video-player.js');

  const player = document.createElement('video-player');
  // ... setup
  block.append(player);
}
```

---

### 4. Shadow DOM and Styling

#### CSS Custom Properties for Theming

Shadow DOM blocks external styles, but CSS custom properties pierce through:

```javascript
// Inside Web Component
this.shadowRoot.innerHTML = `
  <style>
    :host {
      --wc-primary-color: var(--color-primary, #0066cc);
      --wc-font-family: var(--body-font-family, sans-serif);
    }
    .title {
      color: var(--wc-primary-color);
      font-family: var(--wc-font-family);
    }
  </style>
  <h2 class="title"><slot name="title"></slot></h2>
`;
```

```css
/* EDS styles.css — theme the Web Component from outside */
video-player {
  --color-primary: #ff6600;
  --body-font-family: 'Custom Font', sans-serif;
}
```

#### ::part() for Targeted Styling

```javascript
// Expose parts for external styling
this.shadowRoot.innerHTML = `
  <div part="container">
    <h2 part="title"><slot name="title"></slot></h2>
    <p part="description"><slot></slot></p>
  </div>
`;
```

```css
/* External CSS can style exposed parts */
my-card::part(title) {
  font-size: 1.5rem;
  color: var(--heading-color);
}

my-card::part(container) {
  padding: 1rem;
  border-radius: 8px;
}
```

---

### 5. Slots for Content Projection

```javascript
class InfoCard extends HTMLElement {
  constructor() {
    super();
    this.attachShadow({ mode: 'open' });
    this.shadowRoot.innerHTML = `
      <style>
        :host { display: block; border: 1px solid #e0e0e0; border-radius: 8px; padding: 1rem; }
        .header { display: flex; align-items: center; gap: 0.5rem; margin-bottom: 0.5rem; }
        ::slotted(img) { width: 48px; height: 48px; border-radius: 50%; }
        ::slotted(h3) { margin: 0; }
      </style>
      <div class="header">
        <slot name="icon"></slot>
        <slot name="title"></slot>
      </div>
      <slot></slot>
    `;
  }
}
customElements.define('info-card', InfoCard);
```

Usage in HTML:

```html
<info-card>
  <img slot="icon" src="/icons/star.svg" alt="">
  <h3 slot="title">Featured</h3>
  <p>This is the default slot content — description text.</p>
</info-card>
```

---

### 6. Accessibility in Shadow DOM

```javascript
class ToggleButton extends HTMLElement {
  constructor() {
    super();
    this.attachShadow({ mode: 'open' });
    this.pressed = false;
  }

  connectedCallback() {
    this.shadowRoot.innerHTML = `
      <style>
        button { padding: 8px 16px; border-radius: 4px; cursor: pointer; }
        button[aria-pressed="true"] { background: var(--color-primary, #0066cc); color: white; }
      </style>
      <button role="button" aria-pressed="false" tabindex="0">
        <slot></slot>
      </button>
    `;

    const btn = this.shadowRoot.querySelector('button');
    btn.addEventListener('click', () => this.toggle());
    btn.addEventListener('keydown', (e) => {
      if (e.key === 'Enter' || e.key === ' ') {
        e.preventDefault();
        this.toggle();
      }
    });
  }

  toggle() {
    this.pressed = !this.pressed;
    const btn = this.shadowRoot.querySelector('button');
    btn.setAttribute('aria-pressed', String(this.pressed));

    // Dispatch event that bubbles out of Shadow DOM
    this.dispatchEvent(new CustomEvent('toggle', {
      detail: { pressed: this.pressed },
      bubbles: true,
      composed: true, // crosses Shadow DOM boundary
    }));
  }
}
customElements.define('toggle-button', ToggleButton);
```

---

### 7. State Sharing Between Components

```javascript
// Simple event bus for Web Component communication
class EventBus extends EventTarget {
  emit(name, detail) {
    this.dispatchEvent(new CustomEvent(name, { detail }));
  }

  on(name, callback) {
    this.addEventListener(name, callback);
    return () => this.removeEventListener(name, callback);
  }
}

// Shared instance
export const bus = new EventBus();

// Component A — emits
bus.emit('cart:updated', { count: 3 });

// Component B — listens
bus.on('cart:updated', (e) => {
  this.shadowRoot.querySelector('.count').textContent = e.detail.count;
});
```

---

### 8. Anti-Patterns

#### Web Components for Simple Blocks

```javascript
// WRONG — using Shadow DOM for a simple text block
class SimpleText extends HTMLElement {
  connectedCallback() {
    this.attachShadow({ mode: 'open' });
    this.shadowRoot.innerHTML = `<p>${this.textContent}</p>`;
  }
}

// CORRECT — just use a plain EDS block
export default function decorate(block) {
  const text = block.querySelector('p');
  text.classList.add('highlight');
}
```

#### Heavy Frameworks Inside Web Components

```javascript
// WRONG — importing React/Lit inside a Web Component in EDS
import { html, LitElement } from 'lit';
// Adds 15-20KB+ to the bundle, defeats EDS zero-dependency philosophy

// CORRECT — vanilla JS only, keep it lightweight
```

#### Not Cleaning Up

```javascript
// WRONG — memory leak, listeners keep firing after removal
connectedCallback() {
  window.addEventListener('resize', this.handleResize);
  setInterval(this.poll, 5000);
}

// CORRECT — clean up in disconnectedCallback
connectedCallback() {
  this.controller = new AbortController();
  window.addEventListener('resize', this.handleResize, { signal: this.controller.signal });
  this.intervalId = setInterval(this.poll, 5000);
}

disconnectedCallback() {
  this.controller.abort();
  clearInterval(this.intervalId);
}
```

#### Blocking Render with Custom Element Upgrade

```html
<!-- WRONG — script in head blocks page render -->
<head>
  <script src="/blocks/my-widget/my-widget.js"></script>
</head>

<!-- CORRECT — dynamic import in EDS block, loads only when needed -->
<!-- See section 3 for lazy-loading pattern -->
```
