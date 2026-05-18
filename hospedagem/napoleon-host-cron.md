# Cron jobs na Napoleon Host

> Como agendar tarefas recorrentes (backups, limpezas, notificações) em hospedagem compartilhada via cPanel.

**Contexto:** cPanel da Napoleon Host. Aplicável a qualquer script PHP que precise rodar periodicamente sem intervenção manual.

---

## Pré-requisitos

- [ ] Acesso ao cPanel
- [ ] Script PHP funcional acessível pelo servidor (ou via URL pública com token de proteção)
- [ ] Caminho absoluto do PHP no servidor (geralmente `/usr/bin/php` ou `/opt/cpanel/ea-php82/root/usr/bin/php`)

---

## Passo a passo

### 1. Descobrir o caminho do PHP

No Terminal do cPanel:

```bash
which php
# ou para uma versão específica
which ea-php82
```

Anote o caminho retornado (ex: `/opt/cpanel/ea-php82/root/usr/bin/php`).

### 2. Acessar Cron Jobs

No cPanel, abra **Advanced** → **Cron Jobs**.

### 3. Escolher abordagem

Existem duas formas de rodar um script em cron:

**Abordagem A — Executar PHP via CLI (recomendado para scripts internos)**

```bash
/opt/cpanel/ea-php82/root/usr/bin/php /home/cemaneto/public_html/meuapp/cron/cleanup.php
```

✅ Mais rápido (não passa pelo Apache)
✅ Sem timeout de HTTP
✅ Não precisa de proteção por token

**Abordagem B — Chamar URL pública com `curl` ou `wget` (use quando o script depende do bootstrap web)**

```bash
/usr/bin/curl -s "https://app.seudominio.com/cron/backup-diario?token={SEU_TOKEN_AQUI}"
```

⚠️ **Sempre** proteja com token aleatório no query string.
⚠️ Tem timeout do Apache (geralmente 30-120s).

### 4. Adicionar o cron

Em **Cron Jobs**, role até **Add New Cron Job** e preencha:

| Campo | Significado | Exemplo |
|-------|-------------|---------|
| Minute | minuto (0-59) | `0` |
| Hour | hora (0-23) | `3` |
| Day | dia do mês (1-31) | `*` |
| Month | mês (1-12) | `*` |
| Weekday | dia da semana (0-6, 0=domingo) | `*` |
| Command | comando completo | (ver abaixo) |

**Atalhos comuns:**

| Frequência | Sintaxe |
|------------|---------|
| Diário às 03:00 | `0 3 * * *` |
| A cada hora | `0 * * * *` |
| A cada 15 min | `*/15 * * * *` |
| Domingos às 02:00 | `0 2 * * 0` |
| Toda primeira segunda do mês às 04:00 | `0 4 1-7 * 1` |

### 5. Capturar logs

Por padrão, o cron envia output por email. Para salvar em arquivo:

```bash
/opt/cpanel/ea-php82/root/usr/bin/php /home/cemaneto/public_html/meuapp/cron/cleanup.php >> /home/cemaneto/public_html/meuapp/logs/cron.log 2>&1
```

- `>>` anexa a saída padrão
- `2>&1` redireciona erros para o mesmo arquivo

Para **não receber emails** do cron, descomente "I prefer not to receive an email of the output when the command runs successfully." (no topo da página de Cron Jobs) ou adicione `> /dev/null 2>&1` no final do comando.

### 6. Testar

Antes de confiar no cron, execute o comando manualmente no Terminal do cPanel:

```bash
/opt/cpanel/ea-php82/root/usr/bin/php /home/cemaneto/public_html/meuapp/cron/cleanup.php
echo "Exit code: $?"
```

Exit code `0` = sucesso.

---

## Exemplos do GSAU

```bash
# Backup diário às 03:00 (chama URL com token)
0 3 * * * /usr/bin/curl -s "https://gsau.qsl.br/cron/backup-diario?token={BACKUP_TOKEN}" >> /home/cemaneto/public_html/gsau/logs/cron.log 2>&1

# Backup semanal aos domingos às 02:00
0 2 * * 0 /usr/bin/curl -s "https://gsau.qsl.br/cron/backup-semanal?token={BACKUP_TOKEN}" >> /home/cemaneto/public_html/gsau/logs/cron.log 2>&1

# Limpeza de fotos jurídicas expiradas (diário às 04:00, via CLI)
0 4 * * * /opt/cpanel/ea-php82/root/usr/bin/php /home/cemaneto/public_html/gsau/cron/cleanup_fotos_juridico.php >> /home/cemaneto/public_html/gsau/logs/cron.log 2>&1
```

---

## Token de proteção para endpoints cron

Quando expor `/cron/...` como rota HTTP pública, sempre:

1. Gere um token aleatório longo:
   ```php
   echo bin2hex(random_bytes(32)); // 64 caracteres hex
   ```

2. Armazene em uma tabela `configuracoes` ou variável de ambiente.

3. No controller, valide:
   ```php
   $tokenRecebido = $_GET['token'] ?? '';
   $tokenEsperado = ConfigModel::get('backup_cron_token');
   if (!hash_equals($tokenEsperado, $tokenRecebido)) {
       http_response_code(403);
       exit('Forbidden');
   }
   ```

   `hash_equals` evita timing attacks.

4. Mantenha o token **fora do versionamento** (banco ou `.env`).

---

## Troubleshooting

### Cron não dispara

**Causa:** sintaxe errada, caminho do PHP errado ou hospedagem com limite de crons.

**Solução:**
- Confira o caminho do PHP (passo 1).
- Limite comum: 10 crons por conta. Veja em **Cron Jobs** se já bateu o teto.
- Veja o log do cron em `/var/log/cron` (se SSH liberado) ou peça via ticket.

---

### Script roda mas não conecta no banco

**Causa:** scripts CLI não passam pelo bootstrap normal — algumas constantes podem não estar definidas.

**Solução:** inclua manualmente no topo do script:
```php
require_once __DIR__ . '/../config/config.php';
require_once __DIR__ . '/../config/db.php';
```

---

### Email do cron entulhando a caixa

**Causa:** output sendo enviado por email a cada execução.

**Solução:** marque a opção de não receber emails OU adicione `> /dev/null 2>&1` no final do comando (mas atenção: você não verá erros).

Alternativa: redirecione para arquivo (`>> logs/cron.log 2>&1`) e configure log rotate.

---

## Referências

- [cPanel — Cron Jobs](https://docs.cpanel.net/cpanel/advanced/cron-jobs/)
- [crontab.guru](https://crontab.guru/) — calculadora visual de sintaxe cron

---

**Última revisão:** 2026-05-18
**Testado em:** cPanel v110, PHP 8.0/8.2
