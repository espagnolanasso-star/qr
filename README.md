from flask import Flask, request, send_file
from datetime import datetime
import qrcode
from io import BytesIO
import sqlite3
import requests
import os

app = Flask(__name__)


PORT = int(os.environ.get("PORT", 5000))
PUBLIC_URL = "https://TU-APP.up.railway.app"

PASSWORD = "admin123"

def init_db():
    conn = sqlite3.connect("data.db")
    c = conn.cursor()
    c.execute("""
        CREATE TABLE IF NOT EXISTS logs (
            id INTEGER PRIMARY KEY AUTOINCREMENT,
            ip TEXT,
            device TEXT,
            country TEXT,
            time TEXT
        )
    """)
    conn.commit()
    conn.close()

init_db()

def get_country(ip):
    try:
        res = requests.get(f"http://ip-api.com/json/{ip}", timeout=3)
        data = res.json()
        return data.get("country", "Unknown")
    except:
        return "Unknown"

def detectar_dispositivo(user_agent):
    if user_agent and "Mobile" in user_agent:
        return "📱 Móvil"
    return "💻 PC"

def get_real_ip():
    return request.headers.get("X-Forwarded-For", request.remote_addr)

def index():
    ip = get_real_ip()
    user_agent = request.headers.get("User-Agent")
    device = detectar_dispositivo(user_agent)
    time = datetime.now().strftime("%H:%M:%S")
    country = get_country(ip)

    conn = sqlite3.connect("data.db")
    c = conn.cursor()
    c.execute(
        "INSERT INTO logs (ip, device, country, time) VALUES (?, ?, ?, ?)",
        (ip, device, country, time),
    )
    conn.commit()
    conn.close()

    return """
    <html>
    <body style="text-align:center; font-family:Arial;">
        <h1>Registro completado</h1>
        <p>Puedes cerrar esta página.</p>
    </body>
    </html>
    """

@app.route("/qr")
def qr():
    img = qrcode.make(PUBLIC_URL)
    buffer = BytesIO()
    img.save(buffer)
    buffer.seek(0)
    return send_file(buffer, mimetype="image/png")


@app.route("/login")
def login():
    return """
    <form method="POST" action="/dashboard">
        <input type="password" name="password" placeholder="Password">
        <button type="submit">Entrar</button>
    </form>
    """


@app.route("/dashboard", methods=["GET", "POST"])
def dashboard():
    if request.method == "POST":
        if request.form.get("password") != PASSWORD:
            return "❌ Contraseña incorrecta"

    conn = sqlite3.connect("data.db")
    c = conn.cursor()
    c.execute("SELECT * FROM logs ORDER BY id DESC")
    data = c.fetchall()
    conn.close()

    rows = ""
    for log in data:
        rows += f"""
        <tr>
            <td>{log[4]}</td>
            <td>{log[1]}</td>
            <td>{log[2]}</td>
            <td>{log[3]}</td>
        </tr>
        """

    return f"""
    <html>
    <head>
        <meta http-equiv="refresh" content="3">
        <style>
            body {{
                background:#0d1117;
                color:#00ffcc;
                font-family:Arial;
                text-align:center;
            }}
            table {{
                width:90%;
                margin:auto;
                border-collapse:collapse;
            }}
            th,td {{
                border:1px solid #00ffcc;
                padding:10px;
            }}
        </style>
    </head>
    <body>
        <h1>📊 Dashboard PRO</h1>
        <h2>Total visitas: {len(data)}</h2>

        <img src="/qr" width="200">
        <p>{PUBLIC_URL}</p>

        <table>
            <tr>
                <th>Hora</th>
                <th>IP</th>
                <th>Dispositivo</th>
                <th>País</th>
            </tr>
            {rows}
        </table>
    </body>
    </html>
    """


if __name__ == "__main__":
    app.run(host="0.0.0.0", port=PORT)
