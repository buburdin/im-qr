# im-qr

A single-file QR code generator that produces clean SVGs with a logo embedded in the center.

![Screenshot](screenshot.png)

## Features

- **URL → SVG QR code** with high error correction (level H), so the center can be safely covered by the logo
- **Three module styles**: dots, rounded squares, classic squares
- **Shimmer animation** for the dots style with tunable speed, intensity, fade, wave, and randomness
- **Download** as `.svg` or **copy** the SVG markup to clipboard
- No build step, no dependencies beyond a single CDN script (`qrcode-generator`)

## Usage

Open `index.html` in a browser. Paste a URL, click **Generate**, pick a style, and optionally enable the shimmer animation.

## How it works

The center logo sits inside a rounded "knockout" area — modules whose centers fall inside that area are skipped during rendering. The QR's level-H error correction handles the missing data. Finder patterns at the three corners are drawn separately with rounding that matches the chosen module style.

The shimmer animation generates one `<circle>` per module and assigns each a negative `animation-delay` derived from a deterministic wave function over its `(row, column)` position, so the pulse flows organically across the code.

## Minimal "square" version

If you just want the square-style QR with a centered logo — no UI, no animation, no other styles — here's a self-contained snippet. Drop it into an HTML file and call `buildSquareQR(url)` to get an SVG string back.

```html
<script src="https://cdn.jsdelivr.net/npm/qrcode-generator@1.4.4/qrcode.min.js"></script>
<script>
const LOGO = {
  width: 134,
  height: 80,
  paths: `<path fill-rule="evenodd" clip-rule="evenodd" d="M46.4105 0H87.0076V79.9971H46.4105V0ZM0 11.4757H40.5971V80.0015H0V11.4757ZM133.427 0H92.8297V79.9971H133.427V0ZM2.54314e-06 0H40.5971V5.73567H2.54314e-06V0Z" fill="black"/>`
};

function buildSquareQR(text) {
  const qr = qrcode(0, 'H');        // level-H error correction → tolerates the logo knockout
  qr.addData(text);
  qr.make();

  const n = qr.getModuleCount();
  const margin = 4;
  const size = n + margin * 2;

  // Center logo + rounded knockout box
  const logoAR = LOGO.width / LOGO.height;
  const logoW = size * 0.26;
  const logoH = logoW / logoAR;
  const pad = 2.8;
  const koW = logoW + pad * 2;
  const koH = logoH + pad * 2;
  const koX = (size - koW) / 2;
  const koY = (size - koH) / 2;
  const koR = Math.min(koW, koH) * 0.18;
  const logoX = (size - logoW) / 2;
  const logoY = (size - logoH) / 2;
  const logoScale = logoW / LOGO.width;

  const inKnockout = (cx, cy) => {
    if (cx < koX || cx > koX + koW || cy < koY || cy > koY + koH) return false;
    const dx = Math.min(cx - koX, koX + koW - cx);
    const dy = Math.min(cy - koY, koY + koH - cy);
    if (dx >= koR || dy >= koR) return true;
    const ddx = koR - dx, ddy = koR - dy;
    return ddx * ddx + ddy * ddy <= koR * koR;
  };

  const inFinder = (r, c) =>
    (r < 7 && c < 7) ||
    (r < 7 && c >= n - 7) ||
    (r >= n - 7 && c < 7);

  const off = 0.04;
  const hasModule = (r, c) => {
    if (r < 0 || c < 0 || r >= n || c >= n) return false;
    if (!qr.isDark(r, c)) return false;
    if (inFinder(r, c)) return false;
    return !inKnockout(c + margin + 0.5, r + margin + 0.5);
  };

  let modulesPath = '';
  for (let r = 0; r < n; r++) {
    for (let c = 0; c < n; c++) {
      if (!hasModule(r, c)) continue;
      const N = hasModule(r - 1, c);
      const E = hasModule(r, c + 1);
      const S = hasModule(r + 1, c);
      const W = hasModule(r, c - 1);
      const left   = c + margin + (W ? 0 : off);
      const right  = c + margin + 1 - (E ? 0 : off);
      const top    = r + margin + (N ? 0 : off);
      const bottom = r + margin + 1 - (S ? 0 : off);
      modulesPath += `M${left},${top}H${right}V${bottom}H${left}z`;
    }
  }

  const finders = [
    [margin, margin],
    [margin + n - 7, margin],
    [margin, margin + n - 7],
  ];
  let finderSvg = '';
  for (const [x, y] of finders) {
    finderSvg += `
      <rect x="${x}" y="${y}" width="7" height="7" fill="#000"/>
      <rect x="${x + 1}" y="${y + 1}" width="5" height="5" fill="#fff"/>
      <rect x="${x + 2}" y="${y + 2}" width="3" height="3" fill="#000"/>`;
  }

  return `<svg xmlns="http://www.w3.org/2000/svg" viewBox="0 0 ${size} ${size}" width="512" height="512">
    <rect width="${size}" height="${size}" fill="#ffffff"/>
    <path d="${modulesPath}" fill="#000000"/>
    ${finderSvg}
    <g transform="translate(${logoX} ${logoY}) scale(${logoScale})">${LOGO.paths}</g>
  </svg>`;
}
</script>
```

### What the algorithm is doing

1. **Level-H error correction** (`qrcode(0, 'H')`) recovers up to ~30% damage — that's what makes covering the center with a logo safe.
2. **Knockout test** (`inKnockout`) — a rounded rect centered on the QR; any module whose center falls inside is skipped.
3. **Finder skip** (`inFinder`) — the three corner positioning squares are drawn separately, so the data loop excludes them.
4. **Logo overlay** — drawn last with a `translate + scale` transform so it sits exactly in the cleared area.

To swap the logo: replace `LOGO.paths` with the `<path>`/`<g>` contents of your SVG and update `width`/`height` to match its `viewBox`.
