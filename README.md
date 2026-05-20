# 📚 Docs — Runbooks e Documentação Operacional

> Documentação técnica reutilizável: guias de deploy, configurações de serviços, integrações com APIs, padrões de código e tudo que faz sentido manter fora de um projeto específico.

**Mantenedor:** Thiago Mourão · [@tobrasil](https://github.com/tobrasil)

---

## 🗺️ Índice

### 🌐 Hospedagem
- [Deploy na Napoleon Host](hospedagem/napoleon-host-deploy.md) — passo a passo de subida via cPanel/FTP/Git
- [Cron jobs na Napoleon Host](hospedagem/napoleon-host-cron.md) — agendamento de tarefas, tokens de proteção, exemplos
- [Deploy interativo via FTP (PowerShell)](hospedagem/deploy-interativo-ftp.md) — script PS1 com menu interativo + invocação por IA (modes, RESULT:, exit codes)

### 💾 Backup
- [Backup remoto cPanel → Google Drive](backup/cpanel-google-drive.md) — dump PHP-only (sem mysqldump), upload OAuth sem libs externas, cron CLI, retenção FIFO, painel admin e troubleshooting completo

### 📧 Email
- [SMTP com Gmail](email/smtp-gmail.md) — App Password, configuração em PHP (PHPMailer), troubleshooting

### 🔌 APIs e Integrações
- [Google OAuth 2.0 + Drive API](apis/google-oauth-drive.md) — refresh token, upload de arquivos, sem libs externas
- [API Anthropic (Claude)](apis/anthropic-claude-api.md) — geração de texto, controle de custo, boas práticas

### 🐘 PHP
- [PDO Singleton com prepared statements](php/pdo-singleton.md) — padrão de conexão segura e reutilizável

---

## 🧭 Como usar este repositório

1. **Buscar:** use a barra de busca do GitHub (tecla `/`) ou pesquise por arquivo (`t`) — é mais rápido que clicar pasta por pasta.
2. **Copiar trechos:** cada doc tem snippets prontos pra copiar/colar. Sempre revise os placeholders (`{SEU_CLIENT_ID}`, `seu-dominio.com`, etc.) antes de usar.
3. **Linkar em outros projetos:** no `README.md` do projeto, basta linkar a doc específica:
   ```markdown
   - [Configurar deploy](https://github.com/tobrasil/docs/blob/main/hospedagem/napoleon-host-deploy.md)
   ```

---

## ➕ Adicionar uma doc nova

Veja [CONTRIBUTING.md](CONTRIBUTING.md). Resumo:

1. Identifique a pasta certa (ou crie uma nova com `README.md` próprio).
2. Copie [`templates/TEMPLATE-DOC.md`](templates/TEMPLATE-DOC.md) como ponto de partida.
3. Adicione o link no índice deste `README.md`.
4. Commit com mensagem clara: `docs: adiciona guia de X` / `docs: atualiza Y`.

---

## ⚠️ Aviso sobre credenciais

**Nunca** commite senhas, tokens, API keys ou refresh tokens reais. Use sempre placeholders:

```
DB_PASSWORD={SUA_SENHA_AQUI}
GOOGLE_CLIENT_SECRET={CLIENT_SECRET}
ANTHROPIC_API_KEY=sk-ant-{...}
```

Mantenha os valores reais em um gerenciador de senhas (Bitwarden, 1Password) ou em arquivos `.env` versionados **apenas localmente** (com `.env` no `.gitignore`).

---

## 📜 Licença

[MIT](LICENSE) — sinta-se à vontade para copiar, adaptar e usar.
