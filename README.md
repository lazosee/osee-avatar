# @osee/avatar

A lightweight, zero-dependency, deterministic 512×512 SVG avatar generator designed for Node.js, modern browsers, and edge environments like Cloudflare Workers.

Given the same seed string, `@osee/avatar` always renders the exact same SVG profile picture using a 32-bit FNV-1a hash and a Mulberry32 PRNG.

---

## Features

- **Zero Runtime Dependencies:** Pure string-interpolated SVG generator.
- **Edge-Ready:** Executes in `<0.5ms` per render inside Cloudflare Workers, Vercel Edge Functions, or AWS Lambda@Edge.
- **Deterministic:** The exact same seed phrase produces the exact same avatar every time.
- **Layered Profile Anatomy:** Dynamic generation of heads, hairstyles (Afro/Bob, Pompadour, Beanie, Swept Side-Part), facial hair (Full Beard, Goatee, Chevron Moustache), spectacles, torsos, and adaptive color palettes.
- **Dual Module Exports:** Full ESM and CommonJS support with TypeScript declaration files.

---

## Installation

```bash
npm install @osee/avatar
# or
pnpm add @osee/avatar
# or
yarn add @osee/avatar
```

## Usage
### Basic Usage
```ts
import { generateAvatar } from "@osee/avatar";

const svg = generateAvatar("lazaro@example.com");
console.log(svg); // Output: <svg width="512" height="512" ...>
```

### In a Cloudflare Worker
Serve dynamic avatars just-in-time directly over an edge endpoint:

```ts
import { generateAvatar } from "@osee/avatar";

export default {
  async fetch(request: Request): Promise<Response> {
    const url = new URL(request.url);
    const seed = url.searchParams.get("seed") || url.pathname.slice(1) || "guest";

    const svg = generateAvatar(seed);

    return new Response(svg, {
      headers: {
        "Content-Type": "image/svg+xml",
        "Cache-Control": "public, max-age=31536000, immutable",
      },
    });
  },
};
```

### Using Custom Palettes
You can supply your own palette configurations:
```ts
import { generateAvatar, type AvatarPalette } from "@osee/avatar";

const customPalettes: AvatarPalette[] = [
  {
    bg: "#1E293B",
    skin: "#FCD34D",
    clothes: "#6366F1",
    hair: "#0F172A",
    frame: "#475569",
  },
];

const svg = generateAvatar("custom-seed", customPalettes);
```

### Low-Level Utilities
You can also use the hashing and PRNG utilities independently:
```ts
import { fnv1a, mulberry32 } from "@osee/avatar";

const seedInt = fnv1a("any-string");
const rand = mulberry32(seedInt);

console.log(rand()); // Random float in [0, 1)
```

## API Reference
### (I). `generateAvatar(seed: string, palettes?: AvatarPalette[]): string`

Generates an SVG avatar formatted to a 512×512 circular profile frame.

`seed`: Any string (e.g. user ID, username, email).

`palettes (optional)`: Array of AvatarPalette objects. Defaults to `DEFAULT_PALETTES`.

`Returns`: Valid SVG XML string.


### (II). `AvatarPalette`
```ts
interface AvatarPalette {
  bg: string;      // Circular background fill
  skin: string;    // Head and neck fill
  clothes: string; // Torso and beanie fill
  hair: string;    // Hair and facial hair fill
  frame: string;   // Spectacle rims and accessories
}
```

---
## License
MIT &copy; Lazaro Osee

---

### Source Files (`src/`)

#### `src/types.ts`
```typescript
export interface AvatarPalette {
  bg: string;
  skin: string;
  clothes: string;
  hair: string;
  frame: string;
}
```

#### `src/functions.ts`
```ts
// 32-bit FNV-1a Hash to turn any string into an initial integer seed
export function fnv1a(str: string): number {
  let hash = 2166136261;
  for (let i = 0; i < str.length; i++) {
    hash ^= str.charCodeAt(i);
    hash = Math.imul(hash, 16777619);
  }
  return hash >>> 0;
}


// Fast Mulberry32 PRNG returning deterministic floats in [0, 1)
export function mulberry32(a: number): () => number {
  return function () {
    let t = (a += 0x6d2b79f5);
    t = Math.imul(t ^ (t >>> 15), t | 1);
    t ^= t + Math.imul(t ^ (t >>> 7), t | 61);
    return ((t ^ (t >>> 14)) >>> 0) / 4294967296;
  };
}
```

#### `src/generator.ts`
```ts
import type { AvatarPalette } from "./types.js";
import { fnv1a, mulberry32 } from "./functions.js";

export const DEFAULT_PALETTES: AvatarPalette[] = [
  { bg: "#F472B6", skin: "#FBBF24", clothes: "#4F46E5", hair: "#1E1B4B", frame: "#111827" },
  { bg: "#60A5FA", skin: "#FCD34D", clothes: "#059669", hair: "#7C2D12", frame: "#D97706" },
  { bg: "#34D399", skin: "#FBCFE8", clothes: "#D97706", hair: "#312E81", frame: "#1E293B" },
  { bg: "#A78BFA", skin: "#FED7AA", clothes: "#DC2626", hair: "#111827", frame: "#4B5563" },
  { bg: "#F87171", skin: "#FDE68A", clothes: "#2563EB", hair: "#374151", frame: "#0F172A" },
  { bg: "#38BDF8", skin: "#E2E8F0", clothes: "#4338CA", hair: "#0F172A", frame: "#94A3B8" },
];

export function generateAvatar(seed: string, palettes: AvatarPalette[] = DEFAULT_PALETTES): string {
  const rand = mulberry32(fnv1a(seed));
  const pick = <T>(arr: T[]): T => arr[Math.floor(rand() * arr.length)];
  const range = (min: number, max: number): number => min + rand() * (max - min);

  const palette = pick(palettes);

  // --- Core Geometry ---
  const R = range(106, 118);
  const Y = range(215, 228);
  const eyeOffsetY = Y - range(4, 10);
  const eyeSpacing = range(36, 44);
  const eyeRadius = range(6, 8.5);
  const mouthY = Y + range(48, 54);

  // --- 1. Facial Hair & Integrated Mouth ---
  const hasFacialHair = rand() < 0.45;
  const beardType = Math.floor(rand() * 3); // 0: Full Beard, 1: Goatee, 2: Chevron Moustache
  let facialHairSvg = "";
  let mouthSvg = "";

  const defaultSmileCurve = range(8, 14);

  if (!hasFacialHair) {
    mouthSvg = `<path d="M ${256 - 22} ${mouthY} Q 256 ${mouthY + defaultSmileCurve} ${256 + 22} ${mouthY}" stroke="#1F2937" stroke-width="4.5" stroke-linecap="round" fill="none" />`;
  } else if (beardType === 0) {
    facialHairSvg = `
      <path d="
        M ${256 - R + 4} ${Y + 12}
        C ${256 - R + 2} ${Y + R * 0.75}, ${256 - R * 0.55} ${Y + R + 42}, 256 ${Y + R + 46}
        C ${256 + R * 0.55} ${Y + R + 42}, ${256 + R - 2} ${Y + R * 0.75}, ${256 + R - 4} ${Y + 12}
        L ${256 + R - 14} ${Y + 12}
        C ${256 + R - 16} ${Y + 45}, ${256 + 48} ${mouthY + 28}, 256 ${mouthY + 28}
        C ${256 - 48} ${mouthY + 28}, ${256 - R + 16} ${Y + 45}, ${256 - R + 14} ${Y + 12}
        Z"
        fill="${palette.hair}" />
    `;
    mouthSvg = `<path d="M ${256 - 18} ${mouthY} Q 256 ${mouthY + 10} ${256 + 18} ${mouthY}" stroke="#1F2937" stroke-width="4" stroke-linecap="round" fill="none" />`;
  } else if (beardType === 1) {
    facialHairSvg = `
      <path fill-rule="evenodd" d="
        M 256 ${mouthY - 16}
        C 285 ${mouthY - 16}, 295 ${mouthY + 10}, 288 ${Y + R + 14}
        C 282 ${Y + R + 32}, 270 ${Y + R + 38}, 256 ${Y + R + 38}
        C 242 ${Y + R + 38}, 230 ${Y + R + 32}, 224 ${Y + R + 14}
        C 217 ${mouthY + 10}, 227 ${mouthY - 16}, 256 ${mouthY - 16}
        Z
        M 256 ${mouthY - 6}
        C 240 ${mouthY - 6}, 238 ${mouthY + 20}, 256 ${mouthY + 20}
        C 274 ${mouthY + 20}, 272 ${mouthY - 6}, 256 ${mouthY - 6}
        Z"
        fill="${palette.hair}" />
    `;
    mouthSvg = `<path d="M ${256 - 15} ${mouthY + 4} Q 256 ${mouthY + 12} ${256 + 15} ${mouthY + 4}" stroke="#1F2937" stroke-width="3.5" stroke-linecap="round" fill="none" />`;
  } else {
    facialHairSvg = `
      <path d="
        M 256 ${mouthY - 7}
        C 248 ${mouthY - 15}, 235 ${mouthY - 15}, 222 ${mouthY - 3}
        C 220 ${mouthY + 7}, 234 ${mouthY + 9}, 244 ${mouthY + 3}
        C 252 ${mouthY - 1}, 254 ${mouthY + 2}, 256 ${mouthY + 2}
        C 258 ${mouthY + 2}, 260 ${mouthY - 1}, 268 ${mouthY + 3}
        C 278 ${mouthY + 9}, 292 ${mouthY + 7}, 290 ${mouthY - 3}
        C 277 ${mouthY - 15}, 264 ${mouthY - 15}, 256 ${mouthY - 7}
        Z"
        fill="${palette.hair}" />
    `;
    mouthSvg = `<path d="M ${256 - 20} ${mouthY + 8} Q 256 ${mouthY + 18} ${256 + 20} ${mouthY + 8}" stroke="#1F2937" stroke-width="4.5" stroke-linecap="round" fill="none" />`;
  }

  // --- 2. Glasses Generation ---
  const hasGlasses = rand() < 0.4;
  const glassesType = Math.floor(rand() * 3); // 0: Round, 1: Square, 2: Sunglasses
  let glassesSvg = "";

  if (hasGlasses) {
    const lensRadius = range(22, 26);
    const frameStroke = range(4, 5.5);
    const bridgeY = eyeOffsetY - 1;

    if (glassesType === 0) {
      glassesSvg = `
        <g stroke="${palette.frame}" stroke-width="${frameStroke}" fill="none">
          <circle cx="${256 - eyeSpacing}" cy="${eyeOffsetY}" r="${lensRadius}" />
          <circle cx="${256 + eyeSpacing}" cy="${eyeOffsetY}" r="${lensRadius}" />
          <line x1="${256 - eyeSpacing + lensRadius}" y1="${bridgeY}" x2="${256 + eyeSpacing - lensRadius}" y2="${bridgeY}" />
          <line x1="${256 - eyeSpacing - lensRadius}" y1="${bridgeY}" x2="${256 - R + 6}" y2="${bridgeY}" />
          <line x1="${256 + eyeSpacing + lensRadius}" y1="${bridgeY}" x2="${256 + R - 6}" y2="${bridgeY}" />
        </g>
      `;
    } else if (glassesType === 1) {
      const w = lensRadius * 1.9;
      const h = lensRadius * 1.5;
      glassesSvg = `
        <g stroke="${palette.frame}" stroke-width="${frameStroke}" fill="none">
          <rect x="${256 - eyeSpacing - w / 2}" y="${eyeOffsetY - h / 2}" width="${w}" height="${h}" rx="6" />
          <rect x="${256 + eyeSpacing - w / 2}" y="${eyeOffsetY - h / 2}" width="${w}" height="${h}" rx="6" />
          <line x1="${256 - eyeSpacing + w / 2}" y1="${bridgeY}" x2="${256 + eyeSpacing - w / 2}" y2="${bridgeY}" />
          <line x1="${256 - eyeSpacing - w / 2}" y1="${bridgeY}" x2="${256 - R + 6}" y2="${bridgeY}" />
          <line x1="${256 + eyeSpacing + w / 2}" y1="${bridgeY}" x2="${256 + R - 6}" y2="${bridgeY}" />
        </g>
      `;
    } else {
      const w = lensRadius * 2.1;
      const h = lensRadius * 1.7;
      glassesSvg = `
        <g stroke="${palette.frame}" stroke-width="${frameStroke}" fill="#18181B" opacity="0.96">
          <rect x="${256 - eyeSpacing - w / 2}" y="${eyeOffsetY - h / 2}" width="${w}" height="${h}" rx="8" />
          <rect x="${256 + eyeSpacing - w / 2}" y="${eyeOffsetY - h / 2}" width="${w}" height="${h}" rx="8" />
          <line x1="${256 - eyeSpacing + w / 2}" y1="${bridgeY}" x2="${256 + eyeSpacing - w / 2}" y2="${bridgeY}" stroke-width="${frameStroke + 2}" />
          <line x1="${256 - eyeSpacing - 8}" y1="${eyeOffsetY - 6}" x2="${256 - eyeSpacing + 8}" y2="${eyeOffsetY - 12}" stroke="#FFFFFF" stroke-width="2.5" opacity="0.5" />
          <line x1="${256 + eyeSpacing - 8}" y1="${eyeOffsetY - 6}" x2="${256 + eyeSpacing + 8}" y2="${eyeOffsetY - 12}" stroke="#FFFFFF" stroke-width="2.5" opacity="0.5" />
        </g>
      `;
    }
  }

  // --- 3. Hair Styles ---
  const hairStyle = Math.floor(rand() * 4); // 0: Undercut/Pompadour, 1: Afro/Bob, 2: Fitted Beanie, 3: Side-Part Swoop
  let hairSvg = "";

  if (hairStyle === 0) {
    hairSvg = `
      <path d="M ${256 - R - 2} ${Y + 8}
               C ${256 - R - 6} ${Y - 45}, ${256 - R * 0.7} ${Y - R - 28}, 256 ${Y - R - 32}
               C ${256 + R * 0.7} ${Y - R - 28}, ${256 + R + 6} ${Y - 45}, ${256 + R + 2} ${Y + 8}
               L ${256 + R - 6} ${Y + 8}
               C ${256 + R - 2} ${Y - 30}, ${256 + R * 0.6} ${Y - R * 0.65}, 256 ${Y - R * 0.68}
               C ${256 - R * 0.6} ${Y - R * 0.65}, ${256 - R + 2} ${Y - 30}, ${256 - R + 6} ${Y + 8}
               Z"
            fill="${palette.hair}" />
    `;
  } else if (hairStyle === 1) {
    hairSvg = `<circle cx="256" cy="${Y - 14}" r="${R + 24}" fill="${palette.hair}" />`;
  } else if (hairStyle === 2) {
    hairSvg = `
      <g>
        <path d="M ${256 - R - 4} ${Y - 20}
                 C ${256 - R - 4} ${Y - R - 50}, ${256 + R + 4} ${Y - R - 50}, ${256 + R + 4} ${Y - 20}
                 Z"
              fill="${palette.clothes}" />
        <rect x="${256 - R - 8}" y="${Y - R * 0.55}" width="${(R + 8) * 2}" height="32" rx="10" fill="${palette.clothes}" filter="brightness(0.9)" />
        <line x1="240" y1="${Y - R * 0.55 + 4}" x2="240" y2="${Y - R * 0.55 + 28}" stroke="#000000" stroke-width="2" opacity="0.15" />
        <line x1="256" y1="${Y - R * 0.55 + 4}" x2="256" y2="${Y - R * 0.55 + 28}" stroke="#000000" stroke-width="2" opacity="0.15" />
        <line x1="272" y1="${Y - R * 0.55 + 4}" x2="272" y2="${Y - R * 0.55 + 28}" stroke="#000000" stroke-width="2" opacity="0.15" />
      </g>
    `;
  } else {
    hairSvg = `
      <path d="M ${256 - R - 4} ${Y + 4}
               C ${256 - R - 8} ${Y - R * 0.75}, ${256 - R * 0.2} ${Y - R - 26}, 256 ${Y - R - 24}
               C ${256 + R * 0.7} ${Y - R - 20}, ${256 + R + 8} ${Y - R * 0.5}, ${256 + R + 4} ${Y + 4}
               C ${256 + R - 4} ${Y - 10}, ${256 + R * 0.5} ${Y - R * 0.45}, 256 ${Y - R * 0.5}
               C ${256 - R * 0.2} ${Y - R * 0.5}, ${256 - R * 0.6} ${Y - R * 0.15}, ${256 - R + 2} ${Y + 4}
               Z"
            fill="${palette.hair}" />
    `;
  }

  // --- Final Assembly ---
  return `
<svg width="512" height="512" viewBox="0 0 512 512" xmlns="http://www.w3.org/2000/svg">
  <defs>
    <clipPath id="circleClip">
      <circle cx="256" cy="256" r="240" />
    </clipPath>
  </defs>

  <circle cx="256" cy="256" r="240" fill="${palette.bg}" />

  <g clip-path="url(#circleClip)">
    ${hairStyle === 1 ? hairSvg : ""}

    <!-- Neck -->
    <rect x="218" y="${Y + 40}" width="76" height="140" rx="14" fill="${palette.skin}" />
    <path d="M 218 ${Y + 65} Q 256 ${Y + 95} 294 ${Y + 65} L 294 ${Y + 80} Q 256 ${Y + 110} 218 ${Y + 80} Z" fill="#000000" opacity="0.12" />

    <!-- Shoulders / Torso -->
    <path d="M 72 512 C 78 375, 185 360, 256 360 C 327 360, 434 375, 440 512 Z" fill="${palette.clothes}" />

    <!-- Head -->
    <circle cx="256" cy="${Y}" r="${R}" fill="${palette.skin}" />

    <!-- Facial Hair & Mouth -->
    ${facialHairSvg}
    ${mouthSvg}

    <!-- Eyes -->
    <circle cx="${256 - eyeSpacing}" cy="${eyeOffsetY}" r="${eyeRadius}" fill="#1F2937" />
    <circle cx="${256 + eyeSpacing}" cy="${eyeOffsetY}" r="${eyeRadius}" fill="#1F2937" />

    <!-- Eyebrows -->
    <path d="M ${256 - eyeSpacing - 14} ${eyeOffsetY - 15} Q ${256 - eyeSpacing} ${eyeOffsetY - 21} ${256 - eyeSpacing + 14} ${eyeOffsetY - 14}" stroke="${palette.hair}" stroke-width="4" stroke-linecap="round" fill="none" />
    <path d="M ${256 + eyeSpacing - 14} ${eyeOffsetY - 14} Q ${256 + eyeSpacing} ${eyeOffsetY - 21} ${256 + eyeSpacing + 14} ${eyeOffsetY - 15}" stroke="${palette.hair}" stroke-width="4" stroke-linecap="round" fill="none" />

    <!-- Glasses -->
    ${glassesSvg}

    <!-- Front Hair Styles -->
    ${hairStyle !== 1 ? hairSvg : ""}
  </g>

  <circle cx="256" cy="256" r="240" fill="none" stroke="#FFFFFF" stroke-width="12" opacity="0.35" />
</svg>`.trim();
}
```

#### `src/index.ts`
```ts
export * from "./types.js";
export * from "./functions.js";
export * from "./generator.js";
```

---

