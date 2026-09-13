# BiblioTech - Engenharia de requisitos

**Responsável:** Richard Brandão  
**Escopo:** gestão do acervo, cadastro de leitores, empréstimos, devoluções, reservas, renovações e multas. Este documento descreve o produto solicitado. A implementação da Questão 3 refatora somente as funcionalidades existentes no código fornecido; a rastreabilidade está no final deste arquivo.

## Técnica de levantamento e decisões

Foi realizada análise documental da descrição do cliente, do esquema SQLite e do código inicial. As ações foram convertidas em requisitos funcionais; condições, prazos e valores foram separados como regras de negócio. As user stories organizam entregas de valor verificáveis. Não foi realizada entrevista com o cliente: os RNF são propostas, e as decisões não explícitas estão identificadas para validação posterior.

Prioridades: **Alta** = necessária ao fluxo central descrito; **Média** = apoio operacional proposto. Os critérios de aceitação representam resultados observáveis, sem impor a solução técnica.

## Requisitos Funcionais

| ID | Descrição | Prioridade |
|----|-----------|------------|
| RF01 | O sistema deve permitir ao bibliotecário cadastrar um livro com título, autor, ISBN, categoria e quantidade de exemplares. | Alta |
| RF02 | O sistema deve manter os exemplares do acervo e permitir consultar a disponibilidade de cada livro. | Alta |
| RF03 | O sistema deve permitir ao leitor cadastrar nome, CPF, email e telefone. | Alta |
| RF04 | O sistema deve verificar a existência do livro e a situação cadastral do leitor antes de registrar um empréstimo. | Alta |
| RF05 | O sistema deve permitir ao bibliotecário registrar um empréstimo vinculando leitor, exemplar e data do empréstimo. | Alta |
| RF06 | O sistema deve calcular e registrar a data prevista de devolução segundo RN01. | Alta |
| RF07 | O sistema deve atualizar a disponibilidade ao confirmar um empréstimo ou uma devolução. | Alta |
| RF08 | O sistema deve permitir ao leitor solicitar a reserva de um livro sem exemplares disponíveis e acompanhar sua posição na fila. | Alta |
| RF09 | O sistema deve permitir ao bibliotecário registrar a data efetiva de devolução e encerrar o empréstimo. | Alta |
| RF10 | O sistema deve apurar e registrar multa quando a devolução ou consulta do empréstimo indicar atraso. | Alta |
| RF11 | O sistema deve selecionar a primeira reserva ativa do livro após uma devolução e notificar seu leitor por email. | Alta |
| RF12 | O sistema deve permitir renovar um empréstimo em aberto somente quando não houver reservas ativas para o livro. | Alta |
| RF13 | O sistema deve permitir consultar empréstimos, datas previstas e multas registradas de um leitor. | Média |
| RF14 | O sistema deve emitir comprovante e enviar confirmação de empréstimo por email, preservando as funcionalidades do código inicial. | Média |

RF01 a RF12 derivam da descrição, incluindo decomposição das operações. RF13 é apoio operacional proposto. RF14 deriva do código fornecido, não de uma fala adicional do cliente. A posição da reserva em RF08 é uma proposta de transparência ao leitor.

## Requisitos Não-Funcionais

As métricas abaixo são metas propostas para homologação, não resultados já medidos no protótipo.

| ID | Categoria | Descrição | Métrica |
|----|-----------|-----------|---------|
| RNF01 | Desempenho | Consultas de disponibilidade e cadastro devem responder com baixa latência. | Percentil 95 de até 2 s, com 10 mil livros, 50 mil empréstimos históricos e 20 usuários simultâneos em ambiente de homologação definido. |
| RNF02 | Segurança | Operações restritas ao bibliotecário devem exigir autenticação e autorização por papel. | 100% das tentativas de escrita restrita sem credenciais ou com papel inadequado devem ser negadas nos testes de acesso. |
| RNF03 | Privacidade | Dados pessoais devem aparecer apenas para perfis autorizados e somente quando necessários à operação. | Zero senhas, CPFs completos ou corpos de email nos logs operacionais em uma revisão de todas as rotas de aplicação; transporte externo com TLS. |
| RNF04 | Usabilidade | Um bibliotecário treinado deve registrar um empréstimo com poucos passos. | Ao menos 4 de 5 participantes concluem o cenário em até 60 s, sem ajuda, após treinamento padronizado de 10 min. |
| RNF05 | Integridade | Registro de empréstimo e baixa de disponibilidade devem ser atômicos. | Em 100 tentativas concorrentes pelo último exemplar, no máximo uma é confirmada; falha injetada entre as escritas deixa ambas sem efeito. |
| RNF06 | Recuperação | O sistema deve permitir recuperação a partir de backup verificável. | Backup diário; perda máxima de 24 h de dados e restauração em até 2 h em exercício trimestral. |
| RNF07 | Confiabilidade | Falha de email ou comprovante deve ficar visível e permitir tratamento sem duplicar o empréstimo. | Toda falha gera aviso operacional; no produto completo, notificação de reserva entra em fila persistente e recebe até 3 tentativas em 15 min. |

## Regras de Negócio

| ID | Descrição |
|----|-----------|
| RN01 | O prazo padrão de empréstimo é de 14 dias corridos a partir da data do empréstimo. |
| RN02 | Somente exemplar disponível pode ser emprestado; o estoque disponível não pode ficar negativo. |
| RN03 | Uma reserva só pode ser criada se não houver exemplar disponível para o livro. |
| RN04 | Reservas ativas são atendidas em FIFO, pela data de solicitação; empates são desfeitos pelo identificador crescente. |
| RN05 | Na devolução com reservas ativas, o primeiro leitor da fila deve ser notificado por email. |
| RN06 | A renovação é proibida enquanto houver reserva ativa para o livro. |
| RN07 | A multa é de R$ 2,00 por dia corrido de atraso: max(0, data de referência - data prevista) × R$ 2,00. No vencimento não há multa. |
| RN08 | Após a devolução, a data efetiva de devolução encerra a contagem de atraso; antes dela, a referência é a data atual. |
| RN09 | Um exemplar só pode ter um empréstimo em aberto por vez. Empréstimos encerrados permanecem no histórico. |
| RN10 | Um leitor não pode manter duas reservas ativas para o mesmo livro; repetir a solicitação não altera a posição na fila. |
| RN11 | O mesmo empréstimo não deve receber lançamentos duplicados pela mesma apuração de multa; recalcular atualiza a apuração não paga existente. |
| RN12 | CPF identifica unicamente o leitor; ISBN identifica o registro bibliográfico. Exemplares do mesmo livro possuem identificadores próprios no modelo completo. |

RN01 a RN07 traduzem a descrição, com dias corridos e desempate explicitados. RN08 a RN12 são decisões complementares de consistência propostas para validação. Nenhuma regra de bloqueio automático por multa foi informada pelo cliente.

## Premissas a confirmar

1. **Situação do leitor:** nesta versão, significa cadastro existente. O banco não tem campo de bloqueio. Limite de empréstimos e impedimento por multas exigem uma decisão do cliente e não são inventados como regras vigentes.
2. **Renovação:** propõe-se acrescentar 14 dias à data prevista atual, somente em empréstimos em aberto e sem reservas. Quantidade máxima de renovações e tratamento de atrasos ainda devem ser acordados.
3. **Retirada da reserva:** propõe-se separar o exemplar devolvido para o primeiro leitor até retirada ou cancelamento. Falha de email não faz o leitor perder a vez. Prazo de retirada e expiração não foram definidos.
4. **Multa paga:** não se altera lançamento já pago. Multa adicional após pagamento antecipado e regras de quitação/isenção dependem de definição; o protótipo calcula o atraso, mas não implementa um módulo financeiro.
5. **Cadastro:** validação de formato de CPF, email e ISBN é desejável no produto. O banco fornecido contém dados de exemplo; a refatoração não muda esses registros nem exige migração cadastral.

## User Stories

### US01 - Cadastrar acervo

Como bibliotecário  
Quero cadastrar um livro e seus exemplares  
Para disponibilizar o acervo para circulação.

Critérios de Aceitação:
- [ ] Dado um ISBN novo e dados completos, quando cadastrar título, autor, categoria e 3 exemplares, então o livro poderá ser consultado com 3 exemplares disponíveis.
- [ ] Dada uma quantidade negativa, quando tentar cadastrar, então a operação será recusada sem alterar o acervo.
- [ ] Dado um ISBN já cadastrado, quando repetir o cadastro, então o sistema impedirá outro registro bibliográfico para o mesmo ISBN.

Story Points: 3

### US02 - Cadastrar leitor

Como leitor  
Quero cadastrar meus dados de identificação e contato  
Para poder solicitar empréstimos e receber avisos da biblioteca.

Critérios de Aceitação:
- [ ] Dados nome, CPF, email e telefone válidos, quando concluir o cadastro, então o leitor poderá ser localizado pelo CPF.
- [ ] Dado um CPF já existente, quando tentar criar outro cadastro, então a operação será recusada sem sobrescrever o leitor existente.
- [ ] Dado um campo obrigatório ausente, quando enviar o formulário, então o sistema indicará o campo e não criará um cadastro incompleto.

Story Points: 3

### US03 - Registrar empréstimo

Como bibliotecário  
Quero registrar o empréstimo de um exemplar disponível para um leitor cadastrado  
Para controlar a circulação e informar o prazo de devolução.

Critérios de Aceitação:
- [ ] Dado um livro com 1 exemplar disponível e leitor cadastrado, ao emprestar em 01/09/2026, então será criado um empréstimo com vencimento em 15/09/2026 e disponibilidade 0.
- [ ] Dado um CPF inexistente, quando solicitar o empréstimo, então nenhum empréstimo será criado e o estoque permanecerá igual.
- [ ] Dadas duas solicitações simultâneas pelo último exemplar, então somente uma poderá ser confirmada.
- [ ] Dada falha de email após a confirmação, então o empréstimo permanecerá registrado e o operador receberá um aviso da falha.

Story Points: 5

### US04 - Reservar livro indisponível

Como leitor  
Quero reservar um livro sem exemplares disponíveis  
Para aguardar minha vez sem precisar consultar repetidamente o acervo.

Critérios de Aceitação:
- [ ] Dado estoque disponível 0, quando solicitar uma reserva, então ela será inserida após as reservas ativas anteriores do mesmo livro.
- [ ] Dado um livro com exemplar disponível, quando solicitar reserva, então o sistema oferecerá o fluxo de empréstimo em vez de criar a reserva.
- [ ] Dada reserva ativa do mesmo leitor e livro, quando repetir a solicitação, então a fila e a posição existente permanecerão inalteradas.

Story Points: 3

### US05 - Devolver livro e encaminhar a próxima reserva

Como bibliotecário  
Quero registrar a devolução e encaminhar o livro ao próximo leitor da fila  
Para encerrar o empréstimo corretamente e manter a circulação do acervo.

Critérios de Aceitação:
- [ ] Dado vencimento em 15/09/2026 e devolução em 18/09/2026, quando registrar a devolução, então o empréstimo será encerrado e será apurada multa de R$ 6,00.
- [ ] Dada devolução até a data prevista, então nenhuma multa será criada.
- [ ] Dadas duas reservas ativas, quando devolver o livro, então somente o primeiro leitor será selecionado e receberá a notificação de disponibilidade.
- [ ] Dado um livro sem reservas, quando devolvido, então o exemplar ficará disponível sem notificação de reserva.
- [ ] Dada falha de email, então a devolução continuará registrada, a prioridade será preservada e a notificação ficará pendente de reenvio.

Story Points: 5

### US06 - Renovar empréstimo

Como leitor  
Quero renovar meu empréstimo em aberto quando não houver reservas  
Para continuar a leitura sem prejudicar quem aguarda o livro.

Critérios de Aceitação:
- [ ] Dado empréstimo em aberto com vencimento em 15/09/2026 e nenhuma reserva ativa, quando renovado, então o novo vencimento será 29/09/2026, conforme a premissa proposta.
- [ ] Dada uma reserva ativa para o livro, quando solicitar renovação, então a operação será recusada e a data prevista permanecerá igual.
- [ ] Dado um empréstimo já devolvido, quando solicitar renovação, então a operação será recusada sem reabrir o empréstimo.

Story Points: 3

## Verificação INVEST e rastreabilidade

As histórias são fatias de valor, sem prescrever telas ou SQL. São negociáveis nas premissas identificadas; têm tamanho estimável por pontos relativos e critérios objetivos. Podem ser desenvolvidas e verificadas com cadastros e empréstimos previamente preparados. Há dependências naturais do domínio, especialmente em US05 e US06; independência não significa ausência de pré-condições. Os pontos são estimativas iniciais, não horas nem compromisso de prazo.

| História | Requisitos | Regras principais | Situação na Questão 3 |
|----------|------------|-------------------|----------------------|
| US01 | RF01, RF02 | RN02, RN12 | Modelo completo; repositório disponível, sem interface de cadastro. |
| US02 | RF03 | RN12 | Modelo completo; repositório disponível, sem interface de cadastro. |
| US03 | RF04 a RF07, RF14 | RN01, RN02, RN09 | Empréstimo, estoque, email e comprovante refatorados. |
| US04 | RF08 | RN03, RN04, RN10 | Criação da reserva ao faltar estoque; consulta de posição não implementada. |
| US05 | RF07, RF09 a RF11 | RN04, RN05, RN07, RN08, RN11 | Cálculo e persistência da multa refatorados; devolução e despacho da fila apenas modelados. |
| US06 | RF12 | RN06 | Apenas modelada, como no escopo da Questão 2. |

RNF não implementados integralmente, como autenticação, backup e fila persistente de email, permanecem metas do produto. A distinção evita atribuir à refatoração funcionalidades inexistentes no código inicial.
