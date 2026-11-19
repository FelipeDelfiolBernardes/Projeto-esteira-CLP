# Sistema Automatizado de Esteira de Separação de Produtos

Projeto de automação industrial que implementa, em linguagem Ladder (IEC 61131-3), o controle de uma esteira com triagem automática de dois tipos de produtos, mais um simulador web interativo (`esteira.html`) que reproduz a lógica do CLP em tempo real.

## Sumário

- [Visão geral](#visão-geral)
- [Objetivos](#objetivos)
- [Como funciona](#como-funciona)
- [Tabela de I/O](#tabela-de-io)
- [Lógica Ladder](#lógica-ladder)
- [Simulador HTML](#simulador-html)
- [Resultados dos testes](#resultados-dos-testes)
- [Limitações e melhorias futuras](#limitações-e-melhorias-futuras)
- [Tecnologias](#tecnologias)
- [Autores](#autores)

## Visão geral

Em linhas de manufatura flexível, produtos diferentes compartilham a mesma esteira antes de seguirem para seus destinos. O desafio é identificar cada item no ponto de leitura e acionar o desviador correto mais adiante, sem que os itens seguintes interfiram na lógica em andamento.

O sistema resolve isso com:

- Sensor de presença com detecção por borda de subida
- Sensor de classificação com valor inteiro (0 = tipo A, 1 = tipo B, 2 = não identificado)
- Desviadores solenoides acionados por bobina SET
- Contadores CTU para o controle de produção
- Alarme para produtos não identificados

## Objetivos

**Geral:** desenvolver a documentação técnica e a lógica Ladder de um CLP para controle e supervisão de uma esteira de separação automática, validando o sistema por testes funcionais e análise de eficiência.

**Específicos:**

- Mapear sensores e atuadores e elaborar a tabela de entradas e saídas
- Diferenciar os tipos de item por meio do sensor de classificação e de comparadores
- Estruturar o intertravamento dos desviadores, evitando sobreposição de ciclos de atuação
- Validar o comportamento dinâmico no simulador e propor correções

## Como funciona

1. O operador aciona `B_ligar` e o motor da esteira liga (partida selada).
2. O produto passa pelo `Sensor_presenca`, e a borda de subida gera um único `Pulso_produto`.
3. O `Sensor_classificacao` informa o tipo do produto.
4. Comparadores `[==]` identificam o tipo, incrementam os contadores e acionam o desviador correspondente via SET.
5. Se nenhum desviador for acionado, o produto é tratado como não identificado e o `Alarme` é ativado.
6. `B_desligar` (NF) e `B_emergencia` (NF) desligam o motor, com comportamento fail-safe.

## Tabela de I/O

| Variável | Tipo | Categoria | Descrição |
|---|---|---|---|
| `B_ligar` | Bool | Entrada | Botão START (NA) |
| `B_desligar` | Bool | Entrada | Botão STOP (NF) |
| `B_emergencia` | Bool | Entrada | Botão de emergência (NF no rung de partida) |
| `Sensor_presenca` | Bool | Entrada | Sensor fotoelétrico de presença |
| `Sensor_classificacao` | Int | Entrada | 0 = tipo A, 1 = tipo B, 2 = não identificado |
| `Motor` | Bool | Saída | Contator do motor da esteira |
| `Desvio_a` | Bool | Saída | Solenoide do desviador A |
| `Desvio_b` | Bool | Saída | Solenoide do desviador B |
| `Alarme` | Bool | Saída | Sinaleiro de produto não identificado |
| `Pulso_produto` | Bool | Memória | Pulso único por produto detectado |
| `contador_total` | CTU | Contador | Total de produtos processados |
| `contador_a` | CTU | Contador | Produtos tipo A |
| `contador_b` | CTU | Contador | Produtos tipo B |

## Lógica Ladder

O programa tem 8 rungs organizados em blocos funcionais:

| Rung | Entradas | Saída / Ação | Descrição |
|---|---|---|---|
| R001 | `B_ligar` + `B_desligar` (NF) + `B_emergencia` (NF) + `Motor` (selo) | `Motor` | Partida selada do motor |
| R002 | `Sensor_presenca` (P) + `Motor` | `Pulso_produto` | Pulso único por produto |
| R003 | `Pulso_produto` | CTU `contador_total` | Conta todos os produtos |
| R004 | `Pulso_produto` + `[Sensor_classificacao == 0]` | CTU `contador_a` | Conta tipo A |
| R005 | `Pulso_produto` + `[Sensor_classificacao == 1]` | CTU `contador_b` | Conta tipo B |
| R006 | `Pulso_produto` + `[Sensor_classificacao == 0]` | (S) `Desvio_a` | Ativa desviador A |
| R007 | `Pulso_produto` + `[Sensor_classificacao == 1]` | (S) `Desvio_b` | Ativa desviador B |
| R008 | `Pulso_produto` + `Desvio_a` (NC) + `Desvio_b` (NC) | (S) `Alarme` | Produto não identificado |

**Observação:** como os desviadores usam SET, é necessário um reset após o produto passar pelo desviador (por posição ou temporização). Sem isso, o R008 não dispara o alarme corretamente nos itens seguintes. O simulador HTML já aplica esse reset.

## Simulador HTML

Arquivo único e autocontido: `esteira.html`. Basta abri-lo em qualquer navegador, sem instalação e sem dependências externas.

**Recursos:**

- Animação da esteira com sensor, desviadores, caixas A e B, rejeito e alarme
- Painel de comando com `B_ligar`, `B_desligar`, `B_emergencia` e reset
- Inserção manual de produtos tipo A, tipo B e não identificado
- Controle de velocidade (1x, 2x e 4x)
- Ciclo de varredura do CLP a cada 40 ms
- Tabela de I/O com valores em tempo real
- Tabela de rungs com destaque dos rungs ativos
- Teste automático por lote (10 A, 10 B e 5 não identificados) com cálculo de eficiência

**Como usar:**

1. Abra `esteira.html` no navegador.
2. Clique em `B_ligar` para ligar a esteira.
3. Insira produtos com os botões de tipo e observe a triagem e os contadores.
4. Teste o fail-safe acionando `B_desligar` ou `B_emergencia`.
5. Use "Executar lotes" para reproduzir a validação do relatório.

## Resultados dos testes

Validação realizada no simulador `plcsimulator.online`:

| Lote | Itens inseridos | Triagens corretas | Falhas | Eficiência |
|---|---|---|---|---|
| Lote 01 (Tipo A) | 10 | 10 | 0 | 100% |
| Lote 02 (Tipo B) | 10 | 10 | 0 | 100% |
| Lote 03 (Não identificado) | 5 | 5 (alarme) | 0 | 100% |

## Limitações e melhorias futuras

- Substituir a lógica por sensores de fim de curso nos braços desviadores
- Adicionar sensores nas caixas de destino para detectar enchimento
- Implementar interface supervisória HMI/SCADA
- Integrar com sistemas MES para rastreabilidade
- Adicionar rung explícito de reset dos desviadores no programa Ladder

## Tecnologias

- Linguagem Ladder (IEC 61131-3)
- [plcsimulator.online](https://plcsimulator.online) para validação do CLP
- HTML, CSS e JavaScript puro (simulador web)

## Autores

Projeto acadêmico da disciplina de Sistemas Automatizados, Universidade São Judas Tadeu.

Autores: Felipe Delfiol Bernardes, Cauã Carrilla Molinaro, Lucas Agnelli, Pedro Henrique Sardinha e Vinicius Gomes da Silva.
