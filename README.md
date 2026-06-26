# Python-Flask-O-rta-daraja1
Mana, Flask ilovangiz uchun berilgan barcha talablar asosida to'liq va xavfsiz RESTful API tizimini yaratish bo'yicha qo'llanma. Web UI bilan parallel ravishda ishlashi uchun barcha xatoliklar JSON formatida qaytariladi.

1. API Blueprint va Marshrutlar (api.py)
Barcha CRUD amallari, xavfsiz JSON parslash (silent=True), validatsiya va to'g'ri HTTP status kodlari bilan:

Python
import os
from flask import Blueprint, jsonify, request, url_for
from models import db, Post  # Loyihangizdagi db va Post modeli

api_bp = Blueprint('api', __name__, url_prefix='/api')

# --- ERROR HANDLERS (JSON formatida) ---

@api_bp.app_errorhandler(400)
def bad_request(error):
    return jsonify({'error': getattr(error, 'description', 'Bad Request')}), 400

@api_bp.app_errorhandler(404)
def not_found(error):
    return jsonify({'error': 'Resource not found'}), 404

@api_bp.app_errorhandler(500)
def internal_server_error(error):
    return jsonify({'error': 'Internal server error'}), 500


# --- ENDPOINTS ---

@api_bp.route('/posts', methods=['GET'])
def get_posts():
    """Barcha postlarni qaytarish (200)"""
    posts = Post.query.all()
    return jsonify([post.to_dict() for post in posts]), 200


@api_bp.route('/posts/<int:post_id>', methods=['GET'])
def get_post(post_id):
    """Bitta postni qaytarish (200) yoki 404"""
    post = Post.query.get_or_404(post_id)
    return jsonify(post.to_dict()), 200


@api_bp.route('/posts', methods=['POST'])
def create_post():
    """Yangi post yaratish (201) + Location header"""
    data = request.get_json(silent=True)
    
    # JSON mavjudligi va validatsiya
    if not data or 'title' not in data or not data['title'].strip():
        return jsonify({'error': 'title required'}), 400
        
    new_post = Post(
        title=data['title'].strip(),
        content=data.get('content', '').strip()
    )
    db.session.add(new_post)
    db.session.commit()
    
    response = jsonify(new_post.to_dict())
    response.headers['Location'] = url_for('api.get_post', post_id=new_post.id, _external=True)
    return response, 201


@api_bp.route('/posts/<int:post_id>', methods=['PUT'])
def update_post(post_id):
    """Postni yangilash (200)"""
    post = Post.query.get_or_404(post_id)
    data = request.get_json(silent=True)
    
    if not data:
        return jsonify({'error': 'Invalid JSON'}), 400
        
    if 'title' in data:
        if not data['title'].strip():
            return jsonify({'error': 'title required'}), 400
        post.title = data['title'].strip()
        
    if 'content' in data:
        post.content = data['content'].strip()
        
    db.session.commit()
    return jsonify(post.to_dict()), 200


@api_bp.route('/posts/<int:post_id>', methods=['DELETE'])
def delete_post(post_id):
    """Postni o'chirish (204)"""
    post = Post.query.get_or_404(post_id)
    db.session.delete(post)
    db.session.commit()
    return '', 204
Eslatma: Post modelingizda ma'lumotlarni dict ko'rinishiga o'tkazuvchi to_dict() metodi bo'lishi kerak. Masalan:

Python
def to_dict(self):
    return {'id': self.id, 'title': self.title, 'content': self.content}
2. API'ni ro'yxatdan o'tkazish (app.py)
Web UI va API parallel ishlashi uchun Blueprint'ni asosiy ilovaga ulaymiz:

Python
from flask import Flask
from models import db
from api import api_bp
# UI uchun blueprint'ni ham import qilishingiz mumkin (masalan: from web import web_bp)

app = Flask(__name__)
app.config['SQLALCHEMY_DATABASE_URI'] = 'sqlite:///posts.db'
db.init_app(app)

# API blueprint'ini ulash
app.register_blueprint(api_bp)

# Web UI blueprint'ini ulash (U ham o'z yo'lida ishlayveradi)
# app.register_blueprint(web_bp)

if __name__ == '__main__':
    app.run(debug=True)
3. README.md uchun curl qo'llanmasi
Loyiha hujjatlariga quyidagi qismni qo'shishingiz mumkin:

Markdown
## API Endpoint'lar bo'yicha qo'llanma (cURL)

Asosiy URL: `http://127.0.0.1:5000/api`

### 1. Barcha postlarni olish
```bash
curl -X GET [http://127.0.0.1:5000/api/posts](http://127.0.0.1:5000/api/posts)
2. ID bo'yicha bitta postni olish
Bash
curl -X GET [http://127.0.0.1:5000/api/posts/1](http://127.0.0.1:5000/api/posts/1)
3. Yangi post yaratish
Bash
curl -X POST [http://127.0.0.1:5000/api/posts](http://127.0.0.1:5000/api/posts) \
     -H "Content-Type: application/json" \
     -d '{"title": "Yangi Post", "content": "Post matni shu yerda"}'
4. Xatolik testi (Title berilmaganda 400 Bad Request)
Bash
curl -X POST [http://127.0.0.1:5000/api/posts](http://127.0.0.1:5000/api/posts) \
     -H "Content-Type: application/json" \
     -d '{"content": "Sarlavhasiz post"}'
5. Postni yangilash
Bash
curl -X PUT [http://127.0.0.1:5000/api/posts/1](http://127.0.0.1:5000/api/posts/1) \
     -H "Content-Type: application/json" \
     -d '{"title": "Yangilangan sarlavha"}'
6. Postni o'chirish
Bash
curl -X DELETE [http://127.0.0.1:5000/api/posts/1](http://127.0.0.1:5000/api/posts/1)
