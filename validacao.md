# BiblioTech - Evidências de validação

## Execução funcional

A saída abaixo foi produzida por `python src/demonstracao.py`, com banco temporário copiado do original, relógio controlado e transporte de email substituído por captura local.

```text
BiblioTech - execução local com banco temporário
Data controlada: 01/09/2026. Nenhum email é enviado.
1. Livro inexistente: (False, 'Livro não encontrado')
2. Leitor inexistente: (False, 'Leitor não encontrado')
  Notificação capturada localmente: Empréstimo Realizado
3. Empréstimo válido: (True, 'Empréstimo realizado com sucesso')
   Estoque: 3 -> 2
   Vencimento: 2026-09-15
4. Multa antes do vencimento: 0.0
  Notificação capturada localmente: Multa por Atraso
5. Multa com três dias de atraso: 6.0
6. Mesma apuração repetida: 6.0
   Quantidade de multas: 1
7. Livro indisponível: (False, 'Livro indisponível. Reserva criada.')
8. Reserva repetida: (False, 'Livro indisponível. Reserva já existente.')
   Quantidade de reservas: 1
Comprovante gerado em saida-demonstracao/comprovante_1.pdf
Banco original preservado; cópia temporária removida ao encerrar.
```

## Verificações adicionais em cópias temporárias

Foram realizadas verificações locais de integridade, incluindo falha controlada de persistência e duas conexões concorrentes disputando o último exemplar. Esses cenários não equivalem a teste de carga de 100 solicitações previsto como meta de RNF.

```text
OK - entidades inexistentes recusadas sem escrita
OK - empréstimo, prazo de 14 dias, estoque e notificação local
OK - multa zero, atraso, apuração idempotente e fim da contagem na devolução
OK - rollback de empréstimo e estoque após falha de persistência
OK - falhas externas preservam transação e produzem dois avisos
OK - reserva sem estoque e repetição sem duplicação
OK - contratos buscar e salvar dos repositórios
OK - duas solicitações concorrentes: somente um empréstimo confirmado
OK - banco original preservado byte a byte
```

O comprovante PDF foi gerado pelo adaptador real e revisado visualmente. Mensagens de falha de integração foram provocadas deliberadamente na verificação e não representaram perda da transação.

## Limites

- Não foram enviados emails reais; credenciais e entrega por provedor SMTP não foram testadas.
- Os testes privados do professor não estão disponíveis.
- Não foram comprovados os RNF de carga, segurança de usuários, backup ou reenvio persistente; são metas do produto completo.
- A publicação no GitHub e a submissão à plataforma ainda dependem da conta do aluno; um arquivo local não comprova publicação.

## Diagramas e empacotamento

Os três blocos Mermaid foram analisados e renderizados localmente com Mermaid 11.12.0. As imagens resultantes foram inspecionadas: classes, sequência e atividades. Os SVG correspondentes estão em `diagramas/`. A renderização na conta GitHub ainda deve ser conferida após a publicação, pois a versão usada pela plataforma pode diferir.

O pacote final foi conferido quanto à presença dos arquivos exigidos e à integridade do ZIP. O banco e o código original são idênticos aos anexados.
