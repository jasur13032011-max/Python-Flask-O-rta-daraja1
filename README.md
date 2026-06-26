# Python-Flask-O-rta-daraja1
1. Modelga role ustunini qo'shish (models.py)
Foydalanuvchi modeliga standart qiymati 'user' bo'lgan role ustunini qo'shamiz:

Python
class User(db.Model, UserMixin):
    __tablename__ = 'users'
    
    id = db.Column(db.Integer, primary_key=True)
    username = db.Column(db.String(80), unique=True, nullable=False)
    email = db.Column(db.String(120), unique=True, nullable=False)
    password_hash = db.Column(db.String(255), nullable=False)
    # Yangi rol ustuni
    role = db.Column(db.String(20), default='user', nullable=False)

    @property
    def is_admin(self):
        """Shablon va kod ichida tekshirishni osonlashtirish uchun qulaylik"""
        return self.role == 'admin'
(Eslatma: Model o'zgargani uchun terminalda flask db migrate va flask db upgrade buyruqlarini bajarishni unutmang).

2. Admin Dekoratori (decorators.py)
Faqat 'admin' roliga ega foydalanuvchilarni o'tkazadigan maxsus dekorator yaratamiz. Agar foydalanuvchi login qilmagan bo'lsa 401, admin bo'lmasa 403 xatoligini qaytaradi:

Python
from functools import wraps
from flask import abort
from flask_login import current_user

def admin_required(f):
    @wraps(f)
    def decorated_function(*args, **kwargs):
        if not current_user.is_authenticated:
            abort(401)
        if current_user.role != 'admin':
            abort(403)
        return f(*args, **kwargs)
    return decorated_function
3. Elementni Tahrirlash va O'chirish (Ega yoki Admin tekshiruvi)
Item ustida amallar bajarishda ham server, ham shablon darajasida ruxsatlar tekshiriladi:

Python
from flask import Blueprint, abort, redirect, url_for, flash, request
from flask_login import login_required, current_user
from models import Item, db

items_bp = Blueprint('items', __name__)

@items_bp.route('/items/<int:item_id>/edit', methods=['POST'])
@login_required
def edit_item(item_id):
    item = Item.query.get_or_404(item_id)
    
    # KRIZISLI TEKSHIRUV: Faqat ega yoki admin o'zgartira oladi
    if item.user_id != current_user.id and not current_user.is_admin:
        abort(403)
        
    # Tahrirlash mantiqi...
    item.name = request.form.get('name')
    db.session.commit()
    return redirect(url_for('main.index'))
Shablonda ko'rinishi (item_detail.html):

HTML
{% if current_user.is_authenticated and (item.user_id == current_user.id or current_user.is_admin) %}
    <a href="#" class="btn btn-warning">Tahrirlash</a>
    <button class="btn btn-danger">O'chirish</button>
{% endif %}
4. Admin Panel: Foydalanuvchilar va Rolni O'zgartirish
Faqat adminlar kira oladigan va rollarni boshqaradigan marshrut:

Python
from flask import Blueprint, render_template, request, redirect, url_for, flash
from models import User, db
from decorators import admin_required

admin_bp = Blueprint('admin', __name__)

@admin_bp.route('/admin/users')
@admin_required # Server darajasidagi qat'iy himoya
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
        flash(f"{user.username} roli {new_role}ga o'zgartirildi.", "success")
    else:
        flash("Noto'g'ri rol tanlandi!", "danger")
        
    return redirect(url_for('admin.list_users'))
5. Xato Sahifalari (errors.py yoki app.py)
401 va 403 xatoliklari yuz berganda foydalanuvchiga chiroyli interfeys ko'rsatish:

Python
from flask import render_template

@app.errorhandler(401)
def unauthorized_error(error):
    return render_template('errors/401.html'), 401

@app.errorhandler(403)
def forbidden_error(error):
    return render_template('errors/403.html'), 403
6. Bosh Menyuda Admin Havolasi (base.html)
Agar tizimga kirgan foydalanuvchi admin bo'lsa, unga menyuda qo'shimcha boshqaruv paneli havolasi ko'rinadi:

HTML
{% if current_user.is_authenticated %}
  <li class="nav-item">
    <span class="nav-link">Salom, {{ current_user.username }}!</span>
  </li>
  
  {% if current_user.is_admin %}
    <li class="nav-item">
      <a class="nav-link text-danger font-weight-bold" href="{{ url_for('admin.list_users') }}">Admin Panel</a>
    </li>
  {% endif %}
  
  <li class="nav-item">
    <a class="nav-link" href="{{ url_for('auth.logout') }}">Chiqish</a>
  </li>
{% endif %}
Xavfsizlik Oltin Qoidasi: Shablondagi {% if current_user.is_admin %} tekshiruvi faqat dizaynni yashirish uchun xizmat qiladi. Haqiqiy xavfsizlik va ruxsat berish server tomonida @admin_required dekoratori va abort(403) orqali amalga oshirilmoqda. Foydalanuvchi linkni qo'lda tersa ham server baribir uni to'xtatadi.
