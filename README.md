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
        .login-container { max-width: 400px; margin: 80px auto; background: #fff; border-radius: 10px; box-shadow: 0 0 16px #0001; padding: 32px; }
        .dark .login-container { background: #23272b !important; }
    </style>
</head>
<body>
<div id="login" class="login-container">
    <h2 class="mb-4 text-center">Gerenciador MPasc</h2>
    <div id="login-error" class="alert alert-danger d-none"></div>
    <form id="login-form">
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

<div id="dashboard" class="container py-4 d-none">
    <div class="d-flex justify-content-between align-items-center mb-4">
        <h1 class="mb-0">Gerenciador MPasc</h1>
        <div>
            <button class="btn btn-outline-danger ms-2" onclick="logout()">Sair</button>
            <button class="btn btn-outline-secondary theme-toggle" onclick="toggleTheme()" type="button">
                <span id="theme-icon">☀️</span> Tema
            </button>
        </div>
    </div>
    <form class="row g-3 align-items-end mb-4" onsubmit="event.preventDefault(); filterMachines();">
        <div class="col-auto network-select">
            <label class="form-label">Rede:</label>
            <select id="network-select" class="form-select" onchange="filterMachines()"></select>
        </div>
        <div class="col-auto">
            <label class="form-label">Data:</label>
            <input type="date" id="date-input" class="form-control" onchange="filterMachines()">
        </div>
        <div class="col-auto search-box">
            <label class="form-label">Buscar máquina:</label>
            <input type="text" id="search-input" class="form-control" placeholder="Nome da máquina" oninput="filterMachines()">
        </div>
        <div class="col-auto">
            <button type="button" class="btn btn-primary" onclick="filterMachines()">Pesquisar</button>
            <button type="button" class="btn btn-secondary ms-2" onclick="clearFilters()">Limpar</button>
        </div>
    </form>
    <div class="accordion" id="logsAccordion"></div>
    <div class="footer">
        Desenvolvedor: Elias Melo
    </div>
</div>

<script>
// Dados simulados
const machines = [
    { name: "PC1 Elias", ip: "192.168.15.15", network: "Vivo" },
    { name: "PC2 Breno", ip: "192.168.15.16", network: "Vivo" },
    { name: "PC2 Breno", ip: "10.0.0.16", network: "Claro" },
    { name: "PC3 Elias", ip: "192.168.1.66", network: "Claro" },
    { name: "Maria", ip: "192.168.15.20", network: "Vivo" },
    { name: "Scarlet", ip: "192.168.15.21", network: "Vivo" },
    { name: "Caroline", ip: "192.168.15.22", network: "Vivo" },
    { name: "Kelly", ip: "192.168.15.23", network: "Vivo" },
    { name: "Mariana", ip: "192.168.15.24", network: "Vivo" },
    { name: "Jessica", ip: "192.168.15.25", network: "Vivo" },
    { name: "Daiane", ip: "192.168.15.26", network: "Vivo" },
    { name: "Kathlen", ip: "192.168.15.27", network: "Vivo" },
    { name: "Mariane", ip: "192.168.15.28", network: "Vivo" },
    { name: "Maria", ip: "10.0.0.20", network: "Claro" },
    { name: "Scarlet", ip: "10.0.0.21", network: "Claro" },
    { name: "Caroline", ip: "10.0.0.22", network: "Claro" },
    { name: "Kelly", ip: "10.0.0.23", network: "Claro" },
    { name: "Mariana", ip: "10.0.0.24", network: "Claro" },
    { name: "Jessica", ip: "10.0.0.25", network: "Claro" },
    { name: "Daiane", ip: "10.0.0.26", network: "Claro" },
    { name: "Kathlen", ip: "10.0.0.27", network: "Claro" },
    { name: "Mariane", ip: "10.0.0.28", network: "Claro" },
];
const networks = [...new Set(machines.map(m => m.network))];
const fakeLogs = [
    "2024-06-18 08:00:01 - Sistema iniciado",
    "2024-06-18 08:05:12 - Usuário logado",
    "2024-06-18 09:15:45 - Ação executada",
    "2024-06-18 10:00:00 - Sistema fechado"
];

// Login fake
document.getElementById('login-form').onsubmit = function(e) {
    e.preventDefault();
    const user = this.username.value;
    const pass = this.password.value;
    if (user === "adm" && pass === "2025Deus2026") {
        document.getElementById('login').classList.add('d-none');
        document.getElementById('dashboard').classList.remove('d-none');
        loadNetworks();
        filterMachines();
        setTheme(localStorage.getItem('theme') || 'light');
    } else {
        document.getElementById('login-error').textContent = "Usuário ou senha inválidos.";
        document.getElementById('login-error').classList.remove('d-none');
    }
};

function logout() {
    document.getElementById('dashboard').classList.add('d-none');
    document.getElementById('login').classList.remove('d-none');
    document.getElementById('login-form').reset();
    document.getElementById('login-error').classList.add('d-none');
}

// Tema
function toggleTheme() {
    let body = document.body;
    let theme = body.classList.toggle('dark') ? 'dark' : '';
    document.getElementById('theme-icon').textContent = body.classList.contains('dark') ? '🌙' : '☀️';
    localStorage.setItem('theme', body.classList.contains('dark') ? 'dark' : 'light');
}
function setTheme(theme) {
    if (theme === 'dark') {
        document.body.classList.add('dark');
        document.getElementById('theme-icon').textContent = '🌙';
    } else {
        document.body.classList.remove('dark');
        document.getElementById('theme-icon').textContent = '☀️';
    }
}
window.onload = function() {
    setTheme(localStorage.getItem('theme') || 'light');
};

// Filtros e renderização
function loadNetworks() {
    const select = document.getElementById('network-select');
    select.innerHTML = '';
    networks.forEach(net => {
        const opt = document.createElement('option');
        opt.value = net;
        opt.textContent = net;
        select.appendChild(opt);
    });
}
function filterMachines() {
    const net = document.getElementById('network-select').value;
    const search = document.getElementById('search-input').value.toLowerCase();
    const date = document.getElementById('date-input').value;
    const filtered = machines.filter(m => 
        m.network === net && m.name.toLowerCase().includes(search)
    );
    renderLogs(filtered, date);
}
function clearFilters() {
    document.getElementById('network-select').selectedIndex = 0;
    document.getElementById('search-input').value = '';
    document.getElementById('date-input').value = '';
    filterMachines();
}
function renderLogs(machines, date) {
    const acc = document.getElementById('logsAccordion');
    acc.innerHTML = '';
    machines.forEach((m, idx) => {
        const uid = (m.name + m.ip).replace(/[\s\.\(\)]/g, '');
        let logs = fakeLogs;
        if (date) {
            logs = logs.filter(l => l.includes(date));
        }
        let logsHtml = logs.length
            ? logs.map(l => `<div class="log-line">${l}</div>`).join('')
            : `<p class="text-danger">Não foi possível obter os logs (Sistema está fechado nessa máquina).</p>`;
        acc.innerHTML += `
        <div class="accordion-item">
            <h2 class="accordion-header" id="heading${uid}">
                <button class="accordion-button collapsed" type="button" data-bs-toggle="collapse" data-bs-target="#collapse${uid}" aria-expanded="false" aria-controls="collapse${uid}">
                    ${m.name} (${m.ip}) - ${m.network}
                </button>
            </h2>
            <div id="collapse${uid}" class="accordion-collapse collapse" aria-labelledby="heading${uid}" data-bs-parent="#logsAccordion">
                <div class="accordion-body">
                    <div class="log-list">${logsHtml}</div>
                </div>
            </div>
        </div>
        `;
    });
}
</script>
<script src="https://cdn.jsdelivr.net/npm/bootstrap@5.3.3/dist/js/bootstrap.bundle.min.js"></script>
</body>
</html>
