# Decisões — Realizações feitas pela Equipe de Dados

Engloba todas as decisões tomadas pela Equipe de Dados: Allana Rodrigues Abrego e João Vitor Vargas Pereiras.
Desde a limpeza dos dados até a avaliação do modelo.

---

## 1. Definição do alvo

```
inadimplente_30d = 1  se a parcela de referência não foi paga até D+30
inadimplente_30d = 0  caso contrário
```

**Parcela de referência:** a última parcela de cada contrato com vencimento até
`data_snapshot - 30 dias`, ou seja, até **13/02/2026**. Só nessas parcelas o prazo de 30 dias
já fechou e o desfecho é observável. Todos os 3.000 contratos têm uma parcela elegível.

**Data de referência:** vencimento da parcela de referência **menos 7 dias**. É o momento
simulado da ligação preventiva. Vai de **06/12/2024** a **06/02/2026**.

### Por que uma única regra de data para todos os contratos

O enunciado sugere usar o snapshot como data de referência para contratos "em dia" e
`vencimento - 7` para os que atrasaram. Não seguimos essa leitura, por dois motivos.

Escolher a data de referência com base em quem atrasou significa usar o alvo para construir
a feature. No momento real da previsão não se sabe quem vai atrasar, então essa regra é
impossível de reproduzir em produção e vaza o rótulo para dentro dos dados.

Além disso, o snapshot é de 37 a 464 dias posterior ao `vencimento - 7`. Contratos em dia
receberiam muito mais histórico que os inadimplentes, e o modelo aprenderia a diferença de
janela em vez do risco. O dicionário de dados define `vencimento - 7` sem exceção, e foi
essa a regra implementada.

### Prevalência

| Classe | Contratos | Proporção |
|---|---:|---:|
| 0 (adimplente em 30 dias) | 1.700 | 56,7% |
| 1 (inadimplente em 30 dias) | 1.300 | 43,3% |

---

## 2. Regra de corte temporal

Nenhum evento posterior à data de referência entra em nenhuma feature. A função
`filtra_por_data` aplica esse recorte em cada base de eventos antes de qualquer agregação.

| Base | Chave de junção | Coluna de data |
|---|---|---|
| `pagamentos` | `id_contrato` | `data_vencimento` |
| `ocorrencias_sac` | `id_contrato` | `data_ocorrencia` |
| `score_bureau` | `id_contrato` | `data_consulta` |
| `compras` | `id_cliente` | `data_compra` |

`compras` é o único caso que não liga por contrato. A base de referência carrega
`id_cliente` justamente para permitir esse join sem perder a data de corte do contrato.

Em `pagamentos` o filtro usa `data_vencimento < data_referencia`, estritamente anterior.
A parcela de referência vence 7 dias depois do corte e, por definição, ainda estaria em
aberto para todos os 3.000 contratos. Incluí-la geraria uma coluna constante que só reflete
a forma como a janela foi montada.

O status de pagamento também é avaliado na data de referência, não no snapshot:

```
paga_ate_referencia    = data_pagamento preenchida E data_pagamento <= data_referencia
pendente_na_referencia = não paga_ate_referencia
```

Uma parcela paga depois da data de referência conta como pendente, porque na hora da
ligação ela estava em aberto.

---

## 3. Limpeza aplicada

Os arquivos em `bases/` não foram alterados. Toda a limpeza é reproduzível dentro de
`carregar_e_limpar_dados()`, no notebook.

| Problema | Onde | Linhas | Erro ou artefato | Tratamento |
|---|---|---:|---|---|
| Duplicatas completas | `ocorrencias_sac` | 2 | erro de extração | removidas antes de agregar |
| Idade acima de 90 (máx. 137) | `contratos.idade` | 3 | valor improvável | limitada a 90; coluna não entrou no modelo |
| Renda declarada igual a zero | `contratos.renda_declarada` | 4 | impede razão | convertida em ausente |
| `data_pagamento` vazia | `pagamentos` | 1.690 | artefato esperado | lida como parcela não paga, sem imputar data |
| Duplicatas completas | `motivos_atraso` | 87 | erro de extração | base não utilizada |
| Espaços em torno de categorias | todas | — | artefato | `strip` em colunas de texto |

Datas convertidas de `dd/mm/aaaa` e decimais com vírgula em todos os arquivos.
Unicidade validada em `id_contrato` e na chave `(id_contrato, n_parcela)`.

Idades acima de 90 e renda zero foram tratadas, não descartadas. Remover as linhas jogaria
fora contratos válidos por causa de um campo cadastral ruim.

---

## 4. Features que entraram

Uma linha por contrato. 25 features, mais identificadores e alvo, em `features.csv`.

### Contratos

| Coluna | O que captura |
|---|---|
| `n_parcelas` | prazo contratado |
| `valor_parcela` | valor da parcela |
| `renda_declarada_tratada` | renda informada na aprovação |
| `canal_venda` | Loja física, Site ou Televendas |
| `tenure_dias` | dias entre a venda e a data de referência |
| `comprometimento_renda` | `valor_parcela / renda`, peso da parcela no orçamento |
| `qtd_contratos_parcelados` | contratos do cliente já existentes na data de referência |

### Compras

| Coluna | O que captura |
|---|---|
| `qtd_compras` | frequência de compra na rede |
| `ticket_medio_compras` | valor médio por compra |
| `dias_desde_ultima_compra` | recência do relacionamento |
| `qtd_compra_informatica`, `qtd_compra_moveis`, `qtd_compra_eletro`, `qtd_compra_cama_mesa_banho` | mix de categorias |

### Pagamentos

| Coluna | O que captura |
|---|---|
| `qtd_parcelas_vencidas` | histórico disponível até a referência |
| `qtd_parcelas_pagas` | parcelas quitadas até a referência |
| `qtd_parcelas_pendentes` | parcelas em aberto na referência |
| `qtd_atrasos_positivos` | parcelas pagas com atraso ou ainda em aberto |

### SAC

| Coluna | O que captura |
|---|---|
| `qtd_sac_telefone`, `qtd_sac_whatsapp`, `qtd_sac_loja`, `qtd_sac_site` | volume de contato por canal antes da referência |

### Score de bureau

| Coluna | O que captura |
|---|---|
| `score_bureau` | score da última consulta anterior à referência |
| `dias_desde_consulta_bureau` | quão desatualizado está esse score |
| `tem_consulta_bureau` | existe consulta anterior à referência |

> ### Atualização da Fase 3 — features adicionais de pagamentos

> Na versão revisada da preparação, o histórico de pagamentos foi detalhado além das
> contagens básicas já listadas. Essas features foram adicionadas porque a inadimplência
> de 30 dias é um comportamento de pagamento, e não apenas a existência de parcelas
> vencidas ou pendentes. Todas são calculadas somente com parcelas e pagamentos
> disponíveis até `data_referencia`.

> | Coluna | Por que foi adicionada | O que agrega à análise |
> |---|---|---|
> | `qtd_atrasos_pagos` | Separar atrasos já encerrados de parcelas ainda pendentes. | Mede quantas parcelas foram pagas depois do vencimento, identificando recorrência de atraso já observada. |
> | `payment_late_rate` | Transformar a quantidade de atrasos em uma proporção comparável entre contratos com históricos de tamanhos diferentes. | Mede a frequência relativa de parcelas com atraso no histórico disponível. |
> | `payment_mean_delay` | Resumir a intensidade média do atraso na data de referência. | Diferencia poucos atrasos curtos de um padrão médio de atraso mais longo; parcelas pendentes usam o atraso já materializado no corte. |
> | `payment_max_delay` | Capturar o pior atraso observado até a decisão. | Identifica a maior gravidade de atraso já registrada para o contrato. |
> | `payment_sum_delay` | Medir o volume acumulado de dias de atraso. | Combina frequência e duração, indicando a carga total de atraso no histórico. |
> | `payment_mean_delay_paid` | Isolar a duração média dos atrasos que já foram pagos. | Mostra o comportamento de regularização sem misturá-lo com parcelas ainda abertas. |
> | `payment_max_delay_paid` | Isolar o maior atraso entre parcelas efetivamente pagas. | Captura a pior experiência de atraso encerrada e evita que uma parcela pendente domine essa medida. |

Com essa atualização, a matriz revisada contém 32 features de modelo, além de
`id_contrato`, `id_cliente`, `data_referencia` e `inadimplente_30d`. As features adicionais
não usam `status_contrato`, atributos sensíveis ou eventos posteriores ao corte. Médias e
medidas de atraso que não existem permanecem nulas na Fase 3 e são imputadas apenas no
pipeline de modelagem, usando estatísticas aprendidas no conjunto de treino.

### Identificadores e alvo

`id_contrato`, `id_cliente`, `data_referencia` e `inadimplente_30d` não são features.
`data_referencia` fica no arquivo porque o split temporal da fase de modelagem depende dela.

---

## 5. O que ficou de fora

| Coluna | Motivo |
|---|---|
| `sexo` | atributo sensível; discriminação direta |
| `bairro` | proxy territorial de renda e raça |
| `status_contrato` | situação no snapshot, desconhecida no momento da ligação |
| `data_snapshot` | constante; serviu só para definir a parcela elegível |
| `motivos_atraso` | sem chave de contrato ou cliente |
| `idade` | tratada na limpeza, mas descreve identidade, não comportamento |
| negociação de dívida no SAC | nenhum evento anterior à data de referência |

### `status_contrato` não é o alvo

O status do sistema discorda do alvo de 30 dias em 62 contratos:

| `status_contrato` | alvo 0 | alvo 1 |
|---|---:|---:|
| Em dia | 896 | 18 |
| Inadimplente | 22 | 1.260 |
| Quitado | 782 | 22 |

Os 22 "Quitado" com alvo 1 são contratos em que a parcela de referência foi paga depois de
D+30 e o contrato quitou mais tarde. Pela definição do piloto, houve inadimplência de 30
dias: a ligação preventiva teria valido a pena. O alvo é a fonte de verdade; o status é
uma fotografia posterior.

### Negociação de dívida: achado negativo

As 707 ocorrências do tipo "Negociação de dívida" são **todas** posteriores à data de
referência do respectivo contrato. Nenhuma antes do corte.

Faz sentido: o cliente procura a empresa para negociar **depois** de atrasar. É um evento
reativo, e o piloto roda 7 dias **antes** do vencimento. Usar essa informação seria prever
o atraso com um efeito do próprio atraso.

Por isso as colunas de canal do SAC contam atendimentos de qualquer tipo anteriores à
referência, e não apenas negociação. São 2.468 eventos em 1.586 contratos, com sinal real.
Se o filtro fosse só negociação, as quatro colunas ficariam zeradas nos 3.000 contratos.

Esse é um resultado útil para o negócio: mostra que o SAC de negociação não serve como
gatilho de cobrança preventiva, e que um alerta baseado nele chegaria sempre tarde demais.

---

## 6. Tratamento de ausência

A regra não é a mesma para toda coluna. Ausência de evento é informação; ausência de
cálculo não é.

**Vira zero.** Contagens de eventos que não aconteceram. Um cliente sem compras fez zero
compras — isso é um fato, não um dado faltante.

**Fica ausente.** Médias, recências e razões que não existem quando não há base de cálculo.
Preencher com zero inventaria um valor que o negócio não observou: `ticket_medio_compras = 0`
diria que o cliente comprou de graça, e `dias_desde_ultima_compra = 0` diria que comprou hoje.

Nulos remanescentes em `features.csv`:

| Coluna | Nulos | Motivo |
|---|---:|---|
| `renda_declarada_tratada` | 4 | renda declarada igual a zero |
| `comprometimento_renda` | 4 | sem renda válida para dividir |
| `ticket_medio_compras` | 435 | cliente sem compras antes da referência |
| `dias_desde_ultima_compra` | 435 | cliente sem compras antes da referência |

A imputação deve acontecer no pipeline de modelagem, com estatística calculada **apenas no
conjunto de treino**. Imputar agora, sobre a base inteira, vazaria informação do teste.

`tem_consulta_bureau` vale 1 nos 3.000 contratos: todos têm a consulta de aprovação do
crediário antes da data de referência. A coluna é constante nesta base e não vai separar
nada, mas foi mantida por estar no escopo acordado e por documentar a cobertura.

---

## 7. Cobertura das bases

| Situação | Contratos |
|---|---:|
| Total | 3.000 |
| Clientes únicos | 1.824 |
| Sem compras antes da referência | 435 |
| Sem atendimento SAC antes da referência | 1.414 |
| Com parcela pendente na referência | 108 |
| Com consulta de bureau antes da referência | 3.000 |

Só 108 contratos chegam à data de referência com parcela em aberto. O sinal de risco
precisa vir principalmente do histórico de atraso em parcelas já pagas
(`qtd_atrasos_positivos`), e não do saldo em aberto.

---

## 8. Validações automáticas

O notebook interrompe a execução se alguma destas condições falhar:

- `id_contrato` único em `contratos.csv`
- `(id_contrato, n_parcela)` único em `pagamentos.csv`
- exatamente uma linha por contrato na matriz final
- nenhum valor infinito nas colunas numéricas
- `data_referencia` sempre anterior ao vencimento da parcela de referência
- nenhuma negociação de dívida anterior à data de referência

A última é uma trava de regressão. Hoje ela passa. Se a base mudar e ela quebrar, é sinal
de que o corte temporal precisa ser reavaliado, não de defeito no código.

---

## 9. Como reproduzir

Abrir `enunciado.ipynb` e executar as células na ordem. A geração das features está isolada
em `construir_features(caminho_dados, datas_referencia)`, que lê os CSVs originais e devolve
a matriz pronta. A última célula da Fase 3 grava `features.csv`.

Resultado: 3.000 linhas e 29 colunas.

---

## 10. Atualização da Fase 3 — preparação revisada

Esta seção registra a atualização executada depois da revisão da preparação dos dados.
O conteúdo das seções anteriores foi mantido; esta é a versão efetivamente usada pela
modelagem.

### 10.1 Execução reproduzível

A Fase 3 foi executada novamente a partir dos CSVs originais, depois da remoção do
`features.csv` anterior. A função oficial passou a ser:

```python
def executar_fase3(
		caminho_bases=BASE_DIR,
		arquivo_saida=ARQUIVO_SAIDA,
):
		...
```

Ela mantém as etapas 3.1 a 3.4: carga e limpeza, definição do alvo, corte temporal,
engenharia de features, auditoria e exportação.

### 10.2 Resultado atualizado

| Medida | Resultado |
|---|---:|
| Contratos | 3.000 |
| Inadimplentes em 30 dias | 1.300 |
| Adimplentes em 30 dias | 1.700 |
| Prevalência | 43,3% |
| Colunas exportadas | 36 |

O arquivo atualizado contém uma linha por contrato, o alvo, os identificadores e as
features construídas com corte em `data_referencia`. O número de 29 colunas registrado
anteriormente corresponde à versão anterior da preparação; a execução revisada gerou
36 colunas e é a versão usada na Fase 4.

### 10.3 Decisões preservadas na atualização

- O alvo continua sendo `inadimplente_30d`, definido pela não quitação até D+30.
- A `data_referencia` continua sendo sete dias antes do vencimento da parcela de referência.
- Nenhuma informação posterior à `data_referencia` entra nas features.
- Contagens de eventos ausentes viram zero; médias, razões e recências permanecem nulas
	para serem imputadas somente durante o treino.
- `status_contrato`, `sexo`, `bairro`, `idade`, `data_snapshot`, negociação de dívida e
	`motivos_atraso` continuam fora do conjunto de preditores pelos motivos já registrados.

---

## 11. Fase 4 — modelagem

### 11.1 Preparação para o modelo

Foram retirados do conjunto de entrada `id_contrato`, `id_cliente`, `data_referencia` e
`inadimplente_30d`. Os três primeiros são identificadores ou controlam o corte temporal;
o último é o alvo.

As features numéricas receberam imputação pela mediana e padronização. As categóricas
receberam imputação pela categoria mais frequente e codificação one-hot. Essas operações
ficam dentro de um `Pipeline` e são ajustadas apenas no conjunto de treino, evitando
vazamento de informação do teste.

### 11.2 Modelos comparados

Foram treinados dois modelos:

1. **Árvore de decisão baseline:** modelo simples, raso e explicável, com profundidade
	 máxima 5, mínimo de 20 observações por folha e pesos balanceados.
2. **Regressão logística:** alternativa linear, regularizada pelo padrão do `scikit-learn`,
	 com pesos balanceados e as mesmas etapas de pré-processamento.

A árvore atende ao requisito de baseline simples. A regressão logística foi incluída para
verificar se uma fronteira linear generaliza melhor que a árvore.

### 11.3 Split aleatório estratificado

O split aleatório reservou 25% dos contratos para teste, mantendo a proporção das classes.
Os resultados foram:

| Modelo | Acurácia | ROC AUC | Average precision | Precision@80 | Recall@80 |
|---|---:|---:|---:|---:|---:|
| Árvore de decisão | 0,667 | 0,735 | 0,691 | 0,862 | 0,212 |
| Regressão logística | 0,709 | 0,777 | 0,740 | 0,900 | 0,222 |

A acurácia foi reportada, mas não foi usada para escolher a fila, pois a operação não
classifica todos os contratos com um limiar: ela liga para somente 80.

### 11.4 Split temporal

Os 75% dos contratos mais antigos, ordenados por `data_referencia`, foram usados no treino;
os 25% mais recentes foram usados no teste, sem embaralhamento.

| Modelo | Acurácia | ROC AUC | Average precision | Precision@80 | Recall@80 |
|---|---:|---:|---:|---:|---:|
| Árvore de decisão | 0,655 | 0,702 | 0,618 | 0,800 | 0,203 |
| Regressão logística | 0,665 | 0,718 | 0,666 | 0,800 | 0,203 |

O resultado temporal foi inferior ao aleatório nas métricas gerais. Isso sugere que a
distribuição temporal ou a relação entre as features e o alvo pode mudar, e também que o
treino temporal tem menos dados. Esse resultado não prova causalidade, não prova que o
modelo falhará sempre no futuro e não autoriza concluir que uma feature causa o atraso.

Como os modelos empataram em `precision@80` temporal, a escolha usa um desempate
determinístico: primeiro `recall@80`, depois `average_precision` e, por fim, o nome do
modelo em ordem alfabética. Assim, a seleção não depende da ordem das linhas nem da
versão do `scikit-learn`. Nesta execução, a regressão logística foi selecionada pelo
`average_precision` superior. A escolha não representa superioridade estatística geral
da regressão sobre a árvore.

### 11.5 Importância das variáveis

Na árvore escolhida, as maiores importâncias preditivas foram:

| Variável | Importância aproximada |
|---|---:|
| `payment_mean_delay` | 0,595 |
| `score_bureau` | 0,128 |
| `payment_late_rate` | 0,070 |
| `payment_max_delay` | 0,052 |
| `comprometimento_renda` | 0,034 |

Essas importâncias indicam associação usada pelo modelo nesta amostra. Não são efeitos
causais e não significam que alterar uma variável produziria diretamente alteração no
risco.

---

## 12. Fase 5 — avaliação operacional

### 12.1 Critério de avaliação

A capacidade da equipe é de 80 ligações. Por isso, a avaliação foi feita ordenando o teste
por `predict_proba`, selecionando os 80 maiores riscos e calculando `precision@80` e
`recall@80`. O limiar padrão de 0,50 não define a fila.

O falso positivo é uma ligação usada com um contrato que não atingiria 30 dias de atraso.
O falso negativo é um contrato que atingiria 30 dias e ficou fora da fila, representando
uma oportunidade perdida diante do custo médio estimado de R$ 1.200.

### 12.2 Comparação com filas-base

Os resultados abaixo usam o teste temporal:

| Fila | Positivos na fila | Precision@80 | Recall@80 | Falsos positivos | Falsos negativos |
|---|---:|---:|---:|---:|---:|
| Modelo escolhido | 64 | 0,800 | 0,203 | 16 | 251 |
| Aleatória | 36 | 0,450 | 0,114 | 44 | 279 |
| Maior valor da parcela | 47 | 0,588 | 0,149 | 33 | 268 |

A fila do modelo foi melhor que as duas bases. Ela encontrou 28 positivos a mais que a
fila aleatória e 17 a mais que a fila ordenada por valor da parcela. Ainda assim, o recall
é baixo porque a restrição de 80 ligações é muito menor que o total de inadimplentes no
conjunto de teste.

### 12.3 Tradução financeira

Na fila do modelo, 64 contratos positivos foram selecionados, correspondendo a uma
exposição bruta estimada de `64 x R$ 1.200 = R$ 76.800`. Esse valor não é economia
garantida: representa o prejuízo potencial associado aos casos capturados, antes de medir
quantos seriam recuperados pela ligação.

As 16 ligações restantes são falsos positivos e representam capacidade operacional usada
em contratos que não atingiriam o alvo. Os 251 falsos negativos correspondem a uma
exposição potencial de R$ 301.200 que permaneceu fora da fila, sem afirmar que todo esse
valor seria evitável.

### 12.4 Decisão

A recomendação é **ligar com ressalva**. A fila supera as bases de comparação, mas a
validação temporal foi feita em uma única janela e o recall é baixo. Antes de ampliar o
piloto, é necessário acompanhar semanalmente `precision@80`, recuperação efetiva após a
ligação, custo da renegociação, estabilidade das features e desempenho por grupos para
auditoria de fairness.

---

## 13. Model card da versão modelada

- **Dados:** seis CSVs brutos extraídos em 15/03/2026, processados pela Fase 3 revisada.
- **Alvo:** inadimplência de 30 dias da parcela de referência.
- **Uso pretendido:** ordenar contratos para uma fila de cobrança preventiva limitada a 80
	ligações semanais.
- **Features:** dados do contrato, histórico de compras, histórico de pagamentos, contatos
	SAC anteriores à referência e última consulta de bureau anterior à referência.
- **Exclusões:** atributos sensíveis, proxies territoriais, status posterior, motivos sem
	chave de ligação e eventos de negociação posteriores ao corte.
- **Limitações:** não usar para negar crédito, definir preço ou inferir causalidade. A
	prevalência, os custos e o comportamento dos clientes podem mudar.
- **Monitoramento:** acompanhar a performance temporal, a qualidade da fila, o retorno
	financeiro real e possíveis diferenças de desempenho entre grupos.
