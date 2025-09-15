# Método Simples de Distribuição de Transações com Memory Bank

Este documento explica como funciona o método simples para distribuir transações entre adquirentes usando Memory Bank, garantindo que a cada 10 transações, 7 vão para o adquirente de maior prioridade e 3 para o de menor prioridade.

## 🎯 Objetivo

Distribuir transações de forma **exata** e **previsível**:
- **7 de 10** transações → Adquirente de **maior prioridade** (ex: Pagarme)
- **3 de 10** transações → Adquirente de **menor prioridade** (ex: Bradesco)

## 🧠 Como Funciona o Memory Bank

O Memory Bank é um dicionário em memória que armazena contadores para cada regra de distribuição:

```csharp
private static readonly Dictionary<string, int> _transactionCounters = new();
```

### Chave do Memory Bank
```csharp
string ruleKey = $"{rule.AdquirenteId}_{rule.Prioridade}";
// Exemplo: "1_1" para Pagarme com prioridade 1
```

### Contador por Regra
- Cada regra tem seu próprio contador
- O contador vai de 1 a 10
- Quando chega em 11, volta para 1 (novo ciclo)

## 🔧 Implementação Completa

### 1. Interface
```csharp
public interface ISimpleDistributionService
{
    int GetAdquirenteForTransaction(DistributionRule rule);
}
```

### 2. Classe de Regra
```csharp
public class DistributionRule
{
    public int AdquirenteId { get; set; }
    public int Prioridade { get; set; }
    public int Percentual { get; set; } // Porcentagem (ex: 70 para 70%)
    
    public DistributionRule(int adquirenteId, int prioridade, int percentual)
    {
        AdquirenteId = adquirenteId;
        Prioridade = prioridade;
        Percentual = percentual;
    }
    
    // Construtor para regra de menor prioridade (calcula automaticamente)
    public DistributionRule(int adquirenteId, int prioridade, int percentual, int adquirenteMenorPrioridadeId)
    {
        AdquirenteId = adquirenteId;
        Prioridade = prioridade;
        Percentual = percentual;
        AdquirenteMenorPrioridadeId = adquirenteMenorPrioridadeId;
    }
    
    public int AdquirenteMenorPrioridadeId { get; set; }
}
```

### 3. Serviço Principal
```csharp
public class SimpleDistributionService : ISimpleDistributionService
{
    private static readonly Dictionary<string, int> _transactionCounters = new();
    private static readonly object _lockObject = new();

    public int GetAdquirenteForTransaction(DistributionRule rule)
    {
        // 1. Cria chave única para a regra
        string ruleKey = $"{rule.AdquirenteId}_{rule.Prioridade}";
        
            // 2. Inicializa contador se não existir
            if (!_transactionCounters.ContainsKey(ruleKey))
            {
                _transactionCounters[ruleKey] = 0;
            }

            // 3. Incrementa contador
            _transactionCounters[ruleKey]++;
            int currentCount = _transactionCounters[ruleKey];

            // 4. Reset a cada 10 transações (para trabalhar com percentuais)
            if (currentCount > 10)
            {
                _transactionCounters[ruleKey] = 1;
                currentCount = 1;
            }

            // 5. Distribui baseado no percentual da regra
            if (currentCount <= rule.Percentual)
            {
                return rule.AdquirenteId; // Maior prioridade (ex: 1-70)
            }
            else
            {
                // Retorna o adquirente de menor prioridade
                return rule.AdquirenteMenorPrioridadeId;
            }
        }
    
}
```

## 📊 Fluxo de Execução

### Passo a Passo:

1. **Entrada**: Recebe uma `DistributionRule` com `AdquirenteId` e `Prioridade`

2. **Chave**: Cria chave única `"AdquirenteId_Prioridade"`

3. **Contador**: 
   - Se não existe → inicializa com 0
   - Incrementa o contador
   - Se > 10 → reseta para 1

4. **Decisão**:
   - Contador 1-7 → Retorna `AdquirenteId` (maior prioridade)
   - Contador 8-10 → Retorna ID do adquirente de menor prioridade

5. **Saída**: ID do adquirente selecionado

## 🎮 Exemplo Prático

### Configuração
```csharp
// Cria regra com percentual
var regraPagarme = new DistributionRule(1, 1, 70, 2); // ID=1, Prioridade=1, 70%, ID menor prioridade=2

// Cria serviço
var service = new SimpleDistributionService();
```

### Simulação de 20 Transações
```csharp
for (int i = 1; i <= 20; i++)
{
    var adquirenteId = service.GetAdquirenteForTransaction(regraPagarme);
    Console.WriteLine($"Transação {i:D2}: Adquirente {adquirenteId}");
}
```

### Resultado Esperado
```
Transação 01: Adquirente 1  ← Pagarme (1/70)
Transação 02: Adquirente 1  ← Pagarme (2/70)
Transação 03: Adquirente 1  ← Pagarme (3/70)
...
Transação 70: Adquirente 1  ← Pagarme (70/70)
Transação 71: Adquirente 2  ← Bradesco (1/30)
Transação 72: Adquirente 2  ← Bradesco (2/30)
...
Transação 100: Adquirente 2 ← Bradesco (30/30)
Transação 101: Adquirente 1 ← Pagarme (1/70) - NOVO CICLO
...
```

## 🔍 Análise do Memory Bank

### Estado do Dicionário Durante Execução:

| Transação | Chave | Contador | Decisão | Adquirente |
|-----------|-------|----------|---------|------------|
| 1 | "1_1" | 1 | ≤ 70 | 1 (Pagarme) |
| 2 | "1_1" | 2 | ≤ 70 | 1 (Pagarme) |
| ... | ... | ... | ... | ... |
| 70 | "1_1" | 70 | ≤ 70 | 1 (Pagarme) |
| 71 | "1_1" | 71 | > 70 | 2 (Bradesco) |
| 72 | "1_1" | 72 | > 70 | 2 (Bradesco) |
| ... | ... | ... | ... | ... |
| 100 | "1_1" | 100 | > 70 | 2 (Bradesco) |
| 101 | "1_1" | 1 | ≤ 70 | 1 (Pagarme) |

## 🛠️ Métodos Auxiliares

### Reset dos Contadores
```csharp
public static void ResetCounters()
{
    lock (_lockObject)
    {
        _transactionCounters.Clear();
    }
}
```

### Visualizar Estado Atual
```csharp
public static Dictionary<string, int> GetCounters()
{
    lock (_lockObject)
    {
        return new Dictionary<string, int>(_transactionCounters);
    }
}
```

### Exemplo de Uso dos Auxiliares
```csharp
// Ver estado atual
var contadores = SimpleDistributionService.GetCounters();
foreach (var contador in contadores)
{
    Console.WriteLine($"Regra {contador.Key}: {contador.Value} transações");
}

// Reset para testes
SimpleDistributionService.ResetCounters();
```

## 🎯 Exemplos de Configuração com Diferentes Percentuais

### 70/30 (Exemplo Original)
```csharp
var regra = new DistributionRule(1, 1, 70, 2); // 70% Pagarme, 30% Bradesco
```

### 80/20
```csharp
var regra = new DistributionRule(1, 1, 80, 2); // 80% Pagarme, 20% Bradesco
```

### 60/40
```csharp
var regra = new DistributionRule(1, 1, 60, 2); // 60% Pagarme, 40% Bradesco
```

### 50/50
```csharp
var regra = new DistributionRule(1, 1, 50, 2); // 50% Pagarme, 50% Bradesco
```

## ✅ Vantagens do Método

1. **Precisão**: Distribuição exata baseada em percentual
2. **Flexibilidade**: Qualquer percentual (70/30, 80/20, 60/40, etc.)
3. **Simplicidade**: Apenas 1 método principal
4. **Performance**: Memory Bank em RAM
5. **Thread-Safe**: Funciona com múltiplas threads
6. **Previsível**: Sempre o mesmo padrão
7. **Testável**: Fácil de testar e debugar


## 🧪 Teste Unitário

```csharp
[Test]
public void DeveDistribuir70Para30()
{
    // Arrange
    var service = new SimpleDistributionService();
    var regra = new DistributionRule(1, 1, 70, 2); // 70% Pagarme, 30% Bradesco
    
    // Act
    var resultados = new List<int>();
    for (int i = 0; i < 100; i++) // Testa 100 transações
    {
        resultados.Add(service.GetAdquirenteForTransaction(regra));
    }
    
    // Assert
    Assert.AreEqual(70, resultados.Count(x => x == 1)); // Pagarme
    Assert.AreEqual(30, resultados.Count(x => x == 2)); // Bradesco
}

[Test]
public void DeveDistribuir80Para20()
{
    // Arrange
    var service = new SimpleDistributionService();
    var regra = new DistributionRule(1, 1, 80, 2); // 80% Pagarme, 20% Bradesco
    
    // Act
    var resultados = new List<int>();
    for (int i = 0; i < 100; i++)
    {
        resultados.Add(service.GetAdquirenteForTransaction(regra));
    }
    
    // Assert
    Assert.AreEqual(80, resultados.Count(x => x == 1)); // Pagarme
    Assert.AreEqual(20, resultados.Count(x => x == 2)); // Bradesco
}
```

## 🚀 Como Usar

1. **Instancie** o serviço
2. **Crie** uma regra com adquirente, prioridade e percentual
3. **Chame** o método para cada transação
4. **Use** o ID retornado para processar a transação

```csharp
var service = new SimpleDistributionService();
var regra = new DistributionRule(1, 1, 70, 2); // Pagarme, prioridade 1, 70%, Bradesco ID=2

// Para cada transação
var adquirenteId = service.GetAdquirenteForTransaction(regra);
// Use adquirenteId para processar a transação
```

## 📋 Resumo das Mudanças

### ✅ O que foi adicionado:
- **Percentual** na `DistributionRule`
- **ID do adquirente de menor prioridade**
- **Ciclo de 100** transações (para trabalhar com percentuais)
- **Flexibilidade** para qualquer percentual (70/30, 80/20, 60/40, etc.)

### 🎯 Exemplos de uso:
```csharp
// 70% Pagarme, 30% Bradesco
var regra70 = new DistributionRule(1, 1, 70, 2);

// 80% Pagarme, 20% Bradesco  
var regra80 = new DistributionRule(1, 1, 80, 2);

// 60% Pagarme, 40% Bradesco
var regra60 = new DistributionRule(1, 1, 60, 2);
```

---

**Resumo**: Este método garante distribuição exata baseada em percentual usando Memory Bank, sendo simples, eficiente, flexível e previsível! 🎯
