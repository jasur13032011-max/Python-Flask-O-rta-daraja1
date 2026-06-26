# Python-Flask-O-rta-daraja1
Mana, Flask ilovangiz uchun berilgan barcha talablar asosida tayyorlangan to'liq qo'llanma va kodlar to'plami.

1. Model va Migratsiya (Flask-SQLAlchemy)
Foydalanuvchi modeliga avatar ustunini qo'shamiz.

Python
from database import db  # Loyihangizdagi db obyekti

class User(db.Model):
    __tablename__ = 'users'
    
    id = db.Column(db.Integer, primary_key=True)
    username = db.Column(db.String(80), unique=True, nullable=False)
    # Avatar uchun String(255) ustun
    avatar = db.Column(db.String(255), nullable=True, default='default_avatar.png')
Migratsiya buyruqlari (Terminalda):

Bash
flask db migrate -m "Add avatar to user"
flask db upgrade
2. Flask Konfiguratsiyasi (config.py)
Fayl yuklash hajmini cheklash va yuklanadigan papkani belgilash (static dan tashqarida):

Python
import os

class Config:
    SECRET_KEY = 'sizning_maxfiy_kalitingiz'
    SQLALCHEMY_DATABASE_URI = 'sqlite:///app.db'
    
    # Maksimal fayl hajmi: 5MB
    MAX_CONTENT_LENGTH = 5 * 1024 * 1024  
    
    # Static papkasidan tashqaridagi yuklash joyi
    UPLOAD_FOLDER = os.path.join(os.path.abspath(os.path.dirname(__file__)), 'uploads')
3. Forma (forms.py - Flask-WTF)
FileField, FileRequired va FileAllowed validatorlari bilan forma tuzamiz:

Python
from flask_wtf import FlaskForm
from flask_wtf.file import FileField, FileRequired, FileAllowed

class AvatarForm(FlaskForm):
    avatar = FileField('Avatar yuklang', validators=[
        FileRequired(message="Fayl tanlanmagan!"),
        FileAllowed(['jpg', 'jpeg', 'png', 'gif'], message="Faqat rasmlar (jpg, jpeg, png, gif) ruxsat etilgan!")
    ])
4. Helper funksiya: Haqiqiy turni tekshirish (imghdr)
Faqat kengaytma (extension) yetarli emas, fayl ichki tuzilishini tekshirish uchun imghdr (yoki muqobil puremagic/filetype agar Python 3.13+ bo'lsa) ishlatamiz.

Python
import imghdr

def is_valid_image(file_stream):
    """Faylning haqiqiy rasm ekanligini ichki tuzilishidan tekshiradi"""
    header = file_stream.read(512)  # Boshi o'qiladi
    file_stream.seek(0)  # Ko'rsatkich boshiga qaytariladi
    
    img_type = imghdr.what(None, header)
    return img_type in ['jpeg', 'png', 'gif']
5. View va Faylni xavfsiz saqlash (routes.py)
secure_filename va uuid.uuid4().hex orqali noyob nom yaratish, rasm turini tekshirish va faylni uzatish (send_from_directory):

Python
import os
import uuid
from flask import Blueprint, render_template, redirect, url_for, flash, request, send_from_directory, current_app
from werkzeug.utils import secure_filename
from models import User, db
from forms import AvatarForm
from utils import is_valid_image  # Yuqoridagi helper funksiya

user_bp = Blueprint('user', __name__)

@user_bp.route('/profile/avatar', methods=['GET', 'POST'])
def upload_avatar():
    form = AvatarForm()
    if form.validate_on_submit():
        file = form.avatar.data
        
        # 1. Ichki turini tekshirish (imghdr)
        if not is_valid_image(file.stream):
            flash("Haqiqiy rasm faylini yuklang!", "danger")
            return render_template('upload.html', form=form)
        
        # 2. Xavfsiz va noyob nom yaratish
        filename = secure_filename(file.filename)
        ext = os.path.splitext(filename)[1]
        unique_filename = f"{uuid.uuid4().hex}{ext}"
        
        # 3. Papka mavjudligini tekshirib saqlash
        upload_path = current_app.config['UPLOAD_FOLDER']
        if not os.path.exists(upload_path):
            os.makedirs(upload_path)
            
        file.save(os.path.join(upload_path, unique_filename))
        
        # 4. Bazaga yozish (Simulyatsiya: joriy foydalanuvchi ID=1 deb olindi)
        user = User.query.get(1) 
        user.avatar = unique_filename
        db.session.commit()
        
        flash("Avatar muvaffaqiyatli yangilandi!", "success")
        return redirect(url_for('user.upload_avatar'))
        
    return render_template('upload.html', form=form)

# Static'dan tashqaridagi fayllarni xizmat ko'rsatish (serve) qilish
@user_bp.route('/uploads/<filename>')
def serve_avatar(filename):
    return send_from_directory(current_app.config['UPLOAD_FOLDER'], filename)
6. HTML Shablon (upload.html)
Formada albatta enctype="multipart/form-data" bo'lishi shart:

HTML
<!DOCTYPE html>
<html lang="uz">
<head>
    <meta charset="UTF-8">
    <title>Avatar Yuklash</title>
</head>
<body>
    <h2>Profil avatarini o'zgartirish</h2>
    
    {% with messages = get_flashed_messages(with_categories=true) %}
      {% if messages %}
        {% for category, message in messages %}
          <div style="color: {% if category=='success' %}green{% else %}red{% endif %};">{{ message }}</div>
        {% endfor %}
      {% endif %}
    {% endwith %}

    <form method="POST" enctype="multipart/form-data">
        {{ form.hidden_tag() }}
        <div>
            {{ form.avatar.label }} <br>
            {{ form.avatar() }}
            {% for error in form.avatar.errors %}
                <span style="color: red;">{{ error }}</span>
            {% endfor %}
        </div>
        <br>
        <button type="submit">Yuklash</button>
    </form>
</body>
</html>
7. 413 Request Entity Too Large Xatoligi
Fayl hajmi 5MB dan oshganda chiroyli xato sahifasini ko'rsatish:

Python
from flask import Flask, render_template

app = Flask(__name__)
app.config.from_object('config.Config')

@app.errorhandler(413)
def request_entity_too_large(error):
    return render_template('errors/413.html'), 413
templates/errors/413.html:

HTML
<!DOCTYPE html>
<html>
<head><title>Fayl juda katta</title></head>
<body style="text-align: center; padding: 50px;">
    <h1>413 - Fayl hajmi juda katta!</h1>
    <p>Kechirasiz, yuklanadigan avatar hajmi <b>5 MB</b> dan oshmasligi kerak.</p>
    <a href="/profile/avatar">Orqaga qaytish</a>
</body>
</html>
8. Git xavfsizligi (.gitignore)
Loyiha ildiz (root) katalogidagi .gitignore fayliga yuklangan rasmlar Git gashiga tushib qolmasligi uchun quyidagi qatorni qo'shamiz:

Plaintext
# Yuklangan foydalanuvchi fayllarini git'ga qo'shmaslik
uploads/
