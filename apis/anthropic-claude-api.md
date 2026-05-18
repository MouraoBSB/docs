# API Anthropic (Claude) para geração de texto

> Como integrar a API da Anthropic (Claude) em uma aplicação PHP para geração de relatórios, sumarizações e qualquer tarefa de texto. Inclui controle de custo, escolha de modelo e tratamento de erros.

**Contexto:** Aplicação PHP que precisa gerar texto formal (relatórios, ofícios, sumários) sem o usuário escrever do zero. Validado no GSAU para relatórios de internação, alta, transições G1↔G2 e plano emergencial.

---

## Pré-requisitos

- [ ] Conta na [Anthropic Console](https://console.anthropic.com/)
- [ ] API key criada (formato `sk-ant-api03-...`)
- [ ] Créditos disponíveis (ou plano ativo)
- [ ] PHP 8.0+ com `curl` e `json`
- [ ] Estimativa de uso (custos podem crescer rápido — veja abaixo)

---

## Passo a passo

### 1. Obter a API key

1. Acesse [console.anthropic.com](https://console.anthropic.com/).
2. **Settings** → **API Keys** → **Create Key**.
3. Dê um nome descritivo (ex: "GSAU Produção").
4. Copie a key (`sk-ant-api03-...`) — aparece UMA vez.

### 2. Armazenar a key com segurança

**NUNCA** coloque a key em código versionado.

```php
// config/anthropic.php (no .gitignore)
return [
    'api_key' => 'sk-ant-api03-{...}',
    'model'   => 'claude-sonnet-4-20250514',   // ou claude-haiku-4-5-20251001
    'max_tokens' => 1024,
];
```

### 3. Chamar a API

A API usa o endpoint `/v1/messages`. Estrutura básica:

```php
function chamarClaude(string $prompt, int $maxTokens = 1024): ?string {
    $config = require __DIR__ . '/../config/anthropic.php';

    $body = json_encode([
        'model'      => $config['model'],
        'max_tokens' => $maxTokens,
        'messages'   => [
            ['role' => 'user', 'content' => $prompt],
        ],
    ]);

    $ch = curl_init('https://api.anthropic.com/v1/messages');
    curl_setopt_array($ch, [
        CURLOPT_POST           => true,
        CURLOPT_POSTFIELDS     => $body,
        CURLOPT_RETURNTRANSFER => true,
        CURLOPT_HTTPHEADER     => [
            'Content-Type: application/json',
            'x-api-key: ' . $config['api_key'],
            'anthropic-version: 2023-06-01',
        ],
        CURLOPT_TIMEOUT        => 60,
    ]);

    $response = curl_exec($ch);
    $httpCode = curl_getinfo($ch, CURLINFO_HTTP_CODE);
    curl_close($ch);

    if ($httpCode !== 200) {
        error_log("Anthropic API erro HTTP {$httpCode}: {$response}");
        return null;
    }

    $data = json_decode($response, true);

    // A resposta vem em $data['content'] como array de blocos.
    // Para texto simples, pegue o primeiro bloco de tipo 'text':
    foreach ($data['content'] as $bloco) {
        if (($bloco['type'] ?? '') === 'text') {
            return $bloco['text'];
        }
    }
    return null;
}
```

### 4. Usar System Prompt para definir comportamento

System prompt molda o "papel" da IA e regras gerais:

```php
$body = json_encode([
    'model'      => 'claude-sonnet-4-20250514',
    'max_tokens' => 1024,
    'system'     => 'Você é um assistente que gera relatórios oficiais para policiais penais. ' .
                    'Use parágrafo único corrido, sem markdown. Termine sempre com "Fica o registro." ' .
                    'Nomes próprios em MAIÚSCULAS. Horários no formato HHhMM (ex: 15h08).',
    'messages'   => [
        ['role' => 'user', 'content' => 'Gere o relatório com base nos dados: ...'],
    ],
]);
```

### 5. Salvar prompts no banco (recomendado)

Em vez de hardcoded, mantenha os templates de prompt no banco para edição sem deploy:

```sql
CREATE TABLE prompts_ia (
    id INT PRIMARY KEY AUTO_INCREMENT,
    codigo VARCHAR(50) UNIQUE NOT NULL,    -- ex: 'g1_internacao'
    nome VARCHAR(100) NOT NULL,
    descricao TEXT,
    prompt_template LONGTEXT NOT NULL,     -- com placeholders {nome}, {hora}, etc.
    max_tokens INT DEFAULT 1024,
    modelo VARCHAR(50) DEFAULT 'claude-sonnet-4-20250514',
    ativo TINYINT(1) DEFAULT 1,
    atualizado_em TIMESTAMP DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP
);
```

E substitua placeholders antes de enviar:

```php
$template = PromptModel::buscarPorCodigo('g1_internacao')['prompt_template'];
$prompt = strtr($template, [
    '{nome}'        => $interno['nome_completo'],
    '{hora}'        => date('H\hi', strtotime($internacao['horario'])),
    '{unidade}'     => $unidade['sigla'],
    '{escoltantes}' => implode(', ', $escoltantes),
]);
```

---

## Escolha de modelo (custo vs qualidade)

| Modelo | Quando usar | Custo relativo |
|--------|-------------|----------------|
| `claude-haiku-4-5-20251001` | Tarefas simples, classificação, extração, sumarização curta | 💲 |
| `claude-sonnet-4-20250514` | Geração de texto formal, relatórios, raciocínio médio | 💲💲💲 |
| `claude-opus-4-7` (último Opus) | Tarefas que exigem raciocínio profundo, análise complexa | 💲💲💲💲💲 |

> 💡 Para a maioria das tarefas de geração de relatório, **Sonnet** já entrega qualidade excelente com custo razoável. Use Haiku para volume alto de tarefas curtas. Reserve Opus para casos onde a qualidade extra justifica o custo.

> ⚠️ Os preços e modelos mudam — sempre confira [a página oficial](https://www.anthropic.com/pricing) antes de assumir.

---

## Controle de custo

### Estimar tokens antes de enviar

Cada palavra ≈ 1.3 tokens em português. Um relatório de 200 palavras gasta ~260 tokens de output.

### Limite o `max_tokens` agressivamente

Se você sabe que o output cabe em 500 tokens, não peça 4096. Você não economiza no que não usa, mas evita respostas longas em casos de bug ou prompt mal escrito.

### Implemente cache local

Para prompts que dependem de dados que não mudam, cacheie a resposta:

```php
$hash = md5($prompt);
$cacheKey = "ia_cache_{$hash}";
if ($cached = CacheModel::get($cacheKey)) {
    return $cached;
}
$resposta = chamarClaude($prompt);
CacheModel::set($cacheKey, $resposta, 3600); // 1h
return $resposta;
```

### Use Prompt Caching (feature da Anthropic)

Para prompts longos que se repetem (ex: system prompt grande), use o cabeçalho `anthropic-beta: prompt-caching-2024-07-31` e marque blocos como `cache_control`. Pode reduzir custos em até 90% em prompts repetitivos. Veja [documentação](https://docs.claude.com/en/docs/build-with-claude/prompt-caching).

### Monitore consumo

No [console.anthropic.com](https://console.anthropic.com/) → **Usage** você vê:
- Tokens consumidos por dia/mês
- Custo acumulado
- Distribuição por modelo

Configure **billing alerts** para receber email quando passar de X dólares no mês.

---

## Troubleshooting

### Erro: `401 Unauthorized` / `invalid x-api-key`

**Causa:** API key errada, removida ou com permissão revogada.

**Solução:**
- Confirme que copiou a key inteira (começa com `sk-ant-api03-`).
- Verifique no console se a key ainda está ativa.

---

### Erro: `400 Bad Request` — `model: ... not found`

**Causa:** nome do modelo errado ou modelo descontinuado.

**Solução:**
- Use a string exata da [lista de modelos](https://docs.claude.com/en/docs/about-claude/models).
- Modelos antigos eventualmente são aposentados. Anthropic costuma avisar com antecedência.

---

### Erro: `429 Rate Limit`

**Causa:** muitos requests em pouco tempo (limites variam por plano).

**Solução:**
- Implemente retry com backoff exponencial (1s, 2s, 4s, 8s, 16s).
- Para volume alto, considere processar em fila assíncrona em vez de síncrono.

---

### Resposta truncada no meio

**Causa:** `max_tokens` baixo demais para a resposta gerada.

**Solução:**
- Aumente `max_tokens`.
- Verifique `$data['stop_reason']`. Se for `max_tokens`, foi cortada; se for `end_turn`, terminou naturalmente.

---

### Conteúdo bloqueado por safety

**Causa:** prompt ou contexto disparou um filtro de segurança (raro em uso legítimo).

**Solução:**
- Reformule o prompt sem ambiguidades (ex: "registro oficial de transferência de detento" em vez de termos vagos).
- Adicione contexto no system prompt explicando o papel profissional.

---

## Referências

- [Anthropic API Reference](https://docs.claude.com/en/api/messages)
- [Lista de modelos](https://docs.claude.com/en/docs/about-claude/models)
- [Pricing](https://www.anthropic.com/pricing)
- [Prompt Engineering — Best Practices](https://docs.claude.com/en/docs/build-with-claude/prompt-engineering/overview)
- [Prompt Caching](https://docs.claude.com/en/docs/build-with-claude/prompt-caching)

---

**Última revisão:** 2026-05-18
**Testado em:** PHP 8.2, API version `2023-06-01`, GSAU v2.8.2.0
