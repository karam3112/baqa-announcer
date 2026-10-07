# Baqa death-announcement image maker

A single-page tool that renders death and funeral announcements as 1080×1350 PNG images for the
Telegram channel of Baqa Al-Gharbiyye (<https://t.me/baqaelgharbeya>). The owner, Karam, helps run the
channel and communicates in Arabic. All user-facing copy is Arabic and the page is RTL.

## Files

- `index.html` — the whole app: markup, CSS and one inline script. No build step, no dependencies.
  This is the file to keep working on, and the one served on GitHub Pages.
- `artifact/baqa-announcer.html` — the same app as currently published as a Claude artifact. It is a
  body fragment (Claude wraps it in its own document skeleton) and saves files only through
  `window.claude.use('downloads')`. `index.html` was generated from it by adding the document
  skeleton, a small base reset, and a plain browser download when `window.claude` is absent. It is now a frozen
  reference: it still has the old fonts (IBM Plex / Amiri) and no share button. Do not port edits to it
  unless the owner asks for the artifact again.
- `design/` — the three reference boards from Claude's Design canvas (`*.dc.html`, `canvas.json`) and
  `qr.svg`. The boards need that editor's runtime (`support.js`) and their QR `<img>` points at an
  artifact-only `/_blob/...` path, so they do not render standalone. Treat them as the visual layout
  spec (they still show the old fonts). The real name the owner typed in `Main.dc.html` was replaced with
  `[الاسم الثلاثي]` before the repo went public; never commit real names of the deceased.

## How the image is made

Everything is drawn on an off-screen `<canvas>` (not DOM-to-image), so Arabic shaping is the
browser's own and export is exact. `render(s)` draws three zones:

1. Header band, 255px tall: greeting, title, and the announcement date (e.g. `الأربعاء 7 تشرين الأول 2026`:
   Levantine month names, Latin digits, Gregorian only). The date comes from a date field that defaults to
   today; if it is cleared, the date line is dropped and the title moves back down. Accent is `#1E2423` for a death notice and `#14453B`
   for a funeral notice, so the two are told apart at a glance.
2. Middle, between y=255 and y=1121: built by `buildMiddle(s, k, accent)` as a list of blocks and
   gaps, vertically centred. If the content is taller than the space, `k` (a type-scale factor)
   steps down by 0.03 until it fits (it stops at about 0.55). The name also shrinks on its own to stay on
   one line. `render` returns false when even the smallest scale does not fit, and the page then shows a
   warning asking to shorten the note.
3. Footer: channel line, `t.me/baqaelgharbeya`, and the QR code.

The QR is not an image: `QR` holds the 29×29 module matrix as one hex number per row (error
correction Q, payload `https://t.me/baqaelgharbeya`) and is drawn as 4px squares. Regenerate it
(python `qrcode`, `border=0`) if the link ever changes.

The preview `<img>` is the canvas exported to a blob URL on every change (debounced 120ms), which
also lets phone users long-press to save.

Fonts: Readex Pro for everything, page and image, including the name; Amiri only for the verse. Both
come from Google Fonts. The owner chose Readex Pro over IBM Plex, Noto Sans Arabic, Almarai and Tajawal
after a side-by-side comparison, for legibility to older readers. Small image text was enlarged at the
same time (cell labels 26, note 29, footer 28). Google splits each face into Arabic and Latin files, so
`document.fonts.load` is given a sample with both (the channel link and digits). Otherwise `t.me/...` and
times like `4:30` fall back to Tahoma. The canvas is redrawn once those loads resolve.

## Getting the image to Telegram

- **Share** (`#share`): shown only where `navigator.canShare({files})` is true (phones, and Chrome on
  Windows). It opens the system share sheet with the PNG only. The owner did not want the caption sent along:
  Telegram posted it as a second message. The caption stays in the copy box. The user picks Telegram
  and then the channel. It uses the blob that `refresh()` already made, so `navigator.share` runs inside
  the tap; iOS rejects it after an `await`. If an edit is still debouncing, it asks the user to tap again.
- **Download** stays, and becomes the secondary (ghost) button when share is available.
- Posting directly from the page through a bot was considered and set aside. It needs the bot token on
  a server (for example a Cloudflare Worker) plus a password, because the page is public.

## Hosting

Live at <https://karam3112.github.io/baqa-announcer/>, served by GitHub Pages from `main`, repo root
(repo `karam3112/baqa-announcer`). Pushing to `main` updates the site within a minute or two. The remote
URL carries `karam3112@` because this machine's default GitHub login is a different account.
 The page has `noindex`, but anyone with the link can use
it to make an image in the channel's style. The owner accepted this.

## Wording decisions (agreed with the owner — keep them)

- `انتقل / انتقلت` is spelled without a hamza.
- Death notice, on the image: `لم تُحدَّد ساعة تشييع الجنازة حتى الآن، وسنعلن عنها في وقتها بإذن الله تعالى.`
  — no leading `و`. The copyable caption keeps `، ولم تحدد ...` because there it continues the sentence
  after the name.
- Footer line: `تابعوا الإعلان عن الوفيات في باقة الغربية على تلجرام`.
- Funeral details are shown as three cells (الساعة / من / إلى المقبرة) instead of one sentence.
- Condolence note, default text: men at `ديوان باقة` for three days, starting after العصر on the start
  day and ending an hour after العشاء on the end day (start + 2 days); women at the deceased's home.
  The alternative is the "no condolence house" notice, drawn as a dark box. The note text is an
  editable textarea; once edited it stops auto-regenerating until "reset" is pressed.
- Gender switches: `انتقل/انتقلت`, `أبو/أم`, `جثمانه/جثمانها`, `بيت المرحوم/المرحومة`.
- Alignment (owner's request after testing on a phone): the header lines, the "انتقل" line, the name and the
  identity lines are centred, and so is the death-notice sentence. In the funeral notice the "وسيُشيَّع" label,
  the cells and the note box stay right-aligned. A name too long for one line is split into two balanced
  lines (`balance: true`), so a single word is not left alone on the second line.
- The kunya `(أبو/أم ...)` and the extra description (e.g. `حرم فلان`) are on separate lines under the
  name, with no separator dot. Either may be missing, so each line stands alone.
- Download is blocked while a required field is empty (name; for funerals also time, from, cemetery).

## Poster palette

Ground `#F2F4F3`, ink `#121816`, muted `#55625F`, rules `#CBD3D0`, cell border `#D5DCD9`, note box
`#E4EAE8`, header secondary text `#C9D3D0`, placeholder `#97A3A0`. The owner prefers clean, modern,
minimal design over ornament.

## Not verified yet

- iPhone (iOS Safari), including the share button. The owner's phone works: the page, sharing to Telegram,
  and scanning the QR.
- The artifact version's download button inside Claude.

## Ideas discussed, not built

- Direct posting to the channel through a Telegram bot and a small server (see above).
