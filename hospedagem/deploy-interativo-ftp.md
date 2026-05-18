# Deploy interativo via FTP (PowerShell)

> Como montar um script de deploy em PowerShell que funciona tanto em modo interativo (menu para humano) quanto em modo não interativo (parâmetros + exit codes), pronto para ser invocado por IA ou CI.

**Contexto:** Hospedagem compartilhada na Napoleon Host (cPanel + FTP). Aplicação PHP enviada do Windows. Mesmo script atende dois cenários: o desenvolvedor rodando `./deploy.ps1` sem argumentos (menu com 4 opções) e a IA/automação chamando com `-Mode full -Yes -Quiet` para parsing do `RESULT:` no final.

---

## Pré-requisitos

- [ ] PowerShell 5.1+ (já vem no Windows 10/11)
- [ ] Conta FTP criada no cPanel (Napoleon Host) com permissão de upload na raiz do site
- [ ] Acesso ao host/porta FTP (geralmente porta 21, passivo)
- [ ] Repositório local com `.gitignore` configurado para nunca commitar credenciais
- [ ] Estrutura de projeto onde a raiz a ser publicada bate com a raiz do FTP (ou ajuste o `LOCAL_ROOT`)

---

## Arquitetura da solução

Três arquivos na raiz do projeto que vai ser publicado:

```
projeto/
├── deploy.ps1                    <-- script principal (commitado)
├── deploy.config.example.ps1     <-- modelo de credenciais (commitado)
├── deploy.config.ps1             <-- credenciais reais (NUNCA commitar)
└── deploy-log.txt                <-- log gerado a cada execução (gitignored)
```

A separação é o ponto-chave: o script é versionado, mas as credenciais ficam fora do Git. O `deploy.config.example.ps1` documenta a estrutura esperada para quem clonar o repo.

---

## Passo a passo

### 1. Criar o `deploy.config.example.ps1` (commitado)

Modelo com placeholders. É o que outros devs (ou você mesmo em outra máquina) vão copiar para criar o real.

```powershell
# Renomeie para deploy.config.ps1 e preencha com suas credenciais.
# Este arquivo NAO deve ser commitado.

$FTP_SERVER  = "{IP_OU_HOST_FTP}"
$FTP_PORT    = 21
$FTP_USER    = "{USUARIO_FTP}"
$FTP_PASS    = "{SENHA_AQUI}"
$FTP_USE_SSL = $false
$FTP_USE_PASSIVE = $true

# Arquivos/pastas que nunca devem ser enviados (paths relativos a esta pasta)
$EXCLUDES = @(
    "deploy.ps1",
    "deploy.config.ps1",
    "deploy.config.example.ps1",
    "deploy-log.txt",
    ".git",
    ".gitignore",
    "node_modules",
    ".env",
    "logs",
    "uploads",
    "exports",
    "database/schema.sql",
    "database/migration_*.sql"
)
```

> ⚠️ Coloque na lista de exclusões **tudo** que não pode ir para produção: schemas, migrations, dumps, pastas de upload, logs, `.env`, o próprio `deploy.ps1`.

### 2. Atualizar o `.gitignore`

```gitignore
deploy.config.ps1
deploy-log.txt
```

### 3. Criar o `deploy.config.ps1` (local, fora do Git)

Copie do `.example`, preencha com as credenciais reais. Esse arquivo é dot-sourced pelo script principal: `. $configPath`.

### 4. Esqueleto do `deploy.ps1`

Aceita parâmetros para uso por IA/CI. Sem parâmetros, abre menu interativo.

```powershell
param(
    [ValidateSet("test", "full", "partial", "files")]
    [string]$Mode,
    [int]$Minutes = 60,
    [string]$Files,
    [switch]$Yes,
    [switch]$DryRun,
    [switch]$Quiet
)

$ErrorActionPreference = "Stop"
$LOCAL_ROOT = $PSScriptRoot

# Carrega credenciais (dot-source)
$configPath = Join-Path $LOCAL_ROOT "deploy.config.ps1"
if (-not (Test-Path $configPath)) {
    Write-Host "ERRO: deploy.config.ps1 nao encontrado." -ForegroundColor Red
    Write-Host "RESULT: status=failed sent=0 failed=0 total=0 elapsed=0"
    exit 2
}
. $configPath

$FTP_BASE_URI = "ftp://${FTP_SERVER}:${FTP_PORT}"
$FTP_CRED = New-Object System.Net.NetworkCredential($FTP_USER, $FTP_PASS)
```

### 5. Bloco `RESULT:` — o contrato com a IA

Toda saída de execução termina com **uma linha** padronizada que a IA (ou um shell script) consegue parsear via regex. Essa é a parte que transforma o script em algo invocável por agente.

```powershell
function Write-Result {
    param($Status, $Sent, $Failed, $Total, $Elapsed)
    Write-Host ""
    Write-Host ("RESULT: status={0} sent={1} failed={2} total={3} elapsed={4}" `
        -f $Status, $Sent, $Failed, $Total, [int]$Elapsed)
}
```

E **exit codes** consistentes:

| Exit | Significado                          |
|------|--------------------------------------|
| 0    | Sucesso total                        |
| 1    | Falha de conexão FTP                 |
| 2    | Erro de configuração (sem config)    |
| 3    | Envio parcial (alguns arquivos falharam) |
| 4    | Cancelado pelo usuário               |

Combinado, IA chama `pwsh -File deploy.ps1 -Mode full -Yes -Quiet`, lê stdout até achar `RESULT:`, parseia, e decide o próximo passo.

### 6. Os 4 modos de operação

| Modo      | Uso                                            | Exemplo                                            |
|-----------|------------------------------------------------|----------------------------------------------------|
| `test`    | Testa conexão FTP (lista raiz, conta itens)    | `.\deploy.ps1 -Mode test`                          |
| `full`    | Envia tudo (respeitando `$EXCLUDES`)           | `.\deploy.ps1 -Mode full -Yes`                     |
| `partial` | Envia só arquivos modificados nos últimos N min| `.\deploy.ps1 -Mode partial -Minutes 30 -Yes`      |
| `files`   | Envia lista explícita (separada por vírgula)   | `.\deploy.ps1 -Mode files -Files "app/x.php,app/y.php" -Yes` |

Dispatcher final:

```powershell
switch ($Mode) {
    "test"    { Invoke-Test }
    "full"    { Invoke-Send (Get-DeployFiles) }
    "partial" { Invoke-Send (Get-DeployFiles -MinutesAgo $Minutes) }
    "files"   {
        if (-not $Files) {
            Write-Result "failed" 0 0 0 0
            exit 2
        }
        Invoke-Send (Get-DeployFiles -ExplicitFiles ($Files.Split(",")))
    }
    default   { Show-Menu }   # sem -Mode -> menu interativo
}
```

### 7. Função de exclusão (case-insensitive, com prefixo)

```powershell
function Test-Excluded {
    param([string]$RelativePath)
    $normalized = $RelativePath.Replace("/", "\").TrimStart("\")
    foreach ($exc in $EXCLUDES) {
        $excNorm = $exc.Replace("/", "\").TrimStart("\")
        if ($normalized -eq $excNorm) { return $true }
        if ($normalized.StartsWith($excNorm + "\")) { return $true }
    }
    return $false
}
```

Pega tanto arquivo exato (`.env`) quanto pasta inteira (`logs` exclui `logs/php_errors.log`).

### 8. Coleta de arquivos com filtro por data

```powershell
function Get-DeployFiles {
    param([int]$MinutesAgo = 0, [string[]]$ExplicitFiles = @())
    # ... ramo 1: lista explicita
    # ... ramo 2: varredura recursiva com Test-Excluded + filtro de tempo
}
```

O modo `partial` usa `(Get-Date).AddMinutes(-$MinutesAgo)` como cutoff — útil para enviar só o que mexeu depois do último deploy.

### 9. Upload FTP (sem dependências externas)

Usa `[System.Net.FtpWebRequest]` — nativo do .NET, não precisa de WinSCP nem libs extras.

```powershell
function Send-FtpFile {
    param([string]$LocalPath, [string]$RemoteFtpPath)
    $uri = "$FTP_BASE_URI/$RemoteFtpPath"
    try {
        $req = [System.Net.FtpWebRequest]::Create($uri)
        $req.Credentials  = $FTP_CRED
        $req.Method       = [System.Net.WebRequestMethods+Ftp]::UploadFile
        $req.UseBinary    = $true
        $req.EnableSsl    = $FTP_USE_SSL
        $req.UsePassive   = $FTP_USE_PASSIVE
        $bytes = [System.IO.File]::ReadAllBytes($LocalPath)
        $req.ContentLength = $bytes.Length
        $stream = $req.GetRequestStream()
        $stream.Write($bytes, 0, $bytes.Length); $stream.Close()
        $resp = $req.GetResponse(); $resp.Close()
        return $true
    } catch { return $false }
}
```

Antes de enviar, criar diretórios remotos recursivamente (`MakeDirectory` ignora se já existe):

```powershell
function Ensure-FtpDirectory {
    param([string]$RemotePath)
    $parts = $RemotePath.Replace("\", "/").Split("/") | Where-Object { $_ -ne "" }
    $current = ""
    foreach ($part in $parts) {
        $current = if ($current) { "$current/$part" } else { $part }
        try {
            $req = [System.Net.FtpWebRequest]::Create("$FTP_BASE_URI/$current/")
            $req.Credentials = $FTP_CRED
            $req.Method      = [System.Net.WebRequestMethods+Ftp]::MakeDirectory
            $req.EnableSsl   = $FTP_USE_SSL
            $req.UsePassive  = $FTP_USE_PASSIVE
            $resp = $req.GetResponse(); $resp.Close()
        } catch { }  # ja existe -> ignora
    }
}
```

### 10. Menu interativo (modo humano)

Quando nenhum `-Mode` é passado, abre menu numérico. Cada opção dispara a mesma função do dispatcher, mas pergunta os parâmetros:

```powershell
function Show-Menu {
    while ($true) {
        Write-Host "  [1] Testar conexao FTP"
        Write-Host "  [2] Deploy total"
        Write-Host "  [3] Deploy parcial (arquivos modificados nos ultimos X min)"
        Write-Host "  [4] Deploy de arquivos especificos"
        Write-Host "  [0] Sair"
        $c = Read-Host "  Opcao"
        switch ($c) {
            "1" { Invoke-Test }
            "2" { Invoke-Send (Get-DeployFiles) }
            "3" {
                $m = Read-Host "  Minutos [60]"
                $min = if ($m -match '^\d+$') { [int]$m } else { 60 }
                Invoke-Send (Get-DeployFiles -MinutesAgo $min)
            }
            "4" {
                $f = Read-Host "  Arquivos (separados por virgula)"
                if ($f) { Invoke-Send (Get-DeployFiles -ExplicitFiles ($f.Split(","))) }
            }
            "0" { return }
        }
    }
}
```

### 11. Confirmação dupla + dry-run

No envio, antes de subir, o script pergunta "Confirma deploy? (S/N)" — **a menos que** `-Yes` tenha sido passado. Isso é o que torna o mesmo script seguro para humano (pergunta antes) e usável por IA (`-Yes` pula a pergunta).

```powershell
if ($DryRun) {
    Write-Result "success" 0 0 $FilesToSend.Count 0
    exit 0
}

if (-not $Yes) {
    $confirm = Read-Host "  Confirma deploy? (S/N)"
    if ($confirm -notin @("S","s","Y","y")) {
        Write-Result "cancelled" 0 0 $FilesToSend.Count 0
        exit 4
    }
}
```

`-DryRun` lista o que **seria** enviado e sai sem subir nada — ótimo para validar `$EXCLUDES`.

### 12. Log de auditoria

A cada execução, append em `deploy-log.txt`:

```powershell
$logEntry = @(
    "",
    "=== Deploy $(Get-Date -Format 'yyyy-MM-dd HH:mm:ss') ===",
    "Modo: $Mode | Enviados: $sent / $($FilesToSend.Count) | Tempo: $([int]$elapsed)s"
)
$FilesToSend | ForEach-Object { $logEntry += "  OK $($_.FtpPath)" }
$logEntry | Out-File -FilePath $logPath -Append -Encoding UTF8
```

Fica gitignored, mas útil para reconstituir o que subiu quando algo quebra em produção.

---

## Como a IA invoca

```powershell
pwsh -NoProfile -File .\deploy.ps1 -Mode partial -Minutes 30 -Yes -Quiet
```

E parseia a saída final:

```
RESULT: status=success sent=12 failed=0 total=12 elapsed=8
```

Regex sugerida: `^RESULT:\s+status=(\w+)\s+sent=(\d+)\s+failed=(\d+)\s+total=(\d+)\s+elapsed=(\d+)`

`-Quiet` silencia `Out-Step`/`Out-Info`/`Out-Ok` para deixar o stdout limpo — só o `RESULT:` e erros aparecem.

---

## Fluxo recomendado de uso diário

1. **Antes de commitar:** `.\deploy.ps1 -Mode test` (sanity check do FTP).
2. **Após `git commit`:** `.\deploy.ps1` (menu) → opção 3 com 60 min.
3. **Hotfix de 1 arquivo:** `.\deploy.ps1 -Mode files -Files "app/controllers/X.php" -Yes`.
4. **Migration nova:** rodar separado via `mysql -u ... < arquivo.sql` (migrations estão em `$EXCLUDES`).

---

## Troubleshooting

### Erro: `deploy.config.ps1 nao encontrado`

**Causa:** primeira execução em uma máquina nova ou clone fresco.

**Solução:** copie `deploy.config.example.ps1` → `deploy.config.ps1` e preencha as credenciais.

---

### Erro: `(550) Acesso negado` ou `File unavailable`

**Causa:** credenciais corretas, mas usuário FTP não tem permissão na raiz do destino, ou path remoto inválido.

**Solução:**
- Confirme no cPanel → **FTP Accounts** que o usuário tem como diretório `public_html/` (ou onde sua app está).
- Reveja o `LOCAL_ROOT` — ele deve bater com a raiz que o usuário FTP enxerga.

---

### Erro: `The remote server returned an error: (530) Not logged in.`

**Causa:** usuário/senha errados, ou conta criada com `@dominio` e você esqueceu de incluir o sufixo.

**Solução:** no cPanel, alguns provedores exigem `usuario@dominio.com.br` como `$FTP_USER`. Verifique o formato exato em **FTP Accounts**.

---

### Upload trava em arquivos grandes

**Causa:** modo ativo bloqueado por firewall/NAT.

**Solução:** confirme `$FTP_USE_PASSIVE = $true` no config. PowerShell + FTP passivo + NAT doméstico é a combinação que mais funciona.

---

### IA não consegue parsear o resultado

**Causa:** `Out-Step` e variantes estão poluindo o stdout, ou o script morreu antes de chamar `Write-Result`.

**Solução:**
- Sempre invocar com `-Quiet`.
- Garantir que **todos** os caminhos de saída (sucesso, falha, cancelado, config ausente) chamam `Write-Result` antes do `exit`.

---

### Quero forçar SSL (FTPS)

**Causa:** Provedor exige FTPS Explicit.

**Solução:** `$FTP_USE_SSL = $true` no config. Em PowerShell 5.1 pode ser necessário forçar TLS 1.2 antes:

```powershell
[Net.ServicePointManager]::SecurityProtocol = [Net.SecurityProtocolType]::Tls12
```

Coloque isso logo após o dot-source do config.

---

## Referências

- [FtpWebRequest Class — Microsoft Docs](https://learn.microsoft.com/dotnet/api/system.net.ftpwebrequest)
- [WebRequestMethods.Ftp — Microsoft Docs](https://learn.microsoft.com/dotnet/api/system.net.webrequestmethods.ftp)
- [Deploy na Napoleon Host](napoleon-host-deploy.md) — guia geral de subida do projeto (este doc é a versão automatizada)

---

**Última revisão:** 2026-05-18
**Testado em:** Windows 11 + PowerShell 5.1, FTP Napoleon Host (porta 21, passivo, sem SSL)
