
import os
import sqlite3
import secrets
from functools import wraps
from flask import (
    Flask, request, redirect, url_for,
    session, flash, render_template_string
)
from werkzeug.security import generate_password_hash, check_password_hash

app = Flask(__name__)
app.secret_key = os.environ.get("SECRET_KEY", secrets.token_hex(32))

DB = "sasa_store.db"
ADMIN_USER = os.environ.get("ADMIN_USER", "admin")
ADMIN_PASSWORD = os.environ.get("ADMIN_PASSWORD")

if not ADMIN_PASSWORD:
    raise RuntimeError("Set ADMIN_PASSWORD before starting the app.")

PHONE = "01026966717"


def db():
    con = sqlite3.connect(DB)
    con.row_factory = sqlite3.Row
    return con


def init_db():
    con = db()
    con.executescript("""
    CREATE TABLE IF NOT EXISTS users(
        id INTEGER PRIMARY KEY AUTOINCREMENT,
        username TEXT UNIQUE NOT NULL,
        password TEXT NOT NULL,
        balance REAL DEFAULT 0,
        is_admin INTEGER DEFAULT 0
    );

    CREATE TABLE IF NOT EXISTS products(
        id INTEGER PRIMARY KEY AUTOINCREMENT,
        name TEXT NOT NULL,
        price REAL NOT NULL,
        description TEXT DEFAULT '',
        stock INTEGER DEFAULT 0
    );

    CREATE TABLE IF NOT EXISTS topups(
        id INTEGER PRIMARY KEY AUTOINCREMENT,
        user_id INTEGER,
        amount REAL,
        reference TEXT,
        status TEXT DEFAULT 'Pending'
    );

    CREATE TABLE IF NOT EXISTS orders(
        id INTEGER PRIMARY KEY AUTOINCREMENT,
        user_id INTEGER,
        product_id INTEGER,
        price REAL,
        status TEXT DEFAULT 'Pending'
    );

    CREATE TABLE IF NOT EXISTS transactions(
        id INTEGER PRIMARY KEY AUTOINCREMENT,
        user_id INTEGER,
        amount REAL,
        description TEXT
    );
    """)

    admin = con.execute(
        "SELECT id FROM users WHERE username=?",
        (ADMIN_USER,)
    ).fetchone()

    if not admin:
        con.execute(
            "INSERT INTO users(username,password,is_admin) VALUES(?,?,1)",
            (ADMIN_USER, generate_password_hash(ADMIN_PASSWORD))
        )

    con.commit()
    con.close()


init_db()


STYLE = """
*{box-sizing:border-box}
body{
 margin:0;background:#070914;color:#fff;
 font-family:Arial,sans-serif;direction:rtl
}
header{
 background:#11182b;padding:20px;
 border-bottom:1px solid #00e5ff;
 text-align:center
}
.logo{font-size:30px;font-weight:bold;color:#00e5ff}
.brand{color:#ffd166;font-size:12px;margin-top:5px}
nav{display:flex;flex-wrap:wrap;justify-content:center;gap:9px;padding:15px}
a,button{
 text-decoration:none;border:0;border-radius:10px;
 padding:12px 16px;background:#00cfe8;color:#07101c;
 font-weight:bold;cursor:pointer
}
a:hover,button:hover{filter:brightness(1.2);transform:translateY(-2px)}
.container{max-width:900px;margin:20px auto;padding:15px}
.card{
 background:#11182b;border:1px solid #263450;
 border-radius:16px;padding:18px;margin:14px 0;
 box-shadow:0 5px 20px #0005
}
h2{color:#00e5ff}
input{
 width:100%;padding:13px;margin:8px 0;
 background:#080f1d;color:white;border:1px solid #34445e;
 border-radius:9px
}
button{margin-top:8px}
.purple{background:#a855f7;color:white}
.gold{background:#ffd166}
.red{background:#ef4444;color:white}
.muted{color:#aab5ca}
.price{color:#ffd166;font-weight:bold}
.flash{background:#243d54;padding:12px;border-radius:9px;margin:10px 0}
.grid{display:grid;grid-template-columns:repeat(auto-fit,minmax(220px,1fr));gap:12px}
footer{text-align:center;color:#8491a8;padding:25px}
"""


PAGE = """
<!DOCTYPE html>
<html lang="ar">
<head>
<meta charset="UTF-8">
<meta name="viewport" content="width=device-width,initial-scale=1">
<title>SASA STORE PRO</title>
<style>""" + STYLE + """</style>
</head>
<body>
<header>
 <div class="logo">SASA STORE PRO</div>
 <div class="brand">mostafaelsarnoby</div>
</header>
<nav>
 <a href="/">الرئيسية</a>
 {% if user %}
 <a href="/wallet">رصيدي</a>
 <a href="/products">المنتجات</a>
 <a href="/orders">طلباتي</a>
 {% if user['is_admin'] %}
 <a href="/admin">الإدارة</a>
 {% endif %}
 <a href="/logout" class="red">خروج</a>
 {% else %}
 <a href="/login">دخول</a>
 <a href="/register" class="purple">حساب جديد</a>
 {% endif %}
</nav>
<div class="container">
 {% for message in get_flashed_messages() %}
 <div class="flash">{{message}}</div>
 {% endfor %}
 """ + "{% block content %}{% endblock %}" + """
</div>
<footer>SASA STORE PRO © 2026</footer>
<script>
document.querySelectorAll("button").forEach(btn=>{
 btn.addEventListener("click",()=>{
  try{
   const ctx=new(window.AudioContext||window.webkitAudioContext)();
   const osc=ctx.createOscillator();
   const gain=ctx.createGain();
   osc.connect(gain);gain.connect(ctx.destination);
   osc.frequency.value=650;
   gain.gain.value=.035;
   osc.start();
   osc.stop(ctx.currentTime+.07);
  }catch(e){}
 });
});
</script>
</body>
</html>
"""


def page(content):
    return render_template_string(
        PAGE.replace(
            "{% block content %}{% endblock %}",
            content
        ),
        user=current_user()
    )


def current_user():
    uid = session.get("uid")
    if not uid:
        return None
    con = db()
    user = con.execute(
        "SELECT * FROM users WHERE id=?", (uid,)
    ).fetchone()
    con.close()
    return user


def login_required(fn):
    @wraps(fn)
    def wrapper(*args, **kwargs):
        if not current_user():
            return redirect("/login")
        return fn(*args, **kwargs)
    return wrapper


def admin_required(fn):
    @wraps(fn)
    def wrapper(*args, **kwargs):
        user = current_user()
        if not user or not user["is_admin"]:
            flash("غير مسموح بالدخول")
            return redirect("/")
        return fn(*args, **kwargs)
    return wrapper


@app.route("/")
def home():
    content = """
    <div class="card">
      <h2>أهلاً بك في SASA STORE</h2>
      <p>متجرك للمنتجات الرقمية.</p>
      <p class="muted">إدارة سهلة - رصيد - طلبات - منتجات</p>
    </div>
    <div class="grid">
      <div class="card"><h2>محفظتك</h2><p>تابع رصيدك وطلبات الشحن.</p></div>
      <div class="card"><h2>المنتجات</h2><p>اكتشف المنتجات المتاحة.</p></div>
      <div class="card"><h2>الدعم</h2><p>رقم الشحن: """ + PHONE + """</p></div>
    </div>
    """
    return page(content)


@app.route("/register", methods=["GET", "POST"])
def register():
    if request.method == "POST":
        username = request.form.get("username", "").strip()
        password = request.form.get("password", "")

        if len(username) < 3 or len(password) < 10:
            flash("اسم المستخدم 3 أحرف على الأقل وكلمة المرور 10 أحرف.")
        else:
            con = db()
            try:
                con.execute(
                    "INSERT INTO users(username,password) VALUES(?,?)",
                    (username, generate_password_hash(password))
                )
                con.commit()
                flash("تم إنشاء الحساب. سجل دخولك.")
                return redirect("/login")
            except sqlite3.IntegrityError:
                flash("اسم المستخدم مستخدم بالفعل.")
            finally:
                con.close()

    return page("""
    <div class="card">
    <h2>إنشاء حساب</h2>
    <form method="post">
      <input name="username" placeholder="اسم المستخدم" required>
      <input name="password" type="password" placeholder="كلمة مرور قوية" required>
      <button>إنشاء الحساب</button>
    </form>
    </div>
    """)


@app.route("/login", methods=["GET", "POST"])
def login():
    if request.method == "POST":
        con = db()
        user = con.execute(
            "SELECT * FROM users WHERE username=?",
            (request.form.get("username", "").strip(),)
        ).fetchone()
        con.close()

        if user and check_password_hash(
            user["password"], request.form.get("password", "")
        ):
            session.clear()
            session["uid"] = user["id"]
            return redirect("/")
        flash("بيانات الدخول غير صحيحة.")

    return page("""
    <div class="card">
    <h2>تسجيل الدخول</h2>
    <form method="post">
      <input name="username" placeholder="اسم المستخدم" required>
      <input name="password" type="password" placeholder="كلمة المرور" required>
      <button>دخول</button>
    </form>
    </div>
    """)


@app.route("/logout")
def logout():
    session.clear()
    return redirect("/")


@app.route("/wallet")
@login_required
def wallet():
    user = current_user()
    con = db()
    transactions = con.execute(
        "SELECT * FROM transactions WHERE user_id=? ORDER BY id DESC",
        (user["id"],)
    ).fetchall()
    topups = con.execute(
        "SELECT * FROM topups WHERE user_id=? ORDER BY id DESC",
        (user["id"],)
    ).fetchall()
    con.close()

    content = f"""
    <div class="card">
      <h2>محفظتي</h2>
      <h1 class="price">{user['balance']:.2f} EGP</h1>
      <p>رقم الشحن: <b>{PHONE}</b></p>
      <p class="muted">بعد إرسال المبلغ، قدم طلب شحن وانتظر مراجعة الإدارة.</p>
      <form method="post" action="/topup">
        <input name="amount" type="number" min="1" max="10000"
        step="0.01" placeholder="مبلغ الشحن بالجنيه" required>
        <input name="reference" maxlength="100"
        placeholder="رقم مرجع التحويل أو ملاحظة" required>
        <button>إرسال طلب شحن</button>
      </form>
    </div>
    <div class="card"><h2>طلبات الشحن</h2>
    """
    for t in topups:
        content += f"<p>#{t['id']} - {t['amount']:.2f} جنيه - {t['status']}</p>"
    content += "</div><div class='card'><h2>سجل الرصيد</h2>"
    for t in transactions:
        content += f"<p>{t['description']} - {t['amount']:.2f} جنيه</p>"
    content += "</div>"
    return page(content)


@app.route("/topup", methods=["POST"])
@login_required
def topup():
    try:
        amount = float(request.form.get("amount", "0"))
        reference = request.form.get("reference", "").strip()
        if not 1 <= amount <= 10000 or not reference:
            raise ValueError
    except ValueError:
        flash("راجع مبلغ الشحن والبيانات.")
        return redirect("/wallet")

    con = db()
    con.execute(
        "INSERT INTO topups(user_id,amount,reference) VALUES(?,?,?)",
        (current_user()["id"], amount, reference)
    )
    con.commit()
    con.close()
    flash("تم إرسال الطلب للمراجعة.")
    return redirect("/wallet")


@app.route("/products")
@login_required
def products():
    con = db()
    rows = con.execute(
        "SELECT * FROM products ORDER BY id DESC"
    ).fetchall()
    con.close()

    content = "<h2>المنتجات المتاحة</h2><div class='grid'>"
    for p in rows:
        content += f"""
        <div class="card">
          <h2>{p['name']}</h2>
          <p>{p['description']}</p>
          <p class="price">{p['price']:.2f} EGP</p>
          <p>المخزون: {p['stock']}</p>
          <form method="post" action="/buy/{p['id']}">
            <button {'disabled' if p['stock'] < 1 else ''}>شراء</button>
          </form>
        </div>
        """
    content += "</div>"
    return page(content)


@app.route("/buy/<int:pid>", methods=["POST"])
@login_required
def buy(pid):
    user = current_user()
    con = db()
    p = con.execute(
        "SELECT * FROM products WHERE id=?", (pid,)
    ).fetchone()

    if not p or p["stock"] < 1:
        flash("المنتج غير متوفر.")
    elif user["balance"] < p["price"]:
        flash("رصيدك غير كافٍ.")
    else:
        con.execute(
            "UPDATE users SET balance=balance-? WHERE id=?",
            (p["price"], user["id"])
        )
        con.execute(
            "UPDATE products SET stock=stock-1 WHERE id=?",
            (pid,)
        )
        con.execute(
            "INSERT INTO orders(user_id,product_id,price) VALUES(?,?,?)",
            (user["id"], pid, p["price"])
        )
        con.execute(
            "INSERT INTO transactions(user_id,amount,description) VALUES(?,?,?)",
            (user["id"], -p["price"], "شراء " + p["name"])
        )
        flash("تم تسجيل طلبك بنجاح.")

    con.commit()
    con.close()
    return redirect("/orders")


@app.route("/orders")
@login_required
def orders():
    con = db()
    rows = con.execute("""
        SELECT orders.*, products.name
        FROM orders JOIN products ON products.id=orders.product_id
        WHERE orders.user_id=? ORDER BY orders.id DESC
    """, (current_user()["id"],)).fetchall()
    con.close()

    content = "<div class='card'><h2>طلباتي</h2>"
    for o in rows:
        content += f"<p>طلب #{o['id']} - {o['name']} - {o['price']} جنيه - {o['status']}</p>"
    content += "</div>"
    return page(content)


@app.route("/admin")
@admin_required
def admin():
    con = db()
    users = con.execute("SELECT COUNT(*) FROM users").fetchone()[0]
    products_count = con.execute("SELECT COUNT(*) FROM products").fetchone()[0]
    orders_count = con.execute("SELECT COUNT(*) FROM orders").fetchone()[0]
    pending = con.execute(
        "SELECT COUNT(*) FROM topups WHERE status='Pending'"
    ).fetchone()[0]
    con.close()

    return page(f"""
    <div class="card">
      <h2>لوحة الإدارة</h2>
      <div class="grid">
        <div>العملاء: {users}</div>
        <div>المنتجات: {products_count}</div>
        <div>الطلبات: {orders_count}</div>
        <div>طلبات الشحن المعلقة: {pending}</div>
      </div>
      <p><a href="/admin/product">إضافة منتج</a></p>
      <p><a href="/admin/topups">مراجعة طلبات الشحن</a></p>
      <p><a href="/admin/orders">إدارة الطلبات</a></p>
      <p><a href="/admin/users">إدارة العملاء</a></p>
    </div>
    """)


@app.route("/admin/product", methods=["GET", "POST"])
@admin_required
def admin_product():
    if request.method == "POST":
        name = request.form.get("name", "").strip()
        description = request.form.get("description", "").strip()
        try:
            price = float(request.form.get("price", "0"))
            stock = int(request.form.get("stock", "0"))
            if not name or price <= 0 or stock < 0:
                raise ValueError
        except ValueError:
            flash("راجع بيانات المنتج.")
            return redirect("/admin/product")

        con = db()
        con.execute(
            "INSERT INTO products(name,price,description,stock) VALUES(?,?,?,?)",
            (name, price, description, stock)
        )
        con.commit()
        con.close()
        flash("تمت إضافة المنتج.")
        return redirect("/admin")

    return page("""
    <div class="card">
    <h2>إضافة منتج</h2>
    <form method="post">
      <input name="name" placeholder="اسم المنتج" required>
      <input name="description" placeholder="وصف المنتج">
      <input name="price" type="number" min="0.01" step="0.01" placeholder="السعر" required>
      <input name="stock" type="number" min="0" placeholder="الكمية" required>
      <button>حفظ المنتج</button>
    </form>
    </div>
    """)


@app.route("/admin/topups")
@admin_required
def admin_topups():
    con = db()
    rows = con.execute("""
        SELECT topups.*, users.username
        FROM topups JOIN users ON users.id=topups.user_id
        ORDER BY topups.id DESC
    """).fetchall()
    con.close()

    content = "<div class='card'><h2>طلبات الشحن</h2>"
    for t in rows:
        content += f"""
        <p>#{t['id']} - {t['username']} - {t['amount']} جنيه</p>
        <p>المرجع: {t['reference']} | الحالة: {t['status']}</p>
        <form method="post" action="/admin/topup/{t['id']}/approve">
          <button>موافقة وإضافة الرصيد</button>
        </form>
        <form method="post" action="/admin/topup/{t['id']}/reject">
          <button class="red">رفض الطلب</button>
        </form><hr>
        """
    content += "</div>"
    return page(content)


@app.route("/admin/topup/<int:tid>/<action>", methods=["POST"])
@admin_required
def review_topup(tid, action):
    if action not in ("approve", "reject"):
        return "Invalid action", 400

    con = db()
    t = con.execute(
        "SELECT * FROM topups WHERE id=?", (tid,)
    ).fetchone()

    if not t or t["status"] != "Pending":
        flash("الطلب غير موجود أو تمت مراجعته.")
    elif action == "approve":
        con.execute(
            "UPDATE users SET balance=balance+? WHERE id=?",
            (t["amount"], t["user_id"])
        )
        con.execute(
            "UPDATE topups SET status='Approved' WHERE id=?",
            (tid,)
        )
        con.execute(
            "INSERT INTO transactions(user_id,amount,description) VALUES(?,?,?)",
            (t["user_id"], t["amount"], "شحن رصيد")
        )
        flash("تمت الموافقة وإضافة الرصيد.")
    else:
        con.execute(
            "UPDATE topups SET status='Rejected' WHERE id=?",
            (tid,)
        )
        flash("تم رفض طلب الشحن.")

    con.commit()
    con.close()
    return redirect("/admin/topups")


@app.route("/admin/orders")
@admin_required
def admin_orders():
    con = db()
    rows = con.execute("""
        SELECT orders.*, users.username, products.name
        FROM orders
        JOIN users ON users.id=orders.user_id
        JOIN products ON products.id=orders.product_id
        ORDER BY orders.id DESC
    """).fetchall()
    con.close()

    content = "<div class='card'><h2>إدارة الطلبات</h2>"
    for o in rows:
        content += f"""
        <p>#{o['id']} - {o['username']} - {o['name']} - {o['status']}</p>
        <form method="post" action="/admin/order/{o['id']}">
          <select name="status">
            <option>Pending</option>
            <option>Processing</option>
            <option>Completed</option>
            <option>Cancelled</option>
          </select>
          <button>تحديث الحالة</button>
        </form><hr>
        """
    content += "</div>"
    return page(content)


@app.route("/admin/order/<int:oid>", methods=["POST"])
@admin_required
def update_order(oid):
    status = request.form.get("status")
    if status not in ("Pending", "Processing", "Completed", "Cancelled"):
        flash("حالة غير صحيحة.")
    else:
        con = db()
        con.execute(
            "UPDATE orders SET status=? WHERE id=?",
            (status, oid)
        )
        con.commit()
        con.close()
        flash("تم تحديث الطلب.")
    return redirect("/admin/orders")


@app.route("/admin/users")
@admin_required
def admin_users():
    con = db()
    rows = con.execute(
        "SELECT id,username,balance FROM users ORDER BY id DESC"
    ).fetchall()
    con.close()

    content = "<div class='card'><h2>إدارة العملاء</h2>"
    for u in rows:
        content += f"""
        <p>{u['username']} - الرصيد: {u['balance']:.2f} جنيه</p>
        <form method="post" action="/admin/user/{u['id']}">
          <input name="amount" type="number" step="0.01" min="0.01"
          placeholder="مبلغ تعديل الرصيد" required>
          <button>إضافة رصيد</button>
        </form><hr>
        """
    content += "</div>"
    return page(content)


@app.route("/admin/user/<int:uid>", methods=["POST"])
@admin_required
def admin_user_balance(uid):
    try:
        amount = float(request.form.get("amount", "0"))
        if not 0 < amount <= 100000:
            raise ValueError
    except ValueError:
        flash("المبلغ غير صحيح.")
        return redirect("/admin/users")

    con = db()
    exists = con.execute(
        "SELECT id FROM users WHERE id=?", (uid,)
    ).fetchone()

    if exists:
        con.execute(
            "UPDATE users SET balance=balance+? WHERE id=?",
            (amount, uid)
        )
        con.execute(
            "INSERT INTO transactions(user_id,amount,description) VALUES(?,?,?)",
            (uid, amount, "إضافة رصيد من الإدارة")
        )
        con.commit()
        flash("تمت إضافة الرصيد.")
    con.close()
    return redirect("/admin/users")


if __name__ == "__main__":
    app.run(host="0.0.0.0", port=5000, debug=False)
