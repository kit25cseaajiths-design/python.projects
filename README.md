from flask import Flask, request, redirect, url_for, render_template_string
import sqlite3

app = Flask(__name__)

# DB setup
def get_db():
    conn = sqlite3.connect("students.db")
    conn.row_factory = sqlite3.Row
    return conn

def init_db():
    conn = get_db()
    conn.execute("""
    CREATE TABLE IF NOT EXISTS students (
        id INTEGER PRIMARY KEY AUTOINCREMENT,
        name TEXT,
        marks INTEGER
    )
    """)
    conn.commit()
    conn.close()

init_db()

# BASE TEMPLATE
base = """
<!DOCTYPE html>
<html>
<head>
<title>Student Tracker</title>
<link href="https://cdn.jsdelivr.net/npm/bootstrap@5.3.0/dist/css/bootstrap.min.css" rel="stylesheet">
</head>
<body class="bg-light">

<nav class="navbar navbar-dark bg-dark">
<div class="container">
<a class="navbar-brand" href="/">🎓 Student Tracker</a>
</div>
</nav>

<div class="container mt-4">
{{ content }}
</div>

</body>
</html>
"""

# HOME
@app.route('/')
def index():
    conn = get_db()
    students = conn.execute("SELECT * FROM students").fetchall()

    total = len(students)
    avg = sum([s['marks'] for s in students]) / total if total else 0
    top = max([s['marks'] for s in students]) if total else 0

    conn.close()

    content = f"""
    <h3>Dashboard</h3>

    <div class="row mb-3">
        <div class="col"><div class="card p-2">Total: {total}</div></div>
        <div class="col"><div class="card p-2">Average: {avg:.2f}</div></div>
        <div class="col"><div class="card p-2">Top: {top}</div></div>
    </div>

    <form action="/search" class="mb-3">
        <input name="q" class="form-control" placeholder="Search student">
    </form>

    <a href="/add" class="btn btn-primary mb-3">Add Student</a>

    <table class="table table-bordered">
    <tr><th>ID</th><th>Name</th><th>Marks</th><th>Action</th></tr>
    """

    for s in students:
        content += f"""
        <tr>
            <td>{s['id']}</td>
            <td>{s['name']}</td>
            <td>{s['marks']}</td>
            <td>
                <a href="/edit/{s['id']}" class="btn btn-warning btn-sm">Edit</a>
                <a href="/delete/{s['id']}" class="btn btn-danger btn-sm">Delete</a>
            </td>
        </tr>
        """

    content += "</table>"

    return render_template_string(base, content=content)

# ADD
@app.route('/add', methods=['GET','POST'])
def add():
    if request.method == 'POST':
        name = request.form['name']
        marks = request.form['marks']

        conn = get_db()
        conn.execute("INSERT INTO students (name, marks) VALUES (?,?)", (name, marks))
        conn.commit()
        conn.close()
        return redirect('/')

    content = """
    <h3>Add Student</h3>
    <form method="POST">
        <input name="name" class="form-control mb-2" placeholder="Name">
        <input name="marks" type="number" class="form-control mb-2" placeholder="Marks">
        <button class="btn btn-success">Add</button>
    </form>
    """

    return render_template_string(base, content=content)

# EDIT
@app.route('/edit/<int:id>', methods=['GET','POST'])
def edit(id):
    conn = get_db()

    if request.method == 'POST':
        name = request.form['name']
        marks = request.form['marks']
        conn.execute("UPDATE students SET name=?, marks=? WHERE id=?", (name, marks, id))
        conn.commit()
        conn.close()
        return redirect('/')

    student = conn.execute("SELECT * FROM students WHERE id=?", (id,)).fetchone()
    conn.close()

    content = f"""
    <h3>Edit Student</h3>
    <form method="POST">
        <input name="name" value="{student['name']}" class="form-control mb-2">
        <input name="marks" type="number" value="{student['marks']}" class="form-control mb-2">
        <button class="btn btn-success">Update</button>
    </form>
    """

    return render_template_string(base, content=content)

# DELETE
@app.route('/delete/<int:id>')
def delete(id):
    conn = get_db()
    conn.execute("DELETE FROM students WHERE id=?", (id,))
    conn.commit()
    conn.close()
    return redirect('/')

# SEARCH
@app.route('/search')
def search():
    q = request.args.get('q')
    conn = get_db()
    students = conn.execute("SELECT * FROM students WHERE name LIKE ?", ('%' + q + '%',)).fetchall()
    conn.close()

    content = "<h3>Search Results</h3><a href='/' class='btn btn-secondary mb-2'>Back</a><table class='table'>"
    content += "<tr><th>ID</th><th>Name</th><th>Marks</th></tr>"

    for s in students:
        content += f"<tr><td>{s['id']}</td><td>{s['name']}</td><td>{s['marks']}</td></tr>"

    content += "</table>"

    return render_template_string(base, content=content)

# RUN
if __name__ == "__main__":
    app.run(debug=True)
