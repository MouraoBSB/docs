# SMTP com Gmail (App Password + PHPMailer)

> Como enviar emails transacionais via Gmail SMTP usando uma App Password (necessário desde 2022 — senha normal não funciona mais).

**Contexto:** Aplicações PHP enviando notificações, links de redefinição de senha, comunicados, etc. Validado em hospedagem compartilhada (Napoleon Host) e em servidor dedicado.

---

## Pré-requisitos

- [ ] Conta Google (pessoal ou Workspace) com **verificação em duas etapas ativada**
- [ ] Acesso ao código PHP da aplicação
- [ ] PHPMailer instalado (`composer require phpmailer/phpmailer`)
- [ ] Porta 587 (TLS) ou 465 (SSL) liberada no servidor

> ⚠️ Limites do Gmail: **500 emails/dia** para contas pessoais, **2000/dia** para Workspace. Para volumes maiores, use SendGrid, Amazon SES, Mailgun, Brevo, etc.

---

## Passo a passo

### 1. Ativar verificação em duas etapas (2FA)

Sem 2FA ativada, a opção de criar App Password não aparece.

1. Acesse [myaccount.google.com/security](https://myaccount.google.com/security).
2. Em **Como você faz login no Google**, ative **Verificação em duas etapas**.
3. Conclua o processo (geralmente confirmando seu celular).

### 2. Criar uma App Password

1. Acesse [myaccount.google.com/apppasswords](https://myaccount.google.com/apppasswords).
2. Em **App name** (ou "Nome do app"), digite algo descritivo: `GSAU SMTP` ou `MeuApp Notificacoes`.
3. Clique em **Create**.
4. Copie a senha de **16 caracteres** mostrada (geralmente em blocos: `abcd efgh ijkl mnop`).

⚠️ **Essa senha aparece UMA ÚNICA VEZ.** Anote em local seguro. Se perder, revogue e crie outra.

> 💡 Espaços na App Password podem ser removidos: `abcd efgh ijkl mnop` = `abcdefghijklmnop`.

### 3. Instalar PHPMailer

```bash
composer require phpmailer/phpmailer
```

### 4. Configurar e enviar

```php
<?php
use PHPMailer\PHPMailer\PHPMailer;
use PHPMailer\PHPMailer\SMTP;
use PHPMailer\PHPMailer\Exception;

require __DIR__ . '/vendor/autoload.php';

$mail = new PHPMailer(true);

try {
    // Configuração SMTP
    $mail->isSMTP();
    $mail->Host       = 'smtp.gmail.com';
    $mail->SMTPAuth   = true;
    $mail->Username   = 'seuemail@gmail.com';
    $mail->Password   = '{APP_PASSWORD_DE_16_CHARS}';
    $mail->SMTPSecure = PHPMailer::ENCRYPTION_STARTTLS; // ou ENCRYPTION_SMTPS
    $mail->Port       = 587;                            // ou 465 para SMTPS
    $mail->CharSet    = 'UTF-8';

    // Remetente e destinatário
    $mail->setFrom('seuemail@gmail.com', 'Nome de Exibição');
    $mail->addAddress('destinatario@exemplo.com', 'Nome do Destinatário');
    $mail->addReplyTo('seuemail@gmail.com', 'Nome de Exibição');

    // Conteúdo
    $mail->isHTML(true);
    $mail->Subject = 'Teste de SMTP';
    $mail->Body    = '<p>Olá! Este é um email de teste enviado via Gmail SMTP.</p>';
    $mail->AltBody = 'Olá! Este é um email de teste enviado via Gmail SMTP.';

    $mail->send();
    echo 'Email enviado com sucesso!';
} catch (Exception $e) {
    echo "Erro: {$mail->ErrorInfo}";
}
```

### 5. Boas práticas

**Mantenha as credenciais fora do código:**

```php
// config/mail.php (NÃO versionado)
return [
    'host'     => 'smtp.gmail.com',
    'port'     => 587,
    'username' => 'seuemail@gmail.com',
    'password' => '{APP_PASSWORD}',
    'from'     => 'seuemail@gmail.com',
    'from_name' => 'GSAU Notificações',
];
```

E carregue:

```php
$config = require __DIR__ . '/../config/mail.php';
$mail->Username = $config['username'];
$mail->Password = $config['password'];
// ...
```

**Use uma classe helper:**

```php
class Mailer {
    public static function enviar(string $para, string $assunto, string $corpoHtml): bool {
        $config = require __DIR__ . '/../config/mail.php';
        $mail = new PHPMailer(true);
        try {
            $mail->isSMTP();
            $mail->Host = $config['host'];
            $mail->SMTPAuth = true;
            $mail->Username = $config['username'];
            $mail->Password = $config['password'];
            $mail->SMTPSecure = PHPMailer::ENCRYPTION_STARTTLS;
            $mail->Port = $config['port'];
            $mail->CharSet = 'UTF-8';
            $mail->setFrom($config['from'], $config['from_name']);
            $mail->addAddress($para);
            $mail->isHTML(true);
            $mail->Subject = $assunto;
            $mail->Body = $corpoHtml;
            return $mail->send();
        } catch (Exception $e) {
            error_log('Erro SMTP: ' . $mail->ErrorInfo);
            return false;
        }
    }
}
```

---

## Troubleshooting

### Erro: `SMTP connect() failed`

**Causa:** porta bloqueada pelo provedor de hospedagem ou firewall.

**Solução:**
- Tente porta **465** com `ENCRYPTION_SMTPS` em vez de **587** com `ENCRYPTION_STARTTLS`.
- Verifique se a hospedagem permite conexões SMTP externas (algumas exigem usar o SMTP local).
- Teste de outra máquina pra isolar se é problema da rede.

---

### Erro: `SMTP Error: Could not authenticate`

**Causa:** senha errada (provavelmente está usando a senha normal do Gmail, não a App Password).

**Solução:**
- Confirme que está usando a App Password de 16 caracteres.
- Remova espaços da senha.
- Crie uma App Password nova se desconfiar que foi revogada.

---

### Erro: `Username and Password not accepted`

**Causa:** 2FA não está ativada, ou Less Secure Apps foi descontinuado (desde maio/2022).

**Solução:** ative 2FA e use **somente App Password**. Não existe mais opção de "permitir apps menos seguros".

---

### Email vai pro spam

**Causa:** envios de `@gmail.com` em volume ou com conteúdo suspeito vão pro spam fácil.

**Solução:**
- Use um domínio próprio com SPF, DKIM e DMARC configurados (idealmente via Workspace ou SES).
- Evite linguagem agressiva ("CLIQUE AQUI URGENTE!!!"), muitos `<img>` sem `alt`, ou ratio texto/imagem muito baixo.
- Inclua header `Reply-To` válido.

---

### Limite de envios estourado

**Causa:** Gmail bloqueou temporariamente por exceder 500/2000 emails/dia.

**Solução:**
- Aguarde 24h.
- Para volume real, migre para **SendGrid** (100/dia grátis), **Brevo** (300/dia grátis) ou **Amazon SES** (~$0.10 / 1000 emails).

---

## Referências

- [Google — App Passwords](https://support.google.com/accounts/answer/185833)
- [PHPMailer no GitHub](https://github.com/PHPMailer/PHPMailer)
- [Limites de envio do Gmail](https://support.google.com/a/answer/166852)

---

**Última revisão:** 2026-05-18
**Testado em:** PHPMailer 6.9, PHP 8.0/8.2, Gmail pessoal e Workspace
