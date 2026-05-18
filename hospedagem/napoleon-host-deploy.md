# Deploy de aplicação PHP na Napoleon Host

> Como subir uma aplicação PHP (MVC custom ou framework) em hospedagem compartilhada da Napoleon Host, com domínio próprio, SSL e estrutura `public/` separada.

**Contexto:** Hospedagem compartilhada com cPanel. Validado com PHP 8.0+ e MariaDB/MySQL 8.0. Aplicação com separação clara entre código-fonte e webroot (`public/` apontado como `DocumentRoot`).

---

## Pré-requisitos

- [ ] Conta ativa na Napoleon Host com plano que suporte PHP 8.0+
- [ ] Acesso ao cPanel (URL, usuário, senha)
- [ ] Domínio apontado para os DNS da Napoleon (ou subdomínio configurado)
- [ ] Acesso SSH ou ao Terminal do cPanel (recomendado, opcional)
- [ ] Banco MySQL/MariaDB criado pelo cPanel (usuário, senha, nome do banco)

---

## Passo a passo

### 1. Preparar o banco de dados

No cPanel:

1. Abra **MySQL® Databases** (Bancos de Dados MySQL).
2. Crie um novo banco — o nome final terá prefixo da conta (ex: `cemaneto_meuapp`).
3. Crie um usuário do MySQL com senha forte.
4. Associe o usuário ao banco com **TODOS os privilégios**.

Anote:
- Host do banco: geralmente `localhost`
- Nome completo do banco: `prefixo_nome`
- Usuário: `prefixo_usuario`
- Senha: a que você definiu

### 2. Apontar o domínio para a pasta correta

Por padrão, o cPanel serve arquivos da pasta `public_html/`. Se sua aplicação tem estrutura tipo:

```
meuapp/
├── app/
├── config/
├── public/      <-- precisa ser o webroot
│   └── index.php
└── vendor/
```

Você tem duas opções:

**Opção A — Subdomínio com Document Root customizado (recomendado)**

1. No cPanel, abra **Domains** → **Create A New Domain** (ou **Subdomains**).
2. Crie o subdomínio (ex: `app.seudominio.com`).
3. Aponte o **Document Root** para `public_html/meuapp/public`.

**Opção B — `.htaccess` na raiz redirecionando para `public/`**

Crie `public_html/.htaccess` com:

```apache
RewriteEngine On
RewriteRule ^(.*)$ public/$1 [L]
```

E `public_html/public/.htaccess` com o rewrite normal da sua aplicação.

> ⚠️ Opção A é mais limpa e segura. Use Opção B só se não tiver outra escolha.

### 3. Subir os arquivos

**Via Git (recomendado se a hospedagem permitir)**

No Terminal do cPanel:

```bash
cd ~/public_html
git clone https://github.com/seu-user/seu-repo.git meuapp
cd meuapp
composer install --no-dev --optimize-autoloader
```

**Via Gerenciador de Arquivos do cPanel**

1. Compacte o projeto local em `.zip`.
2. Upload via **File Manager**.
3. Extraia no destino.

**Via FTP/SFTP (FileZilla, WinSCP)**

1. Host: `ftp.seudominio.com`
2. Usuário/senha: criados em **FTP Accounts** do cPanel.
3. Faça upload mantendo a estrutura.

### 4. Configurar variáveis de ambiente

Crie/edite `config/db.php` (ou `.env`, conforme sua aplicação):

```php
return [
    'host'     => 'localhost',
    'database' => 'cemaneto_meuapp',
    'username' => 'cemaneto_user',
    'password' => '{SUA_SENHA_AQUI}',
    'charset'  => 'utf8mb4',
];
```

> ⚠️ Garanta que esse arquivo **não está versionado** (`.gitignore`).

### 5. Importar o banco

No cPanel:

1. Abra **phpMyAdmin**.
2. Selecione o banco criado no passo 1.
3. Aba **Import** → faça upload do `schema.sql` (e `seed.sql`, se houver).

> Para arquivos `.sql` maiores que ~50 MB, importe via SSH:
> ```bash
> mysql -u cemaneto_user -p cemaneto_meuapp < schema.sql
> ```

### 6. Permissões de pastas

Pastas que recebem upload ou escrevem em disco precisam de permissão de escrita:

```bash
chmod -R 775 uploads/ storage/ logs/
```

Pastas com conteúdo sensível devem ter `.htaccess` com `Deny from all`:

```apache
# storage/.htaccess
Order allow,deny
Deny from all
```

### 7. Ativar SSL (HTTPS)

1. No cPanel, abra **SSL/TLS Status**.
2. Selecione o domínio.
3. Clique em **Run AutoSSL** (Let's Encrypt grátis).
4. Aguarde a propagação (alguns minutos).

Force redirect HTTP → HTTPS no `.htaccess`:

```apache
RewriteEngine On
RewriteCond %{HTTPS} off
RewriteRule ^(.*)$ https://%{HTTP_HOST}%{REQUEST_URI} [L,R=301]
```

### 8. Testar

Acesse `https://app.seudominio.com` e confira:
- [ ] Página inicial carrega
- [ ] Conexão com banco funciona
- [ ] Cadeado verde no navegador (HTTPS)
- [ ] Logs de erro estão em pasta segura (não acessível via web)

---

## Troubleshooting

### Erro: `500 Internal Server Error`

**Causa:** permissões erradas (`777` em pastas é rejeitado por padrão) ou erro de sintaxe PHP.

**Solução:**
- Verifique `logs/php_errors.log` (ou ative `display_errors` temporariamente).
- Confirme permissões: arquivos `644`, pastas `755`, pastas de escrita `775`.

---

### Erro: `403 Forbidden` ao acessar a raiz

**Causa:** Document Root apontando para pasta sem `index.php` ou `.htaccess` bloqueando.

**Solução:** ajuste o Document Root (passo 2) ou crie um `index.php` na raiz.

---

### Erro: `SQLSTATE[HY000] [1045] Access denied for user`

**Causa:** credenciais erradas no `config/db.php` ou usuário sem privilégios.

**Solução:**
- Reconfira nome completo do banco e usuário (com prefixo da conta).
- No cPanel → MySQL Databases → confirme usuário associado ao banco com **ALL PRIVILEGES**.

---

### Site lento ou com `Connection timed out`

**Causa:** proxy/firewall da Napoleon bloqueia certas chamadas externas (CNA OAB, alguns CDNs).

**Solução:**
- Use cURL com `CURLOPT_PROXY` se necessário.
- Suporte da Napoleon pode liberar IPs específicos via ticket.

---

## Referências

- [Documentação oficial da Napoleon Host](https://napoleonhost.com.br/) <!-- confirmar URL -->
- [Documentação cPanel — MySQL Databases](https://docs.cpanel.net/cpanel/databases/mysql-databases/)
- [Let's Encrypt via AutoSSL](https://docs.cpanel.net/cpanel/security/ssltls-status/)

---

**Última revisão:** 2026-05-18
**Testado em:** PHP 8.0/8.2, MariaDB 10.6, cPanel v110
