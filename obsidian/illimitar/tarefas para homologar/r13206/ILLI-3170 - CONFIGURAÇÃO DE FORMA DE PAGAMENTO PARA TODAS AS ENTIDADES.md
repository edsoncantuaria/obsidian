### 1) Resumo da demanda

- Tipo: `FEAT` (com lacuna funcional percebida como comportamento de bug)
- Contexto funcional: No fluxo `Sistema -> Forma de Pagamento -> Associar Conta -> Incluir`, a associação conta/entidade é operacional por item e não atende a necessidade operacional de replicação em massa para todas as entidades.
- Objetivo da análise: Definir validação orientada a risco para garantir replicação em massa segura, com possibilidade de ajuste individual por entidade, sem regressão financeira/operacional.

### 2) Escopo e fora de escopo

- Escopo: Validar criação/replicação em massa de configuração de conta por forma de pagamento entre entidades; validar edição individual após replicação; validar contrato e retorno dos endpoints já envolvidos no fluxo (`/fluxo/prazo_conta/api/*`, `/fluxo/prazo/api/alterar_configuracao_financeiro`, `/fluxo/prazo/api/duplicar_configuracao_financeiro`); validar rastreabilidade operacional.
- Fora de escopo: Redesenho de arquitetura financeira, revisão de regras fiscais fora do vínculo prazo/conta, alteração de regras de cálculo de título e tarifa, mudanças de cadastro de entidade/conta fora do fluxo de Forma de Pagamento.

### 3) Premissas e dúvidas

- Premissas adotadas: A necessidade é aplicar uma configuração base de forma de pagamento para múltiplas entidades com opção de sobrescrita individual posterior; a ação deve respeitar permissões existentes; entidades desativadas não devem receber aplicação.
- Lacunas de contexto: Definição exata de “todas as entidades”; escopo exato do que é “configuração” (somente conta vs conta + configuração financeira); comportamento para entidades sem associação prévia.
- Perguntas mínimas de clarificação (priorizadas):
1. A replicação deve **criar associações inexistentes** de `prazo_conta` ou apenas atualizar entidades já associadas?
2. O pacote replicado inclui só `conta` ou também propriedades financeiras (`natureza`, `tipo título`, `previsão`, `parceiro`, `observação`)?
3. “Todas as entidades” significa todas as entidades ativas da base, do grupo do usuário ou de um recorte selecionável?
4. Quais perfis podem executar replicação em massa e quais podem apenas editar individualmente?
5. Para conflitos (entidade já possui conta/configuração distinta), a regra será sobrescrever, manter, ou exigir confirmação explícita?

### 4) Comportamento atual vs esperado

- Atual: Existe ação de “Duplicar Configuração” no grid de associação, porém o fluxo atual trabalha com destinos já existentes e replica propriedades financeiras por IDs de `prazo_conta`, sem evidência de criação em massa de associações inexistentes. Evidências: [Lista.js:26](/var/www/mesclagem/frontend/app/view/financeiro/prazo/conta/Lista.js:26), [duplicarConfiguracao/Lista.js:40](/var/www/mesclagem/frontend/app/view/financeiro/prazo/conta/duplicarConfiguracao/Lista.js:40), [Prazo.js:210](/var/www/mesclagem/frontend/app/controller/Prazo.js:210), [prazo.php:1091](/var/www/mesclagem/backend/application/controllers/fluxo/prazo.php:1091), [prazo_conta.php:52](/var/www/mesclagem/backend/application/controllers/fluxo/prazo_conta.php:52).
- Esperado: Deve existir forma operacional de aplicar uma configuração única para todas as entidades-alvo, com manutenção de ajuste individual por entidade.
- Diferença crítica: Gap entre replicação de propriedades em destinos já associados e necessidade de replicação ampla/operacional para todas as entidades.

### 5) Cenários de simulação

- Fluxo feliz: Selecionar forma de pagamento e origem válida; aplicar para todas as entidades-alvo; sistema retornar total processado; ao abrir entidades destino, conta/configuração refletidas; edição individual posterior persistindo sem reverter demais entidades.
- Cenários de borda: Base com alto volume de entidades; mistura de entidades ativas/inativas; destinos já configurados com conta diferente; origem sem configuração financeira completa; execução concorrente por dois usuários.
- Cenários de erro: Nenhum destino selecionado; origem inexistente/desativada; destino inválido/incompatível; timeout/rede; permissão insuficiente; falha parcial durante processamento exigindo rollback.
- Cenários de regressão: PDV e movimentação continuam buscando conta por entidade/prazo sem divergência ([vendarapida.php:3947](/var/www/mesclagem/backend/application/controllers/pdv/vendarapida.php:3947), [movimentacao.php:9497](/var/www/mesclagem/backend/application/controllers/movimentacao/movimentacao.php:9497)); associação manual unitária permanece funcional.

### 6) Possíveis falhas por camada

- UI: Ação de massa indisponível para perfil correto; seleção ambígua de destinos; mensagem de sucesso sem refletir quantidades reais.
- API: Aceitação de destinos fora do escopo esperado; ausência de validação robusta para destino inválido; retorno de erro genérico sem detalhe operacional.
- Banco/persistência: Duplicidade de vínculo prazo/entidade; sobrescrita indevida de configuração individual; inconsistência entre `prazo_conta` e `propriedade`.
- Integrações: Conta incorreta propagada para lançamento financeiro em PDV/movimentação.
- Permissões: Usuário sem privilégio executando replicação em massa; auditoria sem rastreio de autor/quantidade afetada.
- Dados: Replicação de campos nulos/incompletos; divergência entre origem e destino por transformação indevida.

### 7) Plano de testes

- Unitário: Validação de regras de negócio de replicação (seleção de destinos, conflito, permissão, rollback lógico); validação de mapeamento de campos replicados.
- Integração: Contrato dos endpoints de associação/replicação; códigos de retorno; atomicidade de transação sob falha parcial; persistência em `prazo_conta` e `propriedade`.
- E2E: Fluxo completo pela UI no caminho informado pelo solicitante, incluindo aplicação em massa e posterior ajuste individual.
- Manual exploratório: Cenários de usabilidade, mensagens, concorrência, filtros de entidade e consistência pós-atualização de grid.
- Onde testar (ambiente, tela, endpoint, processo, integração): Homolog com base multi-entidade; telas `financeiro/prazo/Lista` e `financeiro/prazo/conta/*`; endpoints `/fluxo/prazo_conta/api/alterar`, `/fluxo/prazo/api/duplicar_configuracao_financeiro`, `/fluxo/prazo/api/alterar_configuracao_financeiro`; processos de venda/baixa que consomem `prazo_conta`.
- Ordem de execução sugerida por risco:
1. Fluxo feliz de replicação em massa com validação em banco (`ALTO`).
2. Regras de conflito/sobrescrita e rollback (`ALTO`).
3. Regressão em PDV/movimentação (`ALTO`).
4. Permissões e auditoria (`MEDIO`).
5. Borda de volume/performance (`MEDIO`).
6. Mensageria e UX de erro controlado (`BAIXO`).

### 8) Matriz "o que testar x impacto esperado"

| Item a testar | Onde testar | Tipo de teste | Risco | Impacto esperado se falhar | Evidência |
| --- | --- | --- | --- | --- | --- |
| Replicar para todas as entidades ativas | UI + API + banco | E2E + integração | ALTO | Operação manual massiva e erro de configuração por entidade | Print da execução + query antes/depois |
| Criação de associação para entidade sem vínculo prévio | API + banco | Integração | ALTO | Entidades ficam sem conta; fluxo financeiro quebra | Payload/response + registro persistido |
| Sobrescrita em entidade já configurada | API + banco | Integração | ALTO | Perda de configuração individual sem controle | Log de alteração + diff de propriedades |
| Edição individual após replicação | UI + API | E2E | ALTO | Impossibilidade de ajuste fino por entidade | Print + consulta de entidade específica |
| Bloqueio por permissão insuficiente | UI + API | E2E + integração | MEDIO | Risco de segurança e alteração indevida | Evidência de perfil e resposta 403/negativa |
| Destino inválido/incompatível | API | Integração | MEDIO | Erro não controlado e possível inconsistência | Log de erro + rollback comprovado |
| Concorrência de duas replicações simultâneas | API + banco | Integração | MEDIO | Condição de corrida e estado final incorreto | Timestamps + estado final validado |
| Regressão no PDV (conta usada por entidade/prazo) | Processo PDV | E2E integrado | ALTO | Lançamento em conta errada e impacto financeiro | Comprovante de lançamento + id_conta |
| Regressão em movimentação financeira | Processo movimentação | E2E integrado | ALTO | Divergência contábil/operacional | Lançamento gerado + trilha de auditoria |
| Mensagens de erro/sucesso | UI | Manual exploratório | BAIXO | Suporte elevado por falta de clareza | Captura de mensagens |
| Performance em lote (ex.: 500 entidades) | API | Não funcional | MEDIO | Timeout e indisponibilidade operacional | Tempo de execução + logs |
| Rastreabilidade de auditoria da replicação | Backend/log | Integração | MEDIO | Sem trilha para suporte e compliance | Registro de controle com autor e quantidade |

### 9) Impactos potenciais

- Negócio: Alto, por reduzir esforço manual e evitar inconsistência operacional entre filiais/entidades.
- Fiscal/contábil: Médio/alto, por risco de lançamento em conta inadequada em processos financeiros.
- Segurança: Médio, devido necessidade de segregação de acesso para ação em massa.
- Performance: Médio, especialmente em bases com grande número de entidades.
- Observabilidade: Alto, pois a operação precisa ser auditável (quem aplicou, quando, quantas entidades).
- Suporte: Alto, sem mensagens claras e evidência de processamento.

### 10) Dependências e riscos

- Dependências: Base de homolog com múltiplas entidades e contas; perfis com e sem permissão; dados de forma de pagamento com e sem configuração financeira; acesso a logs e banco para evidência.
- Riscos: Ambiguidade funcional sobre escopo da replicação; replicação parcial sem rollback; alteração indevida em entidade crítica; regressão em rotinas de venda/movimentação.
- Mitigações recomendadas: Congelar critérios funcionais antes da validação; priorizar suíte de risco ALTO; exigir evidência de banco + processo de negócio; bloquear encerramento sem validação de regressão crítica.

### 11) Critérios de aceite

- [ ] Critério funcional 1: A aplicação em massa atende ao escopo acordado de entidades e retorna quantidade processada coerente.
- [ ] Critério funcional 2: É possível ajustar entidade individualmente após a replicação sem efeito colateral nas demais.
- [ ] Critério não funcional 1: Operação em lote no volume acordado executa sem inconsistência de dados e com tempo aceitável.

### 12) Critérios de conclusão (Definition of Done)

`Pode fechar` quando:

- [ ] Todos os critérios de aceite atendidos.
- [ ] Testes de risco ALTO executados e aprovados.
- [ ] Regressões críticas sem falhas abertas.
- [ ] Evidências anexadas e rastreáveis.
- [ ] Sem bloqueios de negócio/fiscal/operação.

`Não pode fechar` quando:

- [ ] Existe falha crítica sem plano aprovado.
- [ ] Existe dúvida funcional crítica sem definição.
- [ ] Não há evidência mínima dos testes críticos.
- [ ] Há risco fiscal/financeiro sem validação.

### 13) Checklist final de validação e evidências

- [ ] Evidência de fluxo feliz.
- [ ] Evidência de erro controlado.
- [ ] Evidência de regressão adjacente.
- [ ] Evidência de permissões/perfis.
- [ ] Evidência de integridade de dados.
- [ ] Evidência de impacto em integração externa (se aplicável).
- [ ] Rastreabilidade entre cenário, resultado e decisão.
