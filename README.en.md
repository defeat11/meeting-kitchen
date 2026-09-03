# meeting-kitchen

[العربية](README.md)

**One tap on a meeting-room tablet → an order on the kitchen screen and a message in Power Automate.**

> **This system runs in production at my employer, and the code belongs to them. This repository documents the architecture, the engineering decisions, and how it was built.**

A full hospitality system in 5,851 lines. The parts: a PWA for the room tablet, a PHP API with 15
endpoints, and a live kitchen screen. An Android app with a foreground service raises alerts.
A webhook sends every call into Power Automate.

The hosting is shared: PHP and SQLite only. No Node, no WebSocket, no cloud push service.
Every decision in the system works inside that limit.

---

## The problem

A hospitality request in a meeting goes through a person. Someone leaves the room, or makes a
call, or waits.

Calling IT support is worse. There is no record of when the request came in, or who took it.

What was needed: one button inside the room. The order must reach the kitchen with the screen
off. It must also survive a network drop.

---

## How it works

| Component | What it does |
|---|---|
| Room tablet (PWA) | Order screen, no login. The browser stores the room ID, so every request carries its room |
| Offline operation | A service worker caches the UI. A request made during an outage goes to a local queue. It is sent when the connection returns |
| PHP API | One file, `?action=` style, 15 actions, 6 SQLite tables. Every query that takes user input is a prepared statement (22 of them). The rest is fixed SQL with no string building |
| Kitchen screen | Polls `feed` every 5 seconds with an `after` cursor. So only new and active items come down. Sound and vibration on arrival |
| Android app | WebView plus a foreground service. It polls every 8 seconds and raises a system notification with the screen off |
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

The Flow URL lives in the `settings` table. The kitchen settings screen edits it.
It is in no code file, and on no tablet.

---

## The key design decision

**The problem:** the call has to reach Power Automate. But an HTTP call to an outside cloud can
stall or fail. If the server waits for it, the employee stands at a button that hangs.

**The decision:** the server writes the call to SQLite first. Then it fires the webhook inside a
`try/catch`. The connect timeout is 2 seconds, and the total timeout is 3 seconds. A failed Flow
never changes the reply the UI gets.

**Cost and payoff:** worst case, 3 extra seconds on one request. In exchange, the call is stored
and shows on the kitchen screen. This holds even if Power Automate is fully down.

The database is the source of truth. The Flow is an extra alert channel, not a condition for
success.

The same reason put the trigger on the server, not the browser. So the Flow URL never reaches a
tablet sitting in an open room.

---

## What is in this repository

The code belongs to my employer, so it is not here. This repository documents the architecture, and how the system is built and deployed.

The production layout:

```
htdocs/
  index.html app.js kitchen.html kitchen.js settings.html settings.js
  config.js styles.css manifest.json sw.js icons/
  api/  index.php  db.php  data/   ← created on first request, protected by .htaccess
```

How deployment actually works:

1. Upload to the root of shared hosting. The first visit creates and seeds the database.
2. Open `settings.html` on the tablet. Set the room name, then pin the page to the home screen.
3. No password lives in the code. `KITCHEN_USER` and `KITCHEN_PASSWORD` come from the environment.
   Or the system makes a random password. It writes it once to a file, and deletes that file after first login.
4. Paste the Flow URL into the settings tab. From that point on, every call reaches Power Automate.

Before any upload, an automated gate runs PHP lint on every server file. It also checks every JSON file. Nothing ships until it prints `VERIFY: PASSED`.

---

## Why I built it

I work as an IT supervisor. Meeting-room requests used to reach me by word of mouth, or late.

Now one call reaches 3 destinations: the kitchen screen, a phone notification, and Power Automate.
