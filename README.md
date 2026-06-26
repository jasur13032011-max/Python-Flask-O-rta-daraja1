# Python-Flask-O-rta-daraja1
Flask-Login kutubxonasi yordamida autentifikatsiya tizimini to'liq yakunlash uchun quyidagi o'zgarishlar va yangi marshrutlarni (routes) qo'shishingiz kerak.

1. Modelga UserMixin qo'shish va Login Manager (models.py & extensions.py)
UserMixin foydalanuvchi modeliga Flask-Login uchun kerakli bo'lgan xususiyatlarni (is_authenticated, is_active va h.k.) avtomat qo'shadi.

Python
from flask_sqlalchemy import SQLAlchemy
from flask_login import UserMixin, LoginManager
from werkzeug.security import generate_password_hash, check_password_hash

db = SQLAlchemy()
login_manager = LoginManager()
# Login talab qilinadigan sahifaga ruxsatsiz kirilganda yo'naltiriladigan manzil
login_manager.login_view = 'auth.login'
login_manager.login_message_category = 'info'

@login_manager.user_loader
def load_user(user_id):
    """Sessiyadagi foydalanuvchini ID orqali bazadan yuklab olish"""
    return User.query.get(int(user_id))

class User(db.Model, UserMixin): # UserMixin meros olindi
    __tablename__ = 'users'
    
    id = db.Column(db.Integer, primary_key=True)
    username = db.Column(db.String(80), unique=True, nullable=False)
    email = db.Column(db.String(120), unique=True, nullable=False)
    password_hash = db.Column(db.String(255), nullable=False)
    
    # Item modeli bilan aloqa (Relationship)
    items = db.relationship('Item', backref='author', lazy=True)

    def set_password(self, raw_password):
        self.password_hash = generate_password_hash(raw_password)

    def check_password(self, raw_password):
        return check_password_hash(self.password_hash, raw_password)

class Item(db.Model):
    __tablename__ = 'items'
    id = db.Column(db.Integer, primary_key=True)
    name = db.Column(db.String(100), nullable=False)
    # Tashqi kalit (Foreign Key) foydalanuvchiga bog'lash uchun
    user_id = db.Column(db.Integer, db.ForeignKey('users.id'), nullable=False)
2. Login va Logout Marshrutlari (auth.py)
Login jarayonida foydalanuvchi ham username, ham email orqali kira olishi mantiqini yozamiz hamda ?next=... parametrini inobatga olamiz:

Python
from flask import Blueprint, render_template, request, redirect, url_for, flash
from flask_login import login_user, logout_user, login_required, current_user
from models import User, db
from urllib.parse import urlparse

auth = Blueprint('auth', __name__)

@auth.route('/auth/login', methods=['GET', 'POST'])
def login():
    if current_user.is_authenticated:
        return redirect(url_for('main.index')) # Agar login bo'lgan bo'lsa, bosh sahifaga

    if request.method == 'POST':
        login_input = request.form.get('login_input', '').strip() # username yoki email
        password = request.form.get('password', '')
        remember = True if request.form.get('remember') else False

        # Username Yoki Email bo'yicha qidirish
        user = User.query.filter((User.username == login_input) | (User.email == login_input)).first()

        if user and user.check_password(password):
            login_user(user, remember=remember)
            
            # ?next=... parametrini tekshirish (Xavfsiz yo'naltirish bilan)
            next_page = request.args.get('next')
            if not next_page or urlparse(next_page).netloc != '':
                next_page = url_for('main.index') # default sahifa
                
            flash("Tizimga muvaffaqiyatli kirdingiz!", "success")
            return redirect(next_page)
        
        flash("Logindagi ma'lumotlar xato!", "danger")
        
    return render_template('login.html')

@auth.route('/auth/logout')
@login_required # Faqat login bo'lganlar chiqa oladi
def logout():
    logout_user()
    flash("Tizimdan chiqdingiz.", "info")
    return redirect(url_for('main.index'))
3. Yangi Item yaratish (items.py)
Element yaratilganda avtomatik ravishda uni joriy login bo'lgan foydalanuvchiga bog'laymiz:

Python
from flask import Blueprint, render_template, request, redirect, url_for, flash
from flask_login import login_required, current_user
from models import Item, db

items_bp = Blueprint('items', __name__)

@items_bp.route('/items/new', methods=['GET', 'POST'])
@login_required # Faqat login bo'lgan foydalanuvchilar kiradi
def new_item():
    if request.method == 'POST':
        item_name = request.form.get('name', '').strip()
        
        if not item_name:
            flash("Item nomi bo'sh bo'lishi mumkin emas!", "danger")
            return render_template('new_item.html')

        # Yangi item yaratish va joriy foydalanuvchi ID sini biriktirish
        item = Item(name=item_name, user_id=current_user.id)
        
        db.session.add(item)
        db.session.commit()
        
        flash("Yangi element muvaffaqiyatli qo'shildi!", "success")
        return redirect(url_for('main.index'))

    return render_template('new_item.html')
4. Bosh menyu (Shablon - base.html)
Jinjada current_user.is_authenticated orqali menyuni dinamik o'zgartiramiz:

HTML
<nav class="navbar navbar-expand-lg navbar-light bg-light">
  <div class="container-fluid">
    <a class="navbar-brand" href="/">Mening Saytim</a>
    <div class="collapse navbar-collapse">
      <ul class="navbar-nav ms-auto mb-2 mb-lg-0">
        
        {% if current_user.is_authenticated %}
          <li class="nav-item">
            <span class="nav-link text-dark">Salom, <strong>{{ current_user.username }}</strong>!</span>
          </li>
          <li class="nav-item">
            <a class="nav-link" href="{{ url_for('items.new_item') }}">Yangi Item</a>
          </li>
          <li class="nav-item">
            <a class="nav-link btn btn-outline-danger btn-sm text-danger" href="{{ url_for('auth.logout') }}">Chiqish</a>
          </li>
        {% else %}
          <li class="nav-item">
            <a class="nav-link" href="{{ url_for('auth.login') }}">Kirish</a>
          </li>
          <li class="nav-item">
            <a class="nav-link" href="{{ url_for('auth.register') }}">Ro'yxatdan o'tish</a>
          </li>
        {% endif %}
        
      </ul>
    </div>
  </div>
</nav>
