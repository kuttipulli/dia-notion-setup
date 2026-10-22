# Dia + Notion + Claude Setup (for Claude Code)

> **How to use this:** Paste this file's GitHub link into Claude Code and say:
> "Follow the instructions in this file to set up my computer."
> Claude Code will do the work and stop whenever it needs you.

---

## Instructions for Claude Code

You are setting up the user's browser and note-taking workflow. Read this whole file first, then work through each phase in order.

### Ground rules

- **Narrate briefly.** Before each phase, tell the user in one line what you're about to do.
- **Stop for the user** whenever a step needs a click in a GUI, a sign-in, a macOS permission prompt, or their Mac password. Tell them exactly what to click, then wait for them to say "done".
- **Never type, store, or ask the user to paste their passwords.** They sign in themselves.
- **Don't use Homebrew** unless it's already installed. Installing it requires a sudo password you can't enter.
- **Don't delete anything** on the user's computer. If something needs removing, tell the user and let them do it.
- **Privacy:** if you read browser history, only compute aggregate counts. Don't print, save, or upload the raw history. Delete any temp copies when finished.
- **If a step fails twice**, explain what went wrong in plain English, give the user the manual way to do it, and move on.
- Keep a checklist as you go and show it at the end.

---

## Phase 0 — Check the computer

1. Run `sw_vers` and `uname -m` to get the macOS version and chip (Apple Silicon `arm64` vs Intel `x86_64`).
2. Requirements:
   - Dia: macOS on Apple Silicon is best supported. If the user is on Intel or Windows, check diabrowser.com for current support and tell the user before continuing.
   - Notion AI Meeting Notes (Phase 5): macOS 13 or newer.
3. Check what's already installed: `ls /Applications | grep -iE "dia|notion"`. Skip any install that's already done (but still check versions).
4. Ask the user one question: **"What school or job are you using this for, and do you already have Notion or Dia accounts?"** Use the answer to tailor Phase 4.

---

## Phase 1 — Install Dia and Notion

For each app:

1. Find the official download link from the official site only:
   - Dia: https://www.diabrowser.com
   - Notion desktop: https://www.notion.com/desktop
2. Download the `.dmg` to `~/Downloads` with `curl -L -o`.
3. Mount it with `hdiutil attach`, copy the `.app` into `/Applications` with `cp -R`, then `hdiutil detach`.
4. Remove the quarantine flag only on the app you just installed from the official site: `xattr -dr com.apple.quarantine "/Applications/<App>.app"`. If that fails, it's fine; macOS will just ask the user to confirm on first open.
5. Open it with `open -a "<App>"`.

If the download link can't be found or `curl` fails, open the official download page with `open <url>` and walk the user through dragging the app into Applications.

**Verify:** Notion desktop must be version **4.7.0 or later**. Check with:
`defaults read /Applications/Notion.app/Contents/Info.plist CFBundleShortVersionString`
If it's older, have the user open Notion → menu bar → Check for Updates.

---

## Phase 2 — Accounts

Walk the user through each one and wait for "done" after each:

1. **Notion:** Open the Notion app → sign up or log in. Recommend "Continue with Google" using the email they use most for school/work.
   - Students: mention Notion has a free plan for students with a school email (notion.com/product/notion-for-education).
2. **Dia:** Open Dia → sign in or create an account. When Dia offers to import from another browser (Chrome, Safari, Arc), recommend saying **yes** so bookmarks and passwords come over.
3. Set Dia as the default browser if the user wants (Dia → Settings → Default browser). Ask first.

---

## Phase 3 — Extensions in Dia

Dia is built on Chromium, so it uses the Chrome Web Store. For each extension, open the store page **in Dia** with `open -a "Dia" "<url>"` (search the Chrome Web Store if you need the exact URL), then tell the user to click **Add to Chrome** → **Add extension**.

1. **Claude in Chrome** (official extension by Anthropic)
   - After it's added: click the puzzle-piece icon → pin Claude to the toolbar → sign in with their Claude account.
   - Note: Claude in Chrome needs a paid Claude plan. If the extension says it's unavailable on their plan, tell the user and continue.
2. **Notion Web Clipper** (official, by Notion)
   - After it's added: pin it, click it once, and sign in to Notion.

**Verify:** Ask the user to confirm both icons are visible in Dia's toolbar.

---

## Phase 4 — Pinned tabs (most-used sites)

Goal: pin the 5–10 sites the user actually uses most, like a student pinning their LMS (Brightspace, Canvas, Blackboard), homework sites (WebWork, Gradescope, Pearson), Gmail, Google Calendar, Google Drive, and Notion.

### Step 4a — Ask

Ask: **"Which websites do you open every day or every week? List as many as you want."**

### Step 4b — Check history

Find history files from the browser(s) they've been using. Common locations:

| Browser | History file |
|---|---|
| Chrome | `~/Library/Application Support/Google/Chrome/Default/History` (also `Profile 1`, `Profile 2`, …) |
| Arc | `~/Library/Application Support/Arc/User Data/Default/History` |
| Brave | `~/Library/Application Support/BraveSoftware/Brave-Browser/Default/History` |
| Edge | `~/Library/Application Support/Microsoft Edge/Default/History` |
| Dia | `~/Library/Application Support/Dia/User Data/Default/History` |
| Safari | `~/Library/Safari/History.db` |

For each one that exists:

1. Copy it to a temp folder first (the original is locked while the browser is open):
   `mkdir -p /tmp/histcheck && cp "<path>" /tmp/histcheck/<browser>.db`
2. Query visit counts by domain for the last 60 days. Count the individual visits in that window. Don't use the `visit_count` column: it's an all-time total, so a site used heavily last year and opened once last month would still rank high.
   - **Chromium browsers** (`visits` + `urls`; `visit_time` is microseconds since 1601-01-01; transition types 3 and 4 are embedded frames like ads and video players, so skip them):
     ```sql
     SELECT u.url, COUNT(*) FROM visits v
     JOIN urls u ON u.id = v.url
     WHERE v.visit_time > (strftime('%s','now','-60 days') + 11644473600) * 1000000
       AND (v.transition & 255) NOT IN (3, 4)
     GROUP BY u.url;
     ```
   - **Safari** (`history_items` + `history_visits`; `visit_time` is seconds since 2001-01-01):
     ```sql
     SELECT i.url, COUNT(*) FROM history_visits v
     JOIN history_items i ON i.id = v.history_item
     WHERE v.visit_time > strftime('%s','now','-60 days') - 978307200
     GROUP BY i.url;
     ```
3. Group by domain in a short Python script, add up counts across browsers, and drop noise (search engines, login/SSO pages, `localhost`, ad/redirect domains, `chrome://` pages).
4. If Safari fails with "Operation not permitted", tell the user: System Settings → Privacy & Security → Full Disk Access → turn on their terminal app (Terminal, iTerm, or Claude) → restart it. Or skip Safari.
5. Delete `/tmp/histcheck` when done.

### Step 4c — Combine and confirm

Show the user one list of **up to 10** sites, marking where each came from ("you said" / "history" / "both"), with the specific page that's most useful (for example, the course page of their LMS, not just the homepage). Let them add, remove, or reorder. **Don't pin anything until they approve.**

### Step 4d — Pin

For each approved site, open it in Dia: `open -a "Dia" "<url>"`. Then tell the user to pin each tab (right-click the tab → **Pin**, or drag it into the pinned section). Ask them to confirm when all are pinned.

---

## Phase 5 — Notion asks to transcribe every meeting

**Goal:** whenever the user joins a call (Zoom, Google Meet, Teams, etc.), Notion pops up asking "Start AI Meeting Notes?", and with one click it transcribes, summarizes, and saves the notes into their Notion.

> **How it works:** the Notion desktop app notices when another app (like Zoom) is using the microphone and offers to start AI Meeting Notes. It doesn't record unless the user clicks to start.

### Step 5a — Check that Meeting Notes is available

AI Meeting Notes works on Notion's free and student plans too, so don't assume a paid plan is needed. Check it directly:
- Have the user open any Notion page in the desktop app and type `/meet`. If an **AI Meeting Notes** block appears, it's available. Delete the test block and continue.
- If it doesn't appear, check Settings → Notion AI to make sure AI features are turned on for their workspace, then try again.
- If Notion shows a usage limit or an upgrade prompt, tell the user what it says in plain English and continue with the setup anyway. Don't push them to buy anything.

### Step 5b — Create a Meetings database

Have the user (or do it via the Notion API/MCP if available) create a database page called **Meetings** with properties: `Name` (title), `Date` (date), `Type` (select: Class, Club, Interview, Work, Other).

### Step 5c — Turn on the settings

Walk the user through, in the Notion desktop app:
1. **Settings → Notifications → "Desktop meeting detection notifications"** → ON.
2. **Settings → Notion AI → AI Meeting Notes** → set the default location to the **Meetings** database.
3. Optional: connect their calendar in Notion (or Notion Calendar) so meetings get titles and attendees automatically.

### Step 5d — macOS permissions

Open the right panes for them:
- `open "x-apple.systempreferences:com.apple.preference.security?Privacy_Microphone"` → turn on **Notion**
- `open "x-apple.systempreferences:com.apple.preference.security?Privacy_ScreenCapture"` → turn on **Notion** (needed to capture other people's audio)
- System Settings → Notifications → **Notion** → Allow notifications, style **Alerts** (so the prompt doesn't vanish).
- Make Notion open at login: `osascript -e 'tell application "System Events" to make login item at end with properties {path:"/Applications/Notion.app", hidden:true}'`

Tell the user: **Notion must be running (it can be in the background) for the prompt to appear.**

### Step 5e — Test it

1. Have the user open https://meet.google.com/new in Dia and join with the mic on.
2. Confirm the Notion prompt appears. Click it, talk for about a minute, then stop.
3. Check the **Meetings** database for the new page with a transcript and summary.
4. If no prompt appears: confirm Notion is open and updated, notifications are on, the setting in 5c is on, and the mic permission is granted. Then retry once.

Remind the user: **get consent from everyone in the meeting before recording.**

---

## Phase 6 — Wrap up

Show the user a checklist:

- [ ] Dia installed and signed in
- [ ] Notion installed (v4.7.0+) and signed in
- [ ] Claude in Chrome added to Dia and signed in
- [ ] Notion Web Clipper added to Dia
- [ ] Pinned tabs: (list them)
- [ ] Meeting detection on, Meetings database set as default, permissions granted
- [ ] Test meeting saved to Notion

Then list anything that was skipped or still needs them to do manually, in plain English.
