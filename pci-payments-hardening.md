# PCI DSS 4.0 – Hardening da API de Pagamentos (DB) + Tokenização com HashiCorp Vault (OSS) + Exemplos (.NET + Angular)

> **Objetivo:** este documento é um “playbook” prático para colocar no GitHub. Ele resume os riscos típicos encontrados no Swagger da API de pagamentos do DB e entrega um **mapa de melhorias**, com **exemplos de código em .NET** e **front-end em Angular**, além de **cURLs** e **diagramas Mermaid**.
>
> **Escopo:** foco em **reduzir exposição de PAN/CVV**, reforçar **segurança de API**, e “apertar” controles para **passar na certificação PCI DSS 4.0**.

---

## 1) Por que o PCI DSS costuma dar errado nesses ambientes

Em gateways “in-house”, a certificação geralmente falha por 4 motivos:

1. **SAD (CVV/CVC) sendo armazenado ou logado**  
   PCI é rígido: **CVV não pode ser armazenado após autorização** (nem criptografado). Se o sistema tem campos, logs, request/response raw, ou persistência com CVV, **é não conformidade imediata**.

2. **PAN (número do cartão) trafegando e ficando espalhado**  
   Se a API recebe PAN em claro e/ou grava o payload, você aumenta o **escopo PCI** e o risco de vazamento.  
   O correto é: **minimizar o manuseio de PAN**, **tokenizar cedo**, e **nunca logar PAN completo**.

3. **Ausência de segmentação/segregação do CDE**  
   Sem segmentação, “tudo entra no escopo” e a auditoria fica muito mais cara e difícil.

4. **APIs sem controles essenciais de pagamento**  
   - Sem **Idempotency-Key** → cobrança duplicada.  
   - Sem **BOLA protections** (merchant scope) → vazamento/abuso (consultar/cancelar transações alheias).  
   - Swagger público em produção → amplia superfície de ataque.

---

## 2) “Mapa de melhorias” (priorizado) – o que corrigir primeiro

### P0 – bloqueia certificação / risco crítico
- **Remover CVV de qualquer armazenamento/log** (inclusive “raw request/response”).
- **Sanitizar logs e respostas**: PAN mascarado (primeiros 6 e últimos 4) ou apenas **last4**.
- **Desativar Swagger público em produção** (ou proteger por Auth/IP allowlist).
- **Tokenizar PAN** (Vault) e armazenar **token/ciphertext** no banco.
- **Controle de autorização por merchant/tenant** em consultas/cancelamentos (BOLA).

### P1 – robustez e antifraude operacional
- **Idempotência** em endpoints de criação/charge/capture.
- **Amount como `decimal`** (não `double`).
- **Normalizar HTTP verbs** (GET sem side effects; capture/cancel → POST).
- **Rate-limit + WAF rules** para endpoints sensíveis.

### P2 – maturidade/observabilidade
- Auditoria segura (req id, correlation id), sem dados sensíveis.
- Rotação de segredos e políticas de chaves do Vault.
- Alertas e detecção de anomalias.

---

## 3) Arquiteturas recomendadas (Mermaid)

### 3.1 Tokenização no backend (mais simples para começar)
```mermaid
flowchart LR
  A[Front-end Angular] -->|POST /payments| B[API .NET]
  B -->|encrypt/tokenize PAN| V[HashiCorp Vault - Transit (OSS)]
  V -->|ciphertext token| B
  B -->|store token + last4| DB[(DB)]
  B -->|detokenize just-in-time| V
  B -->|PAN+CVV -> adquirente| G[Getnet/Stone/Itaú]
```

### 3.2 “Client-side tokenization” (recomendado: via BFF proxy, NÃO exponha Vault direto)
> Não é recomendado expor o Vault direto ao browser. O padrão seguro é **BFF (Backend For Frontend)** que entrega um token de curto prazo (encode-only) e/ou proxy controlado.

```mermaid
flowchart LR
  A[Angular] -->|GET /vault/encode-token| BFF[Tokenization BFF (.NET)]
  BFF -->|AppRole login| V[Vault Transit]
  BFF -->|short-lived token (encode-only)| A
  A -->|POST /vault/encrypt (proxy)| BFF
  BFF -->|transit/encrypt| V
  V -->|ciphertext| BFF -->|token| A
  A -->|POST /payments {cardToken,...}| API[Payments API .NET]
  API -->|detokenize & send| V --> API --> Getnet[Adquirente]
```

---

## 4) HashiCorp Vault (Open Source) – Setup para “tokenização” (Transit)

> **Importante:** o Vault OSS não tem o **Transform Engine (FPE)** (tokenização com preservação de formato).  
> No OSS, o caminho mais comum é usar **Transit** e tratar o **ciphertext** como token.  
> Se você precisa de token “tipo cartão” (FPE), aí é Vault Enterprise/Transform ou outra solução FPE.

### 4.1 Habilitar Transit e criar chave
```bash
# habilitar transit (uma vez)
vault secrets enable transit

# criar chave para PAN (uma vez)
vault write -f transit/keys/pan
```

### 4.2 Policy (HCL) – frontend (encode-only) vs backend (decode)
**policy-encode.hcl**
```hcl
path "transit/encrypt/pan" {
  capabilities = ["update"]
}
```

**policy-decode.hcl**
```hcl
path "transit/decrypt/pan" {
  capabilities = ["update"]
}
```

Aplicar policies:
```bash
vault policy write payments-encode policy-encode.hcl
vault policy write payments-decode policy-decode.hcl
```

### 4.3 AppRole (recomendado) para apps
```bash
vault auth enable approle

vault write auth/approle/role/payments-encode \
  token_policies="payments-encode" \
  token_ttl="10m" token_max_ttl="30m"

vault write auth/approle/role/payments-decode \
  token_policies="payments-decode" \
  token_ttl="10m" token_max_ttl="30m"
```

Obter RoleID / SecretID (para provisionar na app via secret manager):
```bash
vault read auth/approle/role/payments-encode/role-id
vault write -f auth/approle/role/payments-encode/secret-id
```

### 4.4 cURL – login AppRole, encrypt e decrypt (Transit)
**Login AppRole**:
```bash
curl -sS -X POST "$VAULT_ADDR/v1/auth/approle/login" \
  -H "Content-Type: application/json" \
  -d '{"role_id":"<ROLE_ID>","secret_id":"<SECRET_ID>"}'
```

**Encrypt (PAN → ciphertext token)**:
```bash
PAN="4111111111111111"
PLAINTEXT_B64=$(printf "%s" "$PAN" | base64)

curl -sS -X POST "$VAULT_ADDR/v1/transit/encrypt/pan" \
  -H "X-Vault-Token: <VAULT_TOKEN>" \
  -H "Content-Type: application/json" \
  -d "{\"plaintext\":\"$PLAINTEXT_B64\"}"
```

Resposta típica:
```json
{"data":{"ciphertext":"vault:v1:...."}}
```

**Decrypt (ciphertext → PAN)**:
```bash
curl -sS -X POST "$VAULT_ADDR/v1/transit/decrypt/pan" \
  -H "X-Vault-Token: <VAULT_TOKEN>" \
  -H "Content-Type: application/json" \
  -d '{"ciphertext":"vault:v1:...."}'
```

Você receberá:
```json
{"data":{"plaintext":"NDAxMTExMTExMTExMTExMTExMTExMTEx"}}
```

Decodificando:
```bash
echo "NDAxMTExMTExMTExMTExMTExMTExMTEx" | base64 --decode
```

---

## 5) Modelo de API (contratos) – como deveria ficar

### 5.1 Request do Front-end → Backend (sem PAN)
```json
{
  "amount": 150.00,
  "currency": "BRL",
  "cardToken": "vault:v1:....", 
  "cardLast4": "1111",
  "cardExpMonth": 12,
  "cardExpYear": 2028,
  "cvv": "123",
  "merchantId": "MERCHANT_ABC",
  "idempotencyKey": "f1a9b2d2-0c6d-4e87-9b56-7f7d6a9e08df"
}
```

**Regras PCI:**
- `cvv` **nunca** deve ser persistido nem logado.
- `cardToken` (ciphertext) pode ser armazenado.
- `cardLast4` pode ser armazenado (não é PAN completo).

---

## 6) .NET (ASP.NET Core) – Exemplos práticos

### 6.1 DTOs com `decimal`
```csharp
public sealed record CreatePaymentRequest(
    decimal Amount,
    string Currency,
    string CardToken,
    string CardLast4,
    int CardExpMonth,
    int CardExpYear,
    string Cvv,                 // usar e descartar
    string MerchantId,
    string IdempotencyKey
);

public sealed record CreatePaymentResponse(
    string PaymentId,
    string Status
);
```

### 6.2 Cliente Vault Transit (.NET) – Encrypt/Decrypt
```csharp
using System.Net.Http.Json;
using System.Text;
using System.Text.Json;

public sealed class VaultTransitClient
{
    private readonly HttpClient _http;
    private readonly string _vaultAddr;
    private readonly Func<Task<string>> _tokenProvider; // pega token (AppRole) ou de cache

    public VaultTransitClient(HttpClient http, string vaultAddr, Func<Task<string>> tokenProvider)
    {
        _http = http;
        _vaultAddr = vaultAddr.TrimEnd('/');
        _tokenProvider = tokenProvider;
    }

    public async Task<string> EncryptPanAsync(string pan)
    {
        var token = await _tokenProvider();
        var b64 = Convert.ToBase64String(Encoding.UTF8.GetBytes(pan));

        using var req = new HttpRequestMessage(HttpMethod.Post, $"{_vaultAddr}/v1/transit/encrypt/pan");
        req.Headers.Add("X-Vault-Token", token);
        req.Content = JsonContent.Create(new { plaintext = b64 });

        using var resp = await _http.SendAsync(req);
        resp.EnsureSuccessStatusCode();

        var doc = await resp.Content.ReadFromJsonAsync<JsonDocument>();
        return doc!.RootElement.GetProperty("data").GetProperty("ciphertext").GetString()!;
    }

    public async Task<string> DecryptPanAsync(string ciphertext)
    {
        var token = await _tokenProvider();

        using var req = new HttpRequestMessage(HttpMethod.Post, $"{_vaultAddr}/v1/transit/decrypt/pan");
        req.Headers.Add("X-Vault-Token", token);
        req.Content = JsonContent.Create(new { ciphertext });

        using var resp = await _http.SendAsync(req);
        resp.EnsureSuccessStatusCode();

        var doc = await resp.Content.ReadFromJsonAsync<JsonDocument>();
        var b64 = doc!.RootElement.GetProperty("data").GetProperty("plaintext").GetString()!;
        var bytes = Convert.FromBase64String(b64);
        return Encoding.UTF8.GetString(bytes);
    }
}
```

### 6.3 Idempotência (middleware + storage)
> A ideia: `Idempotency-Key` (ou `idempotencyKey`) identifica uma intenção única.

**Interface de storage (Redis/DB)**
```csharp
public interface IIdempotencyStore
{
    Task<(bool Found, string ResponseJson, int StatusCode)> TryGetAsync(string key, string merchantId);
    Task SaveAsync(string key, string merchantId, string responseJson, int statusCode, TimeSpan ttl);
}
```

**Exemplo de uso no Controller**
```csharp
[ApiController]
[Route("payments")]
public sealed class PaymentsController : ControllerBase
{
    private readonly IIdempotencyStore _idem;
    private readonly VaultTransitClient _vault;
    private readonly IAcquirerClient _acquirer;
    private readonly IPaymentsRepository _repo;

    public PaymentsController(IIdempotencyStore idem, VaultTransitClient vault, IAcquirerClient acquirer, IPaymentsRepository repo)
    {
        _idem = idem; _vault = vault; _acquirer = acquirer; _repo = repo;
    }

    [HttpPost]
    public async Task<IActionResult> Create([FromBody] CreatePaymentRequest req)
    {
        if (string.IsNullOrWhiteSpace(req.IdempotencyKey))
            return BadRequest(new { error = "idempotencyKey obrigatório" });

        var cached = await _idem.TryGetAsync(req.IdempotencyKey, req.MerchantId);
        if (cached.Found)
            return StatusCode(cached.StatusCode, JsonDocument.Parse(cached.ResponseJson).RootElement);

        // Decrypt só no momento de enviar ao adquirente
        var pan = await _vault.DecryptPanAsync(req.CardToken);

        var acqResult = await _acquirer.AuthorizeAsync(new AuthorizeCardPayment(
            Amount: req.Amount,
            Currency: req.Currency,
            Pan: pan,
            ExpMonth: req.CardExpMonth,
            ExpYear: req.CardExpYear,
            Cvv: req.Cvv,
            MerchantId: req.MerchantId
        ));

        // NUNCA persistir CVV; PAN só em memória
        pan = string.Empty;

        var paymentId = await _repo.SaveAsync(new PaymentEntity
        {
            MerchantId = req.MerchantId,
            Amount = req.Amount,
            Currency = req.Currency,
            CardToken = req.CardToken,
            CardLast4 = req.CardLast4,
            Status = acqResult.Approved ? "APPROVED" : "DECLINED",
            AcquirerTid = acqResult.Tid
        });

        var responseObj = new CreatePaymentResponse(paymentId, acqResult.Approved ? "APPROVED" : "DECLINED");
        var responseJson = JsonSerializer.Serialize(responseObj);

        await _idem.SaveAsync(req.IdempotencyKey, req.MerchantId, responseJson, 200, TimeSpan.FromHours(24));

        return Ok(responseObj);
    }
}
```

### 6.4 BOLA (autorização por objeto) – consulta/cancelamento
```csharp
[HttpGet("{paymentId}")]
public async Task<IActionResult> Get(string paymentId)
{
    var merchantId = User.FindFirst("merchant_id")?.Value;
    if (merchantId is null) return Unauthorized();

    var p = await _repo.GetAsync(paymentId);
    if (p is null) return NotFound();

    if (p.MerchantId != merchantId) return Forbid();

    return Ok(new {
        p.PaymentId, p.Status, p.Amount, p.Currency,
        CardLast4 = p.CardLast4
    });
}
```

### 6.5 Logging seguro (redação de PAN)
```csharp
public static class SensitiveDataRedactor
{
    private static readonly System.Text.RegularExpressions.Regex PanRegex =
        new(@"\b\d{12,19}\b", System.Text.RegularExpressions.RegexOptions.Compiled);

    public static string Redact(string input)
    {
        if (string.IsNullOrEmpty(input)) return input;
        return PanRegex.Replace(input, m =>
        {
            var v = m.Value;
            var first6 = v.Length >= 6 ? v[..6] : "******";
            var last4 = v.Length >= 4 ? v[^4..] : "****";
            var middle = v.Length > 10 ? new string('*', v.Length - 10) : "**";
            return first6 + middle + last4;
        });
    }
}
```

---

## 7) Integração com adquirente (Getnet) – padrão Adapter

### 7.1 Contratos internos
```csharp
public sealed record AuthorizeCardPayment(
    decimal Amount,
    string Currency,
    string Pan,
    int ExpMonth,
    int ExpYear,
    string Cvv,
    string MerchantId
);

public sealed record AcquirerAuthResult(
    bool Approved,
    string Tid,
    string RawCode,
    string Message
);

public interface IAcquirerClient
{
    Task<AcquirerAuthResult> AuthorizeAsync(AuthorizeCardPayment req);
}
```

### 7.2 Client HTTP (esqueleto) – Getnet
```csharp
public sealed class GetnetClient : IAcquirerClient
{
    private readonly HttpClient _http;
    private readonly GetnetAuthProvider _auth;

    public GetnetClient(HttpClient http, GetnetAuthProvider auth)
    {
        _http = http;
        _auth = auth;
    }

    public async Task<AcquirerAuthResult> AuthorizeAsync(AuthorizeCardPayment req)
    {
        var accessToken = await _auth.GetAccessTokenAsync();

        using var httpReq = new HttpRequestMessage(HttpMethod.Post, "/v1/payments/credit");
        httpReq.Headers.Authorization =
            new System.Net.Http.Headers.AuthenticationHeaderValue("Bearer", accessToken);

        var payload = new
        {
            amount = req.Amount,
            currency = req.Currency,
            card = new
            {
                number = req.Pan,
                exp_month = req.ExpMonth,
                exp_year = req.ExpYear,
                cvv = req.Cvv
            },
            merchant_id = req.MerchantId
        };

        httpReq.Content = JsonContent.Create(payload);

        using var resp = await _http.SendAsync(httpReq);
        var json = await resp.Content.ReadAsStringAsync();

        // Nunca logar o payload do cartão. Se logar resposta do adquirente, sanitize antes.
        if (!resp.IsSuccessStatusCode)
            return new AcquirerAuthResult(false, "", resp.StatusCode.ToString(), "Erro no adquirente");

        return new AcquirerAuthResult(true, "TID_EXEMPLO", "00", "Aprovado");
    }
}
```

---

## 8) Angular – “SDK” (client Http) para tokenização via BFF + Checkout

### 8.1 Angular service
```ts
import { HttpClient } from '@angular/common/http';
import { Injectable } from '@angular/core';
import { map, Observable } from 'rxjs';

export interface TokenizeResponse {
  cardToken: string;
  last4: string;
}

@Injectable({ providedIn: 'root' })
export class VaultTokenizationService {
  constructor(private http: HttpClient) {}

  tokenizePan(pan: string): Observable<TokenizeResponse> {
    return this.http.post<any>('/api/tokenization/pan', { pan }).pipe(
      map(r => ({ cardToken: r.cardToken, last4: r.last4 }))
    );
  }
}
```

### 8.2 Component
```ts
import { Component } from '@angular/core';
import { HttpClient } from '@angular/common/http';
import { VaultTokenizationService } from './vault-tokenization.service';
import { switchMap } from 'rxjs';

@Component({
  selector: 'app-checkout',
  template: `<button (click)="pay()">Pagar</button>`
})
export class CheckoutComponent {
  pan = '4111111111111111';
  expMonth = 12;
  expYear = 2028;
  cvv = '123';
  amount = 150.00;

  constructor(private vaultTok: VaultTokenizationService, private http: HttpClient) {}

  pay() {
    const idempotencyKey = crypto.randomUUID();

    this.vaultTok.tokenizePan(this.pan).pipe(
      switchMap(tok => this.http.post('/api/payments', {
        amount: this.amount,
        currency: 'BRL',
        cardToken: tok.cardToken,
        cardLast4: tok.last4,
        cardExpMonth: this.expMonth,
        cardExpYear: this.expYear,
        cvv: this.cvv,
        merchantId: 'MERCHANT_ABC',
        idempotencyKey
      }))
    ).subscribe(console.log);
  }
}
```

---

## 9) Swagger hardening (.NET)
```csharp
if (app.Environment.IsDevelopment())
{
    app.UseSwagger();
    app.UseSwaggerUI();
}
```

---

## 10) Checklist rápido
- [ ] CVV nunca persistido/logado.
- [ ] PAN nunca logado; armazenar apenas token + last4.
- [ ] Vault Transit em rede interna, com AppRole e policies mínimas.
- [ ] Detokenizar só “just-in-time” antes de chamar adquirente.
- [ ] Idempotency-Key obrigatório em POST de pagamento.
- [ ] BOLA: validar merchantId em toda consulta/cancelamento.
- [ ] Amount `decimal`.

---

## 11) Observações finais
- No Vault OSS, o “token” mais comum é o **ciphertext** do Transit.
- Se você precisa de **token determinístico** (mesmo cartão → mesmo token), avalie:
  - convergent/deterministic encryption do Transit (avaliar) ou
  - tabela de mapeamento controlada (com hashing/fingerprint) ou
  - Vault Transform (Enterprise) / solução FPE.

