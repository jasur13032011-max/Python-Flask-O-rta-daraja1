# Python-Flask-O-rta-daraja1
1. Form Klasslari (forms.py)
Kodni ixcham va toza saqlash uchun barcha formalar bitta faylda jamlandi. validate_username va validate_email maxsus validatorlari ValidationError xatoligini qaytaradi.

Python
from flask_wtf import FlaskForm
from wtforms import StringField, PasswordField, SubmitField, BooleanField
from wtforms.validators import DataRequired, Length, Email, EqualTo, ValidationError

# Tasavvur qilamiz, bazada foydalanuvchilar bor deb:
# Haqiqiy loyihada bu yerda ma'lumotlar bazasidan tekshiriladi (masalan, User.query)
EXISTING_USERNAMES = ["ali", "vali", "salim"]
EXISTING_EMAILS = ["ali@example.com", "vali@example.com"]

class RegisterForm(FlaskForm):
    username = StringField('Foydalanuvchi nomi', validators=[DataRequired(), Length(min=3, max=20)])
    email = StringField('Email manzili', validators=[DataRequired(), Email()])
    password = PasswordField('Parol', validators=[DataRequired(), Length(min=6)])
    confirm_password = PasswordField('Parolni tasdiqlang', validators=[DataRequired(), EqualTo('password', message='Parollar bir-biriga mos kelmadi.')])
    submit = SubmitField('Ro\'yxatdan o\'tish')

    # Maxsus validator: username bandligini tekshirish
    def validate_username(self, username):
        if username.data.lower() in EXISTING_USERNAMES:
            raise ValidationError('Bu foydalanuvchi nomi allaqachon band.')

    # Maxsus validator: email bandligini tekshirish
    def validate_email(self, email):
        if email.data.lower() in EXISTING_EMAILS:
            raise ValidationError('Bu email manzili allaqachon ro\'yxatdan o\'tgan.')

class LoginForm(FlaskForm):
    email = StringField('Email manzili', validators=[DataRequired(), Email()])
    password = PasswordField('Parol', validators=[DataRequired()])
    remember = BooleanField('Eslab qolish')
    submit = SubmitField('Tizimga kirish')

class PostForm(FlaskForm):
    title = StringField('Sarlavha', validators=[DataRequired(), Length(max=100)])
    content = StringField('Maqola matni', validators=[DataRequired()])
    submit = SubmitField('Chop etish')

class SearchForm(FlaskForm):
    # GET so'rovi orqali ishlaydigan forma uchun CSRF o'chiriladi
    class Meta:
        csrf = False
        
    query = StringField('Qidiruv', validators=[DataRequired()])
    submit = SubmitField('Izlash')
2. HTML Shablonlar (Templates)
Har bir shablonda {{ form.hidden_tag() }} orqali CSRF token joylashtirilgan (SearchFormdan tashqari, chunki unda csrf = False qilingan). Xato xabarlari tegishli maydonning aynan ostida ko'rsatiladi.

register.html (Ro'yxatdan o'tish)
HTML
<form method="POST" action="">
    {{ form.hidden_tag() }}
    
    <div>
        {{ form.username.label }}<br>
        {{ form.username }}
        {% if form.username.errors %}
            {% for error in form.username.errors %}
                <span style="color: red;">[{{ error }}]</span>
            {% endfor %}
        {% endif %}
    </div>

    <div>
        {{ form.email.label }}<br>
        {{ form.email }}
        {% if
