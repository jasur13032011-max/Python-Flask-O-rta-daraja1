# Python-Flask-O-rta-daraja1
Mana, Flask REST API uchun paginatsiya, filtrlash, qidiruv, saralash va dinamik maydonlarni tanlash (field selection) funksiyalarini o'z ichiga olgan to'liq va xavfsiz endpoint kodlari.

1. API Endpoint Kodu (routes.py)
Ushbu kod SQL injection'dan to'liq himoyalangan, so'rov parametrlarini xavfsiz tekshiradi (whitelist) va barcha talab qilingan meta-ma'lumotlar hamda havolalarni (links) shakllantiradi.

Python
from flask import Blueprint, jsonify, request, url_for
from sqlalchemy import or_
from models import db, Post  # Loyihangizdagi db va Post modeli

api_bp = Blueprint('api', __name__, url_prefix='/api')

@api_bp.route('/posts', methods=['GET'])
def get_posts():
    # 1. Paginatsiya parametrlarini olish va tekshirish
    try:
        page = request.args.get('page', default=1, type=int)
        per_page = request.args.get('per_page', default=20, type=int)
        if page < 1: page = 1
        # Maksimal 50 ta cheklovi
        if per_page > 50: per_page = 50
        if per_page < 1: per_page = 20
    except ValueError:
        page, per_page = 1, 20

    # Base query yaratish
    query = Post.query

    # 2. ?q= parametr bo'yicha qidiruv (title va body bo'yicha or_ + ilike)
    search_query = request.args.get('q', default='', type=str).strip()
    if search_query:
        search_fmt = f"%{search_query}%"
        query = query.filter(or_(
            Post.title.ilike(search_fmt),
            Post.body.ilike(search_fmt)  # Agar modelda content bo'lsa, Post.content deb o'zgartiring
        ))

    # 3. Saralash (Whitelist tekshiruvi - SQL Injection'dan himoya)
    sort_by = request.args.get('sort', default='created_at', type=str).lower()
    order = request.args.get('order', default='desc', type=str).lower()

    # Ruxsat etilgan maydonlar ro'yxati
    allowed_sort_fields = {
        'id': Post.id,
        'title': Post.title,
        'created_at': Post.created_at
    }

    # Whitelist tekshiruvi
    sort_column = allowed_sort_fields.get(sort_by, Post.created_at)
    
    if order == 'asc':
        query = query.order_by(sort_column.asc())
    else:
        query = query.order_by(sort_column.desc())

    # 4. Paginatsiyani amalga oshirish (Flask-SQLAlchemy paginate metodi)
    pagination = query.paginate(page=page, per_page=per_page, error_out=False)

    # 5. Bonus: Field Selection (?fields=id,title)
    fields_arg = request.args.get('fields', default='', type=str)
    requested_fields = [f.strip() for f in fields_arg.split(',') if f.strip()] if fields_arg else []

    items = []
    for post in pagination.items:
        # Agar field selection bo'sh bo'lsa, standart dict qaytadi
        full_dict = post.to_dict() 
        if requested_fields:
            # Faqat so'ralgan maydonlarni saralab olish
            filtered_dict = {k: v for k, v in full_dict.items() if k in requested_fields}
            items.append(filtered_dict)
        else:
            items.append(full_dict)

    # 6. To'liq tashqi URL (links) hosil qilish funksiyasi (_external=True)
    def get_page_link(p_num):
        if p_num is None:
            return None
        # Mavjud query parametrlarni saqlab qolgan holda URL hosil qilish
        args = request.args.to_dict()
        args['page'] = p_num
        args['per_page'] = per_page
        return url_for('api.get_posts', _external=True, **args)

    # Natija tuzilishi
    response_data = {
        'items': items,
        'meta': {
            'page': pagination.page,
            'per_page': pagination.per_page,
            'total': pagination.total,
            'pages': pagination.pages,
            'has_next': pagination.has_next,
            'has_prev': pagination.has_prev
        },
        'links': {
            'self': get_page_link(pagination.page),
            'next': get_page_link(pagination.next_num) if pagination.has_next else None,
            'prev': get_page_link(pagination.prev_num) if pagination.has_prev else None
        }
    }

    return jsonify(response_data), 200
2. To'liq cURL misollari (README.md uchun)
Loyiha hujjatlariga quyidagi batafsil cURL buyruqlarini qo'shishingiz mumkin:

Markdown
### 1. Barcha parametrlar ishtirok etgan to'liq cURL so'rovi
Ushbu so'rov `flask` so'zi bo'yicha qidiradi, natijalarni `title` bo'yicha `asc` (o'sish) tartibida saralaydi, sahifada 5 tadan ko'rsatib, 2-sahifani oladi hamda faqat `id` va `title` maydonlarini qaytaradi:

```bash
curl -G "[http://127.0.0.1:5000/api/posts](http://127.0.0.1:5000/api/posts)" \
     --data-urlencode "q=flask" \
     --data-urlencode "page=2" \
     --data-urlencode "per_page=5" \
     --data-urlencode "sort=title" \
     --data-urlencode "order=asc" \
     --data-urlencode "fields=id,title"
2. Standart (Default) so'rov
Parametrlarsiz yuborilganda page=1, per_page=20, sort=created_at, order=desc qiymatlari avtomatik ishlaydi:

Bash
curl -X GET "[http://127.0.0.1:5000/api/posts](http://127.0.0.1:5000/api/posts)"
3. Faqat qidiruv va maydonlarni cheklash (Field Selection)
Bash
curl -G "[http://127.0.0.1:5000/api/posts](http://127.0.0.1:5000/api/posts)" \
     --data-urlencode "q=python" \
     --data-urlencode "fields=id,title,created_at"
