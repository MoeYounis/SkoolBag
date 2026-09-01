# SkoolBag

A Chrome extension that saves Skool lesson videos as MP4 files so you can watch
them offline.

I built this because I kept losing access to courses I had paid for. Memberships
lapse, communities get archived, instructors take lessons down. If you paid for
something, you should be able to keep a copy of it.

It runs entirely in your browser. There is no account, no licence key, no upload
step and no download limit.

## What it does

- Saves any lesson video as an MP4, at whatever quality it was uploaded in
- Works with Skool's own player plus Loom, Vimeo and Wistia embeds
- Backs up a whole classroom in one go, with lesson notes and attachments
- Writes to any folder on your computer, not just the Downloads folder
- Picks up where it left off if you stop a backup halfway
- Speaks 15 languages

## Install

Not on the Chrome Web Store, so it installs from a folder. Takes about a minute.

1. Download the zip from [Releases](../../releases) and unzip it somewhere you
   will not delete later. Chrome reads the extension from that folder every time
   it starts, so Downloads is a bad choice.
2. Go to `chrome://extensions`
3. Turn on **Developer mode**, top right
4. Click **Load unpacked** and pick the folder holding `manifest.json`

`INSTALL.md` covers the same ground with more hand holding, and ships inside the
zip for anyone you pass it to.

Needs Chrome 111 or newer. Also runs unmodified on Edge, Brave, Opera and
Vivaldi.

## Using it

Open a lesson, press play so the player requests its stream, then click the
toolbar icon. Pick a quality and hit download.

For a whole course, open any lesson in the classroom, switch to **Whole course**
and click **Find lessons**. It scans that classroom and nothing else, so a
backup never wanders into another community you happen to be a member of.

A course backup writes:

```
Skool Downloads/
  Cold Email Mastery/
    01 - Welcome/
      01 - Welcome.mp4
      notes.md
      worksheet.pdf
    02 - Setting up your domain/
      ...
    _download-log.txt
```

Module folders are dropped when no module holds more than one lesson, since a
folder per file just repeats the filename.

Run it again later and it skips whatever is already on disk. Delete a file and
re-run, and it comes back. Add lessons to the course and a re-run picks those up
too. Resuming and starting are the same operation, so there is nothing to
remember.

## Supported players

| Player | Notes |
|---|---|
| Skool native | Mux behind Skool's own domain. Full quality ladder. |
| Loom | Public and classroom-private videos. |
| Vimeo | Needs the share hash for unlisted videos, which it digs out of the lesson. |
| Wistia | Video assets only, no captions or storyboards. |
| YouTube | Detected and listed, never downloaded. See below. |

## What it will not do

**YouTube.** YouTube blocks extension downloads. A YouTube lesson is detected,
listed and given a `.url` shortcut file pointing at the original, and that is
all. Adding it is not on the roadmap.

**DRM.** If a playlist declares a real encryption method, the download stops with
a named error. There is no decryption code in here and there will not be.

**Anything your account cannot already open.** It rides your existing logged in
session. A locked lesson stays locked.

## Settings

Everything is in the options page.

- **Simultaneous downloads** and **parallel connections per video.** Higher is
  faster until the video host starts throttling you. Four to eight connections is
  the sweet spot.
- **Save to.** Choose any folder on your computer. Leave it alone and files go to
  your browser's download folder.
- **Filename template.** Drag the pieces (number, lesson title, course name,
  quality) into the order you want and pick a separator. There is a text mode if
  you want something the picker cannot express.
- **Notes and attachments.** Lesson text is converted to Markdown. Attachments are
  fetched through Skool's own signed URL endpoint.

## How it works

Three separate contexts, and the split is not arbitrary.

**The service worker orchestrates and nothing else.** Manifest V3 service workers
get torn down after about 30 seconds of idle, and they have no `Blob` or
`URL.createObjectURL`. Running a 40 minute download there would mean the browser
killing it halfway.

**The engine does the work.** All the fetching, demuxing, muxing and file writing
happens in `src/engine/engine.js`, hosted either in an offscreen document
(Chromium 109+) or a pinned background tab on forks that lack `chrome.offscreen`.
The same file runs in both because it only touches `fetch`, `Blob`,
`URL.createObjectURL` and runtime messaging. Nothing branches on browser name,
only on whether an API exists.

**Content scripts detect.** They read the lesson currently on screen and report
what they find.

### The media pipeline

Transport stream and fragmented MP4 sources go through separate demuxers that
normalise to one common sample shape, so the remuxer can interleave both by
decode time and hand a single sample list to the muxer.

Sample bytes are never concatenated into one buffer. They stay as an array of
chunks handed to a `Blob`, which lets Chrome spill to disk instead of holding a
multi gigabyte `ArrayBuffer` in memory.

Two things in here look odd and are deliberate:

- Audio timestamps are carried in sample rate units, exactly 1024 ticks per AAC
  frame. Carrying them in 90 kHz rounds every single frame and drifts audibly
  across a long lesson.
- The muxer builds its `moov` atom twice, because chunk offsets depend on the size
  of `moov`. This only terminates because `co64` is fixed width, so the second
  pass comes out the same size as the first.

No ffmpeg.wasm. The muxer is about 100 KB of plain JavaScript with no SIMD
requirement, so it works on old CPUs and inside VMs.

### Detection

The rule the whole detector is built around: never scan page state wholesale.

A Skool classroom page embeds the entire course tree, every lesson in it. A deep
scan reports sibling lessons' videos as though they were on the page in front of
you, which is exactly how one native Skool lesson can show up as five identical
Loom rows. Page state is read only for the lesson named by the `?md=` query
parameter.

Sightings merge rather than deduplicate. The same video seen in markup and seen
on the wire carry different fields, and the network one has the signing token that
makes the download work. Dropping it as a duplicate loses the only useful part.

### Referer rewriting

Vimeo and Wistia CDNs reject requests whose Referer is an extension origin.
`declarativeNetRequest` session rules rewrite the header, scoped by tab id so page
traffic is untouched and a lesson playing in another tab keeps working.

## Development

Zero dependencies. Node 18 or newer, nothing to install.

```bash
npm test                          # selftest, course scoping, dead code
node tools/selftest.mjs           # 113 cases
node tools/selftest.mjs bulk      # only cases matching a name or section
node tools/mux-test.mjs           # native playback and signing tokens
node tools/package.mjs            # validate everything, then zip to dist/
node tools/build-locales.mjs      # regenerate _locales from the source table
node tools/build-icons.mjs        # regenerate assets/*.png from branding/
```

`mux-test.mjs` runs separately because it stubs `fetch` globally.

`package.mjs` is the gate. It fails on missing files, locale key drift,
unresolved `__MSG_*` keys, `data-i18n` keys nothing defines, `src` and `href`
paths that do not exist, Chrome only manifest keys, a `minimum_chrome_version`
below what the manifest actually uses, and any Firefox `browser.*` usage.

A few habits worth keeping if you work on this:

- The test fixtures synthesise byte accurate MPEG-TS with real PAT/PMT tables and
  a real H.264 SPS, so the pipeline is exercised against data it would actually
  meet rather than against the muxer's own assumptions.
- After adding a guard, break the thing it guards and confirm the test fails. A
  guard that still passes with the bug reintroduced is worse than no guard.
- Smoke tests execute the page scripts against a DOM stub built from the real
  markup. An undefined identifier is valid syntax, so `node --check` cannot see
  it, and a load time throw kills a page silently.

`_locales/` is generated. Add strings to the table in `tools/build-locales.mjs`
and regenerate. Do not hand edit the output.

## Layout

```
manifest.json
src/
  background/     service worker, queue, course backup, path rules
  content/        lesson detection, page and embed hooks
  engine/         fetching, demuxing, muxing, file writing
  lib/            format parsers, providers, folder writes, diagnostics
  popup/          toolbar UI
  options/        settings
  ui/             shared theme
  welcome/        first run page
tools/            tests, build scripts, packaging
branding/         source artwork
_locales/         generated, 15 languages
```

Everything under `src/lib/` is platform agnostic. Hand it a Mux, Vimeo, Loom or
Wistia id from anywhere and it works. Supporting another platform means new host
matches and a new detector, not engine changes.

## Use it responsibly

This saves copies of material you can already open in your own browser. It does
not unlock anything and it does not bypass any protection.

Download what you have paid for or been given access to, keep it for personal
use, and follow the terms set by creators and platforms. Republishing, reselling
or sharing that material is between you and the rights holder. I am not a party
to it and I accept no liability for misuse.

## Licence

MIT. See [LICENSE](LICENSE).

Poppins is bundled under the SIL Open Font License 1.1, see
`assets/fonts/OFL.txt`.

Built by Moe Younis. [x.com/MoeYounis](https://x.com/MoeYounis)
