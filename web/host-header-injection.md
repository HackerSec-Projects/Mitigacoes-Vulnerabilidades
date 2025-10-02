# 1) Princípios gerais (regra de ouro)

1. **Nunca** confie no header `Host` do cliente para decisões de segurança (ex.: geração de links de reset, redirects, seleção de backend).
2. **Use uma lista branca (allowlist)** de hostnames válidos. Rejeite (400/403) qualquer request cujo `Host` não esteja na lista.
3. **Normalize** e **valide** o `Host` (sem espaços, sem caracteres inválidos).
4. **Não construa URLs absolutos** com o `Host` do request sem antes validá-lo — prefira usar uma configuração do servidor (e.g. `APP_BASE_URL`) para gerar links externos.
5. **No reverse proxy**, não repasse automaticamente `Host` para backends a menos que seja intencional; defina explicitamente qual host o backend deve ver.

---

# 2) Mitigações práticas por camada

## 2.1 Nginx (exemplo)

* Defina `server_name` explicitamente e use um `default_server` que rejeita hosts desconhecidos:

```nginx
# server block para hosts válidos
server {
    listen 80;
    server_name middleware-api.shapedigital.com;
    # ... proxy_pass etc.
}

# bloco default: rejeita todo o resto
server {
    listen 80 default_server;
    return 444;   # fecha a conexão sem resposta
}
```

* Alternativa: checar e bloquear hosts inválidos:

```nginx
server {
    listen 80;
    server_name middleware-api.shapedigital.com;

    if ($host !~* ^(middleware-api\.shapedigital\.com)$) {
        return 400;
    }

    # ao proxy, forçar Host do backend (para evitar forwarding do client)
    location / {
        proxy_set_header Host middleware-api.shapedigital.com;
        proxy_pass http://backend_upstream;
    }
}
```

## 2.2 Apache (httpd)

* Use `ServerName` / `ServerAlias` e crie um vhost padrão que rejeita outros hosts.

```apache
<VirtualHost *:80>
  ServerName middleware-api.shapedigital.com
  # ...
</VirtualHost>

<VirtualHost *:80>
  ServerName default
  Redirect 400 /   # ou DocumentRoot com página 400
</VirtualHost>
```

* Configure `UseCanonicalName On` se o app usa `Host` internamente (generate canonical names from ServerName).

## 2.3 Express (Node.js)

* Middleware simples para validar `Host`:

```js
const allowedHosts = new Set(['middleware-api.shapedigital.com', 'localhost:3000']);

app.use((req, res, next) => {
  const host = req.headers.host;
  if (!allowedHosts.has(host)) return res.status(400).send('Invalid Host header');
  next();
});
```

* Ao gerar links, use `process.env.APP_BASE_URL` em vez de `req.headers.host`.

## 2.4 Flask / Django (Python)

* **Flask**: compare `request.host` com uma allowlist:

```py
from flask import request, abort

ALLOWED_HOSTS = {'middleware-api.shapedigital.com'}

@app.before_request
def check_host():
    if request.host not in ALLOWED_HOSTS:
        abort(400)
```

* **Django**: use `ALLOWED_HOSTS` no `settings.py` (já faz validação de host).

```py
ALLOWED_HOSTS = ['middleware-api.shapedigital.com']
```

## 2.5 IIS / Azure App Service / Application Gateway

* **IIS**: vincule sites ao nome de host (site bindings) — requests com host não bindado não vão pro site correto.
* **Azure Application Gateway**:

  * Configure regras de roteamento por host corretamente.
  * Nas **HTTP settings** do backend pool, há opção "Override with new host name" — configure para **usar o backend hostname** em vez do Host do cliente, ou configure explicitamente o header Host enviado ao backend.
  * Habilite WAF e regras customizadas que bloqueiem hosts fora da allowlist se possível.
* Em geral, para reverse proxies na nuvem: **não repasse** o header `Host` do cliente automaticamente; substitua-o por um valor confiável (o hostname do backend).

---

# 3) Tratamento de redirects (caso comum de exploração)

* Para `Location`/redirects, **não** reescreva usando `Host` do request sem validação.
* Exemplo seguro (pseudocódigo):

  * Se redirect for interno (path relativo) → redirecione para `/path` (relativo).
  * Se redirect absoluto (externo) → valide que hostname está na allowlist antes de permitir.

---

# 4) Logging, monitoração e alertas

* Logue requests com `Host` inválidos e alerte quando ocorrerem (pelo menos nos primeiros dias).
* Adicione métricas (nº de rejeições de Host) e rotação de logs.

---

# 5) Testes para verificar correção

* Com Burp / curl tente:

  * Modificar `Host` para `evil.oastify.com` — esperado: deveria receber 400/444/403, **não** 200/redirect.
  * Teste `X-Forwarded-Host`, `X-Original-Host` — se você aceita esses, valide-os também.
  * Teste redirects: garantir que `Location` não contenha host vindo do cliente (ou que esteja permitido).
* Curl exemplo para testar:

```bash
curl -I -k -H "Host: evil.oastify.com" https://middleware-api.shapedigital.com/
# deve retornar 400/403/444/404 (controlado) e não conteúdo gerado pelo app
```

---

# 6) Checklist rápido (priorize nesta ordem)

1. Defina allowlist de hosts no app/framework (Django `ALLOWED_HOSTS`, Express middleware, Flask before_request).
2. Configure reverses proxies para **não repassar** Host do cliente ao backend; defina `proxy_set_header Host` explicitamente (Nginx) ou equivalente.
3. Garanta bindings de host no servidor web (IIS/Apache/Nginx) e um vhost default que rejeita requests desconhecidos.
4. Sanitize/validate `X-Forwarded-Host` e outros headers se os usa.
5. Evite criar URLs absolutos com `req.headers.host` — use `APP_BASE_URL`/config.
6. Teste com Burp/Collab/OAST para confirmar que não há vazamento/interação.
7. Log + alertas para `Host` inválido.

---

# 7) Exemplo concreto — Nginx + Express (resumo aplicado)

* Nginx: bloqueia hosts inválidos e sempre envia `Host` do backend.
* Express: valida `Host` mesmo assim como defesa em profundidade.

