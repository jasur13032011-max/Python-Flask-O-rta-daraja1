# Python-Flask-O-rta-daraja1
1. Muhit va Konfiguratsiya (.env & config.py)
Avval .env faylida maxfiy ma'lumotlarni saqlaymiz:

Plaintext
MAIL_SERVER=smtp.gmail.com
MAIL_PORT=587
MAIL_USE_TLS=True
MAIL_USERNAME=your_email@gmail.com
MAIL_PASSWORD=your_app_password
MAIL_DEFAULT_SENDER=your_email@gmail.com
SECRET_KEY=super_secret_salt_key
config.py ichida ularni yuklaymiz:

Python
import os
from dotenv import load_dotenv

load_dotenv()

class Config:
    SECRET_KEY = os.getenv('SECRET_KEY')
    MAIL_SERVER = os.getenv('MAIL_SERVER')
    MAIL_PORT = int(os.getenv('MAIL_PORT', 587))
    MAIL_USE_TLS = os.getenv('MAIL_USE_TLS', 'True') == 'True'
    MAIL_USERNAME = os.getenv('MAIL_USERNAME')
    MAIL_PASSWORD = os.getenv('MAIL_PASSWORD')
    MAIL_DEFAULT_SENDER = os.getenv('MAIL_DEFAULT_SENDER')
    # Test rejimi uchun
    MAIL_SUPPRESS_SEND = os.getenv('MAIL_SUPPRESS_SEND', 'False') == 'True'
2. Asinxron Email Yuborish funksiyasi (mail_utils.py)
Ilova kontekstini saqlagan holda, asosiy oqimni (main thread) band qilmasdan, orqa fonda email yuborish:

Python
from threading import Thread
from flask import current_app, render_template
from flask_mail import Message, Mail
from itsdangerous import URLSafeTimedSerializer

mail = Mail()

def send_async_email(app, msg):
    with app.app_context():
        mail.send(msg)

def send_email(subject, to_email, template_prefix, **kwargs):
    """HTML va Plain Text ko'rinishida email yuborish"""
    app = current_app._get_current_object()
    msg = Message(subject, recipients=[to_email])
    
    # Shablonlarni render qilish
    msg.body = render_template(f'{template_prefix}.txt', **kwargs)
    msg.html = render_template(f'{template_prefix}.html', **kwargs)
    
    # Thread yordamida asinxron yuborish
    thr = Thread(target=send_async_email, args=[app, msg])
    thr.start()
    return thr

# Token yaratish va tekshirish (1 soat = 3600 soniya)
def generate_reset_token(email):
    serializer = URLSafeTimedSerializer(current_app.config['SECRET_KEY'])
    return serializer.dumps(email, salt='password-reset-salt')

def verify_reset_token(token, expires_sec=3600):
    serializer = URLSafeTimedSerializer(current_app.config['SECRET_KEY'])
    try:
        email = serializer.loads(token, salt='password-reset-salt', max_age=expires_sec)
    except Exception:
        return None
    return email
3. Email Shablonlari (templates/email/)
Har bir email turi uchun ham HTML, ham Plain Text (.txt) talab etiladi.

Welcome Email
templates/email/welcome.txt:

Plaintext
Salom, {{ username }}! Saytimizga xush kelibsiz.
templates/email/welcome.html:

HTML
<h3>Salom, {{ username }}!</h3>
<p>Saytimizga <strong>xush kelibsiz</strong>.</p>
Password Reset Email
templates/email/reset.txt:

Plaintext
Parolni tiklash uchun quyidagi havolaga o'ting: {{ url }}
Bu havola 1 soat davomida faol bo'ladi.
templates/email/reset.html:

HTML
<p>Parolni tiklash uchun quyidagi tugmani bosing:</p>
<a href="{{ url }}" style="padding: 10px 20px; background: blue; color: white; text-decoration: none;">Parolni yangilash</a>
<p>Bu havola 1 soat davomida faol bo'ladi.</p>
4. Auth Blueprint & Marshrutlar (auth.py)
Email Enumeration himoyasi: Email bazada bor yoki yo'qligidan qat'iy nazar foydalanuvchiga har doim bir xil xabar ko'rsatiladi.

Python
from flask import Blueprint, render_template, redirect, url_for, flash, request
from werkzeug.security import generate_password_hash
from models import db, User  # Simulyatsiya
from mail_utils import send_email, generate_reset_token, verify_reset_token

auth_bp = Blueprint('auth', __name__, url_prefix='/auth')

@auth_bp.route('/register', methods=['POST'])
def register():
    # ... ro'yxatdan o'tish logikasi ...
    # Muallif muvaffaqiyatli o'tgach:
    user_email = request.form.get('email')
    username = request.form.get('username')
    
    send_email("Xush kelibsiz!", user_email, 'email/welcome', username=username)
    return redirect(url_for('auth.login'))


@auth_bp.route('/forgot', methods=['GET', 'POST'])
def forgot_password():
    if request.method == 'POST':
        email = request.form.get('email')
        user = User.query.filter_by(email=email).first()
        
        # Email Enumeration Himoyasi: Foydalanuvchi mavjud bo'lsa email yuboriladi,
        # mavjud bo'lmasa ham "Email yuborildi" deb javob qaytariladi.
        if user:
            token = generate_reset_token(user.email)
            reset_url = url_for('auth.reset_with_token', token=token, _external=True)
            send_email("Parolni tiklash", user.email, 'email/reset', url=reset_url)
            
        flash("Agar ushbu email ro'yxatdan o'tgan bo'lsa, parolni tiklash ko'rsatmalari yuborildi.", "info")
        return redirect(url_for('auth.forgot_password'))
        
    return render_template('forgot.html')


@auth_bp.route('/reset/<token>', methods=['GET', 'POST'])
def reset_with_token(token):
    email = verify_reset_token(token)
    
    # Token eskirgan yoki yaroqsiz bo'lsa
    if not email:
        flash("Parolni tiklash havolasi yaroqsiz yoki muddati o'tgan!", "danger")
        return redirect(url_for('auth.forgot_password'))
        
    if request.method == 'POST':
        new_password = request.form.get('password')
        user = User.query.filter_by(email=email).first_or_404()
        
        # Parolni yangilash
        user.password = generate_password_hash(new_password)
        db.session.commit()
        
        flash("Parolingiz muvaffaqiyatli yangilandi!", "success")
        return redirect(url_for('auth.login'))
        
    return render_template('reset_password.html', token=token)
5. Unit Testlarda Emailni Tekshirish (test_mail.py)
Test jarayonida haqiqiy email ketib qolmasligi va ularni xotirada ushlab tekshirish uchun mail.record_messages ishlatiladi:

Python
import unittest
from flask import Flask
from config import Config
from mail_utils import mail, send_email

class TestEmailConfig(Config):
    TESTING = True
    MAIL_SUPPRESS_SEND = True  # Haqiqiy email yuborishni bloklash

class EmailTestCase(unittest.TestCase):
    def setUp(self):
        self.app = Flask(__name__)
        self.app.config.from_object(TestEmailConfig)
        mail.init_app(self.app)
        self.ctx = self.app.app_context()
        self.ctx.push()

    def tearDown(self):
        self.ctx.pop()

    def test_welcome_email_content(self):
        # mail.record_messages konteksti ichida xabarlarni yozib olamiz
        with mail.record_messages() as outbox:
            send_email(
                subject="Xush kelibsiz!",
                to_email="test@example.com",
                template_prefix="email/welcome",
                username="Abbos"
            )
            
            # Asinxron Thread tugashini ozgina kutish yoki join qilish kerak bo'lishi mumkin, 
            # lekin TESTING=True rejimida Flask-Mail odatda sinxron ishlaydi.
            self.assertEqual(len(outbox), 1)
            self.assertEqual(outbox[0].subject, "Xush kelibsiz!")
            self.assertEqual(outbox[0].recipients, ["test@example.com"])
            self.assertIn("Abbos", outbox[0].body)

if __name__ == '__main__':
    unittest.main()
