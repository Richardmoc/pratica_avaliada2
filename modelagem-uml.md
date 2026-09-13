# BiblioTech - Modelagem UML

**Responsável:** Richard Brandão

Os três diagramas representam o produto completo da descrição. O código da Questão 3 preserva o recorte do gerenciador original. O SQLite fornecido registra quantidade disponível por livro e não possui tabelas de exemplares individuais ou bibliotecários. Implementar todo este modelo demandaria uma migração, que não foi feita na refatoração.

## 1 Diagrama de classes

```mermaid
classDiagram
    direction TB
    class Livro {
        -String isbn
        -String titulo
        -String autor
        -String categoria
        +adicionarExemplar(exemplar)
        +consultarDisponibilidade() int
    }
    class Exemplar {
        -int id
        -String status
        +emprestar()
        +disponibilizar()
        +separarParaReserva()
    }
    class Leitor {
        -String cpf
        -String nome
        -String email
        -String telefone
        +atualizarContato(email, telefone)
        +verificarSituacao() bool
        +solicitarReserva(livro)
    }
    class Emprestimo {
        -int id
        -Date dataEmprestimo
        -Date dataPrevista
        -Date dataDevolucao
        +registrar()
        +renovar(novaData)
        +encerrar(data)
    }
    class Reserva {
        -int id
        -DateTime dataReserva
        -String status
        -String estadoNotificacao
        +cancelar()
        +marcarNotificada()
        +marcarAtendida()
    }
    class Bibliotecario {
        -int id
        -String nome
        -String login
        +cadastrarLivro(dados)
        +registrarEmprestimo(leitor, exemplar)
        +registrarDevolucao(emprestimo)
    }
    class Multa {
        -int id
        -Decimal valor
        -bool paga
        +calcular(diasAtraso) Decimal
        +registrarPagamento()
    }
    Livro "1" *-- "0..*" Exemplar : possui
    Leitor "1" -- "0..*" Emprestimo : realiza
    Exemplar "1" -- "0..*" Emprestimo : integra no historico
    Bibliotecario "1" -- "0..*" Emprestimo : registra
    Livro "1" -- "0..*" Reserva : recebe
    Leitor "1" -- "0..*" Reserva : solicita
    Emprestimo "1" -- "0..1" Multa : gera
    Reserva "0..1" -- "0..1" Exemplar : separa para retirada
```

### Interpretação das cardinalidades e invariantes

- Um livro pode existir no catálogo sem exemplares e possuir vários exemplares; cada exemplar pertence a um único livro.
- Cada empréstimo vincula exatamente um leitor, um exemplar e um bibliotecário responsável. Um exemplar participa de vários empréstimos ao longo do tempo, mas de **no máximo um em aberto** em cada instante. A cardinalidade histórica `0..*` não autoriza empréstimos simultâneos.
- Cada reserva pertence a um leitor e a um livro, não a um exemplar específico na solicitação. Após uma devolução, poderá ter um exemplar separado; as duas extremidades opcionais indicam que nem toda reserva já recebeu exemplar e nem todo exemplar está separado.
- Um empréstimo gera zero ou uma apuração de multa. Pagamentos e eventual parcelamento não foram modelados.
- `dataDevolucao` é opcional enquanto o empréstimo está aberto. Status propostos para exemplar: DISPONIVEL, EMPRESTADO e SEPARADO; para reserva: ATIVA, NOTIFICADA, ATENDIDA ou CANCELADA. ATIVA e NOTIFICADA continuam bloqueando renovação.
- `verificarSituacao()` nesta versão verifica cadastro existente. Não há bloqueio por multa presumido. Métodos nas entidades expressam responsabilidades conceituais; persistência, transporte de email e PDF pertencem a serviços na implementação.

## 2 Diagrama de sequência - Realizar empréstimo

```mermaid
sequenceDiagram
    actor B as Bibliotecário
    participant S as Sistema
    participant L as Livro
    participant R as Leitor
    participant E as Emprestimo
    B->>S: realizarEmprestimo(isbn, cpf)
    activate S
    S->>L: localizar(isbn)
    activate L
    L-->>S: livro ou ausente
    deactivate L
    alt Livro inexistente
        S-->>B: Informar livro não encontrado
    else Livro localizado
        S->>R: verificarSituacao(cpf)
        activate R
        R-->>S: cadastro existente ou inválido
        deactivate R
        alt Leitor inválido
            S-->>B: Recusar sem alterar acervo
        else Leitor válido
            S->>L: consultarDisponibilidade()
            activate L
            L-->>S: quantidade disponível
            deactivate L
            alt Sem exemplar disponível
                S-->>B: Informar indisponibilidade e encaminhar reserva
            else Há exemplar disponível
                Note over S,E: Iniciar transação e revalidar estoque
                S->>L: alocarExemplarSeDisponivel()
                activate L
                L-->>S: alocado ou perdido por concorrência
                deactivate L
                alt Exemplar alocado
                    S->>E: registrar(leitor, exemplar, hoje, hoje + 14 dias)
                    activate E
                    E-->>S: identificador ou erro de persistência
                    deactivate E
                    alt Registro salvo
                        S->>S: Confirmar transação
                        S-->>B: Confirmar empréstimo e prazo
                    else Erro ao registrar
                        S->>S: Reverter transação e baixa do exemplar
                        S-->>B: Informar falha sem empréstimo parcial
                    end
                else Exemplar não alocado
                    S->>S: Encerrar transação sem empréstimo
                    S-->>B: Informar indisponibilidade e encaminhar reserva
                end
            end
        end
    end
    deactivate S
```

As ativações estão balanceadas. O fluxo distingue livro inexistente, leitor inválido e indisponibilidade. A revalidação evita vender a mesma disponibilidade duas vezes. No código refatorado, a consulta e a baixa condicional ocorrem dentro de `BEGIN IMMEDIATE`; a reserva é criada automaticamente quando não há estoque, preservando o original. A confirmação de email e o comprovante são efeitos posteriores à transação e foram omitidos do desenho para manter os cinco participantes exigidos.

## 3 Diagrama de atividades - Devolver livro e processar reservas

Mermaid não possui um tipo clássico de diagrama de atividades UML equivalente ao de classes. O fluxo é representado por `flowchart TD`, com início/fins, ações e decisões, mantendo a semântica de atividades solicitada.

```mermaid
flowchart TD
    A([Início]) --> B[Localizar empréstimo]
    B --> C{Existe e está aberto?}
    C -- Não --> X([Fim sem alteração])
    C -- Sim --> D[Iniciar transação e registrar devolução]
    D --> E{Há atraso?}
    E -- Sim --> F[Calcular dias de atraso e registrar multa]
    F --> G[Definir resultado com multa]
    E -- Não --> H[Definir resultado sem multa]
    G --> I{Há reserva ativa?}
    H --> I
    I -- Não --> J[Disponibilizar exemplar]
    J --> K[Confirmar transação]
    K --> L{Foi registrada multa?}
    L -- Sim --> M([Fim com multa e sem notificação])
    L -- Não --> N([Fim sem multa e sem notificação])
    I -- Sim --> O[Selecionar primeira reserva por data e ID]
    O --> P[Separar exemplar para o leitor selecionado]
    P --> Q[Registrar email pendente e confirmar transação]
    Q --> R[Enviar email ao primeiro leitor]
    R --> S{Envio aceito pelo provedor?}
    S -- Sim --> T[Marcar notificação realizada]
    T --> U{Foi registrada multa?}
    U -- Sim --> V([Fim com multa e com notificação])
    U -- Não --> W([Fim sem multa e com notificação])
    S -- Não --> Y[Manter prioridade e email pendente para reenvio]
    Y --> Z{Foi registrada multa?}
    Z -- Sim --> Z1([Fim com multa e notificação pendente])
    Z -- Não --> Z2([Fim sem multa e notificação pendente])
```

“Notificação” nos estados finais significa **email de disponibilidade para o próximo leitor da reserva**. Um aviso de multa é um evento separado. O desenho cobre os quatro resultados de sucesso e a falha de email sem desfazer a devolução. Aceite pelo provedor não garante leitura da mensagem pelo destinatário. Falha de persistência antes da confirmação reverte a transação; essa regra vale para ambos os ramos. A fila persistente de notificações e a separação de exemplar são propostas para o produto completo e não constam do esquema SQLite legado.

## Referências de sintaxe

- [Mermaid - Class diagrams](https://mermaid.js.org/syntax/classDiagram.html)
- [Mermaid - Sequence diagrams](https://mermaid.js.org/syntax/sequenceDiagram.html)
- [Mermaid - Flowcharts](https://mermaid.js.org/syntax/flowchart.html)
