
import tkinter as tk
from tkinter import messagebox, simpledialog
import sqlite3
import re
import time

# =====================================
# SASA STORE PRO
# BRAND: mostafaelsarnoby
# =====================================

STORE_NAME = "SASA STORE"
BRAND = "mostafaelsarnoby"

ADMIN_USER = "admin"
ADMIN_PASSWORD = "Sasa@2026"

BG = "#070914"
PANEL = "#11182B"
BLUE = "#00E5FF"
PURPLE = "#A855F7"
GOLD = "#FFD166"
WHITE = "#FFFFFF"
RED = "#EF4444"

db = sqlite3.connect("sasa_store.db")
cur = db.cursor()

cur.executescript("""
CREATE TABLE IF NOT EXISTS products(
 id INTEGER PRIMARY KEY AUTOINCREMENT,
 name TEXT NOT NULL,
 price REAL NOT NULL,
 description TEXT,
 stock INTEGER DEFAULT 0
);

CREATE TABLE IF NOT EXISTS coupons(
 id INTEGER PRIMARY KEY AUTOINCREMENT,
 code TEXT UNIQUE,
 discount REAL
);

CREATE TABLE IF NOT EXISTS orders(
 id INTEGER PRIMARY KEY AUTOINCREMENT,
 customer TEXT,
 product TEXT,
 price REAL,
 status TEXT DEFAULT 'Pending'
);

CREATE TABLE IF NOT EXISTS activity(
 id INTEGER PRIMARY KEY AUTOINCREMENT,
 action TEXT,
 details TEXT,
 date TEXT
);
""")
db.commit()

root = tk.Tk()
root.title("SASA STORE PRO")
root.geometry("410x780")
root.configure(bg=BG)
root.resizable(True, True)

# =====================================
# GENERAL FUNCTIONS
# =====================================

def clear():
    for widget in root.winfo_children():
        widget.destroy()

def log(action, details=""):
    cur.execute(
        "INSERT INTO activity(action,details,date) VALUES(?,?,?)",
        (action, details, time.strftime("%Y-%m-%d %H:%M"))
    )
    db.commit()

def title(text):
    tk.Label(
        root,
        text=text,
        bg=BG,
        fg=BLUE,
        font=("Arial", 23, "bold")
    ).pack(pady=15)

def text(parent, value, color=WHITE, size=12):
    return tk.Label(
        parent,
        text=value,
        bg=parent["bg"],
        fg=color,
        font=("Arial", size, "bold")
    )

def button(parent, name, action, color=BLUE):
    return tk.Button(
        parent,
        text=name,
        command=action,
        bg=color,
        fg=BG,
        activebackground=PURPLE,
        activeforeground=WHITE,
        font=("Arial", 12, "bold"),
        relief="flat",
        cursor="hand2",
        pady=12
    )

def field(parent, hidden=None):
    e = tk.Entry(
        parent,
        font=("Arial", 13),
        bg="#080F1D",
        fg=WHITE,
        insertbackground=WHITE,
        relief="flat",
        show=hidden
    )
    e.pack(fill="x", pady=8, ipady=10)
    return e

def brand():
    text(root, "SASA", BLUE, 32).pack(pady=(15, 0))
    text(root, "STORE PRO", PURPLE, 17).pack()
    text(root, BRAND, GOLD, 11).pack(pady=5)

def card():
    frame = tk.Frame(
        root,
        bg=PANEL,
        padx=18,
        pady=18
    )
    frame.pack(fill="x", padx=18, pady=10)
    return frame

# =====================================
# LOGIN SYSTEM
# =====================================

def login():
    clear()
    brand()

    panel = card()

    text(panel, "ADMIN LOGIN", PURPLE, 19).pack(pady=15)

    text(panel, "Username").pack(anchor="w")
    username = field(panel)

    text(panel, "Password").pack(anchor="w")
    password = field(panel, "*")

    def verify():
        if (
            username.get() == ADMIN_USER
            and password.get() == ADMIN_PASSWORD
        ):
            log("LOGIN", "Admin logged in")
            dashboard()
        else:
            messagebox.showerror(
                "SASA SECURITY",
                "اسم المستخدم أو كلمة المرور غير صحيحة"
            )

    button(panel, "LOGIN TO SASA", verify).pack(
        fill="x", pady=20
    )

    text(
        root,
        "SECURE DIGITAL STORE",
        PURPLE,
        10
    ).pack(side="bottom", pady=20)

# =====================================
# DASHBOARD
# =====================================

def dashboard():
    clear()
    brand()

    text(root, "ADMIN CONTROL CENTER", WHITE, 14).pack(pady=12)

    cur.execute("SELECT COUNT(*) FROM products")
    products = cur.fetchone()[0]

    cur.execute("SELECT COUNT(*) FROM orders")
    orders = cur.fetchone()[0]

    cur.execute("SELECT COUNT(*) FROM coupons")
    coupons = cur.fetchone()[0]

    panel = card()

    for name, value in [
        ("TOTAL PRODUCTS", products),
        ("TOTAL ORDERS", orders),
        ("ACTIVE COUPONS", coupons)
    ]:
        text(panel, name, PURPLE, 11).pack(pady=4)
        text(panel, str(value), BLUE, 22).pack(pady=3)

    options = [
        ("ADD PRODUCT", add_product, BLUE),
        ("PRODUCT MANAGEMENT", products_page, PURPLE),
        ("COUPON MANAGEMENT", coupons_page, GOLD),
        ("ORDERS MANAGEMENT", orders_page, BLUE),
        ("PHONE VALIDATION", phone_page, PURPLE),
        ("ACTIVITY HISTORY", history_page, GOLD),
        ("STORE INFORMATION", about_page, BLUE),
        ("LOG OUT", login, RED)
    ]

    for name, action, color in options:
        button(root, name, action, color).pack(
            fill="x", padx=22, pady=5
        )

# =====================================
# ADD PRODUCT
# =====================================

def add_product():
    clear()
    brand()
    title("ADD NEW PRODUCT")

    panel = card()

    text(panel, "Product Name").pack(anchor="w")
    name = field(panel)

    text(panel, "Price (EGP)").pack(anchor="w")
    price = field(panel)

    text(panel, "Description").pack(anchor="w")
    desc = field(panel)

    text(panel, "Stock Quantity").pack(anchor="w")
    stock = field(panel)

    def save():
        try:
            n = name.get().strip()
            p = float(price.get())
            d = desc.get().strip()
            s = int(stock.get() or "0")

            if not n or p <= 0 or s < 0:
                raise ValueError

            cur.execute("""
                INSERT INTO products(name,price,description,stock)
                VALUES(?,?,?,?)
            """, (n, p, d, s))

            db.commit()
            log("ADD PRODUCT", n)

            messagebox.showinfo(
                "SASA STORE",
                "تم إضافة المنتج بنجاح"
            )
            dashboard()

        except ValueError:
            messagebox.showerror(
                "ERROR",
                "تأكد من الاسم والسعر والكمية"
            )

    button(panel, "SAVE PRODUCT", save).pack(
        fill="x", pady=18
    )

    button(root, "BACK", dashboard).pack(pady=10)

# =====================================
# PRODUCT MANAGEMENT
# =====================================

def products_page():
    clear()
    brand()
    title("PRODUCT MANAGEMENT")

    container = tk.Frame(root, bg=BG)
    container.pack(fill="both", expand=True)

    canvas = tk.Canvas(
        container,
        bg=BG,
        highlightthickness=0
    )
    scroll = tk.Scrollbar(
        container,
        command=canvas.yview
    )
    inside = tk.Frame(canvas, bg=BG)

    inside.bind(
        "<Configure>",
        lambda e: canvas.configure(
            scrollregion=canvas.bbox("all")
        )
    )

    canvas.create_window(
        (0, 0),
        window=inside,
        anchor="nw",
        width=355
    )

    canvas.configure(yscrollcommand=scroll.set)

    canvas.pack(side="left", fill="both", expand=True)
    scroll.pack(side="right", fill="y")

    cur.execute("SELECT * FROM products ORDER BY id DESC")

    for product in cur.fetchall():
        p = product

        panel = tk.Frame(
            inside,
            bg=PANEL,
            padx=12,
            pady=12
        )
        panel.pack(fill="x", padx=8, pady=7)

        text(panel, p[1], BLUE, 15).pack(anchor="w")
        text(panel, f"PRICE: {p[2]} EGP", GOLD).pack(anchor="w")
        text(panel, p[3] or "No description", WHITE, 10).pack(anchor="w")
        text(panel, f"STOCK: {p[4]}", PURPLE).pack(anchor="w")

        def edit(pid=p[0], old=p[2]):
            value = simpledialog.askfloat(
                "EDIT PRICE",
                "Enter new price",
                initialvalue=old
            )
            if value is not None and value > 0:
                cur.execute(
                    "UPDATE products SET price=? WHERE id=?",
                    (value, pid)
                )
                db.commit()
                log("EDIT PRICE", str(pid))
                products_page()

        def delete(pid=p[0], pname=p[1]):
            if messagebox.askyesno(
                "DELETE",
                "هل تريد حذف هذا المنتج؟"
            ):
                cur.execute(
                    "DELETE FROM products WHERE id=?",
                    (pid,)
                )
                db.commit()
                log("DELETE PRODUCT", pname)
                products_page()

        button(panel, "EDIT PRICE", edit, PURPLE).pack(
            side="left", padx=4
        )
        button(panel, "DELETE", delete, RED).pack(
            side="left", padx=4
        )

    button(root, "BACK", dashboard).pack(pady=12)

# =====================================
# COUPON MANAGEMENT
# =====================================

def coupons_page():
    clear()
    brand()
    title("DISCOUNT COUPONS")

    panel = card()

    text(panel, "Coupon Code").pack(anchor="w")
    code = field(panel)

    text(panel, "Discount Percentage").pack(anchor="w")
    discount = field(panel)

    def save():
        try:
            c = code.get().strip().upper()
            d = float(discount.get())

            if not c or not 0 < d <= 100:
                raise ValueError

            cur.execute(
                "INSERT INTO coupons(code,discount) VALUES(?,?)",
                (c, d)
            )
            db.commit()
            log("NEW COUPON", c)

            messagebox.showinfo(
                "SASA",
                "تم إنشاء الكوبون"
            )
            coupons_page()

        except ValueError:
            messagebox.showerror(
                "ERROR",
                "نسبة الخصم يجب أن تكون بين 1 و100"
            )
        except sqlite3.IntegrityError:
            messagebox.showerror(
                "ERROR",
                "الكوبون موجود مسبقاً"
            )

    button(panel, "CREATE COUPON", save).pack(
        fill="x", pady=12
    )

    cur.execute("SELECT code,discount FROM coupons")

    for c, d in cur.fetchall():
        text(
            root,
            f"{c}  |  {d}% OFF",
            GOLD
        ).pack(pady=4)

    button(root, "BACK", dashboard).pack(pady=15)

# =====================================
# ORDERS MANAGEMENT
# =====================================

def orders_page():
    clear()
    brand()
    title("ORDERS MANAGEMENT")

    cur.execute("SELECT * FROM orders ORDER BY id DESC")
    rows = cur.fetchall()

    for order in rows:
        panel = card()

        text(panel, f"ORDER #{order[0]}", BLUE).pack(anchor="w")
        text(panel, f"CUSTOMER: {order[1]}", WHITE).pack(anchor="w")
        text(panel, f"PRODUCT: {order[2]}", WHITE).pack(anchor="w")
        text(panel, f"PRICE: {order[3]} EGP", GOLD).pack(anchor="w")
        text(panel, f"STATUS: {order[4]}", PURPLE).pack(anchor="w")

        def change(oid=order[0]):
            value = simpledialog.askstring(
                "ORDER STATUS",
                "اكتب الحالة الجديدة:"
            )
            if value:
                cur.execute(
                    "UPDATE orders SET status=? WHERE id=?",
                    (value, oid)
                )
                db.commit()
                log("ORDER UPDATE", str(oid))
                orders_page()

        button(panel, "UPDATE STATUS", change, PURPLE).pack(
            fill="x", pady=5
        )

    if not rows:
        text(root, "NO ORDERS YET", PURPLE).pack(pady=20)

    button(root, "BACK", dashboard).pack(pady=12)

# =====================================
# PHONE VALIDATION
# =====================================

def phone_page():
    clear()
    brand()
    title("PHONE VALIDATION")

    panel = card()

    text(panel, "INTERNATIONAL PHONE NUMBER").pack(anchor="w")
    phone = field(panel)
    phone.insert(0, "+20")

    def verify():
        number = phone.get().strip()

        if re.fullmatch(r"\+[1-9][0-9]{7,14}", number):
            messagebox.showinfo(
                "VALID",
                "صيغة الرقم صحيحة فقط"
            )
        else:
            messagebox.showerror(
                "INVALID",
                "رقم الهاتف غير صحيح"
            )

    button(panel, "VERIFY NUMBER", verify).pack(
        fill="x", pady=15
    )

    text(
        panel,
        "لا يؤكد هذا الفحص ملكية الرقم",
        GOLD,
        10
    ).pack()

    button(root, "BACK", dashboard).pack(pady=12)

# =====================================
# ACTIVITY HISTORY
# =====================================

def history_page():
    clear()
    brand()
    title("ACTIVITY HISTORY")

    cur.execute("""
        SELECT action,details,date
        FROM activity
        ORDER BY id DESC LIMIT 50
    """)

    rows = cur.fetchall()

    for action, details, date in rows:
        panel = tk.Frame(root, bg=PANEL, padx=12, pady=9)
        panel.pack(fill="x", padx=12, pady=5)

        text(panel, action, BLUE).pack(anchor="w")
        text(panel, details, WHITE, 10).pack(anchor="w")
        text(panel, date, GOLD, 9).pack(anchor="w")

    button(root, "BACK", dashboard).pack(pady=15)

# =====================================
# ABOUT SASA
# =====================================

def about_page():
    clear()
    brand()

    panel = card()
    

    text(panel, "WELCOME TO SASA STORE", BLUE, 18).pack(pady=12)
    text(panel, "BRAND OWNER", PURPLE).pack()
    text(panel, BRAND, GOLD, 15).pack(pady=10)
    text(panel, "DIGITAL PRODUCTS PLATFORM", WHITE).pack(pady=10)
    text(panel, "VERSION 2.0", BLUE).pack(pady=10)

    button(root, "BACK", dashboard).pack(pady=15)

# =====================================
# START APPLICATION
# =====================================

login()
root.mainloop()

