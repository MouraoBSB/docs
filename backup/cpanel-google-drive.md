# Backup remoto cPanel → Google Drive (PHP puro, sem libs externas)

> Guia completo, copy-pasteable, para adicionar a um projeto PHP em hospedagem compartilhada (cPanel) um módulo de backup automático que: (1) gera dump SQL comprimido sem depender de `mysqldump`, (2) envia para o Google Drive via OAuth 2.0 sem `composer require google/apiclient`, (3) mantém retenção local FIFO, (4) tem painel administrativo + log auditável, (5) roda via cron CLI direto sem passar por HTTP.

**Contexto:** validado em produção no GSAU (Napoleon Host, PHP 8.2, MySQL 8). Custo zero — usa só extensões nativas do PHP (`PDO`, `cURL`, `zlib`).

**Para quem é este guia:** desenvolvedor humano OU agente IA que vai implementar do zero. Está organizado para ser **lido em ordem e implementado sequencialmente** — cada passo inclui o código pronto, e o documento termina com uma seção de armadilhas conhecidas que evita os erros que já foram pagos uma vez.

---

## Índice

1. [Pré-requisitos](#pré-requisitos)
2. [Arquitetura](#arquitetura)
3. [Estrutura de arquivos](#estrutura-de-arquivos)
4. [Passo 1 — Schema do banco](#passo-1--schema-do-banco)
5. [Passo 2 — Constantes de configuração](#passo-2--constantes-de-configuração)
6. [Passo 3 — Helper PDO](#passo-3--helper-pdo)
7. [Passo 4 — Model: BackupLogModel](#passo-4--model-backuplogmodel)
8. [Passo 5 — Service: BackupService (dump SQL)](#passo-5--service-backupservice-dump-sql)
9. [Passo 6 — Service: GoogleDriveService (OAuth + upload)](#passo-6--service-googledriveservice-oauth--upload)
10. [Passo 7 — Service: BackupRunner (orquestrador)](#passo-7--service-backuprunner-orquestrador)
11. [Passo 8 — Controller: BackupController](#passo-8--controller-backupcontroller)
12. [Passo 9 — Rotas](#passo-9--rotas)
13. [Passo 10 — View do painel](#passo-10--view-do-painel)
14. [Passo 11 — Scripts CLI de cron](#passo-11--scripts-cli-de-cron)
15. [Passo 12 — Configurar OAuth no Google Cloud](#passo-12--configurar-oauth-no-google-cloud)
16. [Passo 13 — Obter refresh token via OAuth Playground](#passo-13--obter-refresh-token-via-oauth-playground)
17. [Passo 14 — Cadastrar credenciais no painel](#passo-14--cadastrar-credenciais-no-painel)
18. [Passo 15 — Configurar cron jobs no cPanel ⚠️](#passo-15--configurar-cron-jobs-no-cpanel-)
19. [Passo 16 — Testar](#passo-16--testar)
20. [Restaurar um backup](#restaurar-um-backup)
21. [Adaptações para outros projetos](#adaptações-para-outros-projetos)
22. [Troubleshooting](#troubleshooting)
23. [Segurança](#segurança)
24. [Referências](#referências)

---

## Pré-requisitos

- [ ] Hospedagem com cPanel + acesso ao Terminal e Cron Jobs
- [ ] PHP 8.0+ com extensões: `pdo_mysql`, `curl`, `zlib`, `openssl`
- [ ] MySQL/MariaDB com a base do projeto já criada
- [ ] Conta Google com espaço disponível no Drive (mínimo o tamanho do banco × 5–10)
- [ ] Aplicação PHP já tem: bootstrap (`config/config.php`, `config/db.php` ou equivalente), padrão MVC com Controller + View, helper PDO singleton (`db()`), rotas em string `MÉTODO:/caminho`. Se faltar algo, ajuste os trechos para o seu padrão — a lógica é a mesma.

---

## Arquitetura

```
┌────────────────────────────────────────────────────────────────────────┐
│  Origens do disparo                                                    │
│  ┌────────────────────┐  ┌────────────────────┐  ┌──────────────────┐  │
│  │ Painel web (botão) │  │ Cron CLI (cPanel)  │  │ Cron HTTP (token)│  │
│  └─────────┬──────────┘  └─────────┬──────────┘  └────────┬─────────┘  │
└────────────┼─────────────────────────┼─────────────────────┼───────────┘
             ▼                         ▼                     ▼
        BackupController          backup_diario.php     BackupController
        ::executarManual          backup_semanal.php   ::cronDiario/Semanal
             │                         │                     │
             └─────────────────────────┴─────────────────────┘
                                       ▼
                                  BackupRunner
                                  ├─ executarLocal()
                                  └─ executarRemoto()
                                       │
        ┌──────────────────────────────┼───────────────────────────────┐
        ▼                              ▼                               ▼
  BackupService                BackupLogModel              GoogleDriveService
  (dump .sql.gz via PDO)       (tabela backups_log)        (OAuth + upload)
        │                                                              │
        ▼                                                              ▼
  storage/backups/                                              Google Drive
  (FIFO, máx 5)                                                 (pasta {APP} Backups)
```

**Decisões-chave**:

| Decisão | Por quê |
|---|---|
| Dump via PDO puro (não `mysqldump`) | `exec` muitas vezes bloqueado em compartilhada; e `mysqldump` pode ter versão incompatível |
| OAuth via cURL puro (não `google/apiclient`) | Lib traz ~30 MB e cento de classes; só usamos 3 endpoints |
| Escopo `drive.file` | Não-sensível — não exige verificação OAuth; app só vê o que ele criou |
| Cron CLI direto (não curl/HTTP) | Sem timeout de Apache, sem dependência de DNS interno, sem token a gerenciar |
| Compressão em streaming (`gzopen` + `gzwrite`) | Não estoura memória mesmo em bases grandes |
| Retenção FIFO local | Evita encher disco da hospedagem |

---

## Estrutura de arquivos

Caminhos relativos à raiz do projeto. Renomeie `app/`, `public/`, `storage/` para o padrão que você já usa.

```
app/
├── controllers/
│   └── BackupController.php
├── models/
│   └── BackupLogModel.php
├── services/
│   ├── BackupService.php
│   ├── GoogleDriveService.php
│   └── BackupRunner.php
└── views/
    └── admin/
        └── backup.php
cron/
├── backup_diario.php          ← script CLI (Drive, diário)
└── backup_semanal.php         ← script CLI (local FIFO, semanal)
database/
└── migration_backup.sql       ← schema da tabela backups_log
storage/
└── backups/                   ← gerado em runtime, contém *.sql.gz locais
logs/
└── cron_backup.log            ← saída do cron (criado pelo redirect)
```

---

## Passo 1 — Schema do banco

Arquivo: `database/migration_backup.sql`

```sql
-- ============================================================
-- Migration — Backup do banco de dados
--   • Backup remoto diário no Google Drive
--   • Backup local semanal com retenção FIFO
-- ============================================================

-- 1. Tabela de log de backups
CREATE TABLE IF NOT EXISTS backups_log (
    id                       INT UNSIGNED   NOT NULL AUTO_INCREMENT,
    tipo                     ENUM('local_semanal','remoto_diario','manual_local','manual_remoto') NOT NULL,
    status                   ENUM('em_andamento','sucesso','falha') NOT NULL,
    data_inicio              DATETIME       NOT NULL,
    data_fim                 DATETIME       NULL,
    arquivo_nome             VARCHAR(255)   NULL,
    arquivo_tamanho_bytes    BIGINT UNSIGNED NULL,
    drive_file_id            VARCHAR(128)   NULL,
    acionado_por_usuario_id  INT UNSIGNED   NULL,
    erro_mensagem            TEXT           NULL,
    PRIMARY KEY (id),
    KEY idx_backups_data_inicio (data_inicio),
    KEY idx_backups_tipo_status (tipo, status),
    CONSTRAINT fk_backup_usuario
      FOREIGN KEY (acionado_por_usuario_id) REFERENCES usuarios (id) ON DELETE SET NULL
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci;

-- 2. Seed em tabela "configuracoes" (chave/valor)
-- Pré-requisito: você precisa ter uma tabela:
--   CREATE TABLE configuracoes (
--       chave VARCHAR(100) PRIMARY KEY,
--       valor TEXT NOT NULL,
--       atualizado_por INT UNSIGNED NULL,
--       atualizado_em TIMESTAMP DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP
--   );
INSERT IGNORE INTO configuracoes (chave, valor) VALUES
    ('backup_drive_folder_id',        ''),
    ('backup_oauth_client_id',        ''),
    ('backup_oauth_client_secret',    ''),
    ('backup_oauth_refresh_token',    ''),
    ('backup_cron_token',             ''),
    ('backup_max_locais',             '5');
```

> Se seu projeto **não tem tabela `usuarios`**, remova a `CONSTRAINT fk_backup_usuario` e mantenha só a coluna `acionado_por_usuario_id INT UNSIGNED NULL` (pode ficar sempre `NULL`).
>
> Se seu projeto **não tem tabela `configuracoes`**, crie-a com o `CREATE TABLE` mostrado no comentário, OU mude o código para ler/gravar de um arquivo `config/backup.local.php` (ver [Adaptações para outros projetos](#adaptações-para-outros-projetos)).

Aplicar:
```bash
mysql -u {USER} -p {DBNAME} < database/migration_backup.sql
```

---

## Passo 2 — Constantes de configuração

Em `config/config.php` (ou onde você define constantes globais), adicione:

```php
// Backup
define('APP_BACKUP_DIR',        dirname(__DIR__) . '/storage/backups/');
define('APP_BACKUP_MAX_LOCAIS', 5);          // máx arquivos locais antes do FIFO descartar
define('APP_TIMEZONE',          'America/Sao_Paulo');
define('APP_MYSQL_TIMEZONE',    '-03:00');    // usado no header do dump
define('APP_VERSION',           '1.0.0');     // opcional — vai no header do dump
```

Renomeie `APP_*` para o prefixo do seu projeto. O serviço lê essas constantes em runtime.

Crie o diretório de armazenamento:
```bash
mkdir -p storage/backups
chmod 755 storage/backups
```

---

## Passo 3 — Helper PDO

Os serviços assumem que existe uma função global `db()` que retorna um `PDO` singleton com `ERRMODE_EXCEPTION` e `FETCH_ASSOC`. Se não tiver, veja o guia [PDO Singleton com prepared statements](../php/pdo-singleton.md) deste mesmo repositório, ou use este minimal:

```php
// config/db.php
function db(): PDO {
    static $pdo = null;
    if ($pdo === null) {
        $pdo = new PDO(
            'mysql:host=' . DB_HOST . ';dbname=' . DB_NAME . ';charset=utf8mb4',
            DB_USER,
            DB_PASS,
            [
                PDO::ATTR_ERRMODE            => PDO::ERRMODE_EXCEPTION,
                PDO::ATTR_DEFAULT_FETCH_MODE => PDO::FETCH_ASSOC,
                PDO::ATTR_EMULATE_PREPARES   => false,
            ]
        );
    }
    return $pdo;
}
```

---

## Passo 4 — Model: BackupLogModel

Arquivo: `app/models/BackupLogModel.php`

```php
<?php
class BackupLogModel
{
    private PDO $db;

    public function __construct() { $this->db = db(); }

    public function iniciar(string $tipo, ?int $usuarioId = null): int
    {
        $stmt = $this->db->prepare(
            "INSERT INTO backups_log (tipo, status, data_inicio, acionado_por_usuario_id)
             VALUES (?, 'em_andamento', NOW(), ?)"
        );
        $stmt->execute([$tipo, $usuarioId ?: null]);
        return (int) $this->db->lastInsertId();
    }

    public function finalizarSucesso(int $id, string $arquivoNome, int $tamanhoBytes, ?string $driveFileId = null): void
    {
        $stmt = $this->db->prepare(
            "UPDATE backups_log
                SET status = 'sucesso', data_fim = NOW(),
                    arquivo_nome = ?, arquivo_tamanho_bytes = ?, drive_file_id = ?
              WHERE id = ?"
        );
        $stmt->execute([$arquivoNome, $tamanhoBytes, $driveFileId, $id]);
    }

    public function finalizarFalha(int $id, string $mensagem): void
    {
        $stmt = $this->db->prepare(
            "UPDATE backups_log
                SET status = 'falha', data_fim = NOW(), erro_mensagem = ?
              WHERE id = ?"
        );
        $stmt->execute([mb_substr($mensagem, 0, 4000), $id]);
    }

    public function listarRecentes(int $limite = 30): array
    {
        $limite = max(1, min(200, $limite));
        $stmt = $this->db->prepare(
            "SELECT * FROM backups_log ORDER BY data_inicio DESC LIMIT {$limite}"
        );
        $stmt->execute();
        return $stmt->fetchAll();
    }

    public function ultimoPorTipo(string $tipo): ?array
    {
        $stmt = $this->db->prepare("SELECT * FROM backups_log WHERE tipo = ? ORDER BY data_inicio DESC LIMIT 1");
        $stmt->execute([$tipo]);
        return $stmt->fetch() ?: null;
    }

    public function buscarPorId(int $id): ?array
    {
        $stmt = $this->db->prepare("SELECT * FROM backups_log WHERE id = ? LIMIT 1");
        $stmt->execute([$id]);
        return $stmt->fetch() ?: null;
    }

    public function listarBackupsLocaisDisponiveis(): array
    {
        $stmt = $this->db->prepare(
            "SELECT id, tipo, data_inicio, data_fim, arquivo_nome, arquivo_tamanho_bytes
               FROM backups_log
              WHERE status = 'sucesso'
                AND tipo IN ('local_semanal','manual_local')
                AND arquivo_nome IS NOT NULL
              ORDER BY data_inicio DESC"
        );
        $stmt->execute();
        return $stmt->fetchAll();
    }

    public function removerRegistroLocal(int $id): void
    {
        $stmt = $this->db->prepare("DELETE FROM backups_log WHERE id = ?");
        $stmt->execute([$id]);
    }

    /** Sincroniza log com o que realmente existe em disco (remove órfãos). */
    public function removerRegistrosArquivosInexistentes(array $arquivosExistentes): int
    {
        if ($arquivosExistentes === []) {
            $stmt = $this->db->prepare(
                "DELETE FROM backups_log
                  WHERE status = 'sucesso'
                    AND tipo IN ('local_semanal','manual_local')
                    AND arquivo_nome IS NOT NULL"
            );
            $stmt->execute();
            return $stmt->rowCount();
        }
        $placeholders = implode(',', array_fill(0, count($arquivosExistentes), '?'));
        $stmt = $this->db->prepare(
            "DELETE FROM backups_log
              WHERE status = 'sucesso'
                AND tipo IN ('local_semanal','manual_local')
                AND arquivo_nome IS NOT NULL
                AND arquivo_nome NOT IN ({$placeholders})"
        );
        $stmt->execute($arquivosExistentes);
        return $stmt->rowCount();
    }
}
```

---

## Passo 5 — Service: BackupService (dump SQL)

Arquivo: `app/services/BackupService.php`

Gera dump completo (estrutura + dados) **somente com PDO**, sem chamar `mysqldump`. Stream comprimido em `.sql.gz`. Aplica retenção FIFO em backups locais.

```php
<?php
class BackupService
{
    private PDO $db;
    private string $diretorioLocal;
    private int $maxLocais;

    public function __construct()
    {
        $this->db             = db();
        $this->diretorioLocal = rtrim(APP_BACKUP_DIR, '/\\') . DIRECTORY_SEPARATOR;
        $this->maxLocais      = (int) APP_BACKUP_MAX_LOCAIS;

        if (!is_dir($this->diretorioLocal)) {
            @mkdir($this->diretorioLocal, 0775, true);
        }
    }

    /** Dump persistente em APP_BACKUP_DIR. */
    public function gerarDumpLocal(): array
    {
        $arquivoNome = 'app_' . date('Y-m-d_His') . '.sql.gz';
        $caminho     = $this->diretorioLocal . $arquivoNome;
        $tamanho     = $this->dumparParaArquivo($caminho);
        return ['arquivo_nome' => $arquivoNome, 'caminho' => $caminho, 'tamanho' => $tamanho];
    }

    /** Dump temporário em sys_get_temp_dir() — usado pelo upload e descartado. */
    public function gerarDumpTemporario(): array
    {
        $arquivoNome = 'app_' . date('Y-m-d_His') . '.sql.gz';
        $caminho     = sys_get_temp_dir() . DIRECTORY_SEPARATOR . $arquivoNome;
        $tamanho     = $this->dumparParaArquivo($caminho);
        return ['arquivo_nome' => $arquivoNome, 'caminho' => $caminho, 'tamanho' => $tamanho];
    }

    /** Retenção FIFO. Remove mais antigos até sobrar $max. */
    public function limparAntigos(?int $max = null): array
    {
        $max = $max ?? $this->maxLocais;
        $arquivos = $this->listarArquivosOrdenados();
        $removidos = [];
        while (count($arquivos) > $max) {
            $maisAntigo = array_shift($arquivos);
            if ($maisAntigo !== null && @unlink($maisAntigo['caminho'])) {
                $removidos[] = $maisAntigo['arquivo_nome'];
            }
        }
        return $removidos;
    }

    public function listarLocais(): array
    {
        $arquivos = $this->listarArquivosOrdenados();
        usort($arquivos, static fn($a, $b) => $b['mtime'] <=> $a['mtime']);
        return $arquivos;
    }

    public function caminhoLocal(string $arquivoNome): ?string
    {
        // Whitelist do nome para evitar path traversal
        if (!preg_match('/^app_\d{4}-\d{2}-\d{2}_\d{6}\.sql\.gz$/', $arquivoNome)) {
            return null;
        }
        $caminho = $this->diretorioLocal . $arquivoNome;
        return is_file($caminho) ? $caminho : null;
    }

    public function excluirLocal(string $arquivoNome): bool
    {
        $caminho = $this->caminhoLocal($arquivoNome);
        return $caminho !== null && @unlink($caminho);
    }

    public function nomesArquivosLocais(): array
    {
        return array_map(fn($a) => $a['arquivo_nome'], $this->listarArquivosOrdenados());
    }

    // ─── Internos ──────────────────────────────────────────────

    private function listarArquivosOrdenados(): array
    {
        $padrao = $this->diretorioLocal . 'app_*.sql.gz';
        $itens = [];
        foreach (glob($padrao) ?: [] as $caminho) {
            $itens[] = [
                'arquivo_nome' => basename($caminho),
                'caminho'      => $caminho,
                'tamanho'      => (int) (@filesize($caminho) ?: 0),
                'mtime'        => (int) (@filemtime($caminho) ?: 0),
            ];
        }
        usort($itens, static fn($a, $b) => $a['mtime'] <=> $b['mtime']);
        return $itens;
    }

    private function dumparParaArquivo(string $caminho): int
    {
        @set_time_limit(0);

        $gz = gzopen($caminho, 'wb9');
        if ($gz === false) {
            throw new RuntimeException("Não foi possível criar arquivo em {$caminho}");
        }
        try {
            $this->escreverCabecalho($gz);
            foreach ($this->listarTabelas() as $tabela) {
                $this->dumparEstrutura($gz, $tabela);
                $this->dumparDados($gz, $tabela);
            }
            $this->escreverRodape($gz);
        } finally {
            gzclose($gz);
        }

        clearstatcache(true, $caminho);
        $tamanho = (int) (@filesize($caminho) ?: 0);
        if ($tamanho <= 0) {
            @unlink($caminho);
            throw new RuntimeException('Backup gerado vazio — abortado.');
        }
        return $tamanho;
    }

    private function escreverCabecalho($gz): void
    {
        $linhas = [
            '-- ============================================================',
            '-- Backup do banco de dados',
            '-- Versão: ' . (defined('APP_VERSION') ? APP_VERSION : 'n/a'),
            '-- Banco: ' . DB_NAME,
            '-- Gerado em: ' . date('Y-m-d H:i:s') . ' (' . APP_TIMEZONE . ')',
            '-- ============================================================',
            '',
            "SET NAMES utf8mb4 COLLATE utf8mb4_unicode_ci;",
            "SET time_zone = '" . APP_MYSQL_TIMEZONE . "';",
            "SET FOREIGN_KEY_CHECKS = 0;",
            "SET UNIQUE_CHECKS = 0;",
            "SET SQL_MODE = 'NO_AUTO_VALUE_ON_ZERO';",
            '',
        ];
        gzwrite($gz, implode("\n", $linhas) . "\n");
    }

    private function escreverRodape($gz): void
    {
        gzwrite($gz, "\nSET FOREIGN_KEY_CHECKS = 1;\nSET UNIQUE_CHECKS = 1;\n-- Fim do backup.\n");
    }

    private function listarTabelas(): array
    {
        return $this->db->query(
            "SELECT table_name AS nome FROM information_schema.tables
              WHERE table_schema = DATABASE() AND table_type = 'BASE TABLE'
              ORDER BY table_name"
        )->fetchAll(PDO::FETCH_COLUMN) ?: [];
    }

    private function dumparEstrutura($gz, string $tabela): void
    {
        $q = '`' . str_replace('`', '``', $tabela) . '`';
        gzwrite($gz, "\n-- ── Estrutura de {$q} ──\n");
        gzwrite($gz, "DROP TABLE IF EXISTS {$q};\n");
        $row = $this->db->query("SHOW CREATE TABLE {$q}")->fetch(PDO::FETCH_NUM);
        if (!$row || empty($row[1])) {
            throw new RuntimeException("Falha ao obter estrutura de {$tabela}");
        }
        gzwrite($gz, $row[1] . ";\n");
    }

    private function dumparDados($gz, string $tabela): void
    {
        $q = '`' . str_replace('`', '``', $tabela) . '`';
        $total = (int) $this->db->query("SELECT COUNT(*) FROM {$q}")->fetchColumn();
        if ($total === 0) return;

        gzwrite($gz, "\n-- ── Dados de {$q} ({$total} linhas) ──\n");

        $tamanhoLote = 500;
        $offset = 0;
        while ($offset < $total) {
            $stmt = $this->db->prepare("SELECT * FROM {$q} LIMIT {$tamanhoLote} OFFSET {$offset}");
            $stmt->execute();
            $linhas = $stmt->fetchAll(PDO::FETCH_NUM);
            if ($linhas === []) break;

            $colunasNomes = [];
            for ($i = 0; $i < $stmt->columnCount(); $i++) {
                $meta = $stmt->getColumnMeta($i);
                $colunasNomes[] = '`' . str_replace('`', '``', $meta['name']) . '`';
            }

            $valoresSQL = [];
            foreach ($linhas as $linha) {
                $partes = [];
                foreach ($linha as $valor) {
                    $partes[] = $this->quote($valor);
                }
                $valoresSQL[] = '(' . implode(',', $partes) . ')';
            }
            gzwrite($gz, 'INSERT INTO ' . $q . ' (' . implode(',', $colunasNomes) . ") VALUES\n"
                . implode(",\n", $valoresSQL) . ";\n");

            $offset += $tamanhoLote;
            unset($linhas, $valoresSQL);
        }
    }

    private function quote($valor): string
    {
        if ($valor === null) return 'NULL';
        if (is_int($valor) || is_float($valor)) return (string) $valor;
        if (is_bool($valor)) return $valor ? '1' : '0';
        return $this->db->quote((string) $valor);
    }
}
```

**Por que `gzopen` em vez de `gzcompress` no final?** Streaming evita carregar o dump inteiro na RAM. Bases de 500 MB+ não estouram memory_limit.

**Por que `SHOW CREATE TABLE`?** Reaproveita exatamente o DDL original com índices, FKs, charset e collation — coisa que uma reconstrução manual erra.

---

## Passo 6 — Service: GoogleDriveService (OAuth + upload)

Arquivo: `app/services/GoogleDriveService.php`

```php
<?php
class GoogleDriveService
{
    private const TOKEN_URI   = 'https://oauth2.googleapis.com/token';
    private const FILES_URI   = 'https://www.googleapis.com/drive/v3/files';
    private const UPLOAD_URI  = 'https://www.googleapis.com/upload/drive/v3/files?uploadType=multipart';
    private const FOLDER_MIME = 'application/vnd.google-apps.folder';
    private const FOLDER_NOME = 'APP Backups';   // ← personalize com o nome do projeto

    private string $clientId;
    private string $clientSecret;
    private string $refreshToken;
    private ?string $accessToken = null;
    private int $accessTokenExpiraEm = 0;

    public function __construct(string $clientId, string $clientSecret, string $refreshToken)
    {
        $this->clientId     = $clientId;
        $this->clientSecret = $clientSecret;
        $this->refreshToken = $refreshToken;
    }

    /**
     * Garante uma pasta acessível no Drive. Se $folderIdAtual ainda
     * existe e não está na lixeira, é reaproveitado. Senão, cria nova.
     */
    public function ensureFolder(string $folderIdAtual): string
    {
        if ($folderIdAtual !== '' && $this->pastaAcessivel($folderIdAtual)) {
            return $folderIdAtual;
        }
        return $this->criarPasta();
    }

    public function upload(string $caminhoArquivo, string $nomeRemoto, string $mimeType, string $folderId): array
    {
        if (!is_file($caminhoArquivo) || !is_readable($caminhoArquivo)) {
            throw new RuntimeException("Arquivo a enviar não encontrado: {$caminhoArquivo}");
        }
        if ($folderId === '') {
            throw new RuntimeException('ID da pasta vazio — chame ensureFolder() antes.');
        }

        $token    = $this->obterAccessToken();
        $conteudo = file_get_contents($caminhoArquivo);
        if ($conteudo === false) throw new RuntimeException("Falha ao ler {$caminhoArquivo}");

        $boundary = 'app_' . bin2hex(random_bytes(8));
        $metadata = json_encode(['name' => $nomeRemoto, 'parents' => [$folderId]],
            JSON_UNESCAPED_UNICODE | JSON_UNESCAPED_SLASHES);

        $body  = "--{$boundary}\r\nContent-Type: application/json; charset=UTF-8\r\n\r\n";
        $body .= $metadata . "\r\n--{$boundary}\r\nContent-Type: {$mimeType}\r\n\r\n";
        $body .= $conteudo . "\r\n--{$boundary}--";

        $ch = curl_init(self::UPLOAD_URI . '&fields=id,name,webViewLink');
        curl_setopt_array($ch, [
            CURLOPT_POST           => true,
            CURLOPT_HTTPHEADER     => [
                'Authorization: Bearer ' . $token,
                'Content-Type: multipart/related; boundary=' . $boundary,
                'Content-Length: ' . strlen($body),
            ],
            CURLOPT_POSTFIELDS     => $body,
            CURLOPT_RETURNTRANSFER => true,
            CURLOPT_TIMEOUT        => 600,
        ]);
        $resposta = curl_exec($ch);
        $http     = (int) curl_getinfo($ch, CURLINFO_HTTP_CODE);
        $erroCurl = curl_error($ch);
        curl_close($ch);

        if ($resposta === false) throw new RuntimeException('Falha de conexão com Drive: ' . $erroCurl);
        if ($http < 200 || $http >= 300) $this->lancarErroDrive($http, (string) $resposta);

        $dados = json_decode((string) $resposta, true);
        if (!is_array($dados) || empty($dados['id'])) {
            throw new RuntimeException('Resposta inesperada do Drive no upload.');
        }
        return [
            'id'          => (string) $dados['id'],
            'name'        => (string) ($dados['name'] ?? $nomeRemoto),
            'webViewLink' => isset($dados['webViewLink']) ? (string) $dados['webViewLink'] : '',
        ];
    }

    public function testarCredenciais(): array
    {
        $this->obterAccessToken(true);
        return ['ok' => true, 'client_id_prefixo' => substr($this->clientId, 0, 6) . '...'];
    }

    // ─── Internos ──────────────────────────────────────────────

    private function pastaAcessivel(string $folderId): bool
    {
        $token = $this->obterAccessToken();
        $ch = curl_init(self::FILES_URI . '/' . rawurlencode($folderId) . '?fields=id,trashed');
        curl_setopt_array($ch, [
            CURLOPT_HTTPHEADER     => ['Authorization: Bearer ' . $token],
            CURLOPT_RETURNTRANSFER => true,
            CURLOPT_TIMEOUT        => 30,
        ]);
        $resposta = curl_exec($ch);
        $http     = (int) curl_getinfo($ch, CURLINFO_HTTP_CODE);
        curl_close($ch);

        if ($resposta === false) return false;
        if ($http === 404 || $http === 403) return false;
        if ($http < 200 || $http >= 300) $this->lancarErroDrive($http, (string) $resposta);

        $dados = json_decode((string) $resposta, true);
        return is_array($dados) && !empty($dados['id']) && empty($dados['trashed']);
    }

    private function criarPasta(): string
    {
        $token = $this->obterAccessToken();
        $body  = json_encode(['name' => self::FOLDER_NOME, 'mimeType' => self::FOLDER_MIME],
            JSON_UNESCAPED_UNICODE | JSON_UNESCAPED_SLASHES);

        $ch = curl_init(self::FILES_URI . '?fields=id');
        curl_setopt_array($ch, [
            CURLOPT_POST           => true,
            CURLOPT_HTTPHEADER     => [
                'Authorization: Bearer ' . $token,
                'Content-Type: application/json; charset=UTF-8',
            ],
            CURLOPT_POSTFIELDS     => $body,
            CURLOPT_RETURNTRANSFER => true,
            CURLOPT_TIMEOUT        => 60,
        ]);
        $resposta = curl_exec($ch);
        $http     = (int) curl_getinfo($ch, CURLINFO_HTTP_CODE);
        curl_close($ch);

        if ($http < 200 || $http >= 300) $this->lancarErroDrive($http, (string) $resposta);

        $dados = json_decode((string) $resposta, true);
        if (!is_array($dados) || empty($dados['id'])) {
            throw new RuntimeException('Resposta inesperada do Drive ao criar pasta.');
        }
        return (string) $dados['id'];
    }

    private function obterAccessToken(bool $forcar = false): string
    {
        if (!$forcar && $this->accessToken !== null && time() < $this->accessTokenExpiraEm - 60) {
            return $this->accessToken;
        }

        $body = http_build_query([
            'client_id'     => $this->clientId,
            'client_secret' => $this->clientSecret,
            'refresh_token' => $this->refreshToken,
            'grant_type'    => 'refresh_token',
        ]);

        $ch = curl_init(self::TOKEN_URI);
        curl_setopt_array($ch, [
            CURLOPT_POST           => true,
            CURLOPT_POSTFIELDS     => $body,
            CURLOPT_HTTPHEADER     => ['Content-Type: application/x-www-form-urlencoded'],
            CURLOPT_RETURNTRANSFER => true,
            CURLOPT_TIMEOUT        => 30,
        ]);
        $resposta = curl_exec($ch);
        $http     = (int) curl_getinfo($ch, CURLINFO_HTTP_CODE);
        curl_close($ch);

        if ($http !== 200) $this->lancarErroToken($http, (string) $resposta);

        $dados = json_decode((string) $resposta, true);
        if (!is_array($dados) || empty($dados['access_token'])) {
            throw new RuntimeException('Resposta inesperada do OAuth ao trocar refresh_token.');
        }
        $this->accessToken         = (string) $dados['access_token'];
        $this->accessTokenExpiraEm = time() + (int) ($dados['expires_in'] ?? 3600);
        return $this->accessToken;
    }

    private function lancarErroToken(int $http, string $corpo): void
    {
        $msg = $this->extrairMensagemGoogle($corpo);
        if ($http === 400 && stripos($corpo, 'invalid_grant') !== false) {
            throw new RuntimeException('Refresh token OAuth inválido ou revogado. Gere um novo no OAuth Playground.');
        }
        if ($http === 401) {
            throw new RuntimeException('Client ID ou Client Secret OAuth incorretos.');
        }
        throw new RuntimeException("Google respondeu HTTP {$http}: {$msg}");
    }

    private function lancarErroDrive(int $http, string $corpo): void
    {
        $msg = $this->extrairMensagemGoogle($corpo);
        if ($http === 401) throw new RuntimeException('Client ID ou Secret OAuth incorretos.');
        if ($http === 403) {
            if (stripos($corpo, 'quota') !== false || stripos($corpo, 'storageQuotaExceeded') !== false) {
                throw new RuntimeException('Conta Google sem espaço disponível no Drive.');
            }
            if (stripos($corpo, 'Service Accounts do not have storage quota') !== false) {
                throw new RuntimeException('Service Account não tem quota no Drive. Use OAuth de conta pessoal (refresh_token), não Service Account.');
            }
            throw new RuntimeException("Acesso negado pelo Drive: {$msg}");
        }
        throw new RuntimeException("Drive respondeu HTTP {$http}: {$msg}");
    }

    private function extrairMensagemGoogle(string $corpo): string
    {
        $dados = json_decode($corpo, true);
        if (is_array($dados)) {
            return (string) ($dados['error_description']
                ?? $dados['error']['message']
                ?? (is_string($dados['error'] ?? null) ? $dados['error'] : '')
                ?: substr($corpo, 0, 300));
        }
        return substr($corpo, 0, 300);
    }
}
```

> **Armadilha**: NÃO use Service Account com escopo `drive.file` na sua conta pessoal — Service Accounts **não têm storage quota** e o upload falha com HTTP 403 `Service Accounts do not have storage quota`. O fluxo correto é OAuth user-delegated com `refresh_token`. Detalhes no [Troubleshooting](#troubleshooting).

---

## Passo 7 — Service: BackupRunner (orquestrador)

Arquivo: `app/services/BackupRunner.php`

Une dump + persistência + log em uma única chamada. Reutilizado tanto pelo controller (web) quanto pelos scripts CLI.

```php
<?php
require_once __DIR__ . '/BackupService.php';
require_once __DIR__ . '/GoogleDriveService.php';
require_once __DIR__ . '/../models/BackupLogModel.php';

class BackupRunner
{
    private BackupLogModel $logModel;
    private BackupService $backup;
    private PDO $db;

    public function __construct()
    {
        $this->logModel = new BackupLogModel();
        $this->backup   = new BackupService();
        $this->db       = db();
    }

    /** @return array{ok:bool, arquivo_nome?:string, tamanho?:int, erro?:string} */
    public function executarLocal(string $tipo, ?int $usuarioId): array
    {
        $logId = $this->logModel->iniciar($tipo, $usuarioId);
        try {
            $resultado = $this->backup->gerarDumpLocal();
            $this->backup->limparAntigos();
            $this->logModel->finalizarSucesso($logId, $resultado['arquivo_nome'], $resultado['tamanho']);
            $this->logModel->removerRegistrosArquivosInexistentes($this->backup->nomesArquivosLocais());
            return ['ok' => true, 'arquivo_nome' => $resultado['arquivo_nome'], 'tamanho' => $resultado['tamanho']];
        } catch (Throwable $e) {
            $this->logModel->finalizarFalha($logId, $e->getMessage());
            error_log('[App][Backup] Falha em backup local (' . $tipo . '): ' . $e->getMessage());
            return ['ok' => false, 'erro' => $e->getMessage()];
        }
    }

    /** @return array{ok:bool, arquivo_nome?:string, tamanho?:int, drive_file_id?:string, erro?:string} */
    public function executarRemoto(string $tipo, ?int $usuarioId): array
    {
        $logId      = $this->logModel->iniciar($tipo, $usuarioId);
        $caminhoTmp = null;
        try {
            $cfg           = $this->carregarConfig();
            $clientId      = trim($cfg['backup_oauth_client_id']     ?? '');
            $clientSecret  = trim($cfg['backup_oauth_client_secret'] ?? '');
            $refreshToken  = trim($cfg['backup_oauth_refresh_token'] ?? '');
            $folderIdAtual = trim($cfg['backup_drive_folder_id']     ?? '');

            if ($clientId === '' || $clientSecret === '' || $refreshToken === '') {
                throw new RuntimeException('Credenciais OAuth não configuradas. Acesse o painel admin e preencha os 3 campos.');
            }

            $resultado  = $this->backup->gerarDumpTemporario();
            $caminhoTmp = $resultado['caminho'];

            $drive    = new GoogleDriveService($clientId, $clientSecret, $refreshToken);
            $folderId = $drive->ensureFolder($folderIdAtual);
            if ($folderId !== $folderIdAtual) {
                $this->salvarConfig('backup_drive_folder_id', $folderId, $usuarioId);
            }

            $envio  = $drive->upload($caminhoTmp, $resultado['arquivo_nome'], 'application/gzip', $folderId);
            $this->logModel->finalizarSucesso($logId, $resultado['arquivo_nome'], $resultado['tamanho'], $envio['id']);

            return [
                'ok'            => true,
                'arquivo_nome'  => $resultado['arquivo_nome'],
                'tamanho'       => $resultado['tamanho'],
                'drive_file_id' => $envio['id'],
            ];
        } catch (Throwable $e) {
            $this->logModel->finalizarFalha($logId, $e->getMessage());
            error_log('[App][Backup] Falha em backup remoto (' . $tipo . '): ' . $e->getMessage());
            return ['ok' => false, 'erro' => $e->getMessage()];
        } finally {
            if ($caminhoTmp !== null && is_file($caminhoTmp)) @unlink($caminhoTmp);
        }
    }

    private function carregarConfig(): array
    {
        $chaves = ['backup_drive_folder_id','backup_oauth_client_id','backup_oauth_client_secret','backup_oauth_refresh_token'];
        $placeholders = implode(',', array_fill(0, count($chaves), '?'));
        $stmt = $this->db->prepare("SELECT chave, valor FROM configuracoes WHERE chave IN ({$placeholders})");
        $stmt->execute($chaves);
        $cfg = [];
        foreach ($stmt->fetchAll() as $row) $cfg[$row['chave']] = $row['valor'];
        return $cfg;
    }

    private function salvarConfig(string $chave, string $valor, ?int $usuarioId): void
    {
        $stmt = $this->db->prepare(
            "INSERT INTO configuracoes (chave, valor, atualizado_por)
             VALUES (?, ?, ?)
             ON DUPLICATE KEY UPDATE valor = VALUES(valor), atualizado_por = VALUES(atualizado_por)"
        );
        $stmt->execute([$chave, $valor, $usuarioId ?: null]);
    }
}
```

---

## Passo 8 — Controller: BackupController

Arquivo: `app/controllers/BackupController.php`

Inclui métodos para: (a) painel web, (b) disparo manual via botão, (c) cron HTTP com token (fallback, opcional), (d) download/exclusão de backups locais, (e) salvar credenciais OAuth.

```php
<?php
require_once __DIR__ . '/../../config/db.php';
require_once __DIR__ . '/../models/BackupLogModel.php';
require_once __DIR__ . '/../services/BackupService.php';
require_once __DIR__ . '/../services/GoogleDriveService.php';
require_once __DIR__ . '/../services/BackupRunner.php';

class BackupController
{
    private BackupLogModel $logModel;
    private BackupService $backup;
    private BackupRunner $runner;

    public function __construct()
    {
        $this->logModel = new BackupLogModel();
        $this->backup   = new BackupService();
        $this->runner   = new BackupRunner();
    }

    // GET /admin/backup
    public function painel(): void
    {
        // TODO: Sessao::exigirLogin() + Permissao::exigir() — adapte ao seu padrão.

        $this->backup->limparAntigos();
        $this->logModel->removerRegistrosArquivosInexistentes($this->backup->nomesArquivosLocais());

        $cfg              = $this->carregarConfig();
        $backupsLocais    = $this->logModel->listarBackupsLocaisDisponiveis();
        $ultimoRemoto     = $this->logModel->ultimoPorTipo('remoto_diario');
        $ultimoRemotoMan  = $this->logModel->ultimoPorTipo('manual_remoto');
        $historico        = $this->logModel->listarRecentes(30);
        $tokenCronGerado  = ($cfg['backup_cron_token'] ?? '') !== '';
        $oauthClientIdConfigurado     = ($cfg['backup_oauth_client_id']     ?? '') !== '';
        $oauthClientSecretConfigurado = ($cfg['backup_oauth_client_secret'] ?? '') !== '';
        $oauthRefreshTokenConfigurado = ($cfg['backup_oauth_refresh_token'] ?? '') !== '';
        $oauthCompleto    = $oauthClientIdConfigurado && $oauthClientSecretConfigurado && $oauthRefreshTokenConfigurado;
        $folderIdAtual    = $cfg['backup_drive_folder_id'] ?? '';
        $maxLocais        = (int) APP_BACKUP_MAX_LOCAIS;

        // Caminhos absolutos para exibir as instruções de cron no painel
        $cronDiarioPath  = str_replace('\\', '/', realpath(__DIR__ . '/../../cron/backup_diario.php')  ?: '');
        $cronSemanalPath = str_replace('\\', '/', realpath(__DIR__ . '/../../cron/backup_semanal.php') ?: '');

        include __DIR__ . '/../views/admin/backup.php';
    }

    // POST /admin/backup/executar  (tipo=local|remoto)
    public function executarManual(): void
    {
        // TODO: verificar CSRF + permissão
        $tipo = $_POST['tipo'] ?? '';
        $usuarioId = null; // TODO: pegar do session

        if ($tipo === 'local')        $this->runner->executarLocal('manual_local', $usuarioId);
        elseif ($tipo === 'remoto')   $this->runner->executarRemoto('manual_remoto', $usuarioId);

        header('Location: /admin/backup');
        exit;
    }

    // POST /admin/backup/configurar
    public function salvarConfiguracao(): void
    {
        // TODO: verificar CSRF + permissão
        $regenToken = !empty($_POST['regenerar_token']);
        $usuarioId  = null;

        $campos = [
            'backup_oauth_client_id'     => 'oauth_client_id',
            'backup_oauth_client_secret' => 'oauth_client_secret',
            'backup_oauth_refresh_token' => 'oauth_refresh_token',
        ];
        foreach ($campos as $chave => $campo) {
            $valor = trim($_POST[$campo] ?? '');
            if ($valor !== '') $this->salvarConfig($chave, $valor, $usuarioId);
        }

        if ($regenToken) {
            $novoToken = bin2hex(random_bytes(32));
            $this->salvarConfig('backup_cron_token', $novoToken, $usuarioId);
        }

        header('Location: /admin/backup');
        exit;
    }

    // GET /admin/backup/download/{id}
    public function download(int $id): void
    {
        $registro = $this->logModel->buscarPorId($id);
        if (!$registro
            || !in_array($registro['tipo'], ['local_semanal','manual_local'], true)
            || empty($registro['arquivo_nome'])
            || $registro['status'] !== 'sucesso'
        ) {
            http_response_code(404);
            echo '404 — backup não encontrado';
            return;
        }
        $caminho = $this->backup->caminhoLocal($registro['arquivo_nome']);
        if ($caminho === null) {
            http_response_code(404);
            echo '404 — arquivo ausente em disco';
            return;
        }
        header('Content-Type: application/gzip');
        header('Content-Disposition: attachment; filename="' . basename($caminho) . '"');
        header('Content-Length: ' . (int) (@filesize($caminho) ?: 0));
        header('X-Content-Type-Options: nosniff');
        header('Cache-Control: private, no-store');
        readfile($caminho);
        exit;
    }

    // POST /admin/backup/excluir/{id}
    public function excluir(int $id): void
    {
        // TODO: verificar CSRF + permissão
        $registro = $this->logModel->buscarPorId($id);
        if (!$registro || !in_array($registro['tipo'], ['local_semanal','manual_local'], true)) {
            header('Location: /admin/backup');
            exit;
        }
        if (!empty($registro['arquivo_nome'])) $this->backup->excluirLocal($registro['arquivo_nome']);
        $this->logModel->removerRegistroLocal($id);
        header('Location: /admin/backup');
        exit;
    }

    // GET /cron/backup-diario?token=...  (opcional — só se quiser HTTP em vez de CLI)
    public function cronDiario(): void
    {
        $this->validarTokenCronOuMorrer();
        $this->runner->executarRemoto('remoto_diario', null);
        echo "OK\n";
    }

    // GET /cron/backup-semanal?token=...
    public function cronSemanal(): void
    {
        $this->validarTokenCronOuMorrer();
        $this->runner->executarLocal('local_semanal', null);
        echo "OK\n";
    }

    // ─── Internos ──────────────────────────────────────────────

    private function validarTokenCronOuMorrer(): void
    {
        $tokenInformado = $_GET['token'] ?? '';
        $tokenEsperado  = $this->carregarConfig()['backup_cron_token'] ?? '';
        if ($tokenEsperado === '' || !is_string($tokenInformado) || !hash_equals($tokenEsperado, $tokenInformado)) {
            http_response_code(403);
            error_log('[App][Backup] Token inválido no /cron (IP: ' . ($_SERVER['REMOTE_ADDR'] ?? '?') . ')');
            die('Forbidden');
        }
    }

    private function carregarConfig(): array
    {
        $chaves = ['backup_drive_folder_id','backup_oauth_client_id','backup_oauth_client_secret','backup_oauth_refresh_token','backup_cron_token','backup_max_locais'];
        $placeholders = implode(',', array_fill(0, count($chaves), '?'));
        $stmt = db()->prepare("SELECT chave, valor FROM configuracoes WHERE chave IN ({$placeholders})");
        $stmt->execute($chaves);
        $cfg = [];
        foreach ($stmt->fetchAll() as $row) $cfg[$row['chave']] = $row['valor'];
        return $cfg;
    }

    private function salvarConfig(string $chave, string $valor, ?int $usuarioId): void
    {
        $stmt = db()->prepare(
            "INSERT INTO configuracoes (chave, valor, atualizado_por)
             VALUES (?, ?, ?)
             ON DUPLICATE KEY UPDATE valor = VALUES(valor), atualizado_por = VALUES(atualizado_por)"
        );
        $stmt->execute([$chave, $valor, $usuarioId ?: null]);
    }
}
```

---

## Passo 9 — Rotas

Adapte ao seu roteador. Exemplo com mapa `MÉTODO:/caminho => closure`:

```php
'GET:/admin/backup'                    => fn() => (new BackupController)->painel(),
'POST:/admin/backup/executar'          => fn() => (new BackupController)->executarManual(),
'POST:/admin/backup/configurar'        => fn() => (new BackupController)->salvarConfiguracao(),
'GET:/cron/backup-diario'              => fn() => (new BackupController)->cronDiario(),
'GET:/cron/backup-semanal'             => fn() => (new BackupController)->cronSemanal(),
```

E para rotas com parâmetro de ID:
```php
if (preg_match('#^/admin/backup/download/(\d+)$#', $uri, $m) && $metodo === 'GET') {
    (new BackupController)->download((int) $m[1]); exit;
}
if (preg_match('#^/admin/backup/excluir/(\d+)$#', $uri, $m) && $metodo === 'POST') {
    (new BackupController)->excluir((int) $m[1]); exit;
}
```

> As rotas `/cron/backup-*` por HTTP são **opcionais**. O cron principal usa CLI direto (Passo 11). As HTTP existem como fallback de teste pelo navegador.

---

## Passo 10 — View do painel

Arquivo: `app/views/admin/backup.php`

Painel com 4 blocos: (1) status dos últimos backups com botões manuais, (2) lista de backups locais com download/excluir, (3) formulário OAuth (3 campos), (4) **instruções de cron com aviso da armadilha** (ver Passo 15 — esse bloco é crítico para evitar o bug).

A view completa do GSAU é referência direta — está em [gsau/app/views/admin/backup.php](https://github.com/MouraoBSB/GSAU-Sistema/blob/main/gsau/app/views/admin/backup.php). Trecho essencial do bloco de cron (use **exatamente esse layout** para evitar que quem configurar caia na armadilha):

```php
<?php
    $logDiarioPath  = dirname($cronDiarioPath, 2)  . '/logs/cron_backup.log';
    $logSemanalPath = dirname($cronSemanalPath, 2) . '/logs/cron_backup.log';
    $cmdDiario      = '/usr/local/bin/php ' . $cronDiarioPath  . ' >> ' . $logDiarioPath  . ' 2>&1';
    $cmdSemanal     = '/usr/local/bin/php ' . $cronSemanalPath . ' >> ' . $logSemanalPath . ' 2>&1';
?>

<!-- AVISO DA ARMADILHA — NÃO REMOVER -->
<div class="bg-red-50 border border-red-200 rounded-lg p-3 text-xs text-red-800">
    <strong>Atenção — não cole o agendamento dentro do campo "Comando".</strong>
    O campo Comando deve começar em <code>/usr/local/bin/php</code>. Se "0 3 * * *" entrar no Comando,
    o cron grava <code>bash: 0: command not found</code> no log e o backup não roda.
    Confira com <code>crontab -l</code> no Terminal — cada linha deve ter <strong>apenas um</strong>
    bloco de horário antes do PHP.
</div>

<!-- Cron diário — campos separados + comando isolado -->
<div class="bg-amber-50 border border-amber-100 rounded-lg p-3 space-y-2">
    <p class="text-xs font-semibold">1º cron — Backup remoto diário (Google Drive)</p>
    <div class="grid grid-cols-5 gap-2 text-center">
        <div><div class="text-[10px]">Minuto</div><div class="bg-white rounded px-2 py-1 font-mono">0</div></div>
        <div><div class="text-[10px]">Hora</div><div class="bg-white rounded px-2 py-1 font-mono">3</div></div>
        <div><div class="text-[10px]">Dia</div><div class="bg-white rounded px-2 py-1 font-mono">*</div></div>
        <div><div class="text-[10px]">Mês</div><div class="bg-white rounded px-2 py-1 font-mono">*</div></div>
        <div><div class="text-[10px]">Dia da semana</div><div class="bg-white rounded px-2 py-1 font-mono">*</div></div>
    </div>
    <div>
        <div class="text-[10px]">Comando</div>
        <pre class="bg-white rounded p-2 overflow-x-auto whitespace-pre text-[11px] font-mono"><?= htmlspecialchars($cmdDiario) ?></pre>
    </div>
</div>
```

> **Por que campos separados em vez de uma linha única?** Se a view mostrar `0 3 * * * /usr/local/bin/php ...` em um único bloco, o operador tende a copiar tudo e colar no campo "Comando" do cPanel — o que produz exatamente o bug descrito no Passo 15.

---

## Passo 11 — Scripts CLI de cron

Arquivo: `cron/backup_diario.php`

```php
<?php
// Defesa em profundidade: bloqueia execução via web (script só roda em CLI)
if (php_sapi_name() !== 'cli') {
    $ip = $_SERVER['REMOTE_ADDR'] ?? '';
    if (!in_array($ip, ['127.0.0.1', '::1'], true)) {
        http_response_code(403);
        die('Acesso negado.');
    }
}

require_once __DIR__ . '/../config/db.php';                       // bootstrap mínimo
require_once __DIR__ . '/../app/services/BackupRunner.php';

$inicio = microtime(true);
$log = function (string $msg) {
    echo '[' . date('Y-m-d H:i:s') . "] {$msg}\n";
};

$log('Iniciando backup remoto diário (Google Drive)...');

try {
    $resultado = (new BackupRunner())->executarRemoto('remoto_diario', null);
    if ($resultado['ok']) {
        $log('Sucesso: ' . $resultado['arquivo_nome'] . ' (' . $resultado['tamanho'] . ' bytes) em '
            . round(microtime(true) - $inicio, 2) . 's');
        exit(0);
    }
    $log('FALHA: ' . ($resultado['erro'] ?? 'erro desconhecido'));
    exit(1);
} catch (Throwable $e) {
    $log('ERRO FATAL: ' . $e->getMessage());
    exit(2);
}
```

Arquivo: `cron/backup_semanal.php` (idêntico, troca `executarRemoto('remoto_diario', null)` por `executarLocal('local_semanal', null)`).

> **`php_sapi_name() !== 'cli'`** é proteção defensiva. Se algum dia o arquivo virar acessível via web por engano (`/cron/backup_diario.php`), recusa com 403 em vez de gerar um backup sem auth.

---

## Passo 12 — Configurar OAuth no Google Cloud

Guia detalhado: [apis/google-oauth-drive.md](../apis/google-oauth-drive.md). Resumo do necessário:

1. **[console.cloud.google.com](https://console.cloud.google.com/)** → New Project → `{APP} Backups`.
2. **APIs & Services** → **Library** → procure "Google Drive API" → **Enable**.
3. **OAuth consent screen** → User Type **External** → preencha App name, User support email, Developer contact → Save.
4. **Scopes** → Add → marque **apenas** `.../auth/drive.file` (não-sensível, não exige verificação).
5. **Test users** → adicione o seu próprio email Google.
6. **Publish App** → **Confirm** (importantíssimo — em modo Testing, o refresh_token expira em 7 dias).
7. **Credentials** → **Create Credentials** → **OAuth client ID**:
   - Application type: **Web application**
   - Name: `{APP} Web Client`
   - Authorized redirect URIs: `https://developers.google.com/oauthplayground`
8. **Create** → anote **Client ID** e **Client Secret** (vão pro painel admin no Passo 14).

---

## Passo 13 — Obter refresh token via OAuth Playground

Esse passo é manual, feito **UMA vez**.

1. Acesse [developers.google.com/oauthplayground](https://developers.google.com/oauthplayground).
2. Clique no ⚙️ **OAuth 2.0 configuration** (canto superior direito):
   - Marque **Use your own OAuth credentials**.
   - Cole o **OAuth Client ID** e **OAuth Client secret** do passo anterior.
   - Close.
3. À esquerda, na lista de APIs, busque **Drive API v3** → expanda → marque **`https://www.googleapis.com/auth/drive.file`**.
4. Clique **Authorize APIs** → faça login com a conta Google que vai **dona** dos backups → autorize.
5. Volta pro Playground com o "Authorization code" preenchido → clique **Exchange authorization code for tokens**.
6. Copie o **Refresh token** (`1//0g...`) — esse vai pro painel admin.

> **Se o refresh_token vier vazio**: você já tinha autorizado o app antes. Vá em [myaccount.google.com/permissions](https://myaccount.google.com/permissions), revogue o acesso do app, e refaça os passos 4-6.

---

## Passo 14 — Cadastrar credenciais no painel

1. Acesse `https://{seu-dominio}/admin/backup`.
2. Na seção **Google Drive — OAuth 2.0**, preencha:
   - **Client ID OAuth**: do Passo 12.
   - **Client Secret OAuth**: do Passo 12.
   - **Refresh Token**: do Passo 13.
3. Clique **Salvar Configurações**.
4. Clique **Fazer backup remoto agora**.
5. Confirme em [drive.google.com](https://drive.google.com): apareceu pasta `{APP} Backups` com um arquivo `app_YYYY-MM-DD_HHMMSS.sql.gz`.

Se o teste manual funcionar, **as credenciais estão OK**. Pode ir para o Passo 15.

---

## Passo 15 — Configurar cron jobs no cPanel ⚠️

> Este é o passo onde **todo mundo erra**. Leia com atenção.

### 15.1 — Descobrir o caminho do PHP

Terminal do cPanel:
```bash
which php
# ou para uma versão específica:
which ea-php82
```

Anote a saída (geralmente `/usr/local/bin/php` ou `/opt/cpanel/ea-php82/root/usr/bin/php`).

### 15.2 — Abrir Cron Jobs

cPanel → **Avançado** → **Cron Jobs**.

### 15.3 — Adicionar o cron diário (backup remoto)

Em **Add New Cron Job**, preencha **cada campo separadamente**:

| Campo | Valor |
|---|---|
| Minuto | `0` |
| Hora | `3` |
| Dia | `*` |
| Mês | `*` |
| Dia da semana | `*` |
| **Comando** | `/usr/local/bin/php /home/{USUARIO}/{seu-dominio}/cron/backup_diario.php >> /home/{USUARIO}/{seu-dominio}/logs/cron_backup.log 2>&1` |

Clique **Add New Cron Job**.

### 15.4 — Adicionar o cron semanal (backup local)

| Campo | Valor |
|---|---|
| Minuto | `0` |
| Hora | `2` |
| Dia | `*` |
| Mês | `*` |
| Dia da semana | `0` *(domingo)* |
| **Comando** | `/usr/local/bin/php /home/{USUARIO}/{seu-dominio}/cron/backup_semanal.php >> /home/{USUARIO}/{seu-dominio}/logs/cron_backup.log 2>&1` |

### 15.5 — A armadilha (LEIA ANTES DE SALVAR)

> 🛑 **NUNCA cole a linha inteira `0 3 * * * /usr/local/bin/php ...` dentro do campo Comando.**
>
> O cPanel **já tem campos separados** para o agendamento (Minuto/Hora/Dia/Mês/Dia da semana). Se você colocar `0 3 * * *` também no Comando, o crontab final fica:
>
> ```
> 0 3 * * * 0 3 * * * /usr/local/bin/php /home/.../backup_diario.php >> ... 2>&1
> ```
>
> O cron daemon usa os 5 primeiros tokens como agendamento (`0 3 * * *`) e o **resto vira o comando**, que começa com `0` (um número). O shell tenta executar `0` como programa e grava no log:
>
> ```
> /bin/bash: 0: command not found
> ```
>
> **O PHP nunca chega a rodar.** O sintoma é: backup manual funciona, log de erros do PHP não tem nenhuma menção a backup, e `cron_backup.log` só tem `bash: 0: command not found`.

### 15.6 — Auditar o crontab depois de salvar

No Terminal do cPanel:
```bash
crontab -l
```

Cada linha deve ter exatamente **uma** sequência de 5 campos antes do `/usr/local/bin/php`. Exemplo correto:

```
0 3 * * * /usr/local/bin/php /home/cliente/seusite.com.br/cron/backup_diario.php >> /home/cliente/seusite.com.br/logs/cron_backup.log 2>&1
0 2 * * 0 /usr/local/bin/php /home/cliente/seusite.com.br/cron/backup_semanal.php >> /home/cliente/seusite.com.br/logs/cron_backup.log 2>&1
```

Errado (com prefixo duplicado):
```
0 3 * * * 0 3 * * * /usr/local/bin/php ...    ← BUG
```

### 15.7 — Garantir pasta de logs

```bash
mkdir -p /home/{USUARIO}/{seu-dominio}/logs
chmod 755 /home/{USUARIO}/{seu-dominio}/logs
```

---

## Passo 16 — Testar

### 16.1 — Teste imediato (manual via CLI)

Antes de esperar até 03:00, rode o que o cron vai rodar:

```bash
/usr/local/bin/php /home/{USUARIO}/{seu-dominio}/cron/backup_diario.php
```

Saída esperada em poucos segundos:
```
[2026-MM-DD HH:MM:SS] Iniciando backup remoto diário (Google Drive)...
[2026-MM-DD HH:MM:SS] Sucesso: app_2026-MM-DD_HHMMSS.sql.gz (XXXXXX bytes) em X.XXs
```

Conferir:
- Google Drive → pasta `{APP} Backups`: novo arquivo `app_*.sql.gz`.
- Painel admin → Histórico: novo registro `Diário (auto)` com status sucesso.

### 16.2 — Teste com cron daemon (sem esperar 03:00)

Se quiser validar o caminho exato do cron daemon (não só o PHP):

1. No cPanel, edite o cron diário e mude **Hora** para daqui a 2-3 minutos (ex.: se são 14:37, coloque Hora=`14` e Minuto=`40`).
2. Aguarde passar o horário.
3. Cheque o `cron_backup.log`:
   ```bash
   tail -30 /home/{USUARIO}/{seu-dominio}/logs/cron_backup.log
   ```
   Deve mostrar `Iniciando... Sucesso: ...`. **Não pode** mostrar `bash: 0: command not found`.
4. Volte a hora para `3`.

### 16.3 — Limpeza opcional do log antes do teste

```bash
> /home/{USUARIO}/{seu-dominio}/logs/cron_backup.log
```
Zera o arquivo para ver só a próxima execução sem o ruído antigo.

---

## Restaurar um backup

### Baixar o arquivo

- **Backup remoto**: no Drive, baixe `app_YYYY-MM-DD_HHMMSS.sql.gz`.
- **Backup local**: pelo painel → Arquivos Locais → **Baixar**.

### Descomprimir e restaurar

```bash
# Descomprime
gunzip app_2026-05-20_093726.sql.gz
# Resultado: app_2026-05-20_093726.sql

# Restaura (apaga e recria todas as tabelas — o dump tem DROP TABLE IF EXISTS)
mysql -u {USER} -p {DBNAME} < app_2026-05-20_093726.sql
```

> O dump usa `SET FOREIGN_KEY_CHECKS = 0;` no início e `= 1` no fim — restaura na ordem qualquer sem quebrar por dependências circulares.

### Restauração parcial (uma tabela)

```bash
# Extrai SÓ a tabela usuarios
gunzip -c app_2026-05-20_093726.sql.gz \
    | sed -n '/-- ── Estrutura de `usuarios` ──/,/-- ──/p' \
    > restore_usuarios.sql

mysql -u {USER} -p {DBNAME} < restore_usuarios.sql
```

---

## Adaptações para outros projetos

### Sem tabela `configuracoes`

Se seu projeto não tem chave/valor, troque as chamadas a `carregarConfig()` e `salvarConfig()` por leitura/escrita de um arquivo PHP local:

```php
// config/backup.secret.php — NUNCA versionar
return [
    'backup_oauth_client_id'     => '{CLIENT_ID}',
    'backup_oauth_client_secret' => '{CLIENT_SECRET}',
    'backup_oauth_refresh_token' => '{REFRESH_TOKEN}',
    'backup_drive_folder_id'     => '',
];
```

Adicione `config/backup.secret.php` ao `.gitignore`.

### Sem tabela `usuarios`

Remova a `CONSTRAINT fk_backup_usuario` da migration. Coluna `acionado_por_usuario_id` fica sempre `NULL` em ambientes single-user.

### Sem painel admin

Os scripts CLI funcionam sem painel. Configure credenciais via `config/backup.secret.php` (acima) ou via INSERT manual em `configuracoes`. Skip dos passos 10 e 14 (use só `crontab` e SSH).

### Caminho do PHP CLI diferente

Em hospedagens com múltiplas versões PHP, ajuste o cron para a versão específica:

```bash
/opt/cpanel/ea-php82/root/usr/bin/php /home/.../cron/backup_diario.php
# OU
/usr/local/bin/ea-php82 /home/.../cron/backup_diario.php
```

### Projeto sem o helper `db()`

Substitua todas as chamadas a `db()` por sua função/classe singleton. Os serviços só precisam de um `PDO` com `ERRMODE_EXCEPTION` e `FETCH_ASSOC`.

### Renomear "GSAU"/"app" para o nome do projeto

Faça find/replace global por:
- `app_` (nome dos arquivos de backup) → `{APP_SLUG}_` (ex.: `farol_`)
- `APP_BACKUP_DIR`, `APP_BACKUP_MAX_LOCAIS`, `APP_TIMEZONE`, `APP_MYSQL_TIMEZONE`, `APP_VERSION` → constantes do seu projeto
- `[App][Backup]` (prefixo dos `error_log`) → `[{NOME-DO-PROJETO}][Backup]`
- `'APP Backups'` em `GoogleDriveService::FOLDER_NOME` → nome da pasta no Drive

E na regex do `BackupService::caminhoLocal()`:
```php
if (!preg_match('/^app_\d{4}-\d{2}-\d{2}_\d{6}\.sql\.gz$/', $arquivoNome)) {
```
substitua `app_` pelo mesmo prefixo usado na geração.

---

## Troubleshooting

### `bash: 0: command not found` no `cron_backup.log` (e backup não roda automaticamente)

**Causa:** o campo "Comando" do cron no cPanel está com `0 3 * * *` no início. Ver [Passo 15.5](#155--a-armadilha-leia-antes-de-salvar).

**Como confirmar:**
```bash
crontab -l
```
Linha com sequência tipo `0 3 * * * 0 3 * * * /usr/local/bin/php ...` (5 campos de tempo + 5 tokens duplicados antes do PHP).

**Solução:** edite o cron no cPanel. O campo **Comando** deve começar em `/usr/local/bin/php` — sem prefixo de horário. Os 5 campos de agendamento ficam separados em Minuto/Hora/Dia/Mês/Dia da semana.

---

### Backup manual funciona, mas cron não dispara nada (log vazio)

**Causa A:** o caminho do PHP no cron está errado. Confira com `which php`.

**Causa B:** o cPanel limita número de crons (geralmente 10). Já bateu o teto? Veja a página de Cron Jobs.

**Causa C:** o script `backup_diario.php` não tem permissão de leitura.
```bash
chmod 644 /home/{USUARIO}/{seu-dominio}/cron/backup_diario.php
```

---

### `Refresh token OAuth inválido ou revogado`

**Causa A:** app OAuth ainda está em **Testing** (não publicado) e passou de 7 dias.

**Causa B:** você revogou o acesso em [myaccount.google.com/permissions](https://myaccount.google.com/permissions).

**Causa C:** o token guarda quebra de linha ou espaço extra do copy-paste.

**Solução:**
1. Publique o app no Google Cloud: **OAuth consent screen** → **Publish App** → **Confirm**.
2. Refaça o [Passo 13](#passo-13--obter-refresh-token-via-oauth-playground) para gerar novo token.
3. Atualize o **Refresh Token** no painel admin (com trim).

---

### `Service Accounts do not have storage quota` (HTTP 403)

**Causa:** você tentou usar uma **Service Account** em vez de OAuth user-delegated. Service Accounts não têm storage no Drive — só conseguem usar Shared Drives.

**Solução:** use o fluxo OAuth do Passo 12-13 (Web application + refresh_token). Esquece Service Account para esse caso de uso.

---

### `Conta Google sem espaço disponível no Drive` (HTTP 403, `storageQuotaExceeded`)

**Causa:** a conta Google que autorizou está com 15 GB usados.

**Solução:** limpe o Drive **OU** mude para uma conta com Google One **OU** reduza o tamanho dos backups (limpando dados antigos da base antes do dump).

---

### Upload retorna `redirect_uri_mismatch`

**Causa:** o redirect URI configurado no Google Cloud não bate com o que o Playground/seu app usa.

**Solução:** em **Credentials** → o OAuth Client → **Authorized redirect URIs**, adicione exatamente `https://developers.google.com/oauthplayground`. Atenção a barra final, http vs https.

---

### `Falha de conexão com Drive` ou timeout

**Causa A:** firewall da hospedagem bloqueando saída pra `googleapis.com`.

**Causa B:** servidor sem DNS configurado pra resolver `oauth2.googleapis.com`.

**Solução:** teste no Terminal do cPanel:
```bash
curl -v https://oauth2.googleapis.com/token
```
Se falhar conexão, abra ticket com o suporte da hospedagem.

---

### Dump gera arquivo vazio ou 0 bytes

**Causa A:** PDO sem permissão de `SHOW TABLES` no schema.

**Causa B:** schema vazio (sem tabelas).

**Solução:** confira `SHOW GRANTS FOR CURRENT_USER` no MySQL. Precisa de `SELECT`, `SHOW VIEW`, `LOCK TABLES`, `EVENT`, `TRIGGER`.

---

### Cron silenciosamente para de rodar depois de algum tempo

**Causa:** a hospedagem fez maintenance, mudou o caminho do PHP CLI ou renovou o crontab.

**Solução:** faça `crontab -l` periodicamente. Se quiser monitoramento ativo, adicione um cron que verifica se houve backup nas últimas 26 horas e envia email se não:

```php
// cron/verificar_backup.php
require_once __DIR__ . '/../config/db.php';
$stmt = db()->query("SELECT COUNT(*) FROM backups_log
    WHERE tipo IN ('remoto_diario','manual_remoto')
      AND status = 'sucesso'
      AND data_inicio > DATE_SUB(NOW(), INTERVAL 26 HOUR)");
if ((int) $stmt->fetchColumn() === 0) {
    mail('admin@empresa.com', '[ALERTA] Backup não rodou', 'Nenhum backup remoto nas últimas 26h.');
}
```

Agenda: `0 8 * * * /usr/local/bin/php /home/.../cron/verificar_backup.php`.

---

## Segurança

- **Nunca commite** `client_id`, `client_secret` ou `refresh_token` no Git. Use banco (`configuracoes`) ou `.env`.
- **Sempre proteja** a pasta `storage/backups/` com `.htaccess` (`Deny from all`) — backups SQL contêm senhas hasheadas, PII e tokens.
- O **dump não é criptografado** dentro do .gz. Se for sensível, criptografe antes do upload (`openssl enc -aes-256-cbc -in dump.sql.gz -out dump.sql.gz.enc -pass file:chave.key`).
- O `BackupService::caminhoLocal()` faz whitelist do nome com regex para evitar **path traversal** via URL `/admin/backup/download/?file=../../../etc/passwd`. Mantenha essa validação.
- A view `download()` envia `X-Content-Type-Options: nosniff` e `Cache-Control: private, no-store` — não remova.
- O escopo `drive.file` é o **mais restrito** que serve. Não use `drive` (acesso total) a menos que precise.

---

## Referências

- [cPanel — Cron Jobs](https://docs.cpanel.net/cpanel/advanced/cron-jobs/)
- [Google Drive API v3 — Reference](https://developers.google.com/drive/api/v3/reference)
- [OAuth 2.0 Playground](https://developers.google.com/oauthplayground)
- [PHP — gzopen()](https://www.php.net/manual/en/function.gzopen.php)
- [Google Drive — escopos OAuth](https://developers.google.com/drive/api/guides/api-specific-auth)
- [crontab.guru](https://crontab.guru/) — calculadora visual

Outras docs deste repositório que complementam:
- [Cron jobs na Napoleon Host](../hospedagem/napoleon-host-cron.md)
- [Google OAuth 2.0 + Drive API](../apis/google-oauth-drive.md)
- [PDO Singleton](../php/pdo-singleton.md)
- [Deploy na Napoleon Host](../hospedagem/napoleon-host-deploy.md)

---

**Última revisão:** 2026-05-20
**Testado em:** PHP 8.2, MySQL 8, cPanel v110 (Napoleon Host), GSAU em produção
**Autor:** Thiago Mourão — [github.com/MouraoBSB](https://github.com/MouraoBSB)
