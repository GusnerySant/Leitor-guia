# Guia de arguição – Sistema de Coleta Seletiva

Como o código atende os RF, RNF e RN, com perguntas e respostas para ensaiar.

## 0. Antes de começar

**Sobre o nome.** Não existe `OperacaoService` no projeto. O maior service (584 linhas) é o **`SolicitacaoService`**, e é ele que trato como foco. Se o professor falar "service de operação", é esse: ele concentra as operações sobre a solicitação (criar, agendar, recusar, cancelar, iniciar, concluir, reagendar, consultar).

**O que NÃO repito aqui e onde ler:**

| Assunto | Onde está |
| --- | --- |
| O que é o sistema, atores, fluxo, tabela de situações, as RN01–RN06 | README: seções "Atores e permissões", "Fluxo principal", "Regras de negócio", "Situações da solicitação" |
| Modelo de dados, dicionário, migrações V1–V5 e por que cada decisão de banco | `docs/relatorio.md`: "Modelo de dados" e "Decisões de projeto" |
| Pacotes, URLs, formato de erro, 400 × 409, JPA/enums, `@Version` | Guia de desenvolvimento: §2.1, §2.2, §2.3, §2.6 e §2.8 |
| Unidades de peça e regras da equipe (morador inativo, iniciar antes da data) | Guia §2.9 |
| Ordem de trabalho e divisão da equipe | Guia §3 (só se perguntarem sobre o processo) |

**Regra de ouro para a arguição:** o Guia (§5, "Definição de pronto") diz que *todos serão questionados sobre qualquer parte*. Se não souber um detalhe, diga em qual classe ele mora e qual regra o motiva.

---

## 1. Mapa rápido do código

Pacote base `br.edu.ifs.sistema_coleta_seletiva`. Cada requisição percorre sempre o mesmo caminho:

**Controller** (recebe JSON, `@Valid`, chama o service, não tem regra) → **Service** (regras de negócio e `@Transactional`) → **Repository** (Spring Data JPA) → **Model/Entidade** → banco (PostgreSQL, migrações Flyway). A resposta volta como **DTO** (`record`), nunca como entidade. Qualquer erro vira uma exceção do projeto e o **`TratadorGlobalDeErros`** devolve o JSON padrão.

| Camada | Classes principais |
| --- | --- |
| controller | `SolicitacaoController`, `MaterialController`, `PontoDeColetaController`, `MaterialAceitoController`, `MoradorController`, `EnderecoController`, `OperadorController`, `ResumoGerencialController` |
| service | `SolicitacaoService` (foco), `PontoDeColetaService`, `MaterialAceitoService`, `MaterialService`, `MoradorService`, `EnderecoService`, `OperadorService`, `ResumoGerencialService` e dois utilitários estáticos: `TransicoesDeSituacao` e `UnidadesDeMedida` |
| model | `SolicitacaoModel`, `ItemSolicitacaoModel`, `HistoricoSolicitacaoModel`, `MaterialModel`, `MaterialAceitoModel`, `PontoDeColetaModel`, `MoradorModel`, `EnderecoModel`, `OperadorModel` + enums `SituacaoSolicitacao` e `SituacaoStatus` |
| dto | `requisicao/` (entrada, com Bean Validation) e `resposta/` (saída; `PaginaDTO<T>` padroniza páginas; `ErroDto` padroniza erros) |
| exception | `RecursoNaoEncontradoException` (404), `RegraDeNegocioException` (409), `RequisicaoInvalidaException` (400) e `TratadorGlobalDeErros` |
| config | `OpenApiConfig` e `Atores` (constantes das tags do Swagger) |

**Dois utilitários que sustentam as regras.** São classes `final`, construtor privado, métodos `static`, sem `@Service`:

- `TransicoesDeSituacao`: o único lugar que conhece as transições permitidas (a tabela do README virou código).
- `UnidadesDeMedida`: o único lugar que sabe quais unidades são "de peça" (`un, und, pc, pç, jg`), usado para exigir quantidade inteira (RN02).

---

## 2. `SolicitacaoService` (foco)

### 2.1 Com quem ele conversa

| Colaborador | Para que o service usa |
| --- | --- |
| `SolicitacaoRepository` | `findById`, `save/saveAndFlush`, `existsByProtocolo`, `pesquisar(...)` (RF09, JPQL com `@EntityGraph`) e `contarPorSituacao()` (usado pelo resumo) |
| `MoradorRepository`, `PontoDeColetaRepository`, `EnderecoRepository`, `MaterialRepository` | Carregar e validar morador, ponto, endereço e materiais no `criar` |
| `MaterialAceitoRepository` | `findByPontoDeColetaIdAndMaterialId`: confirma que o ponto aceita o material e traz o **limite** (vínculo do RF05) |
| `TransicoesDeSituacao` | Valida toda mudança de situação (RF10–RF14) |
| `UnidadesDeMedida` | `ehDePeca` para exigir inteiro na quantidade estimada e na efetiva |
| `SolicitacaoModel` → `ItemSolicitacaoModel` e `HistoricoSolicitacaoModel` | Agregado: a solicitação tem `itens` e `historicos` com `cascade = ALL` |
| `SolicitacaoRespostaDTO.de(...)` | Converte a entidade na resposta (traz `versao`, itens com unidade e histórico) |
| `PaginaDTO.de(page, conversor)` | Converte a `Page` do Spring em `PaginaDTO` |

Quem usa o service: apenas `SolicitacaoController`. Quem é ligado a ele por consequência: `PontoDeColetaService.inativar` e `MaterialAceitoService.remover` (RN05) olham as solicitações abertas, e `ResumoGerencialService` lê o que ele gravou (itens efetivos e histórico de conclusão).

### 2.2 O agregado que ele grava

`SolicitacaoModel` é a raiz. Os pontos que o professor pode cobrar:

- `@Version Integer versao` (coluna `sol_nr_versao`): base da RN06.
- `copiaEndereco` (texto): a cópia do endereço no momento do pedido (RN04). Não há FK para `EnderecoModel`, então mudar/apagar o endereço do morador não altera a solicitação.
- `OffsetDateTime` em `dataCriacao` e `dataAgendamento` com coluna `timestamptz` (RNF05).
- `itens` e `historicos`: `@OneToMany(mappedBy="solicitacao", cascade = ALL, orphanRemoval = true)`. Por isso, salvar a solicitação salva itens e histórico **juntos**, na mesma transação, sem repository próprio. `@OrderBy` garante leitura em ordem (itens por `id`, histórico por `data, id`).
- `ItemSolicitacaoModel.unidade`: cópia da unidade do material (RN04). Guarda `quantidadeEstimada` e `quantidadeEfetiva` em colunas separadas (RF13: a estimada nunca é alterada).
- `HistoricoSolicitacaoModel`: situação anterior (nula no primeiro registro), nova situação, data, responsável, motivo e informações (RN03).
- Relacionamentos `@ManyToOne(fetch = LAZY)`, convenção do Guia §2.6.

### 2.3 O roteiro comum das ações (RF10 a RF14)

Definido no Guia (§3, "Em cada método do service que muda a situação"). Todas as ações seguem a mesma ordem, e é isso que dá previsibilidade aos códigos HTTP:

1. `buscarEntidade(id)`: carrega ou lança 404.
2. `conferirVersao(...)`: versão recebida ≠ versão do banco → 409.
3. `TransicoesDeSituacao.validar(...)`: transição inexistente ou situação final → 409.
4. Regras específicas da ação e só então altera a entidade.
5. `registrarHistorico(...)` e `saveAndFlush(...)`.

Tudo dentro de um método `@Transactional`: qualquer exceção desfaz tudo (RN06, "situação e histórico na mesma transação"). Como as validações vêm **antes** de qualquer `set`, uma rejeição deixa a solicitação exatamente como estava (critério "continua em SOLICITADA").

**Divisão de trabalho entre DTO e service (RNF03, Guia §2.3).** O que dá para saber olhando só o JSON é 400, pela Bean Validation no DTO: `@Future` na data, `@NotBlank` no motivo, `@PositiveOrZero` na quantidade efetiva, `@Positive` na estimada. O que depende do banco é 409 (ou 404), no service. É por isso que "data no passado" nem chega ao service.

### 2.4 `criar` (RF08, linhas 47–135)

`POST /v1/solicitacoes` → 201. Passos do método, já com a regra que cada um cumpre:

1. **Morador**: existe (404) e está `ativo` (409, decisão do Guia §2.9).
2. **Ponto**: existe (404), `ativo` e com `recolhimentoDomiciliar` verdadeiro (409). É a RN01.
3. **Endereço**: existe (404) e `endereco.getMorador().getId()` é o mesmo do morador (409): "o endereço pertence a quem faz a solicitação".
4. **Material repetido na lista**: `HashSet.add` devolve `false` se o id já entrou → 409 (RN01).
5. **Monta a solicitação**: protocolo (`gerarProtocoloUnico`), `copiaEndereco` (`formatarCopiaEndereco`), situação `SOLICITADA`, `OffsetDateTime.now()`.
6. **Para cada item**: material existe (404); está ativo (409); o ponto aceita (`findByPontoDeColetaIdAndMaterialId`, senão 409); `quantidadeEstimada` não passa do `limite` do vínculo (se houver); se a unidade é de peça, a quantidade tem de ser inteira (`remainder(BigDecimal.ONE).signum() != 0` → 409, RN02). Cria o `ItemSolicitacaoModel` já com `setUnidade(material.getUnidade())` (RN04).
7. **Histórico inicial**: `registrarHistorico(solicitacao, null, SOLICITADA, ...)`, anterior nulo, responsável = nome do morador (RN03).
8. **`save`**: grava solicitação, itens e histórico juntos (RN06).

**Por que "nada é gravado" quando algo falha** (critério do RF08): o `save` só acontece no último passo e o método é `@Transactional`. Se uma exceção ocorre no item 3, os itens 1 e 2 só existem em memória e a transação sofre rollback. Não há gravação parcial de solicitação, itens ou histórico.

**Pontos finos:**

- Quantidade estimada ≤ 0 e lista de itens vazia são barradas no DTO (`@Positive`, `@NotEmpty`), com 400, antes do service.
- O "limite" é comparado com `compareTo` (`BigDecimal` não se compara com `equals`) e é na **unidade do material** (RN02).
- `gerarProtocoloUnico`: formato `SOL-<epoch em segundos>-<4 dígitos aleatórios>`, em laço `do…while` enquanto `existsByProtocolo` for verdadeiro. A restrição `UNIQUE` do banco (`sol_tx_protocolo`) é a barreira final contra concorrência.
- `formatarCopiaEndereco` troca o marcador `--` por `S/N` (número) e omite o complemento `--`. Isso responde a uma consequência que o relatório deixou em aberto ("a cópia do endereço herdará o marcador, a menos que o service o omita": relatório, decisão "Número e complemento com valor padrão `--`"). Essa decisão está no service.

### 2.5 `agendar` (RF10, linhas 143–183)

`POST /{id}/agendar`, corpo: `versao`, `responsavel`, `dataHora`.

- Usa a **sobrecarga de 3 parâmetros** `TransicoesDeSituacao.validar(atual, SOLICITADA, AGENDADA)`. Motivo (explicado no código): `AGENDADA→AGENDADA` e `EM_RECOLHIMENTO→AGENDADA` existem na tabela por causa do RF14. Se só usasse `validar(atual, destino)`, "agendar" passaria numa solicitação já agendada e o critério "só SOLICITADA" quebraria.
- Data futura: `@Future` no DTO (400, com offset obrigatório, RNF05). O service só faz `setDataAgendamento(dto.dataHora())`.
- Guarda `anterior` **antes** de `setSituacao`, porque o histórico precisa dela.
- Motivo no histórico = `null` (agendar não exige justificativa).
- RN04 ("itens e endereço travados após agendar"): não há código específico, porque **não existe endpoint que altere itens ou endereço de uma solicitação**. O comentário do método diz isso e avisa que uma futura ação de correção só poderia aceitar `SOLICITADA`.

### 2.6 `recusar` (RF11, linhas 190–225)

É a ação mais simples do roteiro. `validar(atual, RECUSADA)` (2 parâmetros) basta porque só `SOLICITADA` tem `RECUSADA` como destino. O motivo é `@NotBlank` no DTO (400) e vai trimado para o histórico, que é o que o morador vê no RF09. `RECUSADA` é final: qualquer ação posterior cai na mensagem "situação final" (409).

### 2.7 `iniciarRecolhimento` (RF13, linhas 234–273)

`AGENDADA → EM_RECOLHIMENTO`. Depois da transição, aplica a regra do Guia §2.9: se `agora.isBefore(dataAgendamento)` → 409 (não pode iniciar antes da data). O `!= null` é só proteção contra dado de teste sem data. O comentário sobre **`saveAndFlush`** é importante: o `@Version` só incrementa quando o Hibernate faz *flush*; com `save` simples a resposta traria a versão antiga e a próxima ação do cliente tomaria 409. O flush também faz o conflito de lock otimista estourar dentro do método.

### 2.8 `concluirRecolhimento` (RF13, linhas 279–372)

O método mais denso. Depois dos passos 1 a 3 (`validar(atual, CONCLUIDA)`), **valida tudo antes de alterar qualquer item**:

1. Monta `Map<materialId, item>` com os itens da solicitação.
2. Para cada item informado: o material tem de ser da solicitação; não pode repetir (`Set.add`); se a unidade **copiada no item** (RN04, não a do material atual) for de peça, a quantidade efetiva tem de ser inteira.
3. Marca `algumPositivo` se alguma quantidade > 0.
4. Exige que a lista cubra **todos** os itens (tamanho do `Set` = tamanho do `Map`), e lista os que faltam na mensagem.
5. Se nenhuma quantidade é positiva → 409 ("concluir exige ao menos uma quantidade positiva"). O relatório (decisão "Quantidades validadas pela Bean Validation") antecipou isso: regra que olha vários itens não cabe na validação de campo e fica no service.
6. Só então faz `setQuantidadeEfetiva` em cada item (sem `save` por item: dirty checking do Hibernate), muda a situação e grava o histórico.

Detalhes: negativa é barrada no DTO (`@PositiveOrZero`, 400) e zero é aceito (item não recolhido). A **data do histórico é a informada pelo operador** (`dto.dataHora()`, validada com `@PastOrPresent`), não `now()`. Isso é de propósito: essa linha é a "data de conclusão" do RF15 (Guia §2.5).

### 2.9 `cancelar` (RF12, linhas 505–540)

Passos 1 e 2 iguais. Depois: se `dto.moradorId()` veio e é diferente do dono → 409 ("só cancela as próprias"); se `dto.pontoId()` veio e é diferente do ponto da solicitação → 409. Então `validar(atual, CANCELADA)`, que só admite `SOLICITADA` e `AGENDADA`; `EM_RECOLHIMENTO` e as finais dão 409. Motivo obrigatório (`@NotBlank`) e gravado no histórico. Esse método é o "botão de saída" da RN05: o operador cancela as solicitações abertas, com justificativa, para conseguir inativar o ponto.

> Atenção para a arguição: `moradorId` e `pontoId` são **opcionais** no DTO. A checagem só acontece se o cliente os enviar. Ver a seção de limitações no final.

### 2.10 `reagendar` (RF14, linhas 545–584)

Usa a sobrecarga com **conjunto de origens**: `validar(anterior, Set.of(AGENDADA, EM_RECOLHIMENTO), AGENDADA)`. Assim um único método cobre os dois casos do RF14: reagendamento (`AGENDADA→AGENDADA`) e tentativa sem sucesso (`EM_RECOLHIMENTO→AGENDADA`). A diferença só aparece no texto de `informacoes` do histórico. Pontos-chave:

- `AGENDADA→AGENDADA` está permitido **de propósito** na tabela (comentário em `TransicoesDeSituacao`).
- Cada reagendamento **acrescenta** uma linha ao histórico, sem apagar as anteriores (`historicos.add`), então "a tentativa anterior continua visível".
- Justificativa obrigatória e nova data `@Future` com offset, ambas no DTO (400).

### 2.11 Consultas: `listar` e `buscar` (RF09, linhas 395–434)

- `buscar(id)`: `@Transactional(readOnly = true)`, reaproveita `buscarEntidade` (404, RNF03) e devolve o DTO completo (protocolo, situação, agendamento, itens com unidade, histórico).
- `listar(moradorId, pontoId, situacao, dataInicial, dataFinal, pageable)`: como não há login, o **papel vira filtro de contexto** (Guia §2.4): morador manda `moradorId`, operador manda `pontoId`, administrador não manda nenhum. Isso cumpre "cada pessoa vê só o que o seu papel permite" em nível de API simulada.
- Data inicial > final → `RequisicaoInvalidaException` (400, sabe-se só pelos parâmetros).
- Conversão do período: `dataInicial.atStartOfDay(FUSO)` e **início do dia seguinte** ao `dataFinal` com `<` na consulta, para o último dia entrar inteiro. `FUSO` = `America/Sao_Paulo`.
- Quando não há período, usa `1900-01-01` e `9999-01-01` em vez de `null`, porque o PostgreSQL pode não inferir o tipo de um parâmetro nulo. O mesmo raciocínio faz a `situacao` virar sempre uma lista (`EnumSet.of` ou `EnumSet.allOf`).
- O repositório (`SolicitacaoRepository.pesquisar`) usa `(:moradorId IS NULL OR ...)` para filtros opcionais e `@EntityGraph(morador, pontoDeColeta)` para evitar N+1.
- Paginação: `@PageableDefault(size = 20, sort = "dataCriacao", DESC)` no controller e teto de 100 em `spring.data.web.pageable.max-page-size=100` (RNF04).

### 2.12 Métodos de apoio (privados)

- `buscarEntidade`: `findById` + 404.
- `conferirVersao`: compara `versao` do banco com a recebida. É a **camada manual** da RN06 (Guia §2.8, tabela das duas camadas).
- `registrarHistorico`: cria a linha, preenche anterior/nova/data/responsável(trim)/motivo/informações e a adiciona em `solicitacao.getHistoricos()`. Como o `cascade` é `ALL`, não precisa de `HistoricoRepository` (e por isso existe `HistoricoSolicitacaoRepository`, mas o service não o usa).

### 2.13 A tabela de transições em uma imagem mental

| De | Pode ir para | Ação (requisito) |
| --- | --- | --- |
| SOLICITADA | AGENDADA, RECUSADA, CANCELADA | agendar (RF10), recusar (RF11), cancelar (RF12) |
| AGENDADA | EM\_RECOLHIMENTO, CANCELADA, AGENDADA | iniciar (RF13), cancelar (RF12), reagendar (RF14) |
| EM\_RECOLHIMENTO | CONCLUIDA, AGENDADA | concluir (RF13), tentativa sem sucesso (RF14) |
| CONCLUIDA, RECUSADA, CANCELADA | (nenhuma) | situações finais |

Implementação: `Map.of(situação, Set.of(destinos))`, coleções **imutáveis**, como `static final`. Três métodos: `permite` (booleano, sem exceção), `validar(atual, destino)` (lança 409; mensagem muda se a situação for final) e as duas sobrecargas com origem esperada (usadas por agendar e reagendar). Regra do Guia: nenhuma ação escreve `if` próprio sobre a situação.

### 2.14 Concorrência (RN06) em duas camadas

Explicação completa no Guia §2.8. Resumo para falar de cabeça: o cliente lê a solicitação (versão N) e devolve `versao` na ação. `conferirVersao` rejeita se o banco já está em outra (caso comum: o operador B agiu em cima de um dado velho). O `@Version` do Hibernate cobre o caso raro de duas requisições simultâneas: o `UPDATE ... WHERE sol_nr_versao = ?` não atualiza nada e vira `ObjectOptimisticLockingFailureException`, traduzida para 409 em `TratadorGlobalDeErros.tratarConflitoDeVersao`. O `saveAndFlush` garante que a resposta já traga a versão nova.

### 2.15 Simulação de arguição: SolicitacaoService

**P1. Onde a RN01 (compatibilidade de atendimento) foi implementada?** R: No `SolicitacaoService.criar`, passos 2, 4 e 6. Ponto ativo e com recolhimento domiciliar (passo 2), material não repetido (passo 4), material ativo, aceito pelo ponto e dentro do limite (passo 6). O vínculo "ponto aceita material" vem de `MaterialAceitoRepository`, criado no RF05; a regra de só vincular material ativo está em `MaterialAceitoService.vincular`. As mensagens e o 409 vêm de `RegraDeNegocioException`.

**P2. Como vocês garantem que nada é gravado quando uma solicitação é inválida (RN06)?** R: `criar` é `@Transactional` e só chama `save` no final. Qualquer `RegraDeNegocioException` (RuntimeException) interrompe o método e desfaz a transação. Solicitação, itens e histórico são gravados juntos pelo `cascade = ALL` do `SolicitacaoModel`, então não há como gravar um sem o outro.

**P3. Como a RN04 (cópia do endereço e da unidade) aparece no código?** R: Duas cópias. `formatarCopiaEndereco(endereco)` grava um texto em `copiaEndereco`, sem chave estrangeira para `EnderecoModel`. E `itemModel.setUnidade(material.getUnidade())` copia a unidade para o item. O `concluirRecolhimento` e o `ItemSolicitacaoRepository.totalizarRecolhido` usam a unidade do **item**, não a do material. Dá para demonstrar: trocar a unidade do material por `PUT /v1/materiais/{id}` não muda solicitações antigas. O Guia (§6, item sobre trocar a unidade pelo PUT) explica por que o PUT aceita a troca sem checagem.

**P4. Qual a diferença entre `validar(atual, destino)` e `validar(atual, origem, destino)`? Por que duas versões?** R: A primeira só pergunta "essa mudança existe na tabela?". Serve para quem tem um único destino exclusivo (recusar, iniciar, concluir, cancelar). Para chegar em `AGENDADA` existem três origens (`SOLICITADA`, `AGENDADA`, `EM_RECOLHIMENTO`), cada uma de uma ação diferente. Sem a origem esperada, `agendar` aceitaria uma solicitação já agendada. As sobrecargas (uma origem ou um `Set` de origens) exigem a origem correta e mantêm toda a regra dentro de `TransicoesDeSituacao`.

**P5. Por que `AGENDADA → AGENDADA` é permitido?** R: É o reagendamento do RF14: muda a data, não a situação. Está na tabela do README e comentado em `TransicoesDeSituacao` para ninguém "corrigir" bloqueando situação igual à atual. O reagendamento é registrado no histórico mesmo sem mudar de situação (critério do RF14).

**P6. Como a RN06 trata dois operadores agindo sobre a mesma solicitação?** R: Em duas camadas (Guia §2.8). `conferirVersao` (manual, no service) rejeita quem devolve versão antiga. `@Version` (automático, Hibernate) cobre duas requisições simultâneas. Ambas resultam em 409: a primeira lança `RegraDeNegocioException`, a segunda `ObjectOptimisticLockingFailureException`, traduzida por `TratadorGlobalDeErros`. Os 409 das ações aparecem documentados no Swagger de cada endpoint.

**P7. Por que `saveAndFlush` e não `save`?** R: O `@Version` só é incrementado no flush. Sem ele, o DTO de resposta traria a versão antiga e o cliente tomaria 409 na ação seguinte. O flush também faz o conflito aparecer dentro do método.

**P8. Onde fica a regra de "não iniciar antes da data agendada"? É 400 ou 409?** R: 409, em `iniciarRecolhimento`, comparando `OffsetDateTime.now()` com `dataAgendamento`. É decisão da equipe (Guia §2.9, "Início do recolhimento") e depende de um dado gravado no banco, por isso não pode ser 400 (Guia §2.3: "se precisa consultar o banco, é 404 ou 409"). Na demonstração, o README ("Dados de demonstração") traz o `UPDATE` que "adianta o relógio" da `SOL-DEMO-007` e as solicitações `SOL-DEMO-010` (data futura, dá 409) e `SOL-DEMO-011` (data vencida, inicia).

**P9. Como o `concluirRecolhimento` garante que a quantidade estimada não muda e que o RF15 usa só a efetiva?** R: O método só chama `setQuantidadeEfetiva`; a estimada tem coluna própria (`ite_nr_qtd_efetiva` × `ite_nr_qtd_estimada`). A consulta do resumo (`totalizarRecolhido`) soma apenas `i.quantidadeEfetiva`, e só de solicitações `CONCLUIDA`.

**P10. Por que a data do histórico da conclusão é a informada pelo operador e não `now()`?** R: Porque essa linha de histórico é a data de conclusão usada no resumo gerencial por período (Guia §2.5), sem coluna nova. O DTO exige `@PastOrPresent`. Como a data é informada pelo operador, ela pode ficar anterior à de eventos anteriores, o que mexeria na ordem cronológica do histórico (`@OrderBy("data ASC, id ASC")`). É uma limitação conhecida.

**P11. Como um morador vê apenas as próprias solicitações, e o operador só as do ponto dele (RF09)?** R: Sem login, o papel é o filtro de contexto: `moradorId` ou `pontoId` na consulta (Guia §2.4, tabela de atores). `SolicitacaoRepository.pesquisar` aplica `(:moradorId IS NULL OR s.morador.id = :moradorId)`. É simulação de papel, não autenticação (README, nota em "Atores e permissões"; RNF06 pede registrar isso).

**P12. Como o período do RF09 inclui o dia final inteiro?** R: O service converte `dataFinal` em início do dia **seguinte** (`plusDays(1).atStartOfDay(FUSO)`) e a consulta usa `s.dataCriacao < :fim`. O fuso fixo `America/Sao_Paulo` evita que o dia "escorregue" se o servidor estiver em UTC.

---

## 3. Demais classes, em resumo

### 3.1 `PontoDeColetaService` e `MaterialAceitoService` (RF04, RF05, RF06, RN05)

- **`criar`** (ponto): todo ponto nasce `ativo`; só o `PontoDeColetaRepository.save`. `atualizar` troca os dados cadastrais; a situação só muda por `inativar` (RN03: sem exclusão).
- **`inativar` (RN05)**: se já está inativo, retorna 200 (idempotente). Senão, `repository.existeSolicitacaoAbertaNoPonto(id, SITUACOES_ABERTAS)` (JPQL `count(s) > 0` sobre `SolicitacaoModel`). Se há aberta (`SOLICITADA`, `AGENDADA`, `EM_RECOLHIMENTO`) → 409. O operador resolve ou cancela com `SolicitacaoService.cancelar`, e então inativa.
- **`MaterialAceitoService.vincular` (RF05)**: ponto e material existem (404); material `ativo` (409, RN01); par ainda não vinculado (`existsByPontoDeColetaIdAndMaterialId`, 409, com `uk_material_aceito` do banco como barreira final); limite fracionado para unidade de peça → 409 (RN02, via `UnidadesDeMedida`). Limite `<= 0` é 400 (`@Positive` no DTO) e há um `CHECK` no banco como segunda barreira.
- **`remover` (RN05)**: único `DELETE` físico do domínio de cadastro, porque as solicitações apontam para o **material**, não para o vínculo (comentário no código; Guia §2.9). Bloqueia com 409 se `existeSolicitacaoAbertaAfetada(pontoId, materialId)` (SQL nativo: solicitação aberta do ponto que contém o material).
- **`MaterialAceitoModel`** é entidade e não `@ManyToMany` porque a tabela associativa tem dados próprios (observações, limite). Está explicado no comentário da classe.
- **RF06 (`PontoDeColetaService.pesquisar`)**: filtros opcionais e combináveis (nome, município, bairro, `materialId`), só pontos ativos, paginados. O repositório evita `null` (texto ausente vira `""`, `materialId` ausente vira `0`) por causa da inferência de tipos do PostgreSQL, e usa `exists (subconsulta)` em vez de `JOIN`, para um ponto com vários materiais não aparecer repetido nem estragar a contagem da página. Depois, uma **segunda consulta** traz os materiais ativos de todos os pontos da página de uma vez (`buscarMateriaisAtivosDosPontos`, `join fetch`), evitando N+1 e a paginação em memória que um `JOIN FETCH` de coleção causaria. A resposta (`PontoDeColetaRespostaDTO`) não tem dados de morador nem operador (RNF06).

**P13. Como "inativar ponto bloqueado enquanto houver solicitação aberta" interage com o RF12?** R: O RF12 (`cancelar`) é a saída: ao cancelar as solicitações abertas, `existeSolicitacaoAbertaNoPonto` passa a dar `false`. O Guia §2.9 já dizia que o RN05 "só se confirma por completo depois do RF08"; o RF08 criou as solicitações, e `PontoDeColetaRepository` usa JPQL sobre `SolicitacaoModel` (no RF05 era SQL nativo porque a entidade ainda não existia).

**P14. Por que a pesquisa do RF06 faz duas consultas?** R: Paginar uma consulta que já traz coleções com `JOIN FETCH` obriga o Hibernate a carregar tudo em memória e paginar lá (ruim para o RNF04). Buscar os materiais ponto a ponto geraria 1 consulta por ponto (N+1). A solução é paginar só os pontos e depois buscar os vínculos dos ids da página numa única consulta (comentário em `pesquisar`).

### 3.2 `ResumoGerencialService` (RF15)

Só leitura. Duas partes: (1) contagem por situação, partindo de um `EnumMap` com as 6 situações em zero e sobrescrevendo com o `GROUP BY` do banco, para situações sem solicitações também aparecerem; (2) quantidades recolhidas via `ItemSolicitacaoRepository.totalizarRecolhido`: só `CONCLUIDA`, só `quantidadeEfetiva`, agrupada por **material e unidade do item** (kg e un nunca se somam), com período filtrado pela data do histórico de conclusão. O filtro `h.novaSituacao = :situacao` é essencial: sem ele o `JOIN` com o histórico repetiria cada item por linha de histórico e o `SUM` ficaria multiplicado (comentário da consulta). Período: `inicio` inclusivo e `fim` exclusivo, mesma lógica do RF09 (as constantes estão duplicadas de propósito, comentário no service).

**P15. Por que o resumo está num service separado?** R: É outro assunto (indicadores), só leitura, sem transição, versão ou histórico, e evita conflito de merge com as ações do `SolicitacaoService` (comentário na classe).

**P16. Como o RF15 evita misturar unidades?** R: O `GROUP BY m.id, m.nome, i.unidade` usa a unidade copiada no item (RN04). Se a unidade do material mudar depois, os atendimentos antigos continuam em linha separada com a unidade antiga.

### 3.3 Cadastros: `MaterialService`, `MoradorService`, `EnderecoService`, `OperadorService`

| Classe | Regra que importa |
| --- | --- |
| `MaterialService` (RF03) | `trim` no nome; `existsByNomeIgnoreCase` (409); `atualizar` com `existsByNomeIgnoreCaseAndIdNot` (o próprio material pode manter o nome); descrição ausente vira `""` (coluna `NOT NULL`); nasce `ativo`; `inativar` idempotente. A barreira final é o índice único `uk_material_nome` sobre `lower(mat_tx_nome)` (V5). Violação → `DataIntegrityViolationException` → 409 |
| `MoradorService` (RF01) | E-mail normalizado (`trim().toLowerCase()`) antes de checar `existsByEmailIgnoreCase`; `atualizar` usa `...AndIdNot`; nasce `ativo`; `inativar` em vez de excluir (RN03). Formato do e-mail e nome obrigatório vêm da Bean Validation (`@Email`, `@NotBlank`) |
| `EnderecoService` (RF02) | Vincula ao morador da URL (404 se não existir), `semNumero` grava `--`, complemento vazio grava `--`, CEP só com dígitos, UF em maiúsculas. `converterParaDto` devolve `semNumero` e oculta o `--`. `atualizar` confere que o endereço é do morador (409) |
| `OperadorService` (RF07) | Cria operador vinculado a um ponto existente (404 se o ponto não existe). Relação `@ManyToOne` (V2, 1:N). Ver limitações na seção 5 |

**P17. Onde está o "e-mail repetido, mesmo com caixa diferente ou espaços" do RF01?** R: Em `MoradorService.cadastrar`: `dto.email().trim().toLowerCase()` e `existsByEmailIgnoreCase`. A normalização está no service e a unicidade no banco (`mor_tx_email UNIQUE`), como decidido no relatório ("E-mail do morador obrigatório").

**P18. Por que o nome do material "Vidro" e "vidro" são barrados também no banco?** R: Verificar só no service deixaria passar duas requisições simultâneas. A V5 criou índice único em `lower(mat_tx_nome)`; o PostgreSQL não aceita expressão em `UNIQUE`, por isso é índice. Detalhes no relatório, decisão "Nome do material único". O espaço nas pontas é tratado pelo `trim()` do service, pois o banco não o ignora.

---

## 4. Requisitos não funcionais: como o código atende

| RNF | Como é atendido | Onde ver |
| --- | --- | --- |
| **RNF01** Arquitetura | Java 21 e Spring Boot 4.1.1, `spring-boot-starter-webmvc`, `data-jpa`, driver PostgreSQL, `validation` e Flyway no `pom.xml`. Camadas em pacotes próprios. Controller só repassa; regra no service; DTOs de entrada com Bean Validation e `@Valid` nos `POST/PUT` | `pom.xml`; Guia §2.1; README "Tecnologias" |
| **RNF03** Contrato e erros | JSON; `POST` cria (201 via `@ResponseStatus(CREATED)`), `GET` consulta, `PUT` atualiza, ações como `POST /{id}/verbo`. Formato único `ErroDto{codigo, mensagem, erros[]}`. 400/404/409/500 no `TratadorGlobalDeErros` | Guia §2.2 e §2.3; `TratadorGlobalDeErros` |
| **RNF04** Desempenho | Paginação em todas as listagens com teto em `spring.data.web.pageable.max-page-size=100` (o Spring corta pedidos maiores). Evita N+1 com `@EntityGraph` e `join fetch`. Medição com 1.000 solicitações fica fora da Unidade 1 | `application.properties`; Guia §1 e §6. Índices: ver limitações |
| **RNF05** Persistência | PKs e FKs em todas as tabelas, `UNIQUE` (e-mail de morador e de operador, protocolo, `(ponto, material)`, `(solicitação, material)`), índice único `lower(nome)` do material, `timestamptz` com `OffsetDateTime`, `@Transactional` onde grava mais de uma tabela. Hibernate em `validate`: só o Flyway altera o banco | Relatório "Modelo de dados" e "Hibernate em modo validate"; migrações V1–V5 |
| **RNF06** Segurança básica | Credenciais por variável de ambiente (`DB_USERNAME` e `DB_PASSWORD` sem padrão), `.env` ignorado pelo Git, `.env.example` só com modelo; dados de demonstração fictícios em script separado; respostas públicas saem de DTOs. Sem login: atores simulados e registrado como não sendo autenticação | README "Configuração" e "Atores e permissões"; Relatório "Credenciais fora do repositório" |
| **RNF07** Documentação | Swagger/OpenAPI (SpringDoc 3.0.3, `/swagger-ui.html`), tags por ator (classe `Atores`), `@ApiResponse` com 400/404/409 em cada endpoint. README, migrações Flyway e `docs/demo/*.sql` | README "Documentação da API" e "Dados de demonstração" |

RNF02 (interface) e RNF08 (testes) estão fora de propósito desta unidade (nota final das issues; Guia §1).

**P19. Como o formato único de erro é garantido, inclusive em erro inesperado?** R: `TratadorGlobalDeErros` (`@RestControllerAdvice`) tem um handler por tipo: `RecursoNaoEncontradoException` → 404, `RegraDeNegocioException` → 409, `MethodArgumentNotValidException` → 400 com a lista `erros` por campo, `HttpMessageNotReadableException` e `MethodArgumentTypeMismatchException` → 400, `RequisicaoInvalidaException` → 400, `ObjectOptimisticLockingFailureException` e `DataIntegrityViolationException` → 409, e `Exception` → 500 com mensagem genérica. O stack trace só vai para o log (`@Slf4j`), nunca para a resposta. Erros do próprio Spring (URL inexistente, método errado) preservam o status via `ErrorResponse`.

**P20. Por que quantidade negativa dá 400 e quantidade fracionada para peça dá 409?** R: Pela regra prática do Guia §2.3. Negativo se descobre só olhando o JSON (`@PositiveOrZero`, 400). Fracionado só é errado em função da **unidade do material**, que vem do banco, então é `RegraDeNegocioException` (409), conforme Guia §2.9.

**P21. Como a validação do corpo acontece e o que acontece se esquecerem o `@Valid`?** R: O controller declara `@Valid @RequestBody`; o Spring executa as anotações do `record` antes de chamar o service e lança `MethodArgumentNotValidException`. Sem `@Valid` as anotações seriam ignoradas. Nas listas aninhadas (`itens`) também há `@Valid` no campo, senão cada item não seria validado (comentário em `ConcluirRecolhimentoRequisicaoDTO`).

**P22. Como a persistência (RNF05) sobrevive a reinício e como o banco é criado?** R: Os dados estão no PostgreSQL, não em memória. A estrutura nasce das migrações do Flyway aplicadas na subida; o Hibernate só valida (`ddl-auto=validate`). Os scripts `docs/demo/dados_demonstracao.sql` e `limpar_dados_demonstracao.sql` carregam e limpam a demonstração fora das migrações.

---

## 5. Limitações conhecidas (respostas honestas)

O professor pode ir direto nestes pontos. Vale preparar a resposta, porque são coisas que o código mostra e que as issues deixam em aberto. Confira cada uma antes da arguição.

**Controle de acesso por ponto/morador (RF07, RF10 a RF14, último critério de cada issue, desmarcados).**

- Decisão assumida (Guia §2.4): sem login, o sistema não verifica se o operador mexe só no próprio ponto. Deve constar como limitação no relatório.
- No código: `agendar`, `recusar`, `iniciarRecolhimento` e `concluirRecolhimento` **não recebem nem comparam ponto**. `cancelar` e `reagendar` comparam `pontoId` (e `moradorId` no cancelar) **apenas se o cliente enviar**, pois são opcionais nos DTOs. Em `cancelar`, o critério "morador só cancela as próprias" só vale se `moradorId` vier.
- Resposta sugerida: é uma simulação de papel por filtro de contexto; num sistema real essa checagem sairia do usuário autenticado, no backend (RNF06).

**`OperadorService` (RF07 e RNF03).**

- `buscarPorId` e `atualizar` lançam `RuntimeException` genérica quando o operador ou o ponto não existe. O tratador global transforma isso em **500**, não em 404. Os outros services usam `RecursoNaoEncontradoException`.
- Não há checagem de e-mail duplicado no service (o método `existsByEmailIgnoreCase` existe no repositório, mas não é chamado). A duplicidade é barrada pelo `UNIQUE` do banco e chega como 409 genérico ("viola uma regra de integridade"), sem normalizar caixa e espaços como no morador.
- Corrigir é simples: trocar para `RecursoNaoEncontradoException` e chamar `existsByEmailIgnoreCase` (normalizado) em `criar` e `atualizar`.

**Índices e RNF04 (critérios ainda desmarcados).** Nas migrações há apenas `idx_ope_operador_pdc_nr_id` e `uk_material_nome`, além dos índices das restrições `UNIQUE` e das chaves primárias. Não há índices específicos para os filtros de `pesquisar` (situação, `dataCriacao`, `pdc_nr_id`, `mor_nr_id`). A medição de desempenho está explicitamente fora da Unidade 1 (Guia §1), mas o Guia inclui "índices" no escopo. Se perguntarem, diga que a paginação e o teto de 100 estão feitos e que índices sob medida (`EXPLAIN`) são o próximo passo.

**RF15, critério "o período pode ser escolhido".** O período vale só para as **quantidades recolhidas**. A contagem por situação é intencionalmente da situação atual de todas as solicitações (comentário em `contarPorSituacao`).

**RF02 e RN03 (endereços).** `EnderecoService.deletar(id)` apaga fisicamente e **não usa** o `moradorId` da URL para conferir o dono do endereço. Como a solicitação guarda uma **cópia** (RN04), apagar endereço não quebra solicitações antigas, mas a falta da conferência de dono é uma brecha. A RN03 cita apenas materiais, pontos e moradores como "não excluíveis" (Relatório "Inativação de moradores").

**Pendência já registrada pela equipe (Guia §6, "em aberto").** Ponto inativo ainda pode receber novos vínculos de material: `MaterialAceitoService.vincular` não confere a situação do ponto.

**RF01 e telefone.** O dicionário do relatório marca `mor_tx_fone` como não nulo, mas o DTO não exige telefone (só nome e e-mail, como diz o RF01). Um cadastro sem telefone pode falhar no banco e sair como 409 genérico. Vale testar no Swagger antes da apresentação e decidir se o telefone vira obrigatório ou se a coluna passa a aceitar nulo (V6).

**RN04, correção antes do agendamento.** A RN04 fala em corrigir itens e endereço antes de agendar, mas não há endpoint para isso; hoje eles já nascem imutáveis. O comentário em `agendar` já registra isso.

**Testes.** Só existe o `MigracoesFlywayTest` (convenções das migrações). Testes de integração e do rollback são da Unidade 2 (Guia §1; Relatório "Testes automatizados").

**Pequenas inconsistências de texto.** Um comentário em `PontoDeColetaRepository` cita o atributo `pontoColeta`, mas a entidade usa `pontoDeColeta`; o Guia cita `existsByPontoDeColetaIdAndMaterialId` para o RF08, mas `criar` usa `findByPontoDeColetaIdAndMaterialId` (para pegar o limite). Não afetam o comportamento.

---

## 6. Matrizes de consulta rápida

### 6.1 Regras de negócio → código

| Regra | Onde |
| --- | --- |
| RN01 compatibilidade | `SolicitacaoService.criar` (passos 1 a 6); `MaterialAceitoService.vincular` (material ativo) |
| RN02 quantidades e unidades | `UnidadesDeMedida.ehDePeca`; `@Positive`/`@PositiveOrZero` nos DTOs; `criar` e `concluirRecolhimento`; `MaterialAceitoService.validarLimiteConformeUnidade` |
| RN03 histórico e inativação | `registrarHistorico` em todas as ações; `inativar` em `MaterialService`, `PontoDeColetaService` e `MoradorService`; sem `DELETE` para o que tem histórico |
| RN04 cópias | `formatarCopiaEndereco`; `itemModel.setUnidade(...)`; resumo agrupa por `i.unidade` |
| RN05 bloqueios no ponto | `PontoDeColetaService.inativar` e `MaterialAceitoService.remover`, com consultas de solicitações abertas |
| RN06 atomicidade e concorrência | `@Transactional` em todas as ações; `cascade = ALL`; `conferirVersao`; `@Version`; `saveAndFlush`; handler de lock otimista no tratador |

### 6.2 Requisitos funcionais → classe principal

| RF | Classe e método | Interage com |
| --- | --- | --- |
| RF01 | `MoradorService.cadastrar/atualizar/inativar` | RF08 (morador ativo) |
| RF02 | `EnderecoService` | RF08 (endereço do morador) |
| RF03 | `MaterialService` | RF05, RF08 (material ativo) |
| RF04 | `PontoDeColetaService` | RF05, RF06, RF08, RN05 |
| RF05 | `MaterialAceitoService` | RF06, RF08 (limite), RN05 |
| RF06 | `PontoDeColetaService.pesquisar` | RF03, RF04, RF05 |
| RF07 | `OperadorService` | RF04 (ponto) |
| RF08 | `SolicitacaoService.criar` | RF01 a RF05 |
| RF09 | `SolicitacaoService.listar/buscar` | RF07 (contexto do operador) |
| RF10 | `agendar` | `TransicoesDeSituacao` |
| RF11 | `recusar` | RF09 (morador vê o motivo) |
| RF12 | `cancelar` | RN05 (libera inativar ponto) |
| RF13 | `iniciarRecolhimento`, `concluirRecolhimento` | RF15 (quantidades efetivas) |
| RF14 | `reagendar` | RF10, RF13 |
| RF15 | `ResumoGerencialService` | RF13, RN04 |

**P23. Como os requisitos RF08, RF10, RF13 e RF15 conversam entre si no código?** R: O RF08 cria a solicitação com itens (estimados) e o primeiro histórico. O RF10 muda para `AGENDADA` e grava a data. O RF13 muda para `EM_RECOLHIMENTO` e depois `CONCLUIDA`, preenchendo `quantidadeEfetiva` e a data de conclusão no histórico. O RF15 lê exatamente esses dois dados: soma `quantidadeEfetiva` de solicitações `CONCLUIDA` e filtra por `his_dt_data` da linha `CONCLUIDA`. Por isso `ResumoGerencialService` não precisa de coluna nova.

**P24. Mostre o fluxo completo na demonstração.** R: Seguir o README ("Solicitações úteis para a apresentação"): `SOL-DEMO-007` (agendar, iniciar e concluir; lembrar do `UPDATE` que simula a passagem do tempo), `SOL-DEMO-009` (recusar ou cancelar), `SOL-DEMO-010` (iniciar antes da data → 409), `SOL-DEMO-014` (tentativa sem sucesso), `SOL-DEMO-001` a `006` (resumo gerencial). Carregue os dados perto da apresentação, porque as datas são relativas a `now()`.

**P25. Se pedissem para provar o 409 de versão no Swagger, como seria?** R: Consultar a solicitação (anotar `versao`), executar uma ação válida (a versão sobe), e repetir outra ação com a versão antiga: o retorno é 409 "A solicitação foi alterada por outra pessoa".
