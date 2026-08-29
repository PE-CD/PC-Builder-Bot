


# PC Builder Bot — Histórico das Etapas 1 a 7

Este documento é um registro e publicação do desenvolvimento do algoritmo de recomendação e montagem de computadores, cobrindo desde a estrutura inicial até a validação por faixas de preços, para os usuários da EconoDiginal.

Durante as etapas 1 a 5, o principal desafio que foi fazer o algoritmo deixar de escolher componentes apenas pelo maior score e passar a produzir configurações coerentes com cada faixa de orçamento. 
Nas etapas 6 e 7, o foco mudou. E passou a ser consolidar a estratégia de seleção, melhorar o aproveitamento do valor disponível e validar o comportamento do Builder em diferentes faixas de custo.




### Etapa 1 — Estrutura Inicial do Algoritmo

**Objetivo:**  
Desenvolver a arquitetura base do sistema de montagem automática de PCs.

**Implementação:**  
Foi estabelecida uma base inicial de configurações dividida em três perfis fixos (**Econômico**, **Intermediário** e **Potente**), onde cada perfil mapeava os seguintes componentes:

* **Processador (CPU)**
* **Placa de Vídeo (GPU)**
* **Placa-mãe (Motherboard)**
* **Memória RAM**
* **Armazenamento (SSD)**
* **Fonte de Alimentação (PSU)**
* **Gabinete (Case)**

> **Problema Encontrado:** 

> A dependência excessiva de presets rígidos limitava o aproveitamento de orçamentos intermediários. Um aumento no limite financeiro nem sempre resultava em um ganho proporcional de desempenho, gerando gargalos na alocação de recursos.



### ETAPA 2 — Banco de componentes e combinação

**Objetivo:**  
Evoluir a arquitetura do algoritmo para permitir a combinação dinâmica de peças a partir de uma base de dados centralizada, superando a limitação de perfis fixos.

**Implementação:**  
Foi estruturado o `COMPONENT_DATABASE`, um repositório centralizado de hardware contendo opções categorizadas de:

* **CPU** (Processador)
* **GPU** (Placa de Vídeo)
* **Motherboard** (Placa-mãe)
* **RAM** (Memória)
* **Storage** (Armazenamento/SSD)
* **PSU** (Fonte de Alimentação)
* **Case** (Gabinete)

A partir dessa estrutura, o algoritmo passou a iterar e testar múltiplas combinações possíveis entre as peças disponíveis.

>  **Problema Encontrado:**

> A combinação livre aumentou a flexibilidade, mas gerou **configurações desbalanceadas**. Sem regras de proporção, o sistema cometia erros de alocação — como parear uma CPU de alto custo com uma GPU de entrada, ou investir excessivamente em gabinete e fonte, reduzindo o orçamento disponível para o desempenho principal da máquina.



###   Etapa 3 — Camada de Compatibilidade Técnica

**Objetivo:**  
Garantir que todas as configurações geradas pelo algoritmo fossem eletricamente e fisicamente viáveis antes de serem aceitas.

**Implementação:**  
Integração do serviço `CompatibilityService.validate()`, responsável por checar a integridade da montagem considerando os seguintes critérios técnicos:

* **Socket:** Compatibilidade entre CPU e placa-mãe (ex: AM4, AM5, LGA)
* **Tipo de RAM:** Compatibilidade de geração da memória com a placa-mãe (ex: DDR4 vs DDR5)
* **Demanda Energética:** Relação do consumo estimado da build em relação à capacidade nominal da fonte (PSU)
* **Dimensões Físicas:** Espaço interno livre no gabinete para o comprimento da placa de vídeo (GPU)

>  **Problema Encontrado:**  

> O algoritmo passou a gerar apenas builds viáveis, mas **compatibilidade técnica não garantia uma boa distribuição financeira**. A máquina funcionava sem falhas físicas, porém o orçamento ainda podia ser alocado de forma ineficiente entre as peças.



### ETAPA 4 — Sistema de pontuação (score)

**Objetivo:**  
Desenvolver uma lógica quantitativa para comparar, ranquear e selecionar a melhor configuração entre as opções válidas.

**Implementação:**  
Foi criada a função `scoreBuild()`, estabelecendo um sistema de pesos, bônus e penalidades focado nos seguintes pilares:

* **Aproveitamento de Orçamento:** Calculado via `budgetUsage = total / budget`. O fator multiplicador foi reajustado de `budgetUsage * 160` para `budgetUsage * 240`, premiando builds que utilizam o teto financeiro sem estourá-lo.
* **Foco Gamer (Priorização de GPU):** Aumento de peso para a placa de vídeo e quantidade de VRAM, integrando regras específicas para modelos como RX 6600, RX 7600 e RX 7800 XT.
* **Capacidade de Memória (RAM):** Trava de descarte automático para montagens com menos de 32 GB RAM em orçamentos a partir de R$ 4.500.
* **Armazenamento (SSD):** Bonificação estratégica para modelos de 1 TB em faixas de preço intermediárias e avançadas.
* **Dimensionamento de Fonte (PSU):** Penalização para fontes superdimensionadas quando uma opção menor supre com margem a demanda energética.



###  Etapa 5 — Refinamento da Seleção por Faixa de Orçamento

**Objetivo:**  
Fazer o algoritmo alinhar o nível dos componentes com a capacidade financeira da busca, garantindo que a escolha de peças mantivesse coerência dentro de cada patamar de preço.

**Implementação:**  
Foi implementada a divisão do motor de escolha em três faixas de valor:

* **Econômico:** `< R$ 4.500`
* **Intermediário:** `R$ 4.500` até `R$ 6.499`
* **Potente:** `>= R$ 6.500`

**Regras Específicas de Controle:**

* **Filtros de GPU:** Sistema de penalização para evitar o uso de GPUs desproporcionais ao orçamento (ex: penalidade para `RX 7800 XT` em builds `< R$ 5.500` ou penalidade severa `< R$ 5.000`).
* **Requisito Mínimo de RAM:** A partir de `R$ 4.500`, a presença de **32 GB RAM** tornou-se obrigatória para avanço da build.
* **Dimensionamento da Fonte (PSU):** Para orçamentos `< R$ 5.000`, fontes de **750W ou mais** foram restringidas quando não justificadas pelo consumo energético real do setup.



###  Etapa 6 — Aproveitamento Proporcional e Algoritmo de Desempate

**Objetivo:**  
Otimizar a alocação do orçamento, reduzindo cenários em que o algoritmo selecionava builds subaproveitadas por pequenas margens na pontuação individual das peças.

**Implementação da Heurística:**  
A função `scoreBuild()` passou a calcular ativamente a taxa de uso financeiro (`budgetUsage = total / budget`), aplicando ponderações estratégicas:

* **Bônus de Aproveitamento:** Recompensa para builds com `budgetUsage >= 0.95` e `budgetUsage >= 0.98`.
* **Penalização de Sobra:** Desconto em pontuação para builds com `budgetUsage < 0.90`.
* **Lógica de Desempate:** Introdução das variáveis `scoreDifference`, `bestBudgetUsage` e `closeScore`. Em cenários de pontuações próximas, a build com maior eficiência orçamentária assume prioridade.

**Regras de Trava e Sanidade:**  
* **Requisito Mínimo de RAM:** Para orçamentos `>= R$ 4.500`, configurações com `< 32 GB RAM` são eliminadas antes da etapa de pontuação.
* **Redimensionamento de PSU:** Para orçamentos `< R$ 5.000`, fontes `>= 750W` são descartadas quando o consumo da máquina não justifica a capacidade.




###  Etapa 7 — Validação Completa por Faixas de Orçamento

**Objetivo:**  
Submeter a versão final do algoritmo a testes práticos em múltiplos níveis de preço, homologando o comportamento das regras e a coerência das montagens geradas.

**Resultados das Baterias de Teste:**

| Orçamento Alvo | Custo Final | Status / Classificação | Peças Principais Selecionadas |
| :--- | :--- | :--- | :--- |
| **R$ 2.000** | R$ 3.000 | ⚠️ **Limite Excedido** | O sistema identificou corretamente a inviabilidade e retornou a build mínima possível (R$ 3.000). |
| **R$ 3.000** | R$ 3.000 | ✅ **Aprovado** (Econômico) | Ryzen 5 5500 + RX 6600 (8GB) + 16GB DDR4 + SSD 500GB |
| **R$ 4.000** | R$ 3.999 | ✅ **Aprovado** (Econômico) | Ryzen 5 5500 + RX 7600 (8GB) + **32GB DDR4** + SSD 1TB |
| **R$ 5.000** | R$ 4.956 | ✅ **Aprovado** (Intermediário) | Ryzen 5 7600 + RX 7600 (8GB) + **32GB DDR5** + SSD 500GB |
| **R$ 5.500** | R$ 5.456 | ✅ **Aprovado** (Intermediário) | Ryzen 5 7600 + RX 7800 XT (16GB) + 32GB DDR5 + SSD 500GB |
| **R$ 6.500** | R$ 6.499 | ✅ **Aprovado** (Potente) | Ryzen 7 7800X3D + RX 7800 XT (16GB) + 32GB DDR5 + SSD 1TB |

> **Conclusão da Validação:**  

> Os testes comprovaram a estabilidade do algoritmo: em todos os cenários viáveis, o aproveitamento do valor ficou acima de 98%, respeitando as exigências técnicas (como 32GB de RAM a partir de R$ 4.500) e classificando o perfil final corretamente.
