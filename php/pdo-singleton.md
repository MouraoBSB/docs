# PDO Singleton com prepared statements

> Padrão de conexão PDO reutilizável, com lazy loading, prepared statements e tratamento de erros. Substitui `mysql_*` e `mysqli_*` em projetos legados.

**Contexto:** Aplicação PHP com arquitetura MVC (com ou sem framework). Validado no GSAU como `config/db.php`.

---

## Pré-requisitos

- [ ] PHP 7.4+ (idealmente 8.0+)
- [ ] Extensão `pdo_mysql` habilitada
- [ ] Banco MySQL/MariaDB criado com usuário e senha

---

## Implementação

### `config/db.php`

```php
<?php

class DB
{
    private static ?PDO $instancia = null;

    public static function conectar(): PDO
    {
        if (self::$instancia !== null) {
            return self::$instancia;
        }

        $config = [
            'host'     => 'localhost',
            'database' => 'cemaneto_meuapp',
            'username' => 'cemaneto_user',
            'password' => '{SENHA}',
            'charset'  => 'utf8mb4',
        ];

        $dsn = "mysql:host={$config['host']};dbname={$config['database']};charset={$config['charset']}";

        $opcoes = [
            PDO::ATTR_ERRMODE            => PDO::ERRMODE_EXCEPTION,
            PDO::ATTR_DEFAULT_FETCH_MODE => PDO::FETCH_ASSOC,
            PDO::ATTR_EMULATE_PREPARES   => false,    // usar prepared statements reais
            PDO::ATTR_PERSISTENT         => false,    // false em hospedagem compartilhada
        ];

        try {
            self::$instancia = new PDO($dsn, $config['username'], $config['password'], $opcoes);
            // Timezone consistente
            self::$instancia->exec("SET time_zone = '-03:00'");
        } catch (PDOException $e) {
            // NUNCA exiba $e->getMessage() em produção (pode vazar credenciais)
            error_log('Erro de conexão DB: ' . $e->getMessage());
            http_response_code(500);
            die('Erro de conexão com o banco de dados.');
        }

        return self::$instancia;
    }
}
```

### Carregar configuração de arquivo separado (recomendado)

```php
// config/db_config.php (NO .gitignore)
return [
    'host'     => 'localhost',
    'database' => 'cemaneto_meuapp',
    'username' => 'cemaneto_user',
    'password' => '{SENHA}',
    'charset'  => 'utf8mb4',
];
```

E em `config/db.php`:

```php
$config = require __DIR__ . '/db_config.php';
```

---

## Uso em Models

```php
<?php

class UsuarioModel
{
    public static function buscarPorMatricula(string $matricula): ?array
    {
        $sql = "SELECT id, matricula, nome, perfil FROM usuarios WHERE matricula = :mat LIMIT 1";
        $stmt = DB::conectar()->prepare($sql);
        $stmt->execute([':mat' => $matricula]);
        $row = $stmt->fetch();
        return $row ?: null;
    }

    public static function listarAtivos(): array
    {
        $sql = "SELECT id, matricula, nome FROM usuarios WHERE ativo = 1 ORDER BY nome";
        return DB::conectar()->query($sql)->fetchAll();
    }

    public static function criar(array $dados): int
    {
        $sql = "INSERT INTO usuarios (matricula, nome, senha, perfil)
                VALUES (:mat, :nome, :senha, :perfil)";
        $stmt = DB::conectar()->prepare($sql);
        $stmt->execute([
            ':mat'    => $dados['matricula'],
            ':nome'   => $dados['nome'],
            ':senha'  => password_hash($dados['senha'], PASSWORD_BCRYPT),
            ':perfil' => $dados['perfil'],
        ]);
        return (int) DB::conectar()->lastInsertId();
    }
}
```

---

## Transações

Para operações que envolvem múltiplas tabelas e precisam ser atômicas:

```php
$pdo = DB::conectar();
$pdo->beginTransaction();
try {
    $pdo->prepare("INSERT INTO internacoes (...) VALUES (...)")->execute([...]);
    $internacaoId = (int) $pdo->lastInsertId();

    $pdo->prepare("INSERT INTO internacao_logs (internacao_id, evento) VALUES (?, ?)")
        ->execute([$internacaoId, 'criacao']);

    $pdo->commit();
} catch (Throwable $e) {
    $pdo->rollBack();
    error_log('Erro na transação: ' . $e->getMessage());
    throw $e;
}
```

---

## Princípios

### ✅ Sempre

- **Use prepared statements** (`:nome` ou `?`) para qualquer valor vindo do usuário.
- **Use placeholders nomeados** (`:nome`) em queries com mais de 2-3 parâmetros — mais legível e menos sujeito a erros.
- **Capture exceções** (`PDOException` ou `Throwable`) ao escrever — não deixe erro de banco vazar 500 cru.
- **Logue erros** em arquivo (`error_log`) e exiba mensagem genérica ao usuário.
- **Use `BINARY`** em comparações `WHERE` quando a colação for case-insensitive e você precisar de match exato (case-sensitive).

### ❌ Nunca

- **Concatenar variáveis em SQL.** Mesmo "só esse aqui é seguro" é convite a SQL injection.
  ```php
  // ❌ NUNCA
  $sql = "SELECT * FROM usuarios WHERE matricula = '{$_GET['m']}'";

  // ✅ SEMPRE
  $sql = "SELECT * FROM usuarios WHERE matricula = :mat";
  $stmt = DB::conectar()->prepare($sql);
  $stmt->execute([':mat' => $_GET['m']]);
  ```
- **Usar `PDO::ATTR_EMULATE_PREPARES = true`** sem necessidade. Prepared statements reais são mais seguros e quase sempre disponíveis.
- **Exibir `$e->getMessage()` ao usuário.** Mensagens podem vazar nomes de tabelas, colunas, ou até credenciais em alguns drivers.
- **Conectar a cada query.** O Singleton existe pra evitar isso.

---

## Troubleshooting

### Erro: `SQLSTATE[HY000] [2002] No such file or directory`

**Causa:** PDO está tentando conectar via socket Unix, mas o socket não está no caminho padrão.

**Solução:** force TCP usando `127.0.0.1` em vez de `localhost`:

```php
'host' => '127.0.0.1',
```

---

### Erro: `SQLSTATE[42S22] Column not found`

**Causa:** nome de coluna errado no SQL ou diferença case-sensitive entre código e banco.

**Solução:**
- Confira o schema (`DESCRIBE tabela`).
- Em MySQL, nomes de coluna são case-insensitive por padrão, mas nomes de tabela podem ser case-sensitive em Linux.

---

### Erro: `Integrity constraint violation: 1062 Duplicate entry`

**Causa:** tentativa de inserir valor único que já existe (UNIQUE constraint).

**Solução:** capture especificamente:

```php
try {
    UsuarioModel::criar($dados);
} catch (PDOException $e) {
    if ($e->getCode() === '23000') {
        // Constraint violation
        return ['erro' => 'Matrícula já cadastrada'];
    }
    throw $e;
}
```

---

### Performance ruim em queries com muitos parâmetros IN

**Causa:** PDO não tem suporte nativo a `IN (?, ?, ?)` com array.

**Solução:** monte placeholders dinamicamente:

```php
$ids = [1, 5, 10, 23];
$placeholders = implode(',', array_fill(0, count($ids), '?'));
$sql = "SELECT * FROM usuarios WHERE id IN ({$placeholders})";
$stmt = DB::conectar()->prepare($sql);
$stmt->execute($ids);
```

---

## Referências

- [PHP Manual — PDO](https://www.php.net/manual/en/book.pdo.php)
- [PHP Manual — Prepared Statements](https://www.php.net/manual/en/pdo.prepared-statements.php)
- [OWASP — SQL Injection Prevention Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/SQL_Injection_Prevention_Cheat_Sheet.html)

---

**Última revisão:** 2026-05-18
**Testado em:** PHP 8.0/8.2, MariaDB 10.6, MySQL 8.0
