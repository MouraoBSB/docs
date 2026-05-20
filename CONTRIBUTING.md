# Como adicionar uma doc nova

Este repositório é pra você (e pra qualquer pessoa que precise das mesmas configurações). Quanto mais padronizado, mais útil.

## Passos

### 1. Escolha a pasta certa

| Pasta | Conteúdo |
|-------|----------|
| `hospedagem/` | Tudo relacionado a servidores, cPanel, deploy, FTP, SSH, DNS |
| `email/` | SMTP, IMAP, providers (Gmail, Zoho, SendGrid) |
| `apis/` | Integrações com APIs de terceiros (Google, Anthropic, OpenAI, etc.) |
| `backup/` | Estratégias de backup (local, remoto, restore, retenção) |
| `php/` | Padrões, snippets e boas práticas em PHP |
| `templates/` | Templates reutilizáveis (não é doc final) |

Não cabe em nenhuma? Crie uma pasta nova e adicione um `README.md` dentro dela explicando o tema.

### 2. Use o template

Copie `templates/TEMPLATE-DOC.md` como base. A estrutura padrão é:

- **Título e contexto** — o que esse guia resolve
- **Pré-requisitos** — o que você precisa ter antes de começar
- **Passo a passo** — instruções enumeradas
- **Troubleshooting** — erros comuns e soluções
- **Referências** — links oficiais

### 3. Nomeie o arquivo

Use **kebab-case** em minúsculas e seja descritivo:

✅ `napoleon-host-deploy.md`
✅ `smtp-gmail.md`
✅ `google-oauth-drive.md`

❌ `Deploy.md`
❌ `meu_guia_de_smtp.md`
❌ `apiClaude.md`

### 4. Adicione ao índice

Edite o `README.md` da raiz e inclua o link na seção apropriada.

### 5. Commit

Use mensagens claras seguindo o padrão `tipo: descrição`:

```
docs: adiciona guia de SMTP Zoho
docs: atualiza passo 3 do deploy Napoleon
docs: corrige link quebrado na seção de APIs
fix: corrige snippet errado em pdo-singleton
```

## Princípios

- **Específico > genérico.** "Como configurar SMTP" é vago. "SMTP Gmail com App Password em PHPMailer" é útil.
- **Mostre o erro, mostre a solução.** Se você quebrou a cabeça uma vez, documente para a próxima.
- **Tenha screenshots quando ajudar.** Salve em uma subpasta `img/` dentro da pasta da doc.
- **Nunca exponha credenciais.** Releia tudo antes do commit.
- **Datas importam.** Coloque "Última revisão: AAAA-MM-DD" no rodapé. Documentações de APIs envelhecem rápido.
