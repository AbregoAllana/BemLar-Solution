# Decisões — Limpeza de dados e engenharia de features

Documento da etapa de preparação dos dados do piloto de cobrança preventiva da BemLar.
Cobre da leitura dos CSVs brutos até a geração de `features.csv`.

As seções de modelagem, avaliação e model card serão preenchidas depois de treinar os modelos.

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
`vencimento - 7` para os que atrasaram. Não segui essa leitura, por dois motivos.

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
