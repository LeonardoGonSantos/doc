# PCI DSS 4.0 – Hardening de Payments API (DB) | Docs (.NET + Angular)

Este repositório contém documentos práticos para apoiar o hardening de uma API de pagamentos com foco em **PCI DSS 4.0**, segurança de APIs e arquitetura de gateway/adquirência (Getnet/Stone/Itaú etc.).

## Como ler (ordem sugerida)
1. **Mapa de riscos e prioridades** → `pci-improvements-prioritized.md`  
2. **Implementação do fluxo crítico (cripto no front / decrypt no back)** → `pci-crypto-front-back.md`  
3. **Materiais auxiliares** → `example-distribution.md` (se aplicável ao seu contexto)

---

## Arquivos

### `pci-improvements-prioritized.md`
Plano de melhorias organizado por prioridade:
- **Críticos (P0)**: itens que tendem a bloquear PCI ou representam risco alto (ex.: PAN/CVV no backend, logs/raw, BOLA, Swagger exposto).
- **Não críticos (P1/P2)**: itens de robustez e maturidade (idempotência, `decimal`, verbos HTTP, padronização de erros, rate limit, observabilidade safe).

### `pci-crypto-front-back.md`
Exemplo completo do padrão com **2 endpoints**:
- `GET /crypto/public-key` (backend fornece **chave pública** JWK + `kid`)
- `POST /payments` (front envia **PAN criptografado** e o backend **descriptografa** para chamar o adquirente)
Inclui:
- **Mermaid (arquitetura)**
- Código **.NET (C# / ASP.NET Core)**
- Código **Angular** (WebCrypto)

### `example-distribution.md`
Documento auxiliar com exemplos e/ou notas adicionais (uso interno do time).  
> Se não fizer sentido no seu fluxo, pode ser removido sem impactar os demais.

---

## Observações rápidas (PCI)
- **Nunca** armazenar ou logar **CVV**.
- Evitar que **PAN** apareça em logs, APM, traces e payloads “raw”.
- Criptografia no front ajuda na confidencialidade, mas **não reduz escopo PCI** se o backend ainda descriptografa e manipula PAN.
- Para reduzir escopo de verdade, considerar **tokenização** (Vault/PSP) e detokenização apenas em componente isolado (CDE mínimo).
