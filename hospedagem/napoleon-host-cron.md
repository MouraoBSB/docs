# Cron jobs na Napoleon Host

> Como agendar tarefas recorrentes (backups, limpezas, notificações) em hospedagem compartilhada via cPanel.

**Contexto:** cPanel da Napoleon Host. Aplicável a qualquer script PHP que precise rodar periodicamente sem intervenção manual.

---

## Pré-requisitos

- [ ] Acesso ao cPanel + ao Terminal do cPanel
- [ ] Script PHP funcional acessível pelo servidor (ou via URL pública com token de proteção)
- [ ] Caminho absoluto do PHP CLI no servidor (na Napoleon, geralmente `/usr/local/bin/php` — confira no passo 1)

---

## Passo a passo

### 1. Descobrir o caminho do PHP

No Terminal do cPanel:

```bash
which php
# ou para uma versão específica
which ea-php82
```

Na Napoleon, a saída típica é `/usr/local/bin/php` (CLI padrão da conta). Em outras hospedagens com EasyApache, pode aparecer `/opt/cpanel/ea-php82/root/usr/bin/php` ou similar. Anote o caminho — ele vai aparecer em todos os comandos abaixo.

> ⚠️ **Não confunda `/usr/bin/php` com `/usr/local/bin/php`.** Em algumas contas da Napoleon ambos existem, mas só o `/usr/local/bin/php` está com a versão selecionada no MultiPHP Manager e com extensões habilitadas. Use sempre o que `which php` retornou.

### 2. Acessar Cron Jobs

No cPanel, abra **Advanced** → **Cron Jobs**.

### 3. Escolher abordagem

Existem duas formas de rodar um script em cron. **Na Napoleon, prefira sempre a Abordagem A (CLI).**

**Abordagem A — Executar PHP via CLI (recomendado, padrão)**

```bash
/usr/local/bin/php /home/cemaneto/seusite.com.br/cron/cleanup.php
```

✅ Mais rápido (não passa pelo Apache)
✅ Sem timeout de HTTP
✅ Não precisa de proteção por token
✅ Roda mesmo se o domínio estiver fora do ar / DNS instável

**Abordagem B — Chamar URL pública com `curl` (use só quando o script depende mesmo do bootstrap web)**

```bash
/usr/bin/curl -s "https://app.seudominio.com/cron/backup-diario?token={SEU_TOKEN_AQUI}"
```

⚠️ **Sempre** proteja com token aleatório no query string.
⚠️ Tem timeout do Apache (geralmente 30-120s).
⚠️ **Histórico de problema na Napoleon:** em maio/2026, identifiquei que o cron de backup do GSAU configurado com `curl → URL HTTP` parou de disparar silenciosamente por 4 dias — o `cPanel` da Napoleon não conseguia executar `curl` chamando o próprio domínio do servidor (provavelmente DNS interno ou firewall). Outros sistemas na mesma hospedagem usando PHP CLI direto funcionavam sem falhas. Resolvido migrando tudo para Abordagem A.

### 4. Adicionar o cron

Em **Cron Jobs**, role até **Add New Cron Job** e preencha **cada campo separadamente**:

| Campo | Significado | Exemplo |
|-------|-------------|---------|
| Minute | minuto (0-59) | `0` |
| Hour | hora (0-23) | `3` |
| Day | dia do mês (1-31) | `*` |
| Month | mês (1-12) | `*` |
| Weekday | dia da semana (0-6, 0=domingo) | `*` |
| **Command** | **apenas o comando, começando em `/usr/local/bin/php`** | `/usr/local/bin/php /home/cemaneto/seusite.com.br/cron/cleanup.php >> /home/cemaneto/seusite.com.br/logs/cron.log 2>&1` |

**Atalhos comuns:**

| Frequência | Sintaxe |
|------------|---------|
| Diário às 03:00 | `0 3 * * *` |
| A cada hora | `0 * * * *` |
| A cada 15 min | `*/15 * * * *` |
| Domingos às 02:00 | `0 2 * * 0` |
| Toda primeira segunda do mês às 04:00 | `0 4 1-7 * 1` |

#### ⚠️ Armadilha — não cole o agendamento dentro do campo "Command"

O cPanel **já tem 5 campos separados** para o agendamento. Se você copiar uma linha como `0 3 * * * /usr/local/bin/php ...` e colar **inteira** no campo **Command** (deixando ou não os campos de tempo preenchidos), o crontab final fica com agendamento duplicado:

```
0 3 * * * 0 3 * * * /usr/local/bin/php /home/.../script.php >> ... 2>&1
```

O cron daemon usa os 5 primeiros tokens como agendamento (`0 3 * * *`), e o **resto vira o comando** — que começa com o número `0`. O shell tenta executar `0` como programa e grava no log:

```
/bin/bash: 0: command not found
```

**O PHP nunca chega a rodar.** Os sintomas são silenciosos: o script funciona quando você roda manualmente no Terminal, não há nenhuma entrada relacionada no `php_errors.log` da aplicação, e o `cron.log` só acumula `bash: 0: command not found` em cada execução agendada.

**Regra de ouro:** o campo **Command** deve começar **direto** em `/usr/local/bin/php`. Os 5 valores de agendamento ficam nos campos separados acima dele.

### 5. Auditar o crontab depois de salvar

Toda vez que adicionar ou editar um cron, abra o Terminal do cPanel e rode:

```bash
crontab -l
```

Cada linha deve ter **exatamente um** bloco de 5 campos de tempo antes do `/usr/local/bin/php`. Exemplo correto:

```
0 3 * * * /usr/local/bin/php /home/cemaneto/seusite.com.br/cron/backup_diario.php >> /home/cemaneto/seusite.com.br/logs/cron.log 2>&1
0 2 * * 0 /usr/local/bin/php /home/cemaneto/seusite.com.br/cron/backup_semanal.php >> /home/cemaneto/seusite.com.br/logs/cron.log 2>&1
```

Errado (prefixo duplicado, vai gerar `bash: 0: command not found`):

```
0 3 * * * 0 3 * * * /usr/local/bin/php ...
0 2 * * 0 0 2 * * 0 /usr/local/bin/php ...
```

### 6. Capturar logs

Por padrão, o cron envia output por email. Para salvar em arquivo:

```bash
/usr/local/bin/php /home/cemaneto/seusite.com.br/cron/cleanup.php >> /home/cemaneto/seusite.com.br/logs/cron.log 2>&1
```

- `>>` anexa a saída padrão
- `2>&1` redireciona erros para o mesmo arquivo

Para **não receber emails** do cron, descomente "I prefer not to receive an email of the output when the command runs successfully." (no topo da página de Cron Jobs) ou adicione `> /dev/null 2>&1` no final do comando.

> Lembre de criar a pasta de logs antes:
> ```bash
> mkdir -p /home/cemaneto/seusite.com.br/logs
> chmod 755 /home/cemaneto/seusite.com.br/logs
> ```

### 7. Testar

Antes de confiar no cron, execute o comando manualmente no Terminal do cPanel:

```bash
/usr/local/bin/php /home/cemaneto/seusite.com.br/cron/cleanup.php
echo "Exit code: $?"
```

Exit code `0` = sucesso. Se quiser testar o disparo real pelo cron daemon sem esperar o horário programado, edite o cron e mude **Hora** para daqui a 2-3 minutos — depois cheque o `cron.log`. Se aparecer `bash: 0: command not found` em vez da saída do script, releia o passo 4 sobre a armadilha do campo Command.

---

## Exemplos reais (GSAU em produção na Napoleon)

> Estrutura de diretórios na Napoleon: cada domínio é uma subpasta direto em `/home/{conta}/`. Não tem `public_html/`. Os arquivos da web ficam em `/home/{conta}/{dominio}/public/`, mas scripts de cron CLI ficam fora dessa pasta web em `/home/{conta}/{dominio}/cron/`.

```
# Backup remoto diário às 03:00 (Google Drive, via PHP CLI direto)
0 3 * * * /usr/local/bin/php /home/cemaneto/gsau.qsl.br/cron/backup_diario.php >> /home/cemaneto/gsau.qsl.br/logs/cron_backup.log 2>&1

# Backup local semanal aos domingos às 02:00 (FIFO 5 arquivos)
0 2 * * 0 /usr/local/bin/php /home/cemaneto/gsau.qsl.br/cron/backup_semanal.php >> /home/cemaneto/gsau.qsl.br/logs/cron_backup.log 2>&1

# Limpeza de fotos jurídicas expiradas (diário às 04:00)
0 4 * * * /usr/local/bin/php /home/cemaneto/gsau.qsl.br/cron/cleanup_fotos_juridico.php >> /home/cemaneto/gsau.qsl.br/logs/cron.log 2>&1
```

> Esses crons usam **Abordagem A (CLI)**. A versão anterior desses mesmos crons usava `curl` chamando `https://gsau.qsl.br/cron/backup-...?token=...` (Abordagem B) e parou de disparar silenciosamente — ver Abordagem B no passo 3.

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

### `/bin/bash: 0: command not found` (ou `1`, `2`...) no log do cron

**Causa:** o campo **Command** do cron tem o prefixo de agendamento no início — algo como `0 3 * * * /usr/local/bin/php ...`. O cron daemon usa os 5 primeiros tokens da linha como agendamento e o restante (que começa com um número) vira o comando. O shell tenta executar `0` como programa.

**Como confirmar:**
```bash
crontab -l
```
Procure por linhas com agendamento duplicado, tipo `0 3 * * * 0 3 * * * /usr/local/bin/php ...`.

**Solução:** edite o cron no cPanel e deixe o campo **Command** começando direto em `/usr/local/bin/php`. Os 5 valores de tempo ficam nos campos separados. Ver passo 4.

---

### Cron via `curl + URL HTTP` (Abordagem B) para de disparar sem erro visível

**Causa:** o cPanel da Napoleon tem problema executando `curl` chamando o próprio domínio do servidor (DNS interno / firewall / PATH). O script HTTP teoricamente funciona, mas o `curl` no contexto do cron daemon falha silenciosamente. Confirmado em maio/2026 com o backup do GSAU.

**Solução:** migre para Abordagem A (CLI direto). Crie um script CLI no projeto (`cron/{nome}.php`), proteja com `if (php_sapi_name() !== 'cli') die('CLI only');`, e agende com `/usr/local/bin/php /home/.../cron/{nome}.php >> logs/cron.log 2>&1`.

---

### Backup manual no painel funciona, mas o cron não roda automaticamente

**Causa A:** armadilha do campo Command (ver primeira entrada deste troubleshooting). 90% dos casos.

**Causa B:** caminho do PHP errado no cron (ex.: `/usr/bin/php` em vez de `/usr/local/bin/php`).

**Causa C:** permissão de leitura no script de cron.

**Solução:**
1. `crontab -l` para auditar a linha completa.
2. `which php` para confirmar o caminho.
3. `chmod 644 /home/.../cron/{script}.php` se o arquivo não estiver legível.
4. Rode o comando manualmente no Terminal (sem o cron). Se rodar, é problema na linha do crontab. Se não, é problema no script.

---

### Cron não dispara (nenhuma linha no log)

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
- [Backup remoto cPanel → Google Drive](../backup/cpanel-google-drive.md) — exemplo completo de cron CLI com retenção FIFO e painel admin

---

**Última revisão:** 2026-05-20
**Testado em:** cPanel v110, PHP 8.0/8.2, Napoleon Host (`/usr/local/bin/php`)
**Changelog:**
- 2026-05-20 — adicionado aviso da armadilha do campo Command (`bash: 0: command not found`), `crontab -l` para auditoria, exemplos do GSAU atualizados de Abordagem B (curl) para A (CLI), caminhos da Napoleon corrigidos (`/home/{conta}/{dominio}/`).
