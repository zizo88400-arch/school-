# school-
تعمل
flask
gunicorn
flask-sqlalchemy
flask-login
flask-wtf
psycopg[binary]
import os, uuid
from datetime import datetime
from flask import (Flask, render_template, request, redirect, url_for,
                   flash, send_from_directory, abort)
from flask_sqlalchemy import SQLAlchemy
from flask_login import (LoginManager, UserMixin, login_user, logout_user,
                         login_required, current_user)
from flask_wtf.csrf import CSRFProtect
from werkzeug.security import generate_password_hash, check_password_hash
from werkzeug.utils import secure_filename
from jinja2 import DictLoader

app = Flask(__name__)
BASE = os.path.dirname(os.path.abspath(__file__))

db_url = os.environ.get("DATABASE_URL", "")
if db_url.startswith("postgres://"):
    db_url = db_url.replace("postgres://", "postgresql+psycopg://", 1)
elif db_url.startswith("postgresql://"):
    db_url = db_url.replace("postgresql://", "postgresql+psycopg://", 1)
if not db_url:
    db_url = "sqlite:///" + os.path.join(BASE, "platform.db")

app.config["SECRET_KEY"] = os.environ.get("SECRET_KEY", "change-me-please")
app.config["SQLALCHEMY_DATABASE_URI"] = db_url
app.config["MAX_CONTENT_LENGTH"] = 20 * 1024 * 1024

UPLOADS = os.path.join(BASE, "uploads")
os.makedirs(UPLOADS, exist_ok=True)
ALLOWED = {"pdf", "doc", "docx", "ppt", "pptx", "txt", "png", "jpg", "jpeg", "zip"}

db = SQLAlchemy(app)
CSRFProtect(app)
login_manager = LoginManager(app)
login_manager.login_view = "login"
login_manager.login_message = "سجّل دخولك الأول"


class User(UserMixin, db.Model):
    __tablename__ = "users"
    id = db.Column(db.Integer, primary_key=True)
    name = db.Column(db.String(100), nullable=False)
    email = db.Column(db.String(150), unique=True, nullable=False)
    password_hash = db.Column(db.String(300), nullable=False)
    role = db.Column(db.String(20), default="student")


class Lesson(db.Model):
    __tablename__ = "lessons"
    id = db.Column(db.Integer, primary_key=True)
    title = db.Column(db.String(200), nullable=False)
    description = db.Column(db.Text, default="")
    filename = db.Column(db.String(300))
    orig_name = db.Column(db.String(300))
    created_at = db.Column(db.DateTime, default=datetime.utcnow)
    teacher_id = db.Column(db.Integer, db.ForeignKey("users.id"))
    teacher = db.relationship("User")


@login_manager.user_loader
def load_user(uid):
    return db.session.get(User, int(uid))


TEMPLATES = {
"base.html": """<!doctype html><html lang="ar" dir="rtl"><head><meta charset="utf-8">
<meta name="viewport" content="width=device-width,initial-scale=1">
<title>منصة الطالب والمعلم</title>
<style>body{font-family:sans-serif;background:#f3f4f6;margin:0}
nav{background:#1e40af;padding:12px}nav a{color:#fff;margin-left:14px;text-decoration:none}
.box{max-width:720px;margin:auto;padding:14px}
.card{background:#fff;border-radius:10px;padding:12px;margin:10px 0}
input,textarea,select,button{width:100%;padding:10px;margin:5px 0;box-sizing:border-box}
button{background:#2563eb;color:#fff;border:0;border-radius:8px}
.red{background:#dc2626}.msg{background:#fef3c7;padding:10px;border-radius:8px;margin:8px 0}
small{color:#6b7280}</style></head><body>
<nav><a href="/">الرئيسية</a>
{% if current_user.is_authenticated %}
{% if current_user.role in ['teacher','admin'] %}<a href="/new">درس جديد</a>{% endif %}
{% if current_user.role == 'admin' %}<a href="/admin">الإدارة</a>{% endif %}
<a href="/logout">خروج</a>
{% else %}<a href="/login">دخول</a><a href="/register">حساب جديد</a>{% endif %}
</nav><div class="box">
{% for m in get_flashed_messages() %}<div class="msg">{{ m }}</div>{% endfor %}
{% block body %}{% endblock %}</div></body></html>""",

"index.html": """{% extends "base.html" %}{% block body %}
<h2>الدروس والملفات</h2>
{% for l in lessons %}<div class="card"><b>{{ l.title }}</b>
<p>{{ l.description }}</p>
<small>المعلم: {{ l.teacher.name if l.teacher else '-' }} | {{ l.created_at.strftime('%Y-%m-%d') }}</small>
{% if l.filename %}<p>{% if current_user.is_authenticated %}<a href="/download/{{ l.id }}">تحميل الملف</a>{% else %}<small>سجّل دخول لتحميل الملف</small>{% endif %}</p>{% endif %}
{% if current_user.is_authenticated and (current_user.id == l.teacher_id or current_user.role == 'admin') %}
<form method="post" action="/delete/{{ l.id }}"><input type="hidden" name="csrf_token" value="{{ csrf_token() }}"><button class="red">حذف</button></form>{% endif %}
</div>{% else %}<p>لا توجد دروس بعد.</p>{% endfor %}{% endblock %}""",

"register.html": """{% extends "base.html" %}{% block body %}
<h2>حساب جديد (طالب)</h2><form method="post">
<input type="hidden" name="csrf_token" value="{{ csrf_token() }}">
<input name="name" placeholder="الاسم" required>
<input name="email" type="email" placeholder="الإيميل" required>
<input name="password" type="password" placeholder="كلمة السر (6 حروف على الأقل)" required>
<button>إنشاء الحساب</button></form>{% endblock %}""",

"login.html": """{% extends "base.html" %}{% block body %}
<h2>تسجيل الدخول</h2><form method="post">
<input type="hidden" name="csrf_token" value="{{ csrf_token() }}">
<input name="email" type="email" placeholder="الإيميل" required>
<input name="password" type="password" placeholder="كلمة السر" required>
<button>دخول</button></form>{% endblock %}""",

"new.html": """{% extends "base.html" %}{% block body %}
<h2>درس جديد</h2><form method="post" enctype="multipart/form-data">
<input type="hidden" name="csrf_token" value="{{ csrf_token() }}">
<input name="title" placeholder="عنوان الدرس" required>
<textarea name="description" rows="4" placeholder="الوصف"></textarea>
<input type="file" name="file">
<button>نشر</button></form>{% endblock %}""",

"admin.html": """{% extends "base.html" %}{% block body %}
<h2>إدارة المستخدمين</h2>
{% for u in users %}<div class="card"><b>{{ u.name }}</b> <small>{{ u.email }} | {{ u.role }}</small>
{% if u.role != 'admin' %}<form method="post">
<input type="hidden" name="csrf_token" value="{{ csrf_token() }}">
<input type="hidden" name="uid" value="{{ u.id }}">
<select name="role"><option value="student">طالب</option><option value="teacher">معلم</option></select>
<button>تغيير الدور</button></form>{% endif %}</div>{% endfor %}{% endblock %}""",
}
app.jinja_env.loader = DictLoader(TEMPLATES)


@app.route("/")
def index():
    lessons = Lesson.query.order_by(Lesson.created_at.desc()).all()
    return render_template("index.html", lessons=lessons)


@app.route("/register", methods=["GET", "POST"])
def register():
    if request.method == "POST":
        name = request.form.get("name", "").strip()
        email = request.form.get("email", "").strip().lower()
        pw = request.form.get("password", "")
        if not name or not email or len(pw) < 6:
            flash("املا كل الخانات، وكلمة السر 6 حروف على الأقل")
        elif User.query.filter_by(email=email).first():
            flash("الإيميل ده مسجل قبل كده")
        else:
            db.session.add(User(name=name, email=email,
                                password_hash=generate_password_hash(pw)))
            db.session.commit()
            flash("تم إنشاء الحساب، سجّل دخولك")
            return redirect(url_for("login"))
    return render_template("register.html")


@app.route("/login", methods=["GET", "POST"])
def login():
    if request.method == "POST":
        email = request.form.get("email", "").strip().lower()
        u = User.query.filter_by(email=email).first()
        if u and check_password_hash(u.password_hash, request.form.get("password", "")):
            login_user(u)
            return redirect(url_for("index"))
        flash("الإيميل أو كلمة السر غلط")
    return render_template("login.html")


@app.route("/logout")
def logout():
    logout_user()
    return redirect(url_for("index"))


@app.route("/new", methods=["GET", "POST"])
@login_required
def new():
    if current_user.role not in ("teacher", "admin"):
        abort(403)
    if request.method == "POST":
        title = request.form.get("title", "").strip()
        desc = request.form.get("description", "").strip()
        f = request.files.get("file")
        fname = orig = None
        if f and f.filename:
            ext = f.filename.rsplit(".", 1)[-1].lower() if "." in f.filename else ""
            if ext not in ALLOWED:
                flash("نوع الملف غير مسموح")
                return render_template("new.html")
            orig = secure_filename(f.filename) or ("file." + ext)
            fname = uuid.uuid4().hex + "." + ext
            f.save(os.path.join(UPLOADS, fname))
        if title:
            db.session.add(Lesson(title=title, description=desc, filename=fname,
                                  orig_name=orig, teacher_id=current_user.id))
            db.session.commit()
            flash("تم نشر الدرس")
            return redirect(url_for("index"))
    return render_template("new.html")


@app.route("/download/<int:lid>")
@login_required
def download(lid):
    l = db.get_or_404(Lesson, lid)
    if not l.filename:
        abort(404)
    return send_from_directory(UPLOADS, l.filename, as_attachment=True,
                               download_name=l.orig_name)


@app.route("/delete/<int:lid>", methods=["POST"])
@login_required
def delete(lid):
    l = db.get_or_404(Lesson, lid)
    if current_user.id != l.teacher_id and current_user.role != "admin":
        abort(403)
    db.session.delete(l)
    db.session.commit()
    flash("تم الحذف")
    return redirect(url_for("index"))


@app.route("/admin", methods=["GET", "POST"])
@login_required
def admin():
    if current_user.role != "admin":
        abort(403)
    if request.method == "POST":
        u = db.session.get(User, int(request.form.get("uid", 0) or 0))
        role = request.form.get("role")
        if u and u.role != "admin" and role in ("student", "teacher"):
            u.role = role
            db.session.commit()
            fl
