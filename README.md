#!/usr/bin/env python3
"""
Daily Task Logger
- Simple CLI app to add tasks, mark them done, and view daily summaries.
- Stores data in JSON at ~/.daily_logger/data.json
Usage:
  python daily_logger.py add "Study algorithms" 45   # add task with estimated minutes
  python daily_logger.py done 3                      # mark task id 3 completed
  python daily_logger.py list                        # list today's tasks
  python daily_logger.py stats                       # show summary for last 7 days
"""

import json
from pathlib import Path
from datetime import datetime, date, timedelta
import sys

DATA_DIR = Path.home() / ".daily_logger"
DATA_FILE = DATA_DIR / "data.json"

def ensure_data():
    DATA_DIR.mkdir(parents=True, exist_ok=True)
    if not DATA_FILE.exists():
        with DATA_FILE.open("w", encoding="utf-8") as f:
            json.dump({"tasks": [], "next_id": 1}, f)

def load():
    with DATA_FILE.open("r", encoding="utf-8") as f:
        return json.load(f)

def save(data):
    with DATA_FILE.open("w", encoding="utf-8") as f:
        json.dump(data, f, indent=2, ensure_ascii=False)

def add_task(text, est_minutes=0):
    data = load()
    tid = data["next_id"]
    task = {
        "id": tid,
        "text": text,
        "est_minutes": int(est_minutes),
        "created": datetime.utcnow().isoformat(),
        "completed": None
    }
    data["tasks"].append(task)
    data["next_id"] = tid + 1
    save(data)
    print(f"Added task #{tid}: {text} ({est_minutes}m)")

def list_tasks(show_all=False):
    data = load()
    tasks = data["tasks"]
    today_iso = date.today().isoformat()
    def is_today(t):
        # created date only compare date part
        return t["created"][:10] == today_iso
    filtered = tasks if show_all else [t for t in tasks if is_today(t)]
    if not filtered:
        print("No tasks found.")
        return
    for t in filtered:
        status = "DONE" if t["completed"] else "TODO"
        created_local = t["created"][:19].replace("T", " ")
        print(f"#{t['id']:>3} [{status}] {t['text']} — est {t['est_minutes']}m — created {created_local}")

def mark_done(task_id):
    data = load()
    for t in data["tasks"]:
        if t["id"] == task_id:
            if t["completed"]:
                print(f"Task #{task_id} already completed.")
                return
            t["completed"] = datetime.utcnow().isoformat()
            save(data)
            print(f"Marked #{task_id} done.")
            return
    print(f"Task #{task_id} not found.")

def stats(days=7):
    data = load()
    tasks = data["tasks"]
    cutoff = date.today() - timedelta(days=days-1)
    summary = {}
    for i in range(days):
        d = (cutoff + timedelta(days=i)).isoformat()
        summary[d] = {"added":0, "completed":0, "est_minutes":0}
    for t in tasks:
        created_day = t["created"][:10]
        if created_day in summary:
            summary[created_day]["added"] += 1
            summary[created_day]["est_minutes"] += t.get("est_minutes",0)
        if t["completed"]:
            comp_day = t["completed"][:10]
            if comp_day in summary:
                summary[comp_day]["completed"] += 1
    print(f"Stats (last {days} days):")
    for d, s in summary.items():
        print(f"{d} — added: {s['added']:>2} | done: {s['completed']:>2} | est mins: {s['est_minutes']:>3}")

def help_text():
    print(__doc__)

def main(argv):
    ensure_data()
    if len(argv) < 2:
        help_text(); return
    cmd = argv[1].lower()
    if cmd == "add" and len(argv) >= 3:
        text = argv[2]
        est = argv[3] if len(argv) >= 4 else 0
        add_task(text, est)
    elif cmd == "list":
        show_all = ("--all" in argv)
        list_tasks(show_all)
    elif cmd == "done" and len(argv) == 3:
        try:
            tid = int(argv[2])
            mark_done(tid)
        except ValueError:
            print("Task id must be a number.")
    elif cmd == "stats":
        stats()
    else:
        help_text()

if __name__ == "__main__":
    main(sys.argv)
