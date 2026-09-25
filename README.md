# Impressões HTML — guia para quem for editar os templates

Este repositório guarda os **templates de impressão** (HTML + CSS) de um ERP. O sistema lê cada template, troca as variáveis por dados do banco e converte o resultado em PDF com **DomPDF**. Este guia explica como isso funciona para você editar sem quebrar nada.

Leia tudo antes da primeira edição. Depois de ler, resuma em poucas linhas o que entendeu e espere o pedido.

---

## 1. Estrutura do repositório

```
Impressão/
  README.md            <- este arquivo
  <Cliente>/           <- uma pasta por cliente (Credbee, Eurotruck, Gambine, Hdlcar, Item, ...)
    <documento>.html   <- um arquivo por tipo de documento
  .history/            <- histórico do editor. IGNORAR (nunca editar nem versionar)
```

Nomes de arquivo seguem o padrão do ERP, por exemplo:
- `orcamento-form-01.html`, `pedido-form-01.html`, `pedido-form-03-global.html`
- `fatur-saida-form-01-normal-tributos.html`, `fatur-nf-saida-form-transporte2-10x6.html`
- `contrato-faturamento-saida.html`, `index.html` (contrato da Credbee)
- sufixos como `-foto`, `-sempreco`, `-semfab` indicam variações do mesmo layout.

Cada cliente tem seu próprio estilo. **Antes de criar um HTML novo, abra um parecido do mesmo módulo e siga o estilo dele.**

---

## 2. Sintaxe dos templates

Todo arquivo começa com `<!--[main]-->` e termina com `<!--[/main]-->`. Essas marcações são removidas na renderização.

**Variáveis:** `{$nome_da_variavel}`. Se a variável não existir ou não tiver dado, ela some do PDF e não dá erro.

**Loops (listas):**
```html
<!--[produtos]-->
<tr>
  <td>{$produto_descricao}</td>
  <td>{$qtde}</td>
</tr>
<!--[/produtos]-->
```
- O bloco se repete uma vez por registro. Dentro dele, as variáveis são as colunas daquele registro.
- Variável de loop usada **fora** do bloco sai em branco. Ela precisa ficar entre as tags.
- Não existe "último item" no loop, então separadores (vírgula, traço) não dá para colocar só entre itens.
- Linhas vazias costumam ser escondidas com CSS do tipo `.linha-prod- { display:none }` combinado com `class="linha-prod-{$produto_id}"`.

**Formatação automática feita pelo motor:**
- Campos numéricos cujo nome contém `preco`, `total`, `valor`, `desconto`, `subtotal`, `frete`, `seguro`, `taxa`, `aluguel`, `condominio`, `agua`, `energia`, `setup` etc. saem como dinheiro (`1.234,50`).
- Campos com `qtde`/`qtd` saem como quantidade.
- Datas `AAAA-MM-DD` viram `dd/mm/aaaa`.
- `{$data_hoje}` é a data/hora da impressão.

**Somas calculadas pelo motor** (existem quando o loop tem `qtde` / `total` / `total_bruto`):
- `{$<loop>_qtde_sum}` e `{$<loop>_total_sum}`, por exemplo `{$produtos_qtde_sum}`, `{$produtos_total_sum}`, `{$servicos_total_sum}`.
- Para os loops `produtos` e `descritivos` também existem `{$qtde_sum}` e `{$total_sum}`.

**Imagens:**
- Logo da unidade: `{$unidade_image}` (o motor converte para base64). O alias antigo `{$pessoa_unidade_logo}` também funciona.
- Logo/blocos do site (CRM): `{$site_cabecalho_01}` e variações `{$site_<tipo>}`, `{$site_<tipo>_img}`, `{$site_<tipo>_descricao}`, `{$site_logo}`.
- Fotos no fim do documento: loop `[pedido_image]` com `{$image}` (modelo em `Item/pedido-item-08-item.html` e `Gambine/*-foto.html`).
- Código de barras: `{$imagem_codigo_barras}` (e dentro de loops de produto, `{$imagem_cod_barra}` / `{$imagem_cod_outro}`).

---

## 3. De onde vêm as variáveis (a parte mais importante)

Cada template pertence a um **módulo**, definido no cadastro do relatório dentro do ERP (o módulo não aparece no HTML). O motor de renderização (`Global_ReportTest.php`, que **não** está neste repositório) escolhe:
1. uma **view "mestre"** do banco, cujas colunas viram as variáveis do cabeçalho do documento;
2. um conjunto de **loops**, cada um lendo uma view de detalhe.

**Regra de ouro:** uma variável só sai preenchida se existir na view mestre do módulo (ou no loop, quando usada dentro dele). Views de módulos diferentes têm colunas diferentes. Nunca copie variáveis de um módulo para outro sem confirmar.

### 3.1 Módulos e loops

| Módulo (nome no ERP) | View mestre | Loops disponíveis |
|---|---|---|
| Pedido de **saída**, orçamento, ordem de serviço, PDV (`tipo = saida` na tabela `pedido`) | `bi_pedido_saida` | `descritivos`, `parcelas`, `produtos`, `produtos_separar`, `servicos`, `equipamentos`, `perifericos`, `pedido_image`, `participantes`, `enderecos`, `produtos_condicao`, `produtos_condicao_null` |
| Pedido de **entrada** / compra (`tipo = entrada`) | `bi_pedido_entrada` | os mesmos loops acima, lendo as views `bi_pedido_entrada_*` |
| Faturamento / nota fiscal | `bi_fatur_nf_saida` ou `bi_fatur_nf_entrada` (conforme o tipo da nota) | `produtos`, `servicos`, `parcelas` |
| Contrato (`contrato`, `fatur_contrato`, `assinatura_digital`, `contrato_clinica`, `contrato_animal`) | `bi_fatur_contrato` | `servicos`, `servicos_plano`, `parcelas`, `produtos`, `imoveis`, `pessoas`, `veiculos`, `animais` |
| Contrato de **garantia** (`contrato_garantia`, ou contrato com `tipo = 'garantia'`) | `vw_fatur_contrato_garantia` + enriquecimento (ver 3.3) | os mesmos loops de contrato |
| Contrato de serviço / produto / veículo | `vw_fatur_contrato_servico` / `_produto` / `_veiculo` | os mesmos loops de contrato |
| PCP (`pcp`, `pcp2`, `pcp_ficha`, `pcp_apontamento`) | `bi_pcp_op`, `bi_pcp2_op`, `bi_pcp_ficha`, `bi_pcp_apontamento` | `equipamentos`, `processos`, `produtos`, `materiais`, `testes_qualidade`, `especificacoes` (varia por módulo) |
| Pessoa / análise de crédito | `bi_pessoa` / `bi_pessoa_analise_pf` | `analises_pf`, `enderecos`, `atividades` / `emails`, `fones`, `negativos`, `participacoes` |
| Produto / Serviço | `bi_produto` / `bi_servico` | `materiais`, `tabelas`, `unidades` / `movimentos`, `planos` |
| Financeiro (boleto, recibo, inadimplência, cartão, contas, pagar, receber, desconto de títulos) | `bi_financ_*`, `bi_pagar`, `bi_receber` | `titulos` (desconto de títulos); os demais sem loop |
| Outros | CRM site/briefing, clínica (`servicos`), compra (`produtos`), currículo, consórcio, balanço de estoque, comissão | ver `getSchemaMapping()` no PHP |

### 3.2 Variáveis do cabeçalho de pedido/orçamento (mais usadas)

`{$id}`, `{$cadastro}`, `{$global_status_descricao}`, `{$pessoa_unidade_descricao}`, `{$pessoa_unidade_cpf_cnpj}`, `{$p_unidade_fone1}`, `{$pessoa_id}`, `{$pessoa_descricao}`, `{$pessoa_cpf_cnpj}`, `{$pessoa_fone1}`, `{$pessoa_email}`, `{$pessoa_responsavel_descricao}`, `{$veiculo_placa}`, `{$veiculo_marca_descricao}`, `{$veiculo_modelo_descricao}`, `{$veiculo_km_atual}`, `{$logradouro}`, `{$numero}`, `{$complemento}`, `{$bairro}`, `{$cep}`, `{$local_municipio_descricao}`, `{$local_uf_descricao}`, `{$obs1}`, `{$obs2}`, `{$subtotal_produto}`, `{$subtotal_servico}`, `{$desconto_produto}`, `{$desconto_servico}`, `{$total_produto}`, `{$total_servico}`, `{$total_frete}`, `{$total_pedido}`, `{$financ_forma_pgto_descricao}`, `{$financ_prazo_pgto_descricao}`, `{$volume_peso_bruto}`, `{$volume_peso_liquido}`, `{$volume_qtde}`.

Loops típicos:
- `produtos`: `{$produto_id}`, `{$produto_descricao}`, `{$produto_referencia_descricao}`, `{$qtde}`, `{$sigla}`, `{$preco}`, `{$total_bruto}`, `{$desconto}`, `{$desc_perc}`, `{$total}`, `{$image}`
- `servicos`: `{$servico_id}`, `{$servico_descricao}`, `{$qtde}`, `{$preco}`, `{$total_bruto}`, `{$desconto}`, `{$total}`
- `parcelas`: `{$parcela}`, `{$vencimento}`, `{$valor}`, `{$financ_forma_pgto_descricao_parcela}`
- `equipamentos`: `{$descricao}`
- `participantes` (vendedor interno): `{$pessoa_descricao_participante}`

### 3.3 Contrato de garantia (Credbee)

O Credbee é uma garantia locatícia. Um contrato de garantia é um `fatur_contrato` com `tipo = 'garantia'`. Os dados vêm de `pessoa_analise` (ligada por `pessoa_analise1_id` e `pessoa_analise2_id`) e `pessoa_analise_imovel`. O PHP monta estas variáveis:

- **Unidade (a empresa da garantia):** `{$pessoa_unidade_descricao}`, `{$pes_un_fantasia}`, `{$pes_un_cpf_cnpj}`, `{$pes_un_logradouro}`, `{$pes_un_numero}`, `{$pes_un_bairro}`, `{$pes_un_end_cep}`, `{$pes_un_local_municipio_descricao}`, `{$pes_un_local_uf_sigla}`, `{$unidade_image}`
- **Locatário:** `{$pes_ana1_pes_descricao}`, `{$pes_ana1_pes_cpf_cnpj}`, `{$pes_ana1_pes_pf_rg}`, `{$pes_ana1_pes_nascimento}`, `{$pes_ana1_pes_fone1}`, `{$pes_ana1_pes_email}`
- **Corresponsável:** os mesmos campos com prefixo `pes_ana2_pes_`
- **Administrador (imobiliária):** `{$pessoa_parceiro_descricao}`, `{$pes_parc_cpf_cnpj}`, `{$pes_parc_logradouro}`, `{$pes_parc_numero}`, `{$pes_parc_complemento}`, `{$pes_parc_bairro}`, `{$pes_parc_end_cep}`, `{$pes_parc_local_municipio_descricao}`, `{$pes_parc_local_uf_sigla}`
- **Contratação:** `{$servico_plano_descricao}`, `{$servico_plano_texto_adicional}`, `{$valor_aluguel}`, `{$pai_valor_condominio}`, `{$pai_valor_agua}`, `{$pai_valor_energia}`, `{$valor_garantia_01}`, `{$taxa}`, `{$valor_setup}`
- **Imóvel:** loop `[imoveis]` com `{$logradouro}`, `{$numero}`, `{$complemento}`, `{$bairro}`, `{$cep}`, `{$local_municipio_descricao}`, `{$local_uf_sigla}`, `{$imovel_tipo_descricao}`, `{$imovel_subtipo_descricao}`
- **Representante da unidade (contrato de imobiliária):** `{$representante_legal}`, `{$representante_rg}`, `{$representante_cpf}`

Se um bloco do PDF sair vazio, o motivo mais comum é que o contrato de teste não tem aquele cadastro (sem corresponsável, sem água/energia preenchidos). Não é necessariamente erro do template.

### 3.4 Armadilhas já encontradas

- **Pedido de entrada não tem as mesmas colunas do de saída.** A view `bi_pedido_entrada` é bem menor que `bi_pedido_saida`. Estas colunas **não existem** na entrada e não devem ser esperadas: `codigo_nf`, `envio`, `envio_data`, `prazo_entrega_dia`, `total_adicional_financ`, `defeito`, `diagnostico`, `p_unidade_pj_fantasia`, `pessoa_unidade_cep`, `pessoa_unidade_logradouro/numero/complemento`, `pessoa_unidade_local_*`, `desconto_produto_perc`, `pessoa_pj_fantasia`. Se a query mestre do PHP pedir colunas que a view não tem, o cabeçalho inteiro do PDF sai vazio (só os loops aparecem).
- Só o pedido de **saída** tem `pessoa_pj_fantasia`, `{$envio}`, `{$codigo_nf}` e `{$prazo_entrega_dia}`.
- Variável com nome errado não gera erro: simplesmente some. Se um campo saiu em branco, confira primeiro o nome na view do módulo.
- Zero e vazio são coisas diferentes: valor `0` sai como `0,00`; valor `NULL` sai em branco.

---

## 4. Regras do DomPDF

- Rodapé/cabeçalho que repete em todas as páginas: `position: fixed` com offset negativo dentro da margem da página (`@page { margin: ... }`).
- `page-break-inside: avoid` funciona por linha (`tr`) ou bloco pequeno; não segura tabelas grandes inteiras.
- **Tabela que quebra de página perde a borda de cima** se usar `border-collapse: separate` com borda só em cima/esquerda da tabela. Use `border-collapse: collapse` com borda completa em cada célula.
- `position: absolute` não ocupa espaço no fluxo (altura zero).
- Se uma variável pode vir vazia, a linha continua ocupando espaço. Use a classe `.linha-*-` para esconder linhas de loop sem dado.
- Etiquetas pequenas (ex.: 10x6 cm) são muito sensíveis a tamanho de fonte e margem; teste sempre no PDF.
- Fonte padrão usada nos templates: Arial/Helvetica. A página é A4 retrato, salvo etiquetas.

---

## 5. Como testar (obrigatório ao mexer em layout)

1. Numa pasta temporária: `composer require dompdf/dompdf`.
2. Script PHP que lê o HTML, remove `<!--[main]-->`/`<!--[/main]-->`, troca as `{$variaveis}` por dados de exemplo e os loops por algumas linhas de exemplo.
3. Gere o PDF (A4 retrato) e **olhe o resultado**. Para testar quebra de página, coloque um espaçador antes da tabela até ela dividir entre duas páginas.
4. Não diga que ficou bom sem ter visto o PDF.

---

## 6. Convenções de trabalho

- Respostas em **português**, curtas e diretas.
- Edite só o arquivo pedido. Nada de refatorar o resto.
- Se receber PDF ou Word como modelo, reproduza o texto fielmente e troque só os dados variáveis por `{$variavel}`.
- **Git:** um commit por arquivo alterado, com mensagem em português dizendo o motivo. Adicione arquivos pelo nome (nunca `git add -A` nem `git add .`). Nunca inclua `.history/`, backups, dumps ou pastas de dados. Só faça `push` quando o dono do repositório pedir.
- Não invente nomes de variável. Se não tiver certeza de que existe no módulo, pergunte ou peça a lista de campos.
