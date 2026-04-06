# WhatsApp Class Tracker Bot — Complete Beginner Guide (v2)

> **Who this is for:** Two people building a Python WhatsApp bot together with no prior bot experience. One handles the database, the other handles the bot logic. Both roles are clearly marked throughout.

---

## What You're Building

A WhatsApp bot that lives in your friend group and does five things:

1. Automatically tags friends who have class in the next 2 hours
2. Shows everyone's class schedule for the day
3. Answers natural questions like "what time is Amaka's class?" or "where is EEE 301?"
4. Sends custom messages through the bot
5. Data is stored in a plain JSON file — no database software needed

---

## The MVP (Minimum Viable Product)

Your MVP is the smallest version that actually works. Build only this first.

**MVP includes:**
- A bot that receives messages from your WhatsApp group
- A `classes.json` file with Name, Phone, Course, Location, and Time per person
- `!remind` — finds and tags anyone with class in the next 2 hours
- `!schedule` — shows today's full class list sorted by time
- `!custom <message>` — sends a message through the bot
- Natural Q&A: answers questions like "when is Chidi's class" or "where is MTH 201"

**NOT in the MVP (add later):**
- Multi-day or weekly scheduling
- Per-person private reminders
- A web dashboard for the database

---

## Choosing a Backend Framework

Your backend is a small web server that receives messages from Twilio and runs your bot logic. Here are your realistic options.

### Option 1 — Flask (Recommended)

```
pip install flask
```

**Advantages:** Most beginner-friendly Python web framework. Enormous number of tutorials. Minimal boilerplate — a working server in 10 lines. Twilio's own documentation uses Flask in all its examples. Works perfectly with the webhook pattern this project uses.

**Disadvantages:** Single-threaded by default; not suited for high traffic. No built-in input validation. Slightly more overhead than Bottle.

**Verdict: The right choice for this project.** Simple, well-documented, and Twilio-native.

---

### Option 2 — Bottle

```
pip install bottle
```

**Advantages:** The entire framework is one Python file with zero dependencies. Syntax is nearly identical to Flask, so switching is trivial. Slightly less overhead.

**Disadvantages:** Smaller community, fewer tutorials than Flask. Less actively maintained. Doesn't meaningfully simplify anything over Flask for a project this size.

**Verdict:** A fine alternative if you want the absolute minimum. The code would look almost identical.

---

### Option 3 — FastAPI

```
pip install fastapi uvicorn
```

**Advantages:** Modern async framework. Automatically generates API docs. Built-in type validation. Great for production APIs with many concurrent users.

**Disadvantages:** Requires a separate `uvicorn` server to run. Async patterns are harder for beginners. Auto-docs and validation are pointless for a bot receiving plain text messages. Overkill for a small friend group.

**Verdict:** Great framework, wrong project. Save it for something larger.

---

### Option 4 — Django

```
pip install django
```

**Advantages:** Full batteries-included framework with admin panel, ORM, and auth system. Excellent for large long-lived projects.

**Disadvantages:** Extremely heavy for a WhatsApp bot. Requires understanding Django's project layout before writing a single line of bot logic. The built-in ORM and admin are pointless when your "database" is a JSON file. Setup time alone would triple your development time.

**Verdict:** Do not use for this project.

---

### Option 5 — `http.server` (Python built-in)

**Advantages:** Zero external dependencies, built into Python.

**Disadvantages:** No routing, no request parsing, no helper functions. You would manually re-implement everything Flask gives you for free. Far more error-prone for beginners.

**Verdict:** Not worth it when Flask exists.

---

**Final choice: Flask.** The rest of this guide uses Flask.

---

## Tools You Will Need

| Tool | What it does | Who sets it up |
|---|---|---|
| Python 3.10+ | Main programming language | Both |
| Flask | Small web server to receive WhatsApp messages | Bot person |
| Twilio | Connects your Python code to WhatsApp | Bot person |
| APScheduler | Runs the 2-hour reminder check automatically | Bot person |
| ngrok | Makes your laptop reachable from the internet for testing | Bot person |
| python-dotenv | Stores secret API keys safely in a `.env` file | Both |
| Jupyter Notebook | Lets the database person manage `classes.json` easily | Database person |
| pandas | Reads and displays the JSON data cleanly in Jupyter | Database person |

No SQL library is needed. The JSON file is read and written with Python's built-in `json` module.

---

## Project Folder Structure

```
class-bot/
│
├── bot.py                     Main Flask server — entry point
├── scheduler.py               Automatic 2-hour reminders
├── database.py                Functions to read/write classes.json
├── commands.py                Handles !remind, !schedule, !custom
├── qa.py                      Answers natural language class questions
├── classes.json               The data file (database person manages this)
├── .env                       Secret API keys (never share or commit this)
├── requirements.txt           Packages to install
└── database_manager.ipynb     Jupyter notebook for managing data
```

---

## The Data Format — `classes.json`

This is the only data storage in the project. Create it manually or let `database.py` create it on first run.

```json
[
  {
    "id": 1,
    "name": "Amaka",
    "phone": "+2348012345678",
    "course": "EEE 301",
    "location": "Engineering Block A",
    "class_time": "14:00"
  },
  {
    "id": 2,
    "name": "Chidi",
    "phone": "+2348098765432",
    "course": "MTH 201",
    "location": "Science Faculty Hall",
    "class_time": "09:30"
  }
]
```

Rules:
- `class_time` must always be 24-hour format: `"14:00"` not `"2pm"`
- `phone` must include the country code with no spaces: `"+2348012345678"`
- `name` is case-insensitive in the Q&A engine but be consistent anyway
- One entry per class session — if someone has two classes in a day, add two entries

---

## Step-by-Step Guide

---

### PHASE 1 — Setup (Both People)

#### Step 1: Install Python

Download Python 3.10 or newer from https://python.org/downloads. On Windows, check **"Add Python to PATH"** during installation.

Confirm it works by opening your terminal and running:
```
python --version
```

#### Step 2: Create the Project and Virtual Environment

```bash
mkdir class-bot
cd class-bot
python -m venv venv
```

Activate it:
- **Windows:** `venv\Scripts\activate`
- **Mac/Linux:** `source venv/bin/activate`

You will see `(venv)` at the start of your terminal line. Always activate this before working.

#### Step 3: Install Packages

Create a file called `requirements.txt` and paste this inside:

```
flask
twilio
apscheduler
pandas
python-dotenv
jupyter
notebook
```

Then install:
```bash
pip install -r requirements.txt
```

---

### PHASE 2 — Twilio Setup (Bot Person)

#### Step 4: Create a Twilio Account

1. Sign up for free at https://www.twilio.com
2. On your dashboard, copy your **Account SID** and **Auth Token**

#### Step 5: Activate the WhatsApp Sandbox

1. In Twilio Console go to **Messaging → Try it out → Send a WhatsApp message**
2. Follow the instructions — you will send a join code from your phone to a Twilio number
3. Every friend who should receive bot messages must do this once from their own phone
4. Note the Twilio sandbox WhatsApp number (looks like `+14155238886`)

#### Step 6: Create the `.env` File

Create a file called `.env` in your project folder and fill in your values:

```
TWILIO_ACCOUNT_SID=ACxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx
TWILIO_AUTH_TOKEN=your_auth_token_here
TWILIO_WHATSAPP_NUMBER=whatsapp:+14155238886
GROUP_WHATSAPP_NUMBER=whatsapp:+2348012345678
```

`GROUP_WHATSAPP_NUMBER` is your WhatsApp group's number. You get this from Twilio's message logs after your first test message from the group.

Create a `.gitignore` file and add `.env` to it. Never share or commit this file.

---

### PHASE 3 — Database Setup (Database Person)

#### Step 7: Create `database.py`

This file handles all reading and writing of `classes.json`. Every other file in the project calls these functions — nothing else touches the JSON file directly.

```python
import json
import os
from datetime import datetime

DB_FILE = "classes.json"

def load_data():
    """Reads classes.json and returns the list of entries."""
    if not os.path.exists(DB_FILE):
        return []
    with open(DB_FILE, "r") as f:
        return json.load(f)

def save_data(data):
    """Writes the list of entries back to classes.json."""
    with open(DB_FILE, "w") as f:
        json.dump(data, f, indent=2)

def add_student(name, phone, course, location, class_time):
    """Adds a new class entry."""
    data = load_data()
    new_id = max((entry["id"] for entry in data), default=0) + 1
    data.append({
        "id": new_id,
        "name": name,
        "phone": phone,
        "course": course,
        "location": location,
        "class_time": class_time
    })
    save_data(data)
    print(f"Added: {name} — {course} at {class_time}")

def get_all_classes():
    """Returns all class entries."""
    return load_data()

def delete_entry(entry_id):
    """Removes an entry by its ID."""
    data = load_data()
    updated = [entry for entry in data if entry["id"] != entry_id]
    save_data(updated)
    print(f"Deleted entry ID {entry_id}")

def get_upcoming_classes(within_hours=2):
    """Returns entries where class starts within the next X hours."""
    data = load_data()
    now = datetime.now()
    upcoming = []
    for entry in data:
        try:
            hour, minute = map(int, entry["class_time"].split(":"))
            class_dt = now.replace(hour=hour, minute=minute, second=0, microsecond=0)
            diff_minutes = (class_dt - now).total_seconds() / 60
            if 0 <= diff_minutes <= within_hours * 60:
                upcoming.append(entry)
        except Exception:
            continue
    return upcoming

if __name__ == "__main__":
    if not os.path.exists(DB_FILE):
        save_data([])
        print("classes.json created.")
    else:
        print("classes.json already exists.")
```

#### Step 8: Set Up the Jupyter Notebook

Start Jupyter from your terminal:
```bash
jupyter notebook
```

Create a new notebook called `database_manager.ipynb`. This is your control panel.

**Cell 1 — View all classes:**
```python
import pandas as pd
from database import load_data, add_student, delete_entry, get_upcoming_classes

def view_all():
    data = load_data()
    if not data:
        print("No entries yet.")
        return
    return pd.DataFrame(data)

view_all()
```

**Cell 2 — Add a new entry:**
```python
add_student(
    name="Tunde",
    phone="+2348011223344",
    course="CSC 402",
    location="Computer Science Lab 2",
    class_time="11:00"   # 24-hour format
)
view_all()
```

**Cell 3 — Delete an entry:**
```python
delete_entry(entry_id=3)   # Use the ID shown in view_all()
view_all()
```

**Cell 4 — Preview upcoming classes:**
```python
upcoming = get_upcoming_classes(within_hours=2)
for e in upcoming:
    print(f"{e['name']} — {e['course']} at {e['class_time']}, {e['location']}")
```

Run these cells whenever you need to update the data. The JSON file updates immediately.

---

### PHASE 4 — Q&A Engine (Bot Person)

#### Step 9: Create `qa.py`

When someone types a natural question like "what time is Amaka's class" or "where is EEE 301", this file finds the answer by scanning the JSON data for matching names or course codes. No AI library is needed — simple keyword matching works perfectly for a small group.

```python
from database import load_data

def answer_question(message):
    """
    Tries to answer a natural language question about the class schedule.
    Returns a reply string if a match is found, or None if no match.
    """
    msg = message.lower().strip()
    data = load_data()

    if not data:
        return "The class database is empty. Ask the database manager to add entries."

    # Detect what kind of information is being asked for
    asking_time     = any(w in msg for w in ["time", "when", "start"])
    asking_location = any(w in msg for w in ["where", "venue", "location", "held", "place"])
    asking_course   = any(w in msg for w in ["course", "studying", "what class"])

    matched = []
    for entry in data:
        name_lower   = entry["name"].lower()
        course_lower = entry["course"].lower()

        # Match by name (full name or first name)
        first_name = name_lower.split()[0]
        if name_lower in msg or first_name in msg:
            matched.append(entry)
            continue

        # Match by course code, with or without the space (e.g. "eee301" or "eee 301")
        if course_lower in msg or course_lower.replace(" ", "") in msg.replace(" ", ""):
            matched.append(entry)

    if not matched:
        return None   # Not a class question — bot stays silent

    replies = []
    for entry in matched:
        name     = entry["name"]
        course   = entry["course"]
        location = entry["location"]
        time_str = entry["class_time"]

        if asking_time and not asking_location:
            replies.append(f"*{name}'s* {course} class starts at *{time_str}*.")

        elif asking_location and not asking_time:
            replies.append(f"*{name}'s* {course} class is at *{location}*.")

        elif asking_course:
            replies.append(f"*{name}* has *{course}* at {time_str} in {location}.")

        else:
            # Generic match — return all info
            replies.append(
                f"*{name}* — {course}\n"
                f"   {time_str}   |   {location}"
            )

    return "\n\n".join(replies)
```

**Example inputs and outputs:**

| Someone types | Bot replies |
|---|---|
| `what time is amaka's class` | Amaka's EEE 301 class starts at 14:00. |
| `where is mth 201` | Chidi's MTH 201 class is at Science Faculty Hall. |
| `when is chidi's class` | Chidi's MTH 201 class starts at 09:30. |
| `tunde venue` | Tunde's CSC 402 class is at Computer Science Lab 2. |
| `what class does amaka have` | Amaka has EEE 301 at 14:00 in Engineering Block A. |
| `eee301 time` | Amaka's EEE 301 class starts at 14:00. |

---

### PHASE 5 — Commands (Bot Person)

#### Step 10: Create `commands.py`

```python
from database import get_all_classes, get_upcoming_classes

def handle_command(message_body):
    """
    Handles explicit bot commands (those starting with !).
    Returns a reply string, or None if not a recognized command.
    """
    msg = message_body.strip()
    msg_lower = msg.lower()

    if msg_lower == "!help":
        return (
            "*Class Bot — Commands*\n\n"
            "!remind — Tag people with class in the next 2 hours\n"
            "!schedule — Show today's full class list\n"
            "!custom <message> — Send a message through the bot\n"
            "!help — Show this list\n\n"
            "*You can also ask questions like:*\n"
            "• What time is Amaka's class?\n"
            "• Where is EEE 301?\n"
            "• When is Chidi's class?"
        )

    elif msg_lower == "!remind":
        upcoming = get_upcoming_classes(within_hours=2)
        if not upcoming:
            return "No one has class in the next 2 hours. Enjoy the break!"
        lines = ["*Heads up! These people have class soon:*\n"]
        for entry in upcoming:
            phone    = entry["phone"].replace("+", "").replace(" ", "")
            course   = entry["course"]
            location = entry["location"]
            time_str = entry["class_time"]
            lines.append(f"@{phone} — {course} at {location}, {time_str}")
        return "\n".join(lines)

    elif msg_lower == "!schedule":
        all_classes = get_all_classes()
        if not all_classes:
            return "No classes in the database yet."
        sorted_classes = sorted(all_classes, key=lambda x: x["class_time"])
        lines = ["*Today's Class Schedule:*\n"]
        for entry in sorted_classes:
            lines.append(
                f"• *{entry['name']}* — {entry['course']}\n"
                f"   {entry['class_time']}   |   {entry['location']}"
            )
        return "\n".join(lines)

    elif msg_lower.startswith("!custom "):
        custom_text = msg[len("!custom "):].strip()
        if not custom_text:
            return "Usage: !custom <your message here>"
        return f"*Bot message:* {custom_text}"

    return None   # Not a recognized command
```

---

### PHASE 6 — Main Bot File (Bot Person)

#### Step 11: Create `bot.py`

```python
from flask import Flask, request
from twilio.twiml.messaging_response import MessagingResponse
from twilio.rest import Client
from dotenv import load_dotenv
import os

from commands import handle_command
from qa import answer_question
from scheduler import start_scheduler

load_dotenv()

app = Flask(__name__)

ACCOUNT_SID   = os.getenv("TWILIO_ACCOUNT_SID")
AUTH_TOKEN    = os.getenv("TWILIO_AUTH_TOKEN")
TWILIO_NUMBER = os.getenv("TWILIO_WHATSAPP_NUMBER")
GROUP_NUMBER  = os.getenv("GROUP_WHATSAPP_NUMBER")

client = Client(ACCOUNT_SID, AUTH_TOKEN)

def send_to_group(message):
    """Sends a message directly to the WhatsApp group. Used by the scheduler."""
    client.messages.create(body=message, from_=TWILIO_NUMBER, to=GROUP_NUMBER)

@app.route("/webhook", methods=["POST"])
def webhook():
    """Receives all incoming WhatsApp messages forwarded by Twilio."""
    incoming_msg = request.form.get("Body", "").strip()
    sender       = request.form.get("From", "")

    print(f"[{sender}]: {incoming_msg}")

    reply = None

    # Step 1: Check if it's a ! command
    if incoming_msg.startswith("!"):
        reply = handle_command(incoming_msg)

    # Step 2: If not a command, try to answer it as a class question
    if reply is None:
        reply = answer_question(incoming_msg)

    # Step 3: Send the reply back, or stay silent if nothing matched
    resp = MessagingResponse()
    if reply:
        resp.message(reply)

    return str(resp)

if __name__ == "__main__":
    start_scheduler(send_to_group)
    app.run(debug=True, port=5000)
```

---

### PHASE 7 — Scheduler (Bot Person)

#### Step 12: Create `scheduler.py`

```python
from apscheduler.schedulers.background import BackgroundScheduler
from database import get_upcoming_classes

def check_and_remind(send_fn):
    """Checks the JSON file for upcoming classes and sends a group reminder."""
    upcoming = get_upcoming_classes(within_hours=2)
    if not upcoming:
        return

    lines = ["*Automatic reminder — someone has class coming up!*\n"]
    for entry in upcoming:
        phone    = entry["phone"].replace("+", "").replace(" ", "")
        course   = entry["course"]
        location = entry["location"]
        time_str = entry["class_time"]
        lines.append(f"@{phone} — {course} at {location}, {time_str}")

    send_fn("\n".join(lines))
    print("Reminder sent.")

def start_scheduler(send_fn):
    """Starts the background job that checks every 30 minutes."""
    scheduler = BackgroundScheduler()
    scheduler.add_job(check_and_remind, "interval", minutes=30, args=[send_fn])
    scheduler.start()
    print("Scheduler running — checking every 30 minutes.")
    return scheduler
```

---

### PHASE 8 — Connecting with ngrok (Bot Person)

#### Step 13: Run the Bot Locally

Open two terminals, both with your virtual environment activated.

**Terminal 1 — start Flask:**
```bash
python bot.py
```

**Terminal 2 — start ngrok:**
```bash
ngrok http 5000
```

Copy the `https://abc123.ngrok.io` URL that appears.

#### Step 14: Link ngrok to Twilio

1. Go to Twilio Console → **Messaging → Settings → WhatsApp Sandbox Settings**
2. In **"When a message comes in"** paste: `https://abc123.ngrok.io/webhook`
3. Set the method to **HTTP POST** and save

Now the flow is: WhatsApp → Twilio → ngrok → Flask → your code → reply sent back.

---

### PHASE 9 — Testing

Test in this order:

**1. Verify the database:**
```bash
python database.py
```
Should print `classes.json created.` Open the file to confirm it is valid JSON.

**2. Add test data via Jupyter:**
Add two or three entries including one with a class time 45 minutes from now to test the reminder.

**3. Test commands in the Python shell:**
```python
from commands import handle_command
print(handle_command("!help"))
print(handle_command("!schedule"))
print(handle_command("!remind"))
```

**4. Test the Q&A engine:**
```python
from qa import answer_question
print(answer_question("what time is amaka's class"))
print(answer_question("where is eee 301"))
print(answer_question("when is chidi's class"))
print(answer_question("hello how are you"))   # Should return None
```

**5. Test via WhatsApp:**
Send `!help` to the sandbox number. Then send a natural question. Both should get replies.

---

### PHASE 10 — Going Live (Optional)

Deploy to Railway so the bot runs 24/7 without keeping your laptop on:

1. Push your project to GitHub — confirm `.env` is in `.gitignore`
2. Create a project at https://railway.app and connect your GitHub repo
3. In Railway → **Variables**, add all the values from your `.env` file
4. Railway runs `bot.py` automatically and gives you a public URL
5. Update that URL in Twilio's sandbox settings to replace ngrok

---

## Division of Work

| Task | Database Person | Bot Person |
|---|---|---|
| Python & venv setup | Yes | Yes |
| Twilio account & sandbox | | Yes |
| `.env` file | Fill in values | Create file |
| `classes.json` initial data | Yes | |
| `database.py` | Yes | |
| `database_manager.ipynb` | Yes | |
| `qa.py` | Review the name/course list | Write it |
| `commands.py` | | Yes |
| `bot.py` | | Yes |
| `scheduler.py` | | Yes |
| ngrok + Twilio webhook | | Yes |
| Ongoing data management | Yes | |
| Testing via WhatsApp | Yes | Yes |

---

## Quick Reference

| Input | What the bot does |
|---|---|
| `!help` | Lists all commands and Q&A examples |
| `!remind` | Tags everyone with class in the next 2 hours |
| `!schedule` | Shows all classes sorted by start time |
| `!custom <text>` | Sends text as a bot announcement |
| `what time is [name]'s class` | Returns start time |
| `where is [name]'s class` | Returns location |
| `where is [course code]` | Returns location for that course |
| `when is [name]'s class` | Returns start time |
| `what class does [name] have` | Returns course, time, and location |
| `[name] venue` | Returns location |

---

## Important Notes

- **Time format:** Always use 24-hour `"HH:MM"`. Both the scheduler and Q&A engine depend on this.
- **The JSON file is your source of truth.** Do not edit it by hand while the bot is running. Use Jupyter.
- **Sandbox limit:** Everyone in the group must send the Twilio join code once. For production with no join code, you need a paid Twilio number.
- **ngrok resets each session.** Every time you restart it you get a new URL and must update Twilio. Railway eliminates this.

---

## Useful Links

- Flask quickstart: https://flask.palletsprojects.com/en/3.0.x/quickstart
- Twilio WhatsApp sandbox docs: https://www.twilio.com/docs/whatsapp/sandbox
- APScheduler docs: https://apscheduler.readthedocs.io/en/stable
- ngrok download: https://ngrok.com/download
- Railway deployment: https://railway.app

---

*Build the MVP first. Once !remind, !schedule, and the Q&A all work in the sandbox, everything else is just adding features on top of a working foundation.*
