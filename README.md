# BiblioTech - Prática Avaliada 2 de Engenharia de Software I

**Aluno:** Richard Brandão  
**Conteúdo:** Engenharia de requisitos, modelagem UML e princípios de design/SOLID.

Este projeto contém os arquivos solicitados nas três questões. O enunciado original está preservado em [ENUNCIADO.md](ENUNCIADO.md). O banco `biblioteca.db` e o arquivo `src/gerenciador_original.py` foram preservados sem alterações.

## Entregas

| Questão | Arquivo | Conteúdo |
|---------|---------|----------|
| 1 | [requisitos.md](requisitos.md) | 14 RF, 7 RNF com métricas, 12 regras de negócio, 6 user stories com aceitação e pontos, premissas e rastreabilidade. |
| 2 | [modelagem-uml.md](modelagem-uml.md) | Diagramas Mermaid de classes, sequência e atividades; cardinalidades e explicações. |
| 3a | [analise-design.md](analise-design.md) | Violações justificadas de SRP/OCP/DIP, análise de LSP/ISP, coesão, acoplamento e decisões de refatoração. |
| 3b | [src/emprestimo_refatorado.py](src/emprestimo_refatorado.py) | Repositórios, serviços, calculadora e gerenciador com injeção por abstrações e operação transacional. |
| Evidências | [validacao.md](validacao.md) | Execução local e alcance das verificações. |

## Executar

Requer Python 3.10 ou superior e SQLite com suporte a UPSERT (3.24 ou superior, incluído nas distribuições Python usuais).

Na raiz do projeto:

```bash
python -m venv .venv
```

Ative o ambiente:

```bash
# Linux ou macOS
source .venv/bin/activate
```

```powershell
# Windows PowerShell
.venv\Scripts\Activate.ps1
```

Instale a dependência e execute:

```bash
python -m pip install -r requirements.txt
python src/demonstracao.py
```

A demonstração usa uma **cópia temporária do banco**, datas controladas e um notificador que captura mensagens localmente. Não envia emails. O comprovante fica em `saida-demonstracao/comprovante_1.pdf`; o diretório é ignorado pelo Git. Não execute o código original para a demonstração: ele contém tentativa de conexão SMTP configurada no próprio código.

## Composição das dependências

No código da aplicação, utilizando a raiz do projeto como diretório de execução:

```python
from src.emprestimo_refatorado import (
    BancoSQLite, INotificador, ServicoRelatorio, criar_gerenciador,
)

class NotificadorLocal(INotificador):
    def enviar(self, destinatario, assunto, mensagem):
        print(f"Notificação capturada: {assunto}")

with BancoSQLite("biblioteca.db") as banco:
    gestor = criar_gerenciador(
        banco,
        NotificadorLocal(),
        ServicoRelatorio("comprovantes"),
    )
    resultado = gestor.realizar_emprestimo("978-0134685991", "12345678901")
    print(resultado)
    print(gestor.avisos)
```

**Esse exemplo altera o banco indicado.** Para experimentar sem alterar o fornecido, prefira `src/demonstracao.py`. Todos os repositórios de uma operação precisam compartilhar o mesmo `BancoSQLite`; a fábrica garante isso. O contexto fecha a conexão ao final.

Para compor manualmente o construtor com as seis dependências exigidas:

```python
from src.emprestimo_refatorado import (
    RepositorioLivro, RepositorioLeitor, RepositorioEmprestimo,
    GerenciadorEmprestimo, CalculadoraMulta,
)

# Dentro do contexto BancoSQLite aberto acima:
gestor = GerenciadorEmprestimo(
    RepositorioLivro(banco),
    RepositorioLeitor(banco),
    RepositorioEmprestimo(banco),
    NotificadorLocal(),
    ServicoRelatorio("comprovantes"),
    CalculadoraMulta(),
)
```

### Adaptador SMTP

O adaptador existe para preservar a funcionalidade original, mas sua integração com um provedor real depende de configuração e credenciais. Nenhum serviço SMTP foi acionado na validação.

```python
import os
from src.emprestimo_refatorado import ServicoNotificacao

notificador = ServicoNotificacao(
    host=os.environ["SMTP_HOST"],
    porta=int(os.environ.get("SMTP_PORT", "587")),
    usuario=os.environ["SMTP_USER"],
    senha=os.environ["SMTP_PASSWORD"],
    remetente=os.environ["SMTP_FROM"],
)
```

A senha deve ser configurada fora do repositório. O adaptador usa STARTTLS e autenticação. O exemplo acima apenas constrói o objeto; o envio ocorre ao executar `enviar` ou ao injetá-lo no caso de uso.

## Limite entre modelagem e implementação

A documentação modela o produto completo. A refatoração mantém o escopo funcional do original: empréstimo, estoque, criação de reserva quando indisponível, apuração de multa, email e comprovante. Devolução, renovação, autenticação, exemplar individualizado, pagamento e reenvio persistente de notificações permanecem requisitos/modelagem e não são apresentados como funções implementadas.

A refatoração corrige duplicação de reservas/apurações, exposição de credenciais, falhas externas silenciosas e risco de operação parcial no estoque. A justificativa de cada decisão está em `analise-design.md`.

## Diagramas

Abra `modelagem-uml.md` no GitHub para visualizar os três blocos Mermaid. As versões SVG, geradas para conferência, ficam em `diagramas/` e servem como alternativa de visualização. O Markdown com Mermaid é a entrega principal exigida.

## Entregar na plataforma

O enunciado exige **um repositório GitHub e o link desse repositório**. O ZIP do projeto é um pacote para preparar a publicação; ele não substitui o link solicitado.

1. Crie um repositório na sua conta, por exemplo `es1-pratica-avaliada-2`.
2. Envie o conteúdo desta pasta para a raiz, mantendo `src/` e `diagramas/`. Inclua `biblioteca.db`, os três documentos e o código original/refatorado.
3. Verifique os diagramas na visualização do GitHub e assegure acesso do avaliador conforme as orientações da disciplina.
4. Copie o endereço do repositório e envie na plataforma de ensino.

Não inclua `.venv`, senhas, bancos temporários ou comprovantes gerados. A atividade é individual: revise as decisões e o código para conseguir explicá-los e ajuste os pontos que considerar necessários antes de entregar.
