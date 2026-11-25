# ORGANISE-IPHONE.md

Tidy a messy iPhone Home Screen from any computer - Linux, Windows or Mac - using Claude Code. No Mac needed, no jailbreak, no app to buy.

**How to use it:** plug your iPhone into the computer with a USB cable, unlock it, save this file in an empty folder, start Claude Code in that folder and say:

> Read ORGANISE-IPHONE.md and follow it.

**What to know before you start**

- This uses an unofficial interface (the same one iTunes once used to arrange icons). It works today but Apple could change it in any iOS update.
- A backup of your layout is saved before anything is written, and one command puts it back.
- Nothing is installed on the phone and no app is deleted. Only the positions of icons change.
- The names of your apps are sent to the AI model so it can sort them. If your app list is private to you, think about that first.
- Close anything else that talks to the phone (iTunes sync, Finder, Xcode) while this runs.
- Status: run end to end on an iPhone 12 with iOS 27.0.1 on 25 November 2026 (13 pages down to 2, nothing lost). Treat the first run on any other iOS version as an experiment. Use at your own risk.

---

## Instructions for Claude Code

You are helping the user reorganise the Home Screen of the iPhone plugged into this computer. Follow these steps in order. Be brief and plain: the user may not be technical.

### Rules that always apply

1. Never write to the phone without showing the plan and getting a clear "yes" first.
2. Only ever change the layout through `iphone_layout.py apply`. Never call `set_icon_state` yourself, and never edit the files in `backups/`.
3. Never edit `iphone_layout.py` to get past a refused plan. Fix the plan instead.
4. Every item in `layout.json` must appear exactly once in your plan. You are moving icons, not removing them.
5. If any step fails twice, stop and tell the user what happened. Do not improvise other ways of reaching the phone.

### Step 1 - Set up

1. Check Python 3.10 or newer is available (`python3 --version`, or `python --version` on Windows).
2. Create a virtual environment in this folder and install the one dependency:
   `python3 -m venv .venv` then `.venv/bin/pip install -U pymobiledevice3` (on Windows: `.venv\Scripts\pip`). Use that environment's Python for every command below.
   If `venv` fails because `ensurepip` is missing (Debian and Ubuntu without the `python3-venv` package), no sudo is needed: run `python3 -m venv --without-pip .venv` then `python3 -m pip --python .venv/bin/python install -U pymobiledevice3`.
   `pymobiledevice3` pulls in about 90 packages. If the user vets new packages before installing, do that first (version 11.63.1 was vetted clean on 25 November 2026).
3. Save the script from the bottom of this file as `iphone_layout.py`, exactly as written.
4. Run `iphone_layout.py check`. It should print the phone's name, model and iOS version.

If `check` cannot find the phone, ask the user to confirm the phone is unlocked and that they tapped "Trust This Computer", then:

- **Linux:** `usbmuxd` must be installed and running (`sudo apt install usbmuxd` or the distro's equivalent). Ask before running anything with sudo.
- **Windows:** Apple's USB driver is needed. It comes with the "Apple Devices" app from the Microsoft Store, or with iTunes.
- **macOS:** nothing extra is needed.

If `check` warns the device is not an iPhone (an iPad, say), tell the user the size checks are built for iPhone and stop unless they want to carry on at their own risk.

### Step 2 - Read the current layout

Run `iphone_layout.py dump`. It saves a backup in `backups/` and writes `layout.json`:

```json
{
  "source": "backups/iconstate-...-dump.plist",
  "dock": [{"id": "com.apple.mobilesafari", "name": "Safari"}],
  "pages": [
    [
      {"id": "com.example.app", "name": "Example"},
      {"folder": "Travel", "pages": [[{"id": "...", "name": "..."}]]},
      {"id": "widget:...", "name": "...", "widget": "medium"}
    ]
  ]
}
```

Tell the user what you found in one or two lines: how many apps, widgets, pages and folders. Apps that live only in the App Library are not listed and will not be touched.

### Step 3 - Ask how they want it

Ask one short question offering these styles, and let them describe their own:

- **Everyday first:** page 1 holds the apps they use most, everything else in named folders on page 2.
- **By category:** one folder per category (Money, Travel, Photos, Social, Work, Shopping, Health, Games, Utilities and so on).
- **Minimal:** a near-empty first page, everything else in a few folders.

Also ask which apps they use every day, since you cannot see usage. Keep the dock as it is unless they say otherwise.

### Step 4 - Write the plan

Create `plan.json` in the same shape as `layout.json`, keeping the same `"source"` value. Entries only need an `"id"`; names are optional.

Limits the script enforces (iPhone):

- A page holds 24 slots. An app or folder takes 1; a small widget 4, medium 8, large 16.
- The dock holds 4 items and no widgets.
- A folder page holds 9 apps. Use several pages inside one folder for bigger groups. No folders inside folders, no widgets inside folders.
- No empty pages or empty folders.

Good practice:

- Use your knowledge of what each app is to categorise it. If you do not recognise an app, ask rather than guess, or put it in a folder called "Other".
- Keep existing widgets on the page they were on, in the same order, unless asked to move them.
- Short folder names, one or two words.
- Aim for two or three pages in total.

Then run `iphone_layout.py apply plan.json --dry-run` and fix anything it refuses.

### Step 5 - Show the plan and get a yes

Show the user the plan in plain text: the dock, then each page with its apps and each folder with its contents. Ask if they want changes. Revise and re-run the dry run until they are happy. Do not go on without a clear yes.

### Step 6 - Apply

Ask the user to unlock the phone and leave it on the Home Screen, then run `iphone_layout.py apply plan.json`.

- **"Done. The phone's layout matches the plan."** - tell the user to look at their phone, and give them the undo command the script printed.
- **"the phone's layout differs from the plan"** (exit code 2) - iOS adjusted something, which newer versions sometimes do. Run `dump` again, show the user in plain words what differs, and ask whether to keep it, try one corrected plan, or undo. Do not loop more than twice.
  The usual cause, seen on iOS 27: the first write makes iOS add a new last page holding apps that were not in the dump at all (apps that had been on hidden pages). The planned pages are intact. Run `dump` again, add those apps to the plan, and apply once more; the second apply then matches.
- **"The phone's apps changed since the dump"** - an app was installed or removed. Go back to Step 2.

### Step 7 - Undo, if asked

Run `iphone_layout.py restore backups/<the before-apply file>`. If the plan was applied more than once, the layout the user started with is the earliest `-dump.plist` in `backups/`; restore that one. Backups are plain files in `backups/` and are never deleted, so the user can restore later too.

### Step 8 - Finish

Say in two lines what changed and where the backups are. Remind the user they can delete this folder when they are happy, but that doing so removes the backups.

---

## The script

Save this as `iphone_layout.py`.

```python
#!/usr/bin/env python3
"""iphone_layout.py - read, back up, rewrite and restore an iPhone Home Screen
layout over USB, using pymobiledevice3.

Commands:
  check              show the connected device
  dump               back up the layout and write layout.json (a simple view)
  apply PLAN.json    validate a plan, back up, write it, read back and compare
  restore BACKUP     put a saved backup (.plist) back on the phone

The plan never contains raw device data. Every app, widget or web clip is
referred to by an id, and the real entries are copied untouched from the
backup named in the plan's "source" field. Nothing can be invented or lost:
a plan that drops, duplicates or adds an id is refused before any write.
"""
import argparse
import asyncio
import copy
import datetime as dt
import json
import plistlib
import sys
from pathlib import Path

BACKUPS = Path("backups")
LAYOUT = Path("layout.json")
PAGE_SLOTS = 24          # iPhone page: 4 columns x 6 rows
FOLDER_PAGE_SLOTS = 9    # folder page: 3 x 3
DOCK_MAX = 4
WIDGET_SLOTS = {"small": 4, "medium": 8, "large": 16, "extraLarge": 24}


# ---------- raw icon state <-> simple view ----------------------------------

def is_folder(item):
    # Widgets on iOS 27 also carry an (empty) "iconLists" key; they are not folders.
    return (isinstance(item, dict) and "iconLists" in item
            and item.get("elementType") != "widget" and not item.get("gridSize"))


def leaf_name(item):
    if isinstance(item, str):
        return item
    name = item.get("displayName")
    if name:
        return name
    for el in item.get("elements") or []:
        if isinstance(el, dict):
            for key in ("displayName", "containerBundleIdentifier", "bundleIdentifier"):
                if el.get(key):
                    return str(el[key])
    return item.get("bundleIdentifier") or item.get("displayIdentifier") or "(unnamed)"


def leaf_kind(item):
    if isinstance(item, str):
        return "app", None
    if item.get("gridSize") or item.get("elementType") or item.get("iconType") == "custom":
        return "widget", item.get("gridSize") or "small"
    return "app", None


class Index:
    """Gives every non-folder item a stable id, in traversal order."""

    def __init__(self):
        self.raw = {}
        self.folders = {}

    def add(self, item):
        if isinstance(item, str):
            base = item
        else:
            base = (item.get("bundleIdentifier") or item.get("displayIdentifier")
                    or item.get("uniqueIdentifier") or "item")
            if leaf_kind(item)[0] == "widget":
                base = "widget:" + str(base)
        ident, n = str(base), 2
        while ident in self.raw:
            ident = f"{base}#{n}"
            n += 1
        self.raw[ident] = item
        return ident


def simplify(state, index=None):
    """Raw icon state -> {"dock": [...], "pages": [[...], ...]}."""
    index = index or Index()

    def leaf(item):
        ident = index.add(item)
        out = {"id": ident, "name": leaf_name(item)}
        kind, size = leaf_kind(item)
        if kind == "widget":
            out["widget"] = size
        return out

    def entry(item):
        if is_folder(item):
            name = item.get("displayName") or ""
            index.folders.setdefault(name, item)
            pages = [[leaf(x) for x in page if not is_folder(x)]
                     for page in item.get("iconLists") or []]
            return {"folder": name, "pages": [p for p in pages if p]}
        return leaf(item)

    lists = [[entry(x) for x in lst] for lst in state]
    view = {"dock": lists[0] if lists else [], "pages": lists[1:]}
    return view, index


def plan_id(entry):
    return entry if isinstance(entry, str) else entry.get("id")


def plan_ids(plan):
    out = []
    for lst in [plan.get("dock", [])] + plan.get("pages", []):
        for e in lst:
            if isinstance(e, dict) and "folder" in e:
                for page in e.get("pages", []):
                    out.extend(plan_id(x) for x in page)
            else:
                out.append(plan_id(e))
    return out


def validate(plan, index):
    errors = []
    ids = plan_ids(plan)
    known = set(index.raw)
    seen = set()
    for i in ids:
        if i not in known:
            errors.append(f"unknown id (not on the phone): {i}")
        elif i in seen:
            errors.append(f"id placed more than once: {i}")
        seen.add(i)
    for i in sorted(known - seen):
        errors.append(f"id missing from plan (would be lost): {i} [{leaf_name(index.raw[i])}]")

    def slots(e):
        if isinstance(e, dict) and "folder" in e:
            return 1
        raw = index.raw.get(plan_id(e))
        kind, size = leaf_kind(raw) if raw is not None else ("app", None)
        return WIDGET_SLOTS.get(size, 4) if kind == "widget" else 1

    dock = plan.get("dock", [])
    if len(dock) > DOCK_MAX:
        errors.append(f"dock has {len(dock)} items, maximum is {DOCK_MAX}")
    for e in dock:
        if slots(e) > 1:
            errors.append(f"widgets cannot go in the dock: {plan_id(e)}")
    if not plan.get("pages"):
        errors.append("plan has no pages")
    for n, page in enumerate(plan.get("pages", []), 1):
        if not page:
            errors.append(f"page {n} is empty")
        used = sum(slots(e) for e in page)
        if used > PAGE_SLOTS:
            errors.append(f"page {n} needs {used} slots, maximum is {PAGE_SLOTS}")
    for lst in [dock] + plan.get("pages", []):
        for e in lst:
            if not (isinstance(e, dict) and "folder" in e):
                continue
            name = e.get("folder") or ""
            if not name.strip():
                errors.append("a folder has no name")
            if not e.get("pages") or any(not p for p in e["pages"]):
                errors.append(f"folder '{name}' is empty or has an empty page")
            for page in e.get("pages", []):
                if len(page) > FOLDER_PAGE_SLOTS:
                    errors.append(f"folder '{name}' has a page of {len(page)}, maximum is {FOLDER_PAGE_SLOTS}")
                for x in page:
                    if isinstance(x, dict) and "folder" in x:
                        errors.append(f"folder '{name}' contains another folder")
                    elif slots(x) > 1:
                        errors.append(f"folder '{name}' contains a widget: {plan_id(x)}")
    return errors


def build_state(plan, index):
    """Plan + backup items -> raw icon state ready to send."""

    def entry(e):
        if isinstance(e, dict) and "folder" in e:
            name = e["folder"]
            folder = copy.deepcopy(index.folders.get(name)) or {"listType": "folder"}
            folder["displayName"] = name
            folder["iconLists"] = [[copy.deepcopy(index.raw[plan_id(x)]) for x in page]
                                   for page in e["pages"]]
            return folder
        return copy.deepcopy(index.raw[plan_id(e)])

    state = [[entry(e) for e in plan.get("dock", [])]]
    state += [[entry(e) for e in page] for page in plan["pages"]]
    return strip_microseconds(state)


def strip_microseconds(obj):
    # SpringBoard has been seen to reject dates carrying sub-second precision.
    if isinstance(obj, dt.datetime):
        return obj.replace(microsecond=0)
    if isinstance(obj, list):
        return [strip_microseconds(x) for x in obj]
    if isinstance(obj, dict):
        return {k: strip_microseconds(v) for k, v in obj.items()}
    return obj


def shape(view):
    """Ids only, for comparing a plan with what the phone reports."""

    def entry(e):
        if isinstance(e, dict) and "folder" in e:
            return ("folder", e["folder"], [[plan_id(x) for x in p] for p in e["pages"]])
        return plan_id(e)

    return [[entry(e) for e in view.get("dock", [])]] + \
           [[entry(e) for e in page] for page in view.get("pages", [])]


def differences(wanted, actual):
    out = []
    w, a = shape(wanted), shape(actual)
    for n in range(max(len(w), len(a))):
        label = "dock" if n == 0 else f"page {n}"
        wl = w[n] if n < len(w) else None
        al = a[n] if n < len(a) else None
        if wl != al:
            out.append(f"{label}: wanted {wl!r}, phone has {al!r}")
    return out


# ---------- files -----------------------------------------------------------

def save_backup(state, tag):
    BACKUPS.mkdir(exist_ok=True)
    stamp = dt.datetime.now().strftime("%Y%m%d-%H%M%S")
    path = BACKUPS / f"iconstate-{stamp}-{tag}.plist"
    with open(path, "wb") as fh:
        plistlib.dump(state, fh)
    return path


def load_backup(path):
    with open(path, "rb") as fh:
        return plistlib.load(fh)


# ---------- device ----------------------------------------------------------

HINTS = """Could not talk to the iPhone. Check that:
  - it is plugged in by USB, unlocked, and you tapped "Trust This Computer"
  - Linux: usbmuxd is installed and running (sudo apt install usbmuxd)
  - Windows: Apple's "Apple Devices" app or iTunes is installed (for the USB driver)
  - macOS: nothing extra is needed"""


class Phone:
    async def __aenter__(self):
        try:
            from pymobiledevice3.lockdown import create_using_usbmux
            from pymobiledevice3.services.springboard import SpringBoardServicesService
        except ImportError:
            sys.exit("pymobiledevice3 is not installed. Run: python3 -m pip install -U pymobiledevice3")
        try:
            self.lockdown = await create_using_usbmux()
            self.sb = SpringBoardServicesService(lockdown=self.lockdown)
        except Exception as exc:  # any connection or pairing problem
            sys.exit(f"{HINTS}\n\nDetail: {type(exc).__name__}: {exc}")
        return self

    async def __aexit__(self, *exc):
        for closer in (getattr(self, "sb", None), getattr(self, "lockdown", None)):
            if closer is not None:
                try:
                    await closer.close()
                except Exception:
                    pass

    def info(self):
        v = self.lockdown.all_values
        return {"name": v.get("DeviceName"), "model": v.get("ProductType"),
                "class": v.get("DeviceClass"), "ios": v.get("ProductVersion")}

    async def get(self):
        return await self.sb.get_icon_state()

    async def set(self, state):
        await self.sb.set_icon_state(state)


# ---------- commands --------------------------------------------------------

async def cmd_check(_args):
    async with Phone() as phone:
        info = phone.info()
        print(json.dumps(info, indent=2))
        if info["class"] != "iPhone":
            print("WARNING: page and dock size checks assume an iPhone.")


async def cmd_dump(_args):
    async with Phone() as phone:
        state = await phone.get()
        info = phone.info()
    backup = save_backup(state, "dump")
    view, index = simplify(state)
    view = {"source": str(backup), "device": info, **view}
    LAYOUT.write_text(json.dumps(view, indent=2, ensure_ascii=False), encoding="utf-8")
    folders = sum(1 for lst in [view["dock"]] + view["pages"] for e in lst if "folder" in e)
    print(f"Backup saved:   {backup}")
    print(f"Layout written: {LAYOUT}")
    print(f"{len(index.raw)} items, {len(view['pages'])} pages, {folders} folders, "
          f"{len(view['dock'])} in dock")


async def cmd_apply(args):
    plan = json.loads(Path(args.plan).read_text(encoding="utf-8"))
    source = plan.get("source")
    if not source or not Path(source).exists():
        sys.exit('Plan needs a "source" field naming the backup made by dump.')
    _, index = simplify(load_backup(source))
    errors = validate(plan, index)
    if errors:
        print("Plan refused, nothing was written:")
        for e in errors:
            print("  -", e)
        sys.exit(1)
    new_state = build_state(plan, index)
    print(f"Plan is valid: {len(index.raw)} items, {len(plan['pages'])} pages.")
    if args.dry_run:
        print("Dry run only, nothing was written.")
        return
    async with Phone() as phone:
        current = await phone.get()
        _, live = simplify(current)
        if set(live.raw) != set(index.raw):
            added = sorted(set(live.raw) - set(index.raw))
            gone = sorted(set(index.raw) - set(live.raw))
            sys.exit("The phone's apps changed since the dump, nothing was written. "
                     f"Run dump again.\n  new: {added}\n  gone: {gone}")
        backup = save_backup(current, "before-apply")
        print(f"Backup saved: {backup}")
        await phone.set(new_state)
        await asyncio.sleep(3)
        after = await phone.get()
    save_backup(after, "after-apply")
    actual, _ = simplify(after)
    diffs = differences(plan, actual)
    if diffs:
        print("Written, but the phone's layout differs from the plan:")
        for d in diffs:
            print("  -", d)
        print(f"To undo: python3 iphone_layout.py restore {backup}")
        sys.exit(2)
    print("Done. The phone's layout matches the plan.")
    print(f"To undo: python3 iphone_layout.py restore {backup}")


async def cmd_restore(args):
    state = strip_microseconds(load_backup(args.backup))
    async with Phone() as phone:
        current = await phone.get()
        backup = save_backup(current, "before-restore")
        await phone.set(state)
        await asyncio.sleep(3)
        after = await phone.get()
    same = shape(simplify(after)[0]) == shape(simplify(state)[0])
    print(f"Backup of the previous layout: {backup}")
    print("Restored and verified." if same else
          "Restore sent, but the phone's layout does not exactly match the backup.")
    sys.exit(0 if same else 2)


def main():
    parser = argparse.ArgumentParser(description=__doc__.split("\n\n")[0])
    sub = parser.add_subparsers(dest="command", required=True)
    sub.add_parser("check").set_defaults(func=cmd_check)
    sub.add_parser("dump").set_defaults(func=cmd_dump)
    p = sub.add_parser("apply")
    p.add_argument("plan")
    p.add_argument("--dry-run", action="store_true")
    p.set_defaults(func=cmd_apply)
    p = sub.add_parser("restore")
    p.add_argument("backup")
    p.set_defaults(func=cmd_restore)
    args = parser.parse_args()
    asyncio.run(args.func(args))


if __name__ == "__main__":
    main()
```
