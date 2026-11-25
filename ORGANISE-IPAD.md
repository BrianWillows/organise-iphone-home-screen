# ORGANISE-IPAD.md

Tidy a messy iPad Home Screen from any computer - Linux, Windows or Mac - using Claude Code. No Mac needed, no jailbreak, no app to buy. The same script also works on an iPhone.

**How to use it:** plug your iPad into the computer with a USB cable, unlock it, save this file in an empty folder, start Claude Code in that folder and say:

> Read ORGANISE-IPAD.md and follow it.

**What to know before you start**

- This uses an unofficial interface (the same one iTunes once used to arrange icons). It works today but Apple could change it in any iPadOS update.
- A backup of your layout is saved before anything is written, and one command puts it back.
- Nothing is installed on the iPad and no app is deleted. Only the positions of icons change.
- The names of your apps are sent to the AI model so it can sort them. If your app list is private to you, think about that first.
- Close anything else that talks to the iPad (iTunes sync, Finder, Xcode) while this runs.
- Plug in one Apple device at a time, and use a separate folder for each device. Every backup records which device it came from, and the script refuses to write a backup or plan to a different device.
- Status: run end to end on an iPad mini (5th generation) with iPadOS 26.7.3 on 25 November 2026 (14 pages down to 2, nothing lost), and its iPhone twin on an iPhone 12 with iOS 27.0.1 the same day. Treat the first run on any other version as an experiment. Use at your own risk.

---

## Instructions for Claude Code

You are helping the user reorganise the Home Screen of the iPad plugged into this computer. Follow these steps in order. Be brief and plain: the user may not be technical.

### Rules that always apply

1. Never write to the iPad without showing the plan and getting a clear "yes" first.
2. Only ever change the layout through `ipad_layout.py apply`. Never call `set_icon_state` yourself, and never edit the files in `backups/`.
3. Never edit `ipad_layout.py` to get past a refused plan. Fix the plan instead.
4. Every item in `layout.json` must appear exactly once in your plan. You are moving icons, not removing them.
5. If any step fails twice, stop and tell the user what happened. Do not improvise other ways of reaching the iPad.

### Step 1 - Set up

1. Check Python 3.10 or newer is available (`python3 --version`, or `python --version` on Windows).
2. Create a virtual environment in this folder and install the one dependency:
   `python3 -m venv .venv` then `.venv/bin/pip install -U pymobiledevice3` (on Windows: `.venv\Scripts\pip`). Use that environment's Python for every command below.
   If `venv` fails because `ensurepip` is missing (Debian and Ubuntu without the `python3-venv` package), no sudo is needed: run `python3 -m venv --without-pip .venv` then `python3 -m pip --python .venv/bin/python install -U pymobiledevice3`.
   `pymobiledevice3` pulls in about 90 packages. If the user vets new packages before installing, do that first (version 11.63.1 was vetted clean on 25 November 2026). An environment already built for the iPhone version of this guide can be reused.
3. Save the script from the bottom of this file as `ipad_layout.py`, exactly as written.
4. Run `ipad_layout.py check`. It should print the device's name, model and version, and the limits the device reports for its own Home Screen (items per page, per folder page and in the dock).

If `check` says "Device is password protected", the iPad is locked: ask the user to unlock it and leave it on the Home Screen. If `check` cannot find the iPad, ask the user to confirm it is unlocked and that they tapped "Trust This Computer", then:

- **Linux:** `usbmuxd` must be installed and running (`sudo apt install usbmuxd` or the distro's equivalent). Ask before running anything with sudo.
- **Windows:** Apple's USB driver is needed. It comes with the "Apple Devices" app from the Microsoft Store, or with iTunes.
- **macOS:** nothing extra is needed.

### Step 2 - Read the current layout

Run `ipad_layout.py dump`. It saves a backup in `backups/` (with a small `.meta.json` file beside it naming the device and its limits) and writes `layout.json`:

```json
{
  "source": "backups/iconstate-...-dump.plist",
  "limits": {"page": 30, "folder_page": 16, "dock": 16, "max_pages": 15, "folder_max_pages": 15},
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

Tell the user what you found in one or two lines: how many apps, widgets, pages and folders. Apps that live only in the App Library or on hidden pages are not listed (see Step 6 for what happens to them).

### Step 3 - Ask how they want it

Ask one short question offering these styles, and let them describe their own:

- **Everyday first:** page 1 holds the apps they use most, everything else in named folders on page 2.
- **By category:** one folder per category (Money, Travel, Photos, Social, Work, Shopping, Health, Games, Utilities and so on).
- **Minimal:** a near-empty first page, everything else in a few folders.

Also ask which apps they use every day, since you cannot see usage. If they do not say, the page 1 they already have is the best evidence of what they use: leave it exactly as it is. Keep the dock as it is unless they say otherwise.

### Step 4 - Write the plan

Create `plan.json` in the same shape as `layout.json`, keeping the same `"source"` value. Entries only need an `"id"`; names are optional.

Limits the script enforces. They come from the device, so read them from `"limits"` in `layout.json` and do not assume them. On the iPad mini tested:

- A page holds 30 slots. An app or folder takes 1. Widgets are counted cautiously: small 4, medium 8, large 16.
- The dock holds up to 16 items and no widgets.
- A folder page holds 16 apps. Use several pages inside one folder for bigger groups. No folders inside folders, no widgets inside folders.
- No empty pages or empty folders.
- A page copied from `layout.json` without any change is always accepted, even if the cautious widget count says it is too full. The device has already shown that it fits.

Good practice:

- Use your knowledge of what each app is to categorise it. If you do not recognise an app, ask rather than guess, or put it in a folder called "Other".
- Do not move widgets between pages unless asked. If the page a widget was on is being replaced, keep the widget on the page with the same number, at the start.
- Short folder names, one or two words.
- Aim for two or three pages in total. Twenty or so folders fit on one page beside a couple of small widgets.

Then run `ipad_layout.py apply plan.json --dry-run` and fix anything it refuses.

### Step 5 - Show the plan and get a yes

Show the user the plan in plain text: the dock, then each page with its apps and each folder with its contents. Ask if they want changes. Revise and re-run the dry run until they are happy. Do not go on without a clear yes.

### Step 6 - Apply

Ask the user to unlock the iPad and leave it on the Home Screen, then run `ipad_layout.py apply plan.json`.

- **"Done. The device's layout matches the plan."** - tell the user to look at their iPad, and give them the undo command the script printed.
- **"the device's layout differs from the plan"** (exit code 2) - iPadOS adjusted something. The script prints one short line per page saying what the device added or what is missing. Run `dump` again, show the user in plain words what differs, and ask whether to keep it, try one corrected plan, or undo. Do not loop more than twice.
  The usual cause, seen on both test devices: the first write makes the device show apps that were not in the dump at all (apps that had been on hidden pages). It puts them in free slots on the last page and on a new page after it. The planned pages are otherwise intact. Run `dump` again, add those apps to the plan, and apply once more; the second apply then matches.
- **"The device's apps changed since the dump"** - an app was installed or removed. Go back to Step 2.
- **"This backup came from ... but the connected device is ..."** - the wrong device is plugged in, or this folder belongs to another device. Nothing was written. Fix that before going on.

### Step 7 - Undo, if asked

Run `ipad_layout.py restore backups/<the before-apply file>`. If the plan was applied more than once, the layout the user started with is the earliest `-dump.plist` in `backups/` that has a `.meta.json` beside it; restore that one. Backups are plain files in `backups/` and are never deleted, so the user can restore later too.

### Step 8 - Finish

Say in two lines what changed and where the backups are. Remind the user they can delete this folder when they are happy, but that doing so removes the backups.

---

## The script

Save this as `ipad_layout.py`.

```python
#!/usr/bin/env python3
"""ipad_layout.py - read, back up, rewrite and restore an iPad (or iPhone) Home
Screen layout over USB, using pymobiledevice3.

Commands:
  check              show the connected device
  dump               back up the layout and write layout.json (a simple view)
  apply PLAN.json    validate a plan, back up, write it, read back and compare
  restore BACKUP     put a saved backup (.plist) back on the phone

The plan never contains raw device data. Every app, widget or web clip is
referred to by an id, and the real entries are copied untouched from the
backup named in the plan's "source" field. Nothing can be invented or lost:
a plan that drops, duplicates or adds an id is refused before any write.

Page, dock and folder sizes are not assumed: dump asks the device for its own
grid and saves it beside the backup. Every backup also records which device it
came from, and apply and restore refuse to write it to a different device.
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
# Used only when the device does not report its own grid.
FALLBACK_LIMITS = {
    "iPhone": {"page": 24, "folder_page": 9, "dock": 4, "max_pages": 15, "folder_max_pages": 15},
    "iPad": {"page": 30, "folder_page": 16, "dock": 16, "max_pages": 15, "folder_max_pages": 15},
}
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


def limits_from(metrics, device_class):
    """Device-reported grid -> the limits a plan must respect."""
    out = dict(FALLBACK_LIMITS.get(device_class) or FALLBACK_LIMITS["iPhone"])
    m = metrics or {}

    def num(key):
        try:
            return int(m[key])
        except (KeyError, TypeError, ValueError):
            return 0

    if num("homeScreenIconColumns") and num("homeScreenIconRows"):
        out["page"] = num("homeScreenIconColumns") * num("homeScreenIconRows")
    if num("homeScreenIconFolderColumns") and num("homeScreenIconFolderRows"):
        out["folder_page"] = num("homeScreenIconFolderColumns") * num("homeScreenIconFolderRows")
    for key, name in (("homeScreenIconDockMaxCount", "dock"), ("homeScreenIconMaxPages", "max_pages"),
                      ("homeScreenIconFolderMaxPages", "folder_max_pages")):
        if num(key):
            out[name] = num(key)
    return out


def validate(plan, index, limits, source_view):
    errors = []
    # A page left exactly as the device had it is always allowed: widget sizes
    # are counted cautiously here, and the device has already shown it fits.
    untouched = [json.dumps(p) for p in shape(source_view)[1:]]
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
    if len(dock) > limits["dock"]:
        errors.append(f"dock has {len(dock)} items, maximum is {limits['dock']}")
    for e in dock:
        if slots(e) > 1:
            errors.append(f"widgets cannot go in the dock: {plan_id(e)}")
    if not plan.get("pages"):
        errors.append("plan has no pages")
    if len(plan.get("pages", [])) > limits["max_pages"]:
        errors.append(f"plan has {len(plan['pages'])} pages, maximum is {limits['max_pages']}")
    for n, page in enumerate(plan.get("pages", []), 1):
        if not page:
            errors.append(f"page {n} is empty")
        used = sum(slots(e) for e in page)
        if used > limits["page"] and json.dumps(shape({"pages": [page]})[1]) not in untouched:
            errors.append(f"page {n} needs {used} slots, maximum is {limits['page']}")
    for lst in [dock] + plan.get("pages", []):
        for e in lst:
            if not (isinstance(e, dict) and "folder" in e):
                continue
            name = e.get("folder") or ""
            if not name.strip():
                errors.append("a folder has no name")
            if not e.get("pages") or any(not p for p in e["pages"]):
                errors.append(f"folder '{name}' is empty or has an empty page")
            if len(e.get("pages", [])) > limits["folder_max_pages"]:
                errors.append(f"folder '{name}' has {len(e['pages'])} pages, maximum is {limits['folder_max_pages']}")
            for page in e.get("pages", []):
                if len(page) > limits["folder_page"]:
                    errors.append(f"folder '{name}' has a page of {len(page)}, maximum is {limits['folder_page']}")
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


def flat_ids(lst):
    out = []
    for e in lst or []:
        if isinstance(e, tuple):
            out.extend(i for page in e[2] for i in page)
        else:
            out.append(e)
    return out


def differences(wanted, actual):
    """One short line per dock/page that differs, naming what was added or is missing."""
    out = []
    w, a = shape(wanted), shape(actual)
    for n in range(max(len(w), len(a))):
        label = "dock" if n == 0 else f"page {n}"
        wl = w[n] if n < len(w) else None
        al = a[n] if n < len(a) else None
        if wl == al:
            continue
        if wl is None:
            out.append(f"{label}: not in the plan, the device added it with {flat_ids(al)}")
            continue
        if al is None:
            out.append(f"{label}: in the plan, missing on the device")
            continue
        extra = [i for i in flat_ids(al) if i not in flat_ids(wl)]
        gone = [i for i in flat_ids(wl) if i not in flat_ids(al)]
        if extra:
            out.append(f"{label}: the device added {extra}")
        if gone:
            out.append(f"{label}: missing from this page {gone}")
        if not extra and not gone:
            out.append(f"{label}: same items, but the order or folder grouping differs")
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


def meta_path(backup):
    return Path(str(backup) + ".meta.json")


def save_meta(backup, info, limits):
    meta_path(backup).write_text(json.dumps({"device": info, "limits": limits}, indent=2), encoding="utf-8")


def load_meta(backup):
    path = meta_path(backup)
    if not path.exists():
        sys.exit(f"{path} is missing, so this backup cannot be tied to a device. Nothing was written.")
    return json.loads(path.read_text(encoding="utf-8"))


def require_same_device(meta, info):
    want, have = meta["device"].get("udid"), info.get("udid")
    if not want or want != have:
        sys.exit(f"This backup came from {meta['device'].get('name')!r}, but the connected device is "
                 f"{info.get('name')!r}. Nothing was written. Unplug other devices and try again.")


# ---------- device ----------------------------------------------------------

HINTS = """Could not talk to the device. Check that:
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
                "class": v.get("DeviceClass"), "ios": v.get("ProductVersion"),
                "udid": v.get("UniqueDeviceID")}

    async def limits(self):
        try:
            metrics = await self.sb.get_homescreen_icon_metrics()
        except Exception:
            metrics = None
        return limits_from(metrics, self.info()["class"])

    async def get(self):
        return await self.sb.get_icon_state()

    async def set(self, state):
        await self.sb.set_icon_state(state)


# ---------- commands --------------------------------------------------------

async def cmd_check(_args):
    async with Phone() as phone:
        info = phone.info()
        print(json.dumps({"device": info, "limits": await phone.limits()}, indent=2))


async def cmd_dump(_args):
    async with Phone() as phone:
        state = await phone.get()
        info = phone.info()
        limits = await phone.limits()
    backup = save_backup(state, "dump")
    save_meta(backup, info, limits)
    view, index = simplify(state)
    view = {"source": str(backup), "device": info, "limits": limits, **view}
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
    source_view, index = simplify(load_backup(source))
    meta = load_meta(source)
    errors = validate(plan, index, meta["limits"], source_view)
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
        info = phone.info()
        require_same_device(meta, info)
        current = await phone.get()
        _, live = simplify(current)
        if set(live.raw) != set(index.raw):
            added = sorted(set(live.raw) - set(index.raw))
            gone = sorted(set(index.raw) - set(live.raw))
            sys.exit("The device's apps changed since the dump, nothing was written. "
                     f"Run dump again.\n  new: {added}\n  gone: {gone}")
        backup = save_backup(current, "before-apply")
        save_meta(backup, info, meta["limits"])
        print(f"Backup saved: {backup}")
        await phone.set(new_state)
        await asyncio.sleep(3)
        after = await phone.get()
    save_meta(save_backup(after, "after-apply"), info, meta["limits"])
    actual, _ = simplify(after)
    diffs = differences(plan, actual)
    if diffs:
        print("Written, but the device's layout differs from the plan:")
        for d in diffs:
            print("  -", d)
        print(f"To undo: ipad_layout.py restore {backup}")
        sys.exit(2)
    print("Done. The device's layout matches the plan.")
    print(f"To undo: ipad_layout.py restore {backup}")


async def cmd_restore(args):
    state = strip_microseconds(load_backup(args.backup))
    meta = load_meta(args.backup)
    async with Phone() as phone:
        info = phone.info()
        require_same_device(meta, info)
        current = await phone.get()
        backup = save_backup(current, "before-restore")
        save_meta(backup, info, meta["limits"])
        await phone.set(state)
        await asyncio.sleep(3)
        after = await phone.get()
    same = shape(simplify(after)[0]) == shape(simplify(state)[0])
    print(f"Backup of the previous layout: {backup}")
    print("Restored and verified." if same else
          "Restore sent, but the device's layout does not exactly match the backup.")
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
