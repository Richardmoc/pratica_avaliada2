# BiblioTech - Análise de design e refatoração

**Responsável:** Richard Brandão

## 1 Diagnóstico do código original

A classe original GerenciadorEmprestimo concentra consultas SQL, controle de estoque, prazo de empréstimo, criação de reservas, cálculo e gravação de multas, montagem e envio de emails, autenticação SMTP e geração de PDF. Essas responsabilidades mudam por motivos diferentes, por isso a classe tem baixa coesão. Os métodos precisam conhecer detalhes de várias tecnologias e do esquema do banco, elevando o acoplamento.

### Princípios SOLID

| Princípio | Diagnóstico fundamentado | Refatoração adotada |
|-----------|-------------------------|---------------------|
| SRP | Violação direta: uma mudança de SMTP, layout de comprovante, esquema SQL ou regra de multa obriga a alterar a mesma classe. | Repositórios, adaptadores de notificação e relatório, calculadora pura e orquestrador separados. |
| OCP | Há pontos de extensão fechados: trocar SMTP por outro canal, PDF por outro formato ou cálculo de multa exige editar o gerenciador. | Contratos INotificador, IRelatorio e ICalculadoraMulta permitem substituir implementações sem mudar o fluxo. |
| DIP | A lógica de aplicação instancia diretamente sqlite3, smtplib e canvas, dependendo de detalhes concretos. | GerenciadorEmprestimo recebe seis dependências por construtor, tipadas por abstrações; uma função de composição escolhe os adaptadores. |
| LSP | Não há hierarquia de subtipos suficiente no original para demonstrar violação de substituição. Seria incorreto alegar violação apenas porque a classe é grande. | Contratos documentam retorno e ausência de entidades. Adaptadores alternativos devem cumprir as mesmas semânticas e propagar erros de integração. |
| ISP | O original não apresenta uma interface com clientes comprovadamente obrigados a depender de operações que não usam. Há concentração de responsabilidades, mas não evidência suficiente de violação específica de ISP. | Interfaces de notificação, relatório e cálculo são pequenas. Interfaces de persistência especializadas acrescentam somente as capacidades exigidas pelo fluxo. |

Injeção de dependência, sozinha, não garante DIP: receber seis classes concretas e continuar preso a seus detalhes seria insuficiente. Por isso o gerenciador se apoia nos contratos abstratos, não em chamadas SQL, SMTP ou ReportLab.

### Problemas concretos

1. **Dependência de índices de coluna.** Expressões como `livro[4]` e `leitor[2]` dependem da ordem do SELECT. A refatoração usa registros com nomes de campos e converte o resultado em dicionário.
2. **Tratamento silencioso de falhas.** `except: pass` oculta erros de email e PDF e captura até sinais de encerramento. As integrações agora capturam `Exception`, registram a categoria da falha em log e expõem avisos ao chamador, sem registrar corpo de email ou dados pessoais.
3. **Credenciais fixas.** Host, conta e senha estavam no método. O adaptador SMTP recebe configuração externa; não há credenciais reais no projeto. O exemplo de uso lê variáveis de ambiente.
4. **Conexões sem proteção global.** Vários retornos fecham a conexão manualmente, mas uma exceção de persistência pode escapar antes do fechamento. BancoSQLite implementa contexto de conexão e rollback em caso de falha.
5. **Risco de corrida no estoque.** Ler disponibilidade e depois reduzir sem uma condição na escrita permite uso de informação desatualizada em cenários concorrentes. A nova operação ocorre em `BEGIN IMMEDIATE`, e o UPDATE exige disponibilidade positiva.
6. **Fronteira transacional difusa.** Criação do empréstimo e baixa do estoque precisam confirmar ou reverter juntas. Os repositórios do caso de uso compartilham uma conexão e uma transação externa.
7. **Captura tardia do ID.** O original consulta `lastrowid` depois de outro comando. No sqlite3, um UPDATE não necessariamente apaga esse valor; não se afirma que isso sempre produz um ID errado. A captura imediatamente após o INSERT é mais clara e evita depender de operações subsequentes.
8. **Multas duplicadas e contagem após devolução.** O original insere uma nova multa a cada chamada com atraso e usa a data atual mesmo em um empréstimo já encerrado. Agora a apuração não paga é atualizada sem duplicação, e a data de devolução encerra a contagem.
9. **Mistura de cálculo com efeitos.** Calcular multa também gravava SQL e enviava email dentro do mesmo método. CalculadoraMulta agora é pura: recebe datas e retorna Decimal. O gerenciador coordena a persistência e notificação.
10. **Relógio implícito.** Várias chamadas a datetime.now dificultam reprodução. Uma função `hoje` injetável fornece uma única referência por operação.

O código original já utiliza parâmetros nas consultas SQL. Não foi identificado SQL injection por concatenação nos trechos fornecidos; essa boa prática foi preservada.

## 2 Estrutura da solução

| Classe ou contrato | Responsabilidade |
|-------------------|------------------|
| IRepositorio | Contrato mínimo buscar/salvar; busca retorna dicionário ou None e salvar retorna identificador. |
| IRepositorioLivro | Acrescenta a retirada condicional de exemplar. |
| IRepositorioEmprestimo | Define persistência do caso de empréstimo, contexto transacional e acesso à criação de reserva/apuração de multa. |
| BancoSQLite | Controla conexão, chaves estrangeiras e commit/rollback; não aplica regras de empréstimo. |
| RepositorioLivro | Consulta e grava livros; executa a baixa condicional do estoque. |
| RepositorioLeitor | Consulta e grava cadastro de leitores. |
| RepositorioEmprestimo | Persiste empréstimos e atua como fachada dos registros associados. |
| RepositorioReserva | Implementa exclusivamente a persistência e deduplicação da reserva. |
| RepositorioMulta | Implementa exclusivamente a persistência idempotente da apuração. |
| ServicoNotificacao | Adaptador SMTP com STARTTLS, timeout e configuração recebida externamente. |
| ServicoRelatorio | Adaptador de comprovante PDF; recebe dados prontos, sem consultar o banco. |
| CalculadoraMulta | Calcula dias de atraso × tarifa; sem banco, email ou PDF. |
| GerenciadorEmprestimo | Coordena validações, operação atômica e efeitos externos pelos contratos recebidos. |
| criar_gerenciador | Monta os objetos concretos e garante conexão compartilhada. |

A fachada RepositorioEmprestimo mantém o construtor de seis dependências solicitado e evita colocar SQL de reservas e multas no gerenciador. Os adaptadores específicos continuam separados. Sua interface inclui o ciclo de persistência do caso de empréstimo, não regras de negócio ou transporte. Em uma aplicação maior, uma unidade de trabalho e portas de reserva/multa explícitas poderiam substituir essa fachada.

### Contrato transacional

Todos os repositórios SQLite de uma operação devem usar **o mesmo objeto BancoSQLite**. A fábrica `criar_gerenciador` garante essa composição. O uso manual com conexões distintas não atende ao contrato atômico e não é suportado. Um adaptador alternativo precisa implementar uma transação equivalente, incluindo estoque e empréstimo.

`BEGIN IMMEDIATE` reserva a escrita antes de consultar e alterar o estoque; o UPDATE condicional acrescenta proteção contra saldo negativo. A gravação do empréstimo retorna o ID imediatamente. Só depois do commit são tentados email e PDF. Uma falha de persistência é propagada e reverte a operação; uma falha de integração é registrada em `avisos` e não apaga um empréstimo já confirmado.

Não há tentativa automática de repetir o empréstimo em caso de falha externa: isso poderia criar duas operações. Retry persistente de notificações exigiria uma outbox, prevista como evolução nos requisitos, mas não implementada no banco legado.

## 3 Funcionalidades preservadas e ajustes

| Cenário | Resultado da refatoração |
|---------|-------------------------|
| Livro inexistente | `(False, "Livro não encontrado")`, sem escrita. |
| Leitor inexistente | `(False, "Leitor não encontrado")`, sem escrita. |
| Exemplar disponível | Empréstimo com 14 dias, redução de estoque, tentativa de email e PDF; retorno de sucesso no formato original. |
| Livro indisponível | Cria reserva e retorna False com mensagem, como no original. Reserva repetida é reconhecida sem nova linha. |
| Empréstimo inexistente ao calcular multa | Retorna 0.0, preservando o resultado numérico original. |
| Sem atraso | Retorna 0.0 e não cria multa. |
| Com atraso | Calcula R$ 2,00 por dia, persiste a apuração e tenta notificar quando o valor é criado ou alterado. |
| Repetição da mesma apuração | Retorna o mesmo valor, sem nova linha nem nova notificação. |
| Falha no email/PDF | Mantém o empréstimo confirmado e expõe avisos operacionais. |

O gerenciador recebe repositórios e serviços; não conserva o construtor antigo por `db_path`. Essa alteração é deliberada para atender à injeção de dependências pedida. Os nomes das classes requeridas e os métodos públicos `realizar_emprestimo` e `calcular_multa` foram mantidos. Não há acesso aos testes privados do professor, portanto a verificação local não é uma garantia sobre esses testes.

## 4 Limites e decisões explícitas

- A modelagem cobre devolução, renovação, exemplares individuais e notificação do próximo reservado; a Questão 3 refatora o código original, que não implementa esses fluxos. Não se afirma que a aplicação completa foi construída.
- Na refatoração, situação do leitor significa cadastro existente. Não se adiciona bloqueio por multas sem regra definida e suporte no esquema.
- O esquema SQLite é preservado. O cálculo usa Decimal; a gravação converte para REAL porque esse é o tipo legado. Para um sistema financeiro completo, centavos inteiros ou armazenamento decimal exigiriam migração.
- Recalcular não altera multa já paga. O retorno representa a apuração de atraso, não um saldo financeiro a cobrar. Pagamento antecipado com atraso posterior exige definição adicional do cliente.
- A idempotência pressupõe uma apuração por empréstimo criada por esta versão. Linhas duplicadas deixadas por sistemas anteriores não são apagadas automaticamente; o banco fornecido começa sem multas.
- Os dados de livro e leitor recebidos por `salvar` são dicionários já validados pela camada de cadastro. A refatoração não é um formulário de cadastro nem valida dígitos de CPF/ISBN.
- BancoSQLite não compartilha a mesma conexão entre threads. Cada trabalhador concorrente abre seu próprio contexto; dentro de cada trabalhador, os repositórios compartilham esse contexto.
- O serviço SMTP foi implementado, mas nenhuma mensagem real foi enviada. A demonstração substitui o transporte por captura local. O PDF foi gerado e conferido localmente.

## 5 Verificação

A demonstração usa uma cópia temporária do banco fornecido e relógio controlado. A validação adicional verificou rollback por falha injetada, concorrência pelo último exemplar, falhas externas visíveis e preservação do banco original. Os resultados e o alcance estão em [validacao.md](validacao.md).

Os scripts de testes privados do professor não foram alterados ou presumidos. Não é necessário instalar pytest para executar a solução; ReportLab é a única dependência externa usada pelo adaptador PDF.

## Referências

- Enunciado fornecido, preservado em [ENUNCIADO.md](ENUNCIADO.md).
- [Python - sqlite3](https://docs.python.org/3/library/sqlite3.html), para transações, acesso por nome de coluna e cursores.
- [Python - abc](https://docs.python.org/3/library/abc.html), para contratos abstratos.
