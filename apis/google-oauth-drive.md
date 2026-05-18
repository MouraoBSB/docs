# Google OAuth 2.0 + Drive API (sem libs externas)

> Como integrar uma aplicação PHP com o Google Drive usando OAuth 2.0 (refresh token), fazendo upload de arquivos com apenas `cURL` e `openssl_*`. Útil para backups automáticos.

**Contexto:** Aplicação PHP em hospedagem compartilhada onde instalar a Google API Client Library (`composer require google/apiclient`) traria muitas dependências pesadas. Validado no GSAU para backup diário e semanal.

---

## Pré-requisitos

- [ ] Conta Google
- [ ] Acesso ao [Google Cloud Console](https://console.cloud.google.com/)
- [ ] PHP 8.0+ com extensões `curl` e `openssl`
- [ ] HTTPS no domínio da aplicação (obrigatório para redirect URI em produção)

---

## Passo a passo

### 1. Criar projeto no Google Cloud

1. Acesse [console.cloud.google.com](https://console.cloud.google.com/).
2. **Select a project** → **New Project** → nomeie (ex: "MeuApp Backups") → **Create**.
3. Aguarde o provisionamento (alguns segundos).

### 2. Habilitar a Drive API

1. Menu lateral → **APIs & Services** → **Library**.
2. Busque por "Google Drive API".
3. Clique em **Enable**.

### 3. Configurar OAuth Consent Screen

1. Menu lateral → **APIs & Services** → **OAuth consent screen**.
2. User Type: **External** (mesmo para uso pessoal) → **Create**.
3. Preencha:
   - **App name:** MeuApp Backups
   - **User support email:** seu email
   - **Developer contact information:** seu email
4. **Save and Continue**.
5. **Scopes** → **Add or Remove Scopes** → selecione:
   - `.../auth/drive.file` (cria/edita só arquivos que o app gera — recomendado)
   - OU `.../auth/drive` (acesso total — use só se realmente precisar)
6. **Save and Continue**.
7. **Test users** → adicione seu próprio email Google → **Save**.

> 💡 Enquanto estiver em **Testing**, o refresh token expira em **7 dias**. Para uso real, publique o app (não precisa de verificação se for `.../auth/drive.file`):
> **OAuth consent screen** → **Publish App** → **Confirm**.

### 4. Criar credenciais OAuth

1. **APIs & Services** → **Credentials** → **Create Credentials** → **OAuth client ID**.
2. **Application type:** Web application.
3. **Name:** MeuApp Web Client.
4. **Authorized redirect URIs:** adicione (com HTTPS em produção):
   - `https://app.seudominio.com/admin/google-oauth/callback`
5. **Create**.
6. Anote **Client ID** e **Client Secret**.

### 5. Obter Refresh Token (autorização inicial)

Esse passo é manual — você faz UMA vez, captura o refresh token e armazena no banco.

**Passo 5.1 — Construir URL de autorização:**

```php
$params = http_build_query([
    'client_id'     => '{SEU_CLIENT_ID}',
    'redirect_uri'  => 'https://app.seudominio.com/admin/google-oauth/callback',
    'response_type' => 'code',
    'scope'         => 'https://www.googleapis.com/auth/drive.file',
    'access_type'   => 'offline',    // <-- ESSENCIAL para receber refresh_token
    'prompt'        => 'consent',    // <-- ESSENCIAL para forçar emissão (se já autorizou antes)
]);

$urlAutorizacao = "https://accounts.google.com/o/oauth2/v2/auth?{$params}";
```

Redirecione o usuário (ou clique no link manualmente).

**Passo 5.2 — Receber `code` no callback e trocar por tokens:**

```php
// No controller do callback
$code = $_GET['code'] ?? '';

$ch = curl_init('https://oauth2.googleapis.com/token');
curl_setopt_array($ch, [
    CURLOPT_POST           => true,
    CURLOPT_POSTFIELDS     => http_build_query([
        'code'          => $code,
        'client_id'     => '{SEU_CLIENT_ID}',
        'client_secret' => '{SEU_CLIENT_SECRET}',
        'redirect_uri'  => 'https://app.seudominio.com/admin/google-oauth/callback',
        'grant_type'    => 'authorization_code',
    ]),
    CURLOPT_RETURNTRANSFER => true,
    CURLOPT_HTTPHEADER     => ['Content-Type: application/x-www-form-urlencoded'],
]);
$response = curl_exec($ch);
$data = json_decode($response, true);
curl_close($ch);

// $data contém:
// - access_token (válido ~1h)
// - refresh_token (válido enquanto não revogado)
// - expires_in (segundos até expirar o access_token)
// - token_type ("Bearer")

// SALVE O refresh_token NO BANCO (criptografado de preferência).
```

> ⚠️ `refresh_token` só vem na **primeira autorização** ou quando você passa `prompt=consent`. Sem isso, autorizações subsequentes retornam só `access_token`.

### 6. Renovar `access_token` quando precisar

```php
function obterAccessToken(string $refreshToken, string $clientId, string $clientSecret): ?string {
    $ch = curl_init('https://oauth2.googleapis.com/token');
    curl_setopt_array($ch, [
        CURLOPT_POST           => true,
        CURLOPT_POSTFIELDS     => http_build_query([
            'refresh_token' => $refreshToken,
            'client_id'     => $clientId,
            'client_secret' => $clientSecret,
            'grant_type'    => 'refresh_token',
        ]),
        CURLOPT_RETURNTRANSFER => true,
    ]);
    $response = curl_exec($ch);
    $data = json_decode($response, true);
    curl_close($ch);

    return $data['access_token'] ?? null;
}
```

### 7. Upload de arquivo (multipart)

```php
function uploadDrive(string $accessToken, string $caminhoArquivo, string $nome, string $mime, string $folderId): array {
    $metadata = json_encode([
        'name'    => $nome,
        'parents' => [$folderId],
    ]);

    $arquivoConteudo = file_get_contents($caminhoArquivo);
    $boundary = '----GsauBoundary' . bin2hex(random_bytes(8));

    $body  = "--{$boundary}\r\n";
    $body .= "Content-Type: application/json; charset=UTF-8\r\n\r\n";
    $body .= $metadata . "\r\n";
    $body .= "--{$boundary}\r\n";
    $body .= "Content-Type: {$mime}\r\n\r\n";
    $body .= $arquivoConteudo . "\r\n";
    $body .= "--{$boundary}--";

    $ch = curl_init('https://www.googleapis.com/upload/drive/v3/files?uploadType=multipart');
    curl_setopt_array($ch, [
        CURLOPT_POST           => true,
        CURLOPT_POSTFIELDS     => $body,
        CURLOPT_HTTPHEADER     => [
            "Authorization: Bearer {$accessToken}",
            "Content-Type: multipart/related; boundary={$boundary}",
            "Content-Length: " . strlen($body),
        ],
        CURLOPT_RETURNTRANSFER => true,
    ]);
    $response = curl_exec($ch);
    curl_close($ch);

    return json_decode($response, true);
}
```

### 8. Criar/garantir pasta no Drive

```php
function ensureFolder(string $accessToken, string $nomePasta, ?string $folderIdAtual = null): string {
    // Se o folderId atual ainda existe, retorna ele.
    if ($folderIdAtual) {
        $ch = curl_init("https://www.googleapis.com/drive/v3/files/{$folderIdAtual}?fields=id");
        curl_setopt_array($ch, [
            CURLOPT_HTTPHEADER     => ["Authorization: Bearer {$accessToken}"],
            CURLOPT_RETURNTRANSFER => true,
        ]);
        $response = curl_exec($ch);
        $info = curl_getinfo($ch, CURLINFO_HTTP_CODE);
        curl_close($ch);
        if ($info === 200) return $folderIdAtual;
    }

    // Senão, cria uma nova.
    $body = json_encode([
        'name'     => $nomePasta,
        'mimeType' => 'application/vnd.google-apps.folder',
    ]);
    $ch = curl_init('https://www.googleapis.com/drive/v3/files');
    curl_setopt_array($ch, [
        CURLOPT_POST           => true,
        CURLOPT_POSTFIELDS     => $body,
        CURLOPT_HTTPHEADER     => [
            "Authorization: Bearer {$accessToken}",
            "Content-Type: application/json",
        ],
        CURLOPT_RETURNTRANSFER => true,
    ]);
    $response = curl_exec($ch);
    curl_close($ch);

    $data = json_decode($response, true);
    return $data['id'];
}
```

---

## Troubleshooting

### Erro: `invalid_grant` ao usar refresh_token

**Causa:** refresh token expirou (app em modo Testing por mais de 7 dias) ou foi revogado.

**Solução:**
- **Publique o app** em **OAuth consent screen** (não precisa de verificação para escopo `drive.file`).
- Refaça o fluxo de autorização (passo 5) para obter novo refresh token.

---

### Erro: `redirect_uri_mismatch`

**Causa:** o `redirect_uri` enviado não está exatamente na lista do Google Cloud.

**Solução:**
- Confira **protocolo** (http vs https), **porta**, **domínio**, **path** e **barra final**.
- `https://app.com/callback` ≠ `https://app.com/callback/`.

---

### Upload retorna 401 ou 403

**Causa:** `access_token` expirado ou escopo insuficiente.

**Solução:**
- Renove o access_token (passo 6) — eles duram só ~1 hora.
- Se usar `drive.file`, lembre que o app só vê arquivos que ele próprio criou. Para acessar pastas criadas manualmente, use escopo `drive` (mais amplo).

---

### `Quota exceeded` ou rate limit

**Causa:** muitas requisições em curto espaço de tempo.

**Solução:**
- Implemente backoff exponencial (espere 1s, 2s, 4s, 8s...).
- Limites padrão Drive API: 1000 requests/100s/usuário. Geralmente sobra.

---

## Referências

- [Google Drive API v3 — Reference](https://developers.google.com/drive/api/v3/reference)
- [OAuth 2.0 for Web Server Applications](https://developers.google.com/identity/protocols/oauth2/web-server)
- [Drive API — Uploading Files](https://developers.google.com/drive/api/v3/manage-uploads)

---

**Última revisão:** 2026-05-18
**Testado em:** PHP 8.2, Drive API v3, GSAU v2.8.2.0 (Fase 22 — módulo de backup)
