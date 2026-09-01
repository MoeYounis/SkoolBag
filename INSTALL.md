# Installing SkoolBag

SkoolBag is not on the Chrome Web Store, so it installs from a folder rather
than from a link. This takes about a minute.

Works in Chrome, Edge, Brave, Opera and Vivaldi.

## 1. Unzip it somewhere permanent

Unzip `skoolbag.zip` to a folder you are not going to delete or move later.
Somewhere like `Documents\SkoolBag` is ideal.

This matters more than it sounds. Chrome loads the extension from wherever that
folder lives and reads it from there every time the browser starts. If the
folder is moved, renamed, or cleaned out of your Downloads, the extension stops
working.

After unzipping you should see a folder containing `manifest.json`, plus
`assets`, `src` and `_locales`. That folder is the one to point Chrome at.

## 2. Open the extensions page

Type this into your address bar and press Enter:

    chrome://extensions

In other browsers it is `edge://extensions`, `brave://extensions`,
`opera://extensions` or `vivaldi://extensions`.

## 3. Turn on Developer mode

There is a toggle labelled **Developer mode**. In Chrome, Brave, Opera and
Vivaldi it is in the top right. In Edge it is in the bottom left.

Turn it on. Leave it on, since the extension will not load without it.

## 4. Load the folder

Click **Load unpacked**, then select the folder from step 1. Pick the folder
itself, not any file inside it.

SkoolBag now appears in your extension list and its icon is in the toolbar. If
you do not see the icon, click the puzzle piece next to the address bar and pin
it.

## Updating

There are no automatic updates when installing this way. To update, download the
new zip, unzip it over the same folder, then return to the extensions page and
click the refresh arrow on the SkoolBag card.

## If something goes wrong

**"Manifest file is missing or unreadable"**
You selected the wrong folder. Pick the one that has `manifest.json` directly
inside it, not the folder above it.

**The extension disappears when the browser restarts**
Developer mode was switched off, or the folder was moved or deleted. Put the
folder back and switch Developer mode on again.

**A warning about developer mode extensions**
Your browser is telling you this extension did not come from its store, which is
true. It is safe to dismiss.

**The icon is there but nothing is detected on a lesson**
Press play on the video first, give it a few seconds, then open SkoolBag again.
Some videos are private to the classroom and only become reachable once their
player has actually started.

## What it can reach

SkoolBag rides your existing logged-in session. It can download exactly what you
could already open yourself in the browser, and nothing beyond that. Nothing is
uploaded and nothing is tracked.
