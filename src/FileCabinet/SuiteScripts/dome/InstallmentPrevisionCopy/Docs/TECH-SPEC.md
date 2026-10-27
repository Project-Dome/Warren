# TECH-SPEC — Cópia de Previsão de Parcelas na Vendor Bill (Warren)

## 1. Entrypoint: User Event — Vendor Bill (afterSubmit)

**Script:** `EntryPoints/InstallmentPrevisionCopy.UE.js` (`afterSubmit`)
→ `UseCases/CopyInstallmentPrevisions.js`
→ `Models/InstallmentPrevision.model.js`

**Registro de deployment:** `vendorbill` (nativo)

**Registro consultado/manipulado:** `customrecord_wr_installment_prevision` — objeto
pré-existente no ambiente Warren (fora da ProjectDome, sem cadastro prévio em MANIFEST).
Cadastrado no MANIFEST.md deste módulo como parte deste projeto, pois passa a ser
manipulado (criação de cópias) pelo novo entrypoint. Ver MANIFEST.md para a definição
completa dos campos.

### 1.1 Regra: Cópia das Previsões de Parcela do PO de origem

**Necessidade funcional:** ao criar uma Vendor Bill a partir de uma Purchase Order, cada
fatura precisa exibir sua própria cópia das previsões de parcela cadastradas no PO de
origem (`customrecord_wr_installment_prevision`), permitindo consulta e exibição
independente por fatura, sem alterar os registros originais vinculados ao PO.

**Contexto de execução:** roda apenas em `context.type === CREATE`. Ignorado em qualquer
outro evento (`EDIT`, `DELETE`, etc.).

**Identificação do(s) PO(s) de origem:** fonte primária é o campo nativo `orderdoc` de cada
linha do sublist `item` da Vendor Bill recém-criada — populado pelo NetSuite no fluxo "Bill
Multiple Purchase Orders", aponta para a PO de origem daquela linha especificamente. Lê-se
**todas** as linhas do sublist `item`, coletando os ids de PO distintos (uma Vendor Bill
pode ter linhas de mais de uma PO); cada PO identificada é processada individualmente,
sempre levando em consideração o que consta em cada PO (suas próprias previsões).

Como fallback, caso nenhuma linha tenha `orderdoc` preenchido, o script tenta o sublist
nativo `purchaseorders` (campo `id` de cada linha). Validado em sandbox que esse sublist
**não está disponível em `afterSubmit` no contexto CREATE** (`getLineCount` retorna `-1`),
portanto hoje essa via nunca encontra PO nesse evento — mantido por segurança/compatibilidade
caso o comportamento do NetSuite mude ou o script passe a rodar em outro contexto. Como
último fallback, usa o campo nativo `createdfrom`, somente quando essa transação for do tipo
Purchase Order (evita usar por engano a transação de origem quando esta não é uma PO, ver
nota sobre Recebimento de Item abaixo). Se nenhuma das três fontes apontar para uma ou mais
Purchase Orders, o script **não faz nada** (fatura sem PO de origem, fora de escopo desta
regra).

**Nota sobre Recebimento de Item:** hoje, no fluxo do Warren, a Vendor Bill não é criada a
partir de um Recebimento de Item (Item Receipt) — é sempre criada diretamente a partir da
PO. Esse cenário pode passar a ocorrer no futuro. O campo `orderdoc` do sublist `item`
continua preenchido com a PO de origem independentemente do caminho ser PO → Item Receipt →
Bill ou PO → Bill direto, portanto a identificação primária por esse campo continua
funcionando sem alteração. O ponto de atenção permanece no fallback via `createdfrom`: nesse
cenário ele apontaria para o Item Receipt, não para a PO — por isso o fallback valida o tipo
da transação antes de usá-la.

**Busca dos registros de origem:** para cada PO identificada, todos os registros de
`customrecord_wr_installment_prevision` cujo campo `custrecord_wr_ip_transaction_ls` seja
igual a essa PO. O resultado final é a união das previsões de todas as POs da fatura. Se a
busca não retornar nenhum registro (em nenhuma das POs), o script não faz nada (sem erro).

**Cópia:** para cada registro de previsão encontrado, cria um **novo** registro
`customrecord_wr_installment_prevision`, copiando os campos:

| Campo (MANIFEST) | Tratamento na cópia |
|---|---|
| `custrecord_wr_ip_number_nu` | copiado como está |
| `custrecord_wr_ip_entity_ls` | copiado como está |
| `custrecord_wr_ip_base_amount_cr` | copiado como está |
| `custrecord_wr_ip_duedate_ts` | copiado como está |
| `custrecord_wr_ip_pymtmethod_ls` | copiado como está |
| `custrecord_wr_ip_obs_ds` | copiado como está |
| `custrecord_wr_ip_transaction_ls` | **não** copiado do original — na cópia, aponta para a Vendor Bill recém-criada (`context.newRecord.id`), não para o PO |

O registro original (que aponta para o PO) permanece intacto — nunca é editado nem
removido.

**Acúmulo em faturamento parcial:** um mesmo PO pode ser faturado em múltiplas Vendor Bills
ao longo do tempo (faturamento parcial). Cada nova Vendor Bill dispara sua própria cópia
completa de **todas** as previsões do PO de origem, sem checar se já foram copiadas por uma
fatura anterior — não há deduplicação. Comportamento intencional, confirmado pelo usuário:
cada fatura precisa exibir o registro de previsão, independentemente de quantas outras
faturas já tenham copiado o mesmo conjunto de previsões do PO.

**Tratamento de erro (best-effort por registro):** a cópia é feita registro a registro. Se a
criação de uma cópia específica falhar (ex.: erro de permissão, validação de campo), o
script registra o erro (`log.error`) identificando qual registro de previsão de origem
falhou, e **continua** tentando copiar os demais registros de previsão do mesmo PO. Uma
falha isolada não interrompe o processamento das demais cópias nem afeta a Vendor Bill já
salva (o `afterSubmit` roda após o commit da transação).

**Fora de escopo:**
- Deduplicação de cópias entre faturamentos parciais do mesmo PO.
- Alteração do registro original de previsão (vinculado ao PO) — a cópia nunca escreve no
  registro de origem.
- Reprocessamento/backfill de Vendor Bills já criadas antes da entrada deste script em
  produção.