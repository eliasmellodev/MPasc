from flask import Flask, render_template_string, request, redirect, url_for, session
import requests
import threading
import webbrowser
from datetime import datetime

app = Flask(__name__)
app.secret_key = "chave_secreta_segura"

machines = [
    {"name": "PC1 Elias", "ip": "192.168.15.15", "network": "Vivo"},
    {"name": "PC2 Breno", "ip": "192.168.15.16", "network": "Vivo"},
    {"name": "PC2 Breno", "ip": "10.0.0.16", "network": "Claro"},
    {"name": "PC3 Elias", "ip": "192.168.1.66", "network": "Claro"},
    {"name": "Maria", "ip": "192.168.15.20", "network": "Vivo"},
    {"name": "Scarlet", "ip": "192.168.15.21", "network": "Vivo"},
    {"name": "Caroline", "ip": "192.168.15.22", "network": "Vivo"},
    {"name": "Kelly", "ip": "192.168.15.23", "network": "Vivo"},
    {"name": "Mariana", "ip": "192.168.15.24", "network": "Vivo"},
    {"name": "Jessica", "ip": "192.168.15.25", "network": "Vivo"},
    {"name": "Daiane", "ip": "192.168.15.26", "network": "Vivo"},
    {"name": "Kathlen", "ip": "192.168.15.27", "network": "Vivo"},
    {"name": "Mariane", "ip": "192.168.15.28", "network": "Vivo"},
    {"name": "Maria", "ip": "10.0.0.20", "network": "Claro"},
    {"name": "Scarlet", "ip": "10.0.0.21", "network": "Claro"},
    {"name": "Caroline", "ip": "10.0.0.22", "network": "Claro"},
    {"name": "Kelly", "ip": "10.0.0.23", "network": "Claro"},
    {"name": "Mariana", "ip": "10.0.0.24", "network": "Claro"},
    {"name": "Jessica", "ip": "10.0.0.25", "network": "Claro"},
    {"name": "Daiane", "ip": "10.0.0.26", "network": "Claro"},
    {"name": "Kathlen", "ip": "10.0.0.27", "network": "Claro"},
    {"name": "Mariane", "ip": "10.0.0.28", "network": "Claro"},
]

networks = sorted(list(set(m["network"] for m in machines)))

LOGIN_TEMPLATE = """
<!DOCTYPE html>
<html lang="pt-br">
<head>
    <meta charset="UTF-8">
    <title>Login - Gerenciador MPasc</title>
    <meta name="viewport" content="width=device-width, initial-scale=1">
    <link href="https://cdn.jsdelivr.net/npm/bootstrap@5.3.3/dist/css/bootstrap.min.css" rel="stylesheet">
    <style>
        body { background: #f8f9fa; }
        .login-container { max-width: 400px; margin: 80px auto; background: #fff; border-radius: 10px; box-shadow: 0 0 16px #0001; padding: 32px; }
        .footer { margin-top: 32px; text-align: center; color: #888; font-size: 0.95rem; }
    </style>
</head>
<body>
    <div class="login-container">
        <h2 class="mb-4 text-center">Gerenciador MPasc</h2>
        {% if error %}
            <div class="alert alert-danger">{{ error }}</div>
        {% endif %}
        <form method="post">
            <div class="mb-3">
                <label class="form-label">Usuário</label>
                <input type="text" name="username" class="form-control" required autofocus>
            </div>
            <div class="mb-3">
                <label class="form-label">Senha</label>
                <input type="password" name="password" class="form-control" required>
            </div>
            <button class="btn btn-primary w-100" type="submit">Entrar</button>
        </form>
        <div class="footer mt-4">
            Desenvolvedor: Elias Melo<br>
            <span>Gerenciamento remoto MPasc</span>
        </div>
    </div>
</body>
</html>
"""

TEMPLATE = """
<!DOCTYPE html>
<html lang="pt-br">
<head>
    <meta charset="UTF-8">
    <title>Gerenciador MPasc</title>
    <meta name="viewport" content="width=device-width, initial-scale=1">
    <link id="theme-link" href="https://cdn.jsdelivr.net/npm/bootstrap@5.3.3/dist/css/bootstrap.min.css" rel="stylesheet">
    <style>
        body.dark { background: #181a1b !important; color: #f8f9fa !important; }
        .dark .accordion-button, .dark .form-label, .dark .form-control, .dark .btn, .dark .accordion-body { background: #23272b !important; color: #f8f9fa !important; }
        .dark .accordion-item { background: #23272b !important; }
        .dark .log-line { background: #111 !important; color: #0f0 !important; }
        .log-line { background: #222; color: #0f0; padding: 8px 12px; border-radius: 8px; font-family: monospace; margin-bottom: 6px; }
        .log-list { margin-top: 10px; }
        .search-box { max-width: 300px; }
        .network-select { max-width: 200px; }
        .theme-toggle { margin-left: 10px; }
        .footer { margin-top: 40px; text-align: center; color: #888; font-size: 0.95rem; }
    </style>
</head>
<body class="{{ 'dark' if theme == 'dark' else '' }}">
<div class="container py-4">
    <div class="d-flex justify-content-between align-items-center mb-4">
        <h1 class="mb-0">Gerenciador MPasc</h1>
        <div>
            <a href="{{ url_for('logout') }}" class="btn btn-outline-danger ms-2">Sair</a>
            <button class="btn btn-outline-secondary theme-toggle" onclick="toggleTheme()" type="button">
                <span id="theme-icon">{% if theme == 'dark' %}🌙{% else %}☀️{% endif %}</span> Tema
            </button>
        </div>
    </div>
    <form method="get" class="row g-3 align-items-end mb-4">
        <div class="col-auto network-select">
            <label class="form-label">Rede:</label>
            <select name="network" class="form-select" onchange="this.form.submit()">
                {% for net in networks %}
                    <option value="{{ net }}" {% if net == selected_network %}selected{% endif %}>{{ net }}</option>
                {% endfor %}
            </select>
        </div>
        <div class="col-auto">
            <label class="form-label">Data:</label>
            <input type="date" name="date" value="{{ selected_date }}" class="form-control">
        </div>
        <div class="col-auto search-box">
            <label class="form-label">Buscar máquina:</label>
            <input type="text" name="search" value="{{ search_query }}" class="form-control" placeholder="Nome da máquina">
        </div>
        <div class="col-auto">
            <button type="submit" class="btn btn-primary">Pesquisar</button>
            <a href="/" class="btn btn-secondary ms-2">Limpar</a>
        </div>
    </form>
    <div class="accordion" id="logsAccordion">
        {% for m in logs %}
        {% set uid = (m['machine'] ~ m['ip']).replace(' ', '').replace('.', '').replace('(', '').replace(')', '') %}
        <div class="accordion-item">
            <h2 class="accordion-header" id="heading{{ uid }}">
                <button class="accordion-button collapsed" type="button" data-bs-toggle="collapse" data-bs-target="#collapse{{ uid }}" aria-expanded="false" aria-controls="collapse{{ uid }}">
                    {{ m['machine'] }} ({{ m['ip'] }}) - {{ m['network'] }}
                </button>
            </h2>
            <div id="collapse{{ uid }}" class="accordion-collapse collapse" aria-labelledby="heading{{ uid }}" data-bs-parent="#logsAccordion">
                <div class="accordion-body">
                    {% if m['logs'] %}
                        <div class="log-list">
                            {% for linha in m['logs'].split('\n') %}
                                {% if linha.strip() %}
                                    {% set partes = linha.split(' - ', 1) %}
                                    {% if partes|length == 2 %}
                                        {% set data = partes[0] %}
                                        {% set resto = partes[1] %}
                                        {% if data|length >= 10 %}
                                            {% set data_formatada = data[8:10] ~ '/' ~ data[5:7] ~ '/' ~ data[0:4] ~ data[10:] %}
                                            <div class="log-line">{{ data_formatada }} - {{ resto }}</div>
                                        {% else %}
                                            <div class="log-line">{{ data }} - {{ resto }}</div>
                                        {% endif %}
                                    {% else %}
                                        <div class="log-line">{{ linha }}</div>
                                    {% endif %}
                                {% endif %}
                            {% endfor %}
                        </div>
                    {% else %}
                        <p class="text-danger">Não foi possível obter os logs (Sistema está fechado nessa máquina).</p>
                    {% endif %}
                </div>
            </div>
        </div>
        {% endfor %}
    </div>
    <div class="footer">
        Desenvolvedor: Elias Melo
    </div>
</div>
<script>
function toggleTheme() {
    let body = document.body;
    let theme = body.classList.toggle('dark') ? 'dark' : '';
    document.getElementById('theme-icon').textContent = body.classList.contains('dark') ? '🌙' : '☀️';
    fetch('/set_theme?theme=' + (body.classList.contains('dark') ? 'dark' : 'light'));
    localStorage.setItem('theme', body.classList.contains('dark') ? 'dark' : 'light');
}
window.onload = function() {
    if (localStorage.getItem('theme') === 'dark') {
        document.body.classList.add('dark');
        document.getElementById('theme-icon').textContent = '🌙';
    }
};
</script>
<script src="https://cdn.jsdelivr.net/npm/bootstrap@5.3.3/dist/js/bootstrap.bundle.min.js"></script>
</body>
</html>
"""

@app.route("/login", methods=["GET", "POST"])
def login():
    if request.method == "POST":
        username = request.form.get("username")
        password = request.form.get("password")
        if username == "adm" and password == "2025Deus2026":
            session["logged_in"] = True
            return redirect(url_for("dashboard"))
        else:
            return render_template_string(LOGIN_TEMPLATE, error="Usuário ou senha inválidos.")
    return render_template_string(LOGIN_TEMPLATE, error=None)

@app.route("/logout")
def logout():
    session.clear()
    return redirect(url_for("login"))

@app.route("/set_theme")
def set_theme():
    theme = request.args.get("theme", "light")
    session["theme"] = theme
    return "", 204

@app.route("/", methods=["GET"])
def dashboard():
    if not session.get("logged_in"):
        return redirect(url_for("login"))
    selected_network = request.args.get("network", networks[0])
    search_query = request.args.get("search", "").strip().lower()
    selected_date = request.args.get("date", "")

    filtered_machines = [m for m in machines if m["network"] == selected_network]
    if search_query:
        filtered_machines = [m for m in filtered_machines if search_query in m["name"].lower()]

    logs = []
    for m in filtered_machines:
        try:
            resp = requests.get(f"http://{m['ip']}:5000/logs", timeout=2)
            data = resp.json()
            linhas = data.get("logs", [])
            if selected_date:
                try:
                    date_br = datetime.strptime(selected_date, "%Y-%m-%d").strftime("%d/%m/%Y")
                    date_barra = datetime.strptime(selected_date, "%Y-%m-%d").strftime("%Y/%m/%d")
                except Exception:
                    date_br = ""
                    date_barra = ""
                linhas = [
                    l for l in linhas
                    if selected_date in l or date_br in l or date_barra in l
                ]
            linhas_str = "\n".join(linhas)
            logs.append({
                "machine": m["name"],
                "ip": m["ip"],
                "network": m["network"],
                "logs": linhas_str
            })
        except Exception:
            logs.append({
                "machine": m["name"],
                "ip": m["ip"],
                "network": m["network"],
                "logs": ""
            })
    theme = session.get("theme", "light")
    return render_template_string(
        TEMPLATE,
        logs=logs,
        machines=machines,
        networks=networks,
        selected_network=selected_network,
        selected_date=selected_date,
        search_query=search_query,
        theme=theme
    )

def open_browser():
    webbrowser.open("http://localhost:8000")

if __name__ == "__main__":
    threading.Timer(1.5, open_browser).start()
    app.run(debug=False, port=8000)
