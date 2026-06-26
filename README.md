# Python-Flask-O-rta-daraja1
1. Loyiha Tuzilishi (Directory Structure)
Plaintext
my_flask_app/
│
├── app/
│   ├── __init__.py          # Application Factory
│   ├── config.py            # Konfiguratsiyalar
│   ├── models.py            # Ma'lumotlar bazasi modellari
│   ├── decorators.py        # @admin_required va boshqalar
│   ├── utils.py             # imghdr va email yordamchi funksiyalari
│   │
│   ├── main/                # Main Blueprint
│   │   ├── __init__.py
│   │   └── routes.py
│   ├── auth/                # Auth Blueprint
│   │   ├── __init__.py
│   │   ├── forms.py
│   │   └── routes.py
│   ├── posts/               # Posts Blueprint
│   │   ├── __init__.py
│   │   ├── forms.py
│   │   └── routes.py
│   └── api/                 # API Blueprint
│       ├── __init__.py
│       ├── errors.py
│       └── routes.py
│
├── templates/               # HTML shablonlar
├── static/                  # Statik fayllar (CSS, JS)
├── uploads/                 # Static'dan tashqaridagi yuklanmalar papkasi
├── migrations/              # Flask-Migrate papkasi
├── .env
├── .env.example
├── .gitignore
├── README.md
├── requirements.txt
└── run.py                   # Kirish nuqtasi
2. Core Konfiguratsiya va Factory
app/config.py
Python
import os
from dotenv import load_dotenv

load_dotenv()
BASE_DIR = os.path.abspath(os.path.dirname(os.path.dirname(__file__)))

class Config:
    SECRET_KEY = os.getenv('SECRET_KEY', 'default-key-for-dev')
    SQLALCHEMY_DATABASE_URI = os.getenv('DATABASE_URL', f"sqlite:///{os.path.join(BASE_DIR, 'app.db')}")
    SQLALCHEMY_TRACK_MODIFICATIONS = False
    
    # Mail config
    MAIL_SERVER = os.getenv('MAIL_SERVER', 'smtp.gmail.com')
    MAIL_PORT = int(os.getenv('MAIL_PORT', 587))
    MAIL_USE_TLS = os.getenv('MAIL_USE_TLS', 'True') == 'True'
    MAIL_USERNAME = os.getenv('MAIL_USERNAME')
    MAIL_PASSWORD = os.getenv('MAIL_PASSWORD')
    MAIL_DEFAULT_SENDER = os.getenv('MAIL_DEFAULT_SENDER')
    
    # Uploads config
    MAX_CONTENT_LENGTH = 2 * 1024 * 1024  # Max 2MB
    UPLOAD_FOLDER = os.path.join(BASE_DIR, 'uploads')
app/__init__.py
Python
from flask import Flask, render_template
from flask_sqlalchemy import SQLAlchemy
from flask_migrate import Migrate
from flask_login import LoginManager
from flask_mail import Mail
from flask_wtf.csrf import CSRFProtect
from app.config import Config

db = SQLAlchemy()
migrate = Migrate()
login_manager = LoginManager()
mail = Mail()
csrf = CSRFProtect()

def create_app(config_class=Config):
    app = Flask(__name__, template_folder='../templates', static_folder='../static')
    app.config.from_object(config_class)

    # Init Extensions
    db.init_app(app)
    migrate.init_app(app, db)
    login_manager.init_app(app)
    mail.init_app(app)
    csrf.init_app(app)

    login_manager.login_view = 'auth.login'
    login_manager.login_message_category = 'info'

    # API ni CSRF himoyasidan chetlatish
    csrf.exempt(api_bp) # Blueprint import qilingach quyida exemption beriladi

    # Register Blueprints
    from app.main.routes import main_bp
    from app.auth.routes import auth_bp
    from app.posts.routes import posts_bp
    from app.api.routes import api_bp

    app.register_blueprint(main_bp)
    app.register_blueprint(auth_bp)
    app.register_blueprint(posts_bp)
    app.register_blueprint(api_bp)
    
    csrf.exempt(api_bp)

    # Global 413 error handler
    @app.errorhandler(413)
    def request_entity_too_large(error):
        return render_template('errors/413.html'), 413

    return app
3. Ma'lumotlar Bazasi Modellari (app/models.py)
Many-to-Many, One-to-Many munosabatlari va Flask-Login integratsiyasi:

Python
from datetime import datetime
from app import db, login_manager
from flask_login import UserMixin
from werkzeug.security import generate_password_hash, check_password_hash

@login_manager.user_loader
def load_user(user_id):
    return User.query.get(int(user_id))

# Many-to-Many yordamchi jadvali (Post <-> Tag)
post_tags = db.Table('post_tags',
    db.Column('post_id', db.Integer, db.ForeignKey('posts.id', ondelete='CASCADE'), primary_key=True),
    db.Column('tag_id', db.Integer, db.ForeignKey('tags.id', ondelete='CASCADE'), primary_key=True)
)

class User(db.Model, UserMixin):
    __tablename__ = 'users'
    id = db.Column(db.Integer, primary_key=True)
    username = db.Column(db.String(80), unique=True, nullable=False)
    email = db.Column(db.String(120), unique=True, nullable=False)
    password_hash = db.Column(db.String(255), nullable=False)
    avatar = db.Column(db.String(255), nullable=True, default='default.png')
    is_admin = db.Column(db.Boolean, default=False)
    
    posts = db.relationship('Post', backref='author', lazy=True, cascade="all, delete-orphan")
    comments = db.relationship('Comment', backref='author', lazy=True, cascade="all, delete-orphan")

    def set_password(self, password):
        self.password_hash = generate_password_hash(password)
        
    def check_password(self, password):
        return check_password_hash(self.password_hash, password)

class Post(db.Model):
    __tablename__ = 'posts'
    id = db.Column(db.Integer, primary_key=True)
    title = db.Column(db.String(255), nullable=False)
    body = db.Column(db.Text, nullable=False)
    created_at = db.Column(db.DateTime, default=datetime.utcnow)
    user_id = db.Column(db.Integer, db.ForeignKey('users.id', ondelete='CASCADE'), nullable=False)
    
    comments = db.relationship('Comment', backref='post', lazy=True, cascade="all, delete-orphan")
    tags = db.relationship('Tag', secondary=post_tags, backref=db.backref('posts', lazy='dynamic'))

    def to_dict(self):
        return {
            'id': self.id,
            'title': self.title,
            'body': self.body,
            'created_at': self.created_at.isoformat(),
            'author': self.author.username,
            'tags': [tag.name for tag in self.tags]
        }

class Tag(db.Model):
    __tablename__ = 'tags'
    id = db.Column(db.Integer, primary_key=True)
    name = db.Column(db.String(50), unique=True, nullable=False)

class Comment(db.Model):
    __tablename__ = 'comments'
    id = db.Column(db.Integer, primary_key=True)
    body = db.Column(db.Text, nullable=False)
    created_at = db.Column(db.DateTime, default=datetime.utcnow)
    user_id = db.Column(db.Integer, db.ForeignKey('users.id', ondelete='CASCADE'), nullable=False)
    post_id = db.Column(db.Integer, db.ForeignKey('posts.id', ondelete='CASCADE'), nullable=False)
4. Admin Dekoratori (app/decorators.py)
Faqat admin huquqiga ega foydalanuvchilar kira olishini ta'minlovchi dekorator:

Python
from functools import wraps
from flask import abort
from flask_login import current_user

def admin_required(f):
    @wraps(f)
    def decorated_function(*args, **kwargs):
        if not current_user.is_authenticated or not current_user.is_admin:
            abort(403)  # Forbidden
        return f(*args, **kwargs)
    return decorated_function
Aniq marshrutda qo'llash misoli (/admin/users uchun):

Python
@main_bp.route('/admin/users')
@admin_required
def admin_users():
    users = User.query.all()
    return render_template('admin/users.html', users=users)
5. REST API Paginatsiya, Qidiruv va Saralash (app/api/routes.py)
Dinamik JSON error handlerlar va whitelist asosida ishlovchi mukammal API tizimi:

Python
from flask import Blueprint, jsonify, request, url_for
from app import db
from app.models import Post
from sqlalchemy import or_

api_bp = Blueprint('api', __name__, url_prefix='/api')

# --- JSON Error Handlers ---
@api_bp.app_errorhandler(400)
def bad_request(e):
    return jsonify({'error': 'Bad Request', 'message': getattr(e, 'description', '')}), 400

@api_bp.app_errorhandler(404)
def not_found(e):
    return jsonify({'error': 'Not Found', 'message': 'Resource could not be found'}), 404

@api_bp.app_errorhandler(500)
def server_error(e):
    return jsonify({'error': 'Internal Server Error', 'message': 'An unexpected error occurred'}), 500


# --- Endpoints ---
@api_bp.route('/posts', methods=['GET'])
def get_posts():
    # Paginatsiya
    page = request.args.get('page', 1, type=int)
    per_page = request.args.get('per_page', 20, type=int)
    if per_page > 50: per_page = 50

    query = Post.query

    # Qidiruv (?q=)
    search_q = request.args.get('q', '').strip()
    if search_q:
        query = query.filter(or_(Post.title.ilike(f"%{search_q}%"), Post.body.ilike(f"%{search_q}%")))

    # Saralash (Whitelist)
    sort_by = request.args.get('sort', 'created_at')
    order = request.args.get('order', 'desc')
    
    sort_whitelist = {'id': Post.id, 'title': Post.title, 'created_at': Post.created_at}
    sort_column = sort_whitelist.get(sort_by, Post.created_at)
    
    if order == 'asc':
        query = query.order_by(sort_column.asc())
    else:
        query = query.order_by(sort_column.desc())

    pagination = query.paginate(page=page, per_page=per_page, error_out=False)
    
    def get_url(p):
        if p is None: return None
        args = request.args.to_dict()
        args.update({'page': p, 'per_page': per_page})
        return url_for('api.get_posts', _external=True, **args)

    return jsonify({
        'items': [post.to_dict() for post in pagination.items],
        'meta': {
            'page': pagination.page,
            'per_page': pagination.per_page,
            'total': pagination.total,
            'pages': pagination.pages,
            'has_next': pagination.has_next,
            'has_prev': pagination.has_prev
        },
        'links': {
            'self': get_url(pagination.page),
            'next': get_url(pagination.next_num),
            'prev': get_url(pagination.prev_num)
        }
    }), 200

@api_bp.route('/posts', methods=['POST'])
def create_post():
    data = request.get_json(silent=True) or {}
    if 'title' not in data or 'body' not in data or not data['title'].strip():
        return jsonify({'error': 'Validation Error', 'message': 'title and body are required'}), 400
    
    post = Post(title=data['title'], body=data['body'], user_id=1) # Default test user id
    db.session.add(post)
    db.session.commit()
    
    response = jsonify(post.to_dict())
    response.headers['Location'] = url_for('api.get_post', post_id=post.id, _external=True)
    return response, 201

@api_bp.route('/posts/<int:post_id>', methods=['GET'])
def get_post(post_id):
    post = Post.query.get_or_404(post_id)
    return jsonify(post.to_dict()), 200

@api_bp.route('/posts/<int:post_id>', methods=['PUT'])
def update_post(post_id):
    post = Post.query.get_or_404(post_id)
    data = request.get_json(silent=True) or {}
    
    if 'title' in data: post.title = data['title']
    if 'body' in data: post.body = data['body']
    
    db.session.commit()
    return jsonify(post.to_dict()), 200

@api_bp.route('/posts/<int:post_id>', methods=['DELETE'])
def delete_post(post_id):
    post = Post.query.get_or_404(post_id)
    db.session.delete(post)
    db.session.commit()
    return '', 204
6. Proyekt Hujjatlari (README.md)
Loyihaning ildiz katalogida joylashishi kerak boʻlgan tayyor README.md fayli tarkibi:

Markdown
# Advanced Flask Application Factory Framework

Ushbu loyiha production talablariga javob beradigan, to'liq konfiguratsiya qilingan Flask ilovasi shablonidir.

## Jonli Demo (Live Demo)
Ilova quyidagi havolada joylashtirilgan: [https://your-app-demo.railway.app](https://your-app-demo.railway.app)

## O'rnatish va Ishga tushirish qadamlari

1. **Repozitoriyani yuklab oling va ichiga kiring:**
   ```bash
   git clone [https://github.com/username/repo-name.git](https://github.com/username/repo-name.git)
   cd repo-name
Virtual muhitni yarating va faollashtiring:

Bash
python3 -m venv venv
source venv/bin/activate  # Windows uchun: venv\Scripts\activate
Kerakli paketlarni o'rnating:

Bash
pip install -r requirements.txt
Muhit o'zgaruvchilarini sozlang:

Bash
cp .env.example .env
# .env faylini ochib, kerakli parollar va email sozlamalarini kiriting.
Ma'lumotlar bazasini migratsiya qiling:

Bash
flask db init
flask db migrate -m "Initial migration with Users, Posts, Tags, Comments."
flask db upgrade
Ilovani ishga tushiring:

Bash
python run.py
REST API cURL Misollari
1. Barcha postlarni saralash va qidiruv bilan olish (GET)
Bash
curl -G "[http://127.0.0.1:5000/api/posts](http://127.0.0.1:5000/api/posts)" \
     --data-urlencode "q=flask" \
     --data-urlencode "page=1" \
     --data-urlencode "per_page=5" \
     --data-urlencode "sort=title" \
     --data-urlencode "order=asc"
2. Yangi post yaratish (POST)
Bash
curl -X POST [http://127.0.0.1:5000/api/posts](http://127.0.0.1:5000/api/posts) \
     -H "Content-Type: application/json" \
     -d '{"title": "Yangi Texnologiyalar", "body": "Flask mukammal framework hisoblanadi."}'
3. Postni yangilash (PUT)
Bash
curl -X PUT [http://127.0.0.1:5000/api/posts/1](http://127.0.0.1:5000/api/posts/1) \
     -H "Content-Type: application/json" \
     -d '{"title": "Yangilangan Sarlavha"}'
4. Postni o'chirish (DELETE)
Bash
curl -X DELETE [http://127.0.0.1:5000/api/posts/1](http://127.0.0.1:5000/api/posts/1)
Bulutli Platformalarga (Render/Railway) Deploy qilish
Platformada yangi Web Service yarating va ushbu repozitoriyani bog'lang.

Environment Variables bo'limiga .env.example ichidagi barcha o'zgaruvchilarni (ayniqsa SECRET_KEY va DATABASE_URL) qo'shing.

Build Command: pip install -r requirements.txt && flask db upgrade

Start Command: gunicorn run:app
