---
title: Using MongoDB in the habit tracker
slug: using-mongodb-in-the-habit-tracker
tags:
  - Recorded
  - How to
categories:
  - Video
section_number: 10
excerpt: Transition to using MongoDB instead of storing data locally in variables for our habit tracker project.
draft: true
---

# Using MongoDB in the habit tracker

[[toc]]

## In this video... (TL;DR)

::: tip
List of all code changes made in this lecture: [https://diff-store.com/diff/section10__09_using_mongodb_in_project](https://diff-store.com/diff/section10__09_using_mongodb_in_project)
:::

We'll store habit data in MongoDB: a unique `_id` for each habit alongside the habit `name`.

MongoDB works well with Python's `datetime` objects, so we'll use that instead of `date`. That does mean we need to do a little bit more work to handle the time aspect, but it's not much and it makes database interactions much easier.

## Code written in this lecture

Let's begin by adding our `.env` file that will contain the MongoDB connection string. Make sure to provide your own connection string here:

```
MONGODB_URI=mongodb+srv://username:password@cluster0.cboqc.mongodb.net/tracker?retryWrites=true&w=majority
```

I'll also create a `.env.example` file to remind developers that they need to provide a `.env` file:

```
MONGODB_URI=
```

With this, we can then go to `app.py` and make a few changes:

- Import `os` for handling environment variables.
- Import `MongoClient`
- Import `load_dotenv` to load the `.env` file
- Use this in `create_app()` to connect to MongoDB and store the `MongoClient` in the app object.

```diff
--- app.py
+++ app.py
@@ -1,9 +1,16 @@
+import os
 from flask import Flask
 from routes import pages
+from pymongo import MongoClient
+from dotenv import load_dotenv
+
+load_dotenv()
 
 
 def create_app():
     app = Flask(__name__)
+    client = MongoClient(os.environ.get("MONGODB_URI"))
+    app.db = client.get_default_database()
+
     app.register_blueprint(pages)
     return app
```

Most of the other changes are in `routes.py`.

Since we'll be using the database, we can get rid of `defaultdict`, and the `habits` and `completions` variables. We'll need to import `uuid` and `current_app` from Flask.

In MongoDB we'll have two collections: one for the habits, and one for the completions.

The `habits` collection will have three fields:

- `_id`, a unique UUID to identify each habit
- `added`, a datetime field telling us when the habit was added to the database.
- `name`, the name of the habit as typed by the user.

The reason we're going to add an `added` field is so we don't show habits as non-completed before they were even added to our database.

The `completions` collection will also have three fields:

- `_id`, a unique ID generated by MongoDB. We won't be using this, but MongoDB generates it for us if we don't provide it.
- `date`, the date of the habit completion.
- `habit`, the unique ID of the habit that was completed on this date.

```diff
 import datetime
-from collections import defaultdict
-from flask import Blueprint, render_template, request, redirect, url_for
+import uuid
+from flask import Blueprint, request, redirect, url_for, render_template, current_app
 
 pages = Blueprint(
     "habits", __name__, template_folder="templates", static_folder="static"
 )
-habits = ["Test habit"]
-completions = defaultdict(list)
```

I'll go ahead and change the `date_range` function to take in a `datetime` object. I'll also create a `today_at_midnight` function that returns a `datetime` object with the time set to midnight. That's so we don't have to worry about the time aspect of things:

```diff

 @pages.context_processor
 def add_calc_date_range():
-    def date_range(start: datetime.date):
+    def date_range(start: datetime.datetime):
         dates = [start + datetime.timedelta(days=diff) for diff in range(-3, 4)]
         return dates
 
     return {"date_range": date_range}
 
 
+def today_at_midnight():
+    today = datetime.datetime.today()
+    return datetime.datetime(today.year, today.month, today.day)

```

Now let's handle adding new habits.

We want to include a unique ID for each habit, as well as when the habit was added to the database and the habit's name.

```diff
 @pages.route("/add", methods=["GET", "POST"])
 def add_habit():
+    today = today_at_midnight()
+
     if request.form:
-        habits.append(request.form.get("habit"))
+        current_app.db.habits.insert_one(
+            {"_id": uuid.uuid4().hex, "added": today, "name": request.form.get("habit")}
+        )
 
     return render_template(
-        "add_habit.html",
-        title="Habit Tracker - Add Habit",
-        selected_date=datetime.date.today(),
+        "add_habit.html", title="Habit Tracker - Add Habit", selected_date=today
     )
```

I'm using the `uuid` module to create the unique ID.

Next up, we have to make a few changes to the `index` route. We'll use `datetime` where we previously used `date`, and we'll access the database for data instead of the variables we had previously:

```diff
 @pages.route("/")
 def index():
     date_str = request.args.get("date")
     if date_str:
-        selected_date = datetime.date.fromisoformat(date_str)
+        selected_date = datetime.datetime.fromisoformat(date_str)
     else:
-        selected_date = datetime.date.today()
+        selected_date = today_at_midnight()
+
+    habits_on_date = current_app.db.habits.find({"added": {"$lte": selected_date}})
+    completions = [
+        habit["habit"]
+        for habit in current_app.db.completions.find({"date": selected_date})
+    ]
 
     return render_template(
         "index.html",
-        habits=habits,
+        habits=habits_on_date,
         selected_date=selected_date,
-        completions=completions[selected_date],
+        completions=completions,
         title="Habit Tracker - Home",
     )
```

Here I'm using MongoDB search filters, like `{"$lte": selected_date}`, to find those habits that were added today or earlier.

Then I'm querying the database for the completions on the `selected_date`, getting only the habit names back.

With that, we've got habits and completions, which we can then pass to our template. However, previously both our `habits` and `completions` lists were just strings.

Now, `habits` contains dictionaries and `completions` contains strings (the ID of each habit).

We have to make some small changes in our `index.html` template to account for this:

```diff
--- C:\Users\jose\Documents\projects\courses\new-python-web\curriculum\section10\lectures\09_using_mongodb_in_project\start\templates\index.html
+++ C:\Users\jose\Documents\projects\courses\new-python-web\curriculum\section10\lectures\09_using_mongodb_in_project\end\templates\index.html
@@ -3,11 +3,11 @@
 {% block main_content %}
     <section class="habit-list">
     {% for habit in habits %}
-        {% set completed = habit in completions %}
+        {% set completed = habit["_id"] in completions %}
         {% if completed %}
             <div class="habit completed">
                 <p class="habit__name">
-                    {{ habit }}
+                    {{ habit["name"] }}
                 </p>
                 <svg class="habit__icon" xmlns="http://www.w3.org/2000/svg" viewBox="0 0 20 20" fill="currentColor">
                     <path fill-rule="evenodd" d="M10 18a8 8 0 100-16 8 8 0 000 16zm3.707-9.293a1 1 0 00-1.414-1.414L9 10.586 7.707 9.293a1 1 0 00-1.414 1.414l2 2a1 1 0 001.414 0l4-4z" clip-rule="evenodd" />
@@ -16,10 +16,10 @@
         {% else %}
             <div class="habit">
                 <form method="POST" class="habit__form" action="{{ url_for('habits.complete') }}">
-                <input type="hidden" id="habitName" name="habitName" value="{{ habit }}" />
+                <input type="hidden" id="habitId" name="habitId" value="{{ habit['_id'] }}" />
                 <input type="hidden" id="date" name="date" value="{{ selected_date }}" />
                 <button type="submit" class="habit__button">
-                        {{ habit }}
+                        {{ habit["name"] }}
                 </button>
                 </form>
             </div>
```

Note that `completions` will contain habit IDs, so the form has to send habit IDs instead of habit names.

Let's deal with the `complete` endpoint next:

```diff
 def complete():
     date_string = request.form.get("date")
-    date = datetime.date.fromisoformat(date_string)
-    habit = request.form.get("habitName")
-    completions[date].append(habit)
+    date = datetime.datetime.fromisoformat(date_string)
+    habit = request.form.get("habitId")
+    current_app.db.completions.insert_one({"date": date, "habit": habit})
 
     return redirect(url_for(".index", date=date_string))
```

As you can see, not much going on there: we're getting the habit ID instead of the name, and inserting that along with the date into the database. Remember to also change your `datetime.date` to `datetime.datetime` since we're setting the time to midnight!

## Finished code changes

### In `app.py`

```diff
--- app.py
+++ app.py
@@ -1,9 +1,16 @@
+import os
 from flask import Flask
 from routes import pages
+from pymongo import MongoClient
+from dotenv import load_dotenv
+
+load_dotenv()
 
 
 def create_app():
     app = Flask(__name__)
+    client = MongoClient(os.environ.get("MONGODB_URI"))
+    app.db = client.get_default_database()
+
     app.register_blueprint(pages)
-
     return app
```

### In `routes.py`:

```diff
--- routes.py
+++ routes.py
@@ -1,36 +1,45 @@
 import datetime
-from collections import defaultdict
-from flask import Blueprint, render_template, request, redirect, url_for
+import uuid
+from flask import Blueprint, request, redirect, url_for, render_template, current_app
 
 pages = Blueprint(
     "habits", __name__, template_folder="templates", static_folder="static"
 )
-habits = ["Test habit"]
-completions = defaultdict(list)
 
 
 @pages.context_processor
 def add_calc_date_range():
-    def date_range(start: datetime.date):
+    def date_range(start: datetime.datetime):
         dates = [start + datetime.timedelta(days=diff) for diff in range(-3, 4)]
         return dates
 
     return {"date_range": date_range}
 
 
+def today_at_midnight():
+    today = datetime.datetime.today()
+    return datetime.datetime(today.year, today.month, today.day)
+
+
 @pages.route("/")
 def index():
     date_str = request.args.get("date")
     if date_str:
-        selected_date = datetime.date.fromisoformat(date_str)
+        selected_date = datetime.datetime.fromisoformat(date_str)
     else:
-        selected_date = datetime.date.today()
+        selected_date = today_at_midnight()
+
+    habits_on_date = current_app.db.habits.find({"added": {"$lte": selected_date}})
+    completions = [
+        habit["habit"]
+        for habit in current_app.db.completions.find({"date": selected_date})
+    ]
 
     return render_template(
         "index.html",
-        habits=habits,
+        habits=habits_on_date,
         selected_date=selected_date,
-        completions=completions[selected_date],
+        completions=completions,
         title="Habit Tracker - Home",
     )
 
@@ -39,19 +48,21 @@
 def complete():
     date_string = request.form.get("date")
-    date = datetime.date.fromisoformat(date_string)
-    habit = request.form.get("habitName")
-    completions[date].append(habit)
+    date = datetime.datetime.fromisoformat(date_string)
+    habit = request.form.get("habitId")
+    current_app.db.completions.insert_one({"date": date, "habit": habit})
 
     return redirect(url_for(".index", date=date_string))
 
 
 @pages.route("/add", methods=["GET", "POST"])
 def add_habit():
+    today = today_at_midnight()
+
     if request.form:
-        habits.append(request.form.get("habit"))
+        current_app.db.habits.insert_one(
+            {"_id": uuid.uuid4().hex, "added": today, "name": request.form.get("habit")}
+        )
 
     return render_template(
-        "add_habit.html",
-        title="Habit Tracker - Add Habit",
-        selected_date=datetime.date.today(),
+        "add_habit.html", title="Habit Tracker - Add Habit", selected_date=today
     )
```

### In `index.html`:

```diff
--- templates/index.html
+++ templates/index.html
@@ -3,11 +3,11 @@
 {% block main_content %}
     <section class="habit-list">
     {% for habit in habits %}
-        {% set completed = habit in completions %}
+        {% set completed = habit["_id"] in completions %}
         {% if completed %}
             <div class="habit completed">
                 <p class="habit__name">
-                    {{ habit }}
+                    {{ habit["name"] }}
                 </p>
                 <svg class="habit__icon" xmlns="http://www.w3.org/2000/svg" viewBox="0 0 20 20" fill="currentColor">
                     <path fill-rule="evenodd" d="M10 18a8 8 0 100-16 8 8 0 000 16zm3.707-9.293a1 1 0 00-1.414-1.414L9 10.586 7.707 9.293a1 1 0 00-1.414 1.414l2 2a1 1 0 001.414 0l4-4z" clip-rule="evenodd" />
@@ -16,10 +16,10 @@
         {% else %}
             <div class="habit">
                 <form method="POST" class="habit__form" action="{{ url_for('habits.complete') }}">
-                <input type="hidden" id="habitName" name="habitName" value="{{ habit }}" />
+                <input type="hidden" id="habitId" name="habitId" value="{{ habit['_id'] }}" />
                 <input type="hidden" id="date" name="date" value="{{ selected_date }}" />
                 <button type="submit" class="habit__button">
-                        {{ habit }}
+                        {{ habit["name"] }}
                 </button>
                 </form>
             </div>
```

### Environment files

Added `.env` and `.env.example` with `MONGODB_URI`.

