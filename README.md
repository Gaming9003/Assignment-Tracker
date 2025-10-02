import os, json, time
from flask import Flask, render_template, request, redirect, url_for, send_from_directory, flash
from werkzeug.utils import secure_filename

app = Flask(__name__)
app.secret_key = os.environ.get("SECRET_KEY", "assignment-tracker-secret")
UPLOAD_FOLDER = os.path.join(app.root_path, "uploads")
os.makedirs(UPLOAD_FOLDER, exist_ok=True)
DATA_FILE = os.path.join(app.root_path, "assignments.json")

def load_data():
    if os.path.exists(DATA_FILE):
        with open(DATA_FILE, "r", encoding="utf-8") as f:
            try:
                return json.load(f)
            except:
                return []
    return []

def save_data(data):
    with open(DATA_FILE, "w", encoding="utf-8") as f:
        json.dump(data, f, indent=4, ensure_ascii=False)

@app.route("/", methods=["GET", "POST"])
def index():
    data = load_data()
    if request.method == "POST":
        title = request.form.get("title","").strip()
        due_date = request.form.get("due_date","").strip()
        file = request.files.get("file")

        if not title or not due_date:
            flash("Title and due date are required.", "warning")
            return redirect(url_for("index"))

        filename = ""
        if file and file.filename:
            safe = secure_filename(file.filename)
            filename = f"{int(time.time())}_{safe}"
            file.save(os.path.join(UPLOAD_FOLDER, filename))

        data.append({
            "title": title,
            "due_date": due_date,
            "file": filename
        })
        save_data(data)
        flash("Assignment added.", "success")
        return redirect(url_for("index"))

    data = sorted(load_data(), key=lambda x: x.get("due_date",""))
    return render_template("index.html", assignments=data)

@app.route("/uploads/<path:filename>")
def uploaded_file(filename):
    return send_from_directory(UPLOAD_FOLDER, filename, as_attachment=True)

if __name__ == "__main__":
    # local debug server (not used by gunicorn in Render)
    port = int(os.environ.get("PORT", 5000))
    app.run(host="0.0.0.0", port=port, debug=True)
    
