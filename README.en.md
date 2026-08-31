# meeting-kitchen

[العربية](README.md)

**One tap on a meeting-room tablet → an order on the kitchen screen and a message in Power Automate.**

> **This system runs in production at my employer, and the code belongs to them. This repository documents the architecture, the engineering decisions, and how it was built.**

A complete hospitality system in 5,851 lines: a PWA for the room tablet, a PHP API with 15
endpoints, a live kitchen screen, an Android app with a foreground service for alerts, and a
webhook into Power Automate.

The hosting is shared: PHP and SQLite only. No Node, no WebSocket, no cloud push service.
Every decision in the system is built inside that constraint.

---

## The problem

A hospitality request in a meeting goes through a person. Someone leaves the room, or makes a
call, or waits.

Calling IT support is worse: no record of when the request came in, or who picked it up.

What was needed: one button inside the room, and delivery to the kitchen that holds even when
the screen is off or the network drops.

---

## How it works

| Component | What it does |
|---|---|
| Room tablet (PWA) | Order screen, no login. The room ID is stored locally, so every request is tagged with its room |
| Offline operation | A service worker caches the UI. A request made during an outage enters a local queue and is sent when the connection returns |
| PHP API | One file, `?action=` style, 15 actions, 6 SQLite tables. Every query that takes user input is a prepared statement (22 of them); the rest is fixed SQL with no string building |
| Kitchen screen | Polls `feed` every 5 seconds with an `after` cursor, so only new and active items come down. Sound and vibration on arrival |
| Android app | WebView plus a foreground service that polls every 8 seconds and raises a system notification with the screen off |
| **Power Automate** | Every call fires a webhook from the server with a `{type, label, room, note, time}` payload |
| Interface | Arabic first (RTL) and full English: 182 translation keys in one file |

**The call path into Power Automate:**

```
Room tablet ──POST action=call──> PHP API ──INSERT──> SQLite
                                      │
                                      ├──> Kitchen screen + Android app (feed poll)
                                      └──> POST JSON ──> Power Automate (HTTP trigger)
                                                              └──> Teams / email / task
```

The Flow URL lives in the `settings` table and is edited from the kitchen settings screen.
It is in no code file, and on no tablet.

---

## The key design decision

**The problem:** the call has to reach Power Automate. But an HTTP call to an outside cloud can
stall or fail. If the server waits on it, the employee is left standing at a button that hangs.

**The decision:** the server writes the call to SQLite first, then fires the webhook inside a
`try/catch` with a 2-second connect timeout and a 3-second total timeout. A failed Flow never
changes what the UI gets back.

**Cost and payoff:** worst case, 3 extra seconds on a single request. In exchange: the call is
stored and visible on the kitchen screen even if Power Automate is completely down.

The database is the source of truth. The Flow is an extra alert channel, not a condition for
success.

The same reasoning put the trigger on the server instead of the browser: the Flow URL never
reaches a tablet sitting in an open room.

---

## What is in this repository

The code belongs to my employer, so it is not here. This repository documents the architecture and how the system is built and deployed.

The production layout:

```
htdocs/
  index.html app.js kitchen.html kitchen.js settings.html settings.js
  config.js styles.css manifest.json sw.js icons/
  api/  index.php  db.php  data/   ← created on first request, protected by .htaccess
```

How deployment actually works:

1. Upload to the root of shared hosting. The database is created and seeded on the first visit.
2. Open `settings.html` on the tablet, set the room name, then pin the page to the home screen.
3. No password lives in the code: `KITCHEN_USER` and `KITCHEN_PASSWORD` come from the environment,
   or the system generates a random password, written once to a file that is deleted after first login.
4. Paste the Flow URL into the settings tab. From that point on, every call reaches Power Automate.

Before any upload, an automated gate runs PHP lint on every server file and validates every JSON file; nothing ships until it prints `VERIFY: PASSED`.

---

## Why I built it

I work as an IT supervisor, and meeting-room requests used to reach me by word of mouth, or late.

Now one call reaches 3 destinations: the kitchen screen, a phone notification, and Power Automate.
