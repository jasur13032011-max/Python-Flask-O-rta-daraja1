# Python-Flask-O-rta-daraja1
Keling, ushbu qadamlarning barchasini o'z ichiga olgan to'liq va tayyor kod strukturasini ko'rib chiqamiz. Parollarni xeshlash uchun Flask-da standart bo'lgan Werkzeug kutubxonasidan foydalanamiz.

1. Model va Metodlar (models.py)
Foydalanuvchi modeliga password_hash ustunini qo'shamiz hamda parolni o'rnatish va tekshirish metodlarini yozamiz:

Python
from flask_sqlalchemy import SQLAlchemy
from werkzeug.security import generate_password_hash, check_password_hash

db = SQLAlchemy()

class User(db.Model):
    __tablename__ = 'users'
    
    id = db.Column(db.Integer, primary_key=True)
    username = db.Column(db.String(80), unique=True, nullable=False)
    email = db.Column(db.String(120), unique=True, nullable=False)
    # Xesh uchun 255 belgi uzunlikdagi ustun
    password_hash = db.Column(db.String(255), nullable=False)

    def set_password(self, raw_password):
        """Xom parolni xeshga o'girib saqlaydi"""
        self.password_hash = generate_password_hash(raw_password)

    def check_password(self, raw_password):
        """Kiritilgan parolni xesh bilan solishtiradi"""
        return check_password_hash(self.password_hash, raw_password)
2. Ma'lumotlar bazasi migratsiyasi (Flask-Migrate)
Terminalda yangi ustunni bazaga qo'shish uchun quyidagi buyruqlarni ketma-ket bajaramiz:

Bash
# 1. Yangi ustun uchun migratsiya faylini yaratish
flask db migrate -m "add password_hash to users"

# 2. O'zgarishlarni ma'lumotlar bazasiga qo'llash
flask db upgrade
3. Register Controller va Validatsiya (routes.py)
Ro'yxatdan o'tish formasi, validatsiya va xavfsizlik tekshiruvlari:

Python
from flask import Blueprint, render_template, request, redirect, url_for, flash
from models import db, User

auth = Blueprint('auth', __name__)

@auth.route('/auth/register', methods=['GET', 'POST'])
def register():
    if request.method == 'POST':
        username = request.form.get('username', '').strip()
        email = request.form.get('email', '').strip()
        password = request.form.get('password', '')

        # 1. Validatsiya: Majburiy maydonlar va parol uzunligi
        if not username or not email:
            flash("Username va Email kiritilishi majburiy!", "danger")
            return render_template('register.html'), 400
            
        if len(password) < 8:
            flash("Parol kamida 8 ta belgidan iborat bo'lishi kerak!", "danger")
            return render_template('register.html'), 400

        # 2. Unikallikni tekshirish (Unique check)
        existing_username = User.query.filter_by(username=username).first()
        if existing_username:
            flash("Bu username allaqachon band!", "danger")
            return render_template('register.html'), 400

        existing_email = User.query.filter_by(email=email).first()
        if existing_email:
            flash("Bu email bilan allaqachon ro'yxatdan o'tilgan!", "danger")
            return render_template('register.html'), 400

        # 3. Yangi foydalanuvchini yaratish
        new_user = User(username=username, email=email)
        new_user.set_password(password) # Parol xeshlanadi
        
        db.session.add(new_user)
        db.session.commit()

        flash("Ro'yxatdan muvaffaqiyatli o'tdingiz!", "success")
        return redirect(url_for('auth.register'))

    # GET so'rovi bo'lganda formani qaytaradi
    return render_template('register.html')
4. Flask Shell orqali test qilish
Tizim to'g'ri ishlayotganini tekshirish uchun terminalda flask shellni ochamiz va quyidagi kodlarni bajarib ko'ramiz:

Python
# Flask shell muhitida:
>>> from models import User, db

# 1. Bazadagi birinchi foydalanuvchini olamiz
>>> user = User.query.first()
>>> print(user.username)
'shaxzod'

# 2. Noto'g'ri parol bilan tekshiramiz (False qaytishi kerak)
>>> user.check_password('noto_g_ri_parol')
False

# 3. To'g'ri parol bilan tekshiramiz (True qaytishi kerak)
>>> user.check_password('my_secret_password123')
True
