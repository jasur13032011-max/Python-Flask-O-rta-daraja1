# Python-Flask-O-rta-daraja1
Siz aytib o'tgan barcha texnik shartlar va mezonlarni o'z ichiga olgan, to'liq ishlaydigan Flask loyiha arxitekturasini taqdim etaman. Loyiha modulli tizimda (Blueprints) tashkil qilingan bo'lib, uning tarkibida modellar, dekoratorlar, marshrutlar va README yo'riqnomasi mavjud.

1. Modellar Muhiti (app/models.py)
Foydalanuvchi va Post modellari o'rtasidagi munosabat db.relationship orqali bog'langan.

Python
from flask_sqlalchemy import SQLAlchemy
from flask_login import UserMixin
from werkzeug.security import generate_password_hash, check_password_hash

db = SQLAlchemy()

class User(db.Model, UserMixin):
    __tablename__ = 'users'
    
    id = db.Column(db.Integer, primary_key=True)
    username = db.Column(db.String(80), unique=True, nullable=False)
    email = db.Column(db.String(120), unique=True, nullable=False)
    password_hash = db.Column(db.String(255), nullable=False)
    role = db.Column(db.String(20), default='user', nullable=False) # default 'user'
    
    # Post modeli bilan aloqa
    posts = db.relationship('Post', backref='author', lazy=True, cascade="all, delete-orphan")

    def set_password(self, raw_password):
        self.password_hash = generate_password_hash(raw_password)

    def check_password(self, raw_password):
        return check_password_hash(self.password_hash, raw_password)

    @property
    def is_admin(self):
        return self.role == 'admin'


class Post(db.Model):
    __tablename__ = 'posts'
    
    id = db.Column(db.Integer, primary_key=True)
    title = db.Column(db.String(200), nullable=False)
    content = db.Column(db.Text, nullable=False)
    user_id = db.Column(db.Integer, db.ForeignKey('users.id'), nullable=False)
2. Maxsus Dekoratorlar (app/decorators.py)
Admin huquqlarini tekshirish uchun server darajasidagi qat'iy dekorator:

Python
from functools import wraps
from flask import abort
from flask_login import current_user

def admin_required(f):
    @wraps(f)
    def decorated_function(*args, **kwargs):
        if not current_user.is_authenticated:
            abort(401)
        if not current_user.is_admin:
            abort(403)
        return f(*args, **kwargs)
    return decorated_function
3. Autentifikatsiya Kontrolleri (app/auth.py)
Ro'yxatdan o'tish (min 8 belgi), login (username/email + remember me) va ?next=... parametrlari boshqaruvi:

Python
from flask import Blueprint, render_template, request, redirect, url_for, flash, abort
from flask_login import login_user, logout_user, login_required, current_user
from app.models import db, User
from urllib.parse import urlparse

auth = Blueprint('auth', __name__)

@auth.route('/auth/register', methods=['GET', 'POST'])
def register():
    if request.method == 'POST':
        username = request.form.get('username', '').strip()
        email = request.form.get('email', '').strip()
        password = request.form.get('password', '')

        if not username or not email or len(password) < 8:
            flash("Validatsiya xatosi: Ma'lumotlarni to'liq kiriting (Parol min 8 belgi)!", "danger")
            return render_template('auth/register.html'), 400

        if User.query.filter((User.username == username) | (User.email == email)).first():
            flash("Username yoki Email allaqachon mavjud!", "danger")
            return render_template('auth/register.html'), 400

        user = User(username=username, email=email)
        user.set_password(password)
        db.session.add(user)
        db.session.commit()
        
        flash("Muvaffaqiyatli ro'yxatdan o'tdingiz!", "success")
        return redirect(url_for('auth.login'))
        
    return render_template('auth/register.html')

@auth.route('/auth/login', methods=['GET', 'POST'])
def login():
    if request.method == 'POST':
        login_input = request.form.get('login_input', '').strip()
        password = request.form.get('password', '')
        remember = True if request.form.get('remember') else False

        user = User.query.filter((User.username == login_input) | (User.email == login_input)).first()

        if user and user.check_password(password):
            login_user(user, remember=remember)
            
            # ?next=... sahifasini xavfsiz tekshirish
            next_page = request.args.get('next')
            if not next_page or urlparse(next_page).netloc != '':
                next_page = url_for('main.index')
                
            return redirect(next_page)
            
        flash("Login yoki parol xato!", "danger")
    return render_template('auth/login.html')

@auth.route('/auth/logout')
@login_required
def logout():
    logout_user()
    return redirect(url_for('main.index'))
4. Postlar Kontrolleri (app/posts.py)
Ega yoki admin ekanligini tekshirish mexanizmiga ega post boshqaruvi:

Python
from flask import Blueprint, render_template, request, redirect, url_for, abort, flash
from flask_login import login_required, current_user
from app.models import db, Post

posts_bp = Blueprint('posts', __name__)

@posts_bp.route('/posts/new', methods=['GET', 'POST'])
@login_required
def new_post():
    if request.method == 'POST':
        title = request.form.get('title')
        content = request.form.get('content')
        
        post = Post(title=title, content=content, user_id=current_user.id)
        db.session.add(post)
        db.session.commit()
        return redirect(url_for('main.index'))
    return render_template('posts/new.html')

@posts_bp.route('/posts/<int:post_id>/edit', methods=['GET', 'POST'])
@login_required
def edit_post(post_id):
    post = Post.query.get_or_404(post_id)
    
    # SERVER-SIDE TEKSHIRUV: Ega yoki Admin
    if post.user_id != current_user.id and not current_user.is_admin:
        abort(403)
        
    if request.method == 'POST':
        post.title = request.form.get('title')
        post.content = request.form.get('content')
        db.session.commit()
        return redirect(url_for('main.index'))
        
    return render_template('posts/edit.html', post=post)

@posts_bp.route('/posts/<int:post_id>/delete', methods=['POST'])
@login_required
def delete_post(post_id):
    post = Post.query.get_or_404(post_id)
    
    # SERVER-SIDE TEKSHIRUV: Ega yoki Admin
    if post.user_id != current_user.id and not current_user.is_admin:
        abort(403)
        
    db.session.delete(post)
    db.session.commit()
    return redirect(url_for('main.index'))
5. Admin Kontrolleri (app/admin.py)
Python
from flask import Blueprint, render_template, request, redirect, url_for, flash
from app.models import db, User
from app.decorators import admin_required

admin_bp = Blueprint('admin', __name__)

@admin_bp.route('/admin/users')
@admin_required
def list_users():
    users = User.query.all()
    return render_template('admin/users.html', users=users)

@admin_bp.route('/admin/users/<int:user_id>/change-role', methods=['POST'])
@admin_required
def change_role(user_id):
    user = User.query.get_or_404(user_id)
    new_role = request.form.get('role')
    if new_role in ['user', 'admin']:
        user.role = new_role
        db.session.commit()
        flash(f"{user.username} roli yangilandi!", "success")
    return redirect(url_for('admin.list_users'))
6. Xatolik Sahifalari va Global Error Handler
Ilova asosiy faylida (app/__init__.py) xatoliklarni chiroyli sahifalarga yo'naltiramiz:

Python
from flask import Flask, render_template

def create_app():
    app = Flask(__name__)
    
    # ... konfiguratsiyalar va blueprint registratsiyalari ...

    @app.errorhandler(401)
    def unauthorized(error):
        return render_template('errors/401.html'), 401

    @app.errorhandler(403)
    def forbidden(error):
        return render_template('errors/403.html'), 403

    return app
7. Shablonda Navigatsiya Tekshiruvi (base.html)
HTML
<nav>
    <a href="/">Bosh sahifa</a>
    {% if current_user.is_authenticated %}
        <span>Salom, {{ current_user.username }}</span>
        <a href="{{ url_for('posts.new_post') }}">Yangi Post</a>
        
        {% if current_user.is_admin %}
            <a href="{{ url_for('admin.list_users') }}" style="color: red;">Admin Panel</a>
        {% endif %}
        
        <a href="{{ url_for('auth.logout') }}">Chiqish</a>
    {% else %}
        <a href="{{ url_for('auth.login') }}">Kirish</a>
        <a href="{{ url_for('auth.register') }}">Ro'yxatdan o'tish</a>
    {% endif %}
</nav>
8. Loyiha bo'yicha Yo'riqnoma (README.md)
Markdown
# Flask Autentifikatsiya va Rol Tizimi Loyihasi

### Admin foydalanuvchi yaratish (Flask Shell orqali)

Tizimga birinchi admin foydalanuvchini qo'shish yoki mavjud foydalanuvchini admin roliga o'tkazish uchun terminalda quyidagi buyruqlarni bajaring:

```bash
# 1. Flask shell muhitini oching
flask shell
Ochilgan interfeysga quyidagi Python kodini kiriting:

Python
from app.models import db, User

# Yangi admin foydalanuvchi qo'shish:
admin = User(username="superadmin", email="admin@example.com", role="admin")
admin.set_password("JudaKuchliParol123!")
db.session.add(admin)
db.session.commit()

print("Admin foydalanuvchi muvaffaqiyatli yaratildi!")
exit()
