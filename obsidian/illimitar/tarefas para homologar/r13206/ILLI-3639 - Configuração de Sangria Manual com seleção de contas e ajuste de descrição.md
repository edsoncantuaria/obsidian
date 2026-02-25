### 1) Resumo da demanda

- Tipo: `FEAT`
- Contexto funcional: A configuração de Sangria Manual deve permitir definir contas autorizadas para uso no caixa; o operador escolhe a conta no momento da sangria. Hoje, a descrição da ajuda induz leitura de conta fixa obrigatória e há risco de comportamento divergente entre configuração e operação.
- Objetivo da análise: Definir validação orientada a risco para garantir aderência funcional, integridade financeira/contábil, rastreabilidade operacional e regressão controlada no fluxo de sangria.

### 2) Escopo e fora de escopo

- Escopo: Validação da configuração `pdv_vendarapida_sangria_conta_manual`; validação da seleção de conta no fluxo de Sangria Manual no PDV; validação do texto descritivo da configuração; validação do contrato de operação (`conta_operacao`/`idContaDestino`) sem ruptura; validação de persistência (`id_conta_operacao`) e evidências de operação/impressão.
- Fora de escopo: Mudança de regra de Sangria Automática; redesign completo de UX do PDV; mudança de arquitetura de permissões; revisão funcional de Suprimento além de regressão adjacente.

### 3) Premissas e dúvidas

- Premissas adotadas: A seleção de conta ocorre no caixa entre contas previamente autorizadas; não há troca de contrato público do endpoint de sangria (permanece conta única por operação); o lançamento continua registrando `id_conta_operacao`; por padrão de validação, se não houver conta autorizada, a operação deve ser bloqueada de forma explícita e auditável.
- Lacunas de contexto: Regra oficial quando Sangria Manual está ativa sem contas autorizadas; necessidade de seleção explícita sem pré-seleção automática; texto final homologado para ajuda; regras de permissão por perfil para configurar e executar; comportamento esperado para ordenação/quantidade de contas autorizadas.
- Perguntas mínimas de clarificação (priorizadas):
1. Com Sangria Manual ativa e zero contas autorizadas, o esperado é bloquear sangria ou liberar todas as contas?
2. No PDV, a conta destino deve exigir escolha explícita do operador (sem valor default)?
3. A lista de contas autorizadas deve respeitar alguma ordenação obrigatória (cadastro, alfabética, última usada)?
4. Quais perfis podem alterar a configuração e quais perfis podem executar sangria manual?
5. Qual redação final aprovada para a ajuda da configuração de Sangria Manual?

### 4) Comportamento atual vs esperado

- Atual: A ajuda da configuração de Sangria Manual descreve obrigatoriedade de “especificar a conta” (interpretação de conta fixa); no PDV, o campo `conta_operacao` é alimentado a partir de `pdv_vendarapida_sangria_conta_manual`, com filtro por IDs configurados e pré-seleção da primeira conta; o backend processa conta única por transação e valida existência/tipo da conta.
- Esperado: Configurar contas permitidas para Sangria Manual e permitir escolha no momento da operação; remover percepção de conta fixa obrigatória; atualizar descrição para refletir seleção no caixa entre contas autorizadas.
- Diferença crítica: Divergência entre semântica operacional desejada e texto/fluxo percebido, com risco de erro operacional e lançamento em conta indevida.

### 5) Cenários de simulação

- Fluxo feliz: Configurar múltiplas contas autorizadas; abrir caixa; executar sangria manual escolhendo uma conta autorizada; confirmar sucesso de operação, impressão e persistência correta do `id_conta_operacao`.
- Cenários de borda: Apenas 1 conta autorizada; múltiplas contas autorizadas com troca durante operação; nenhuma conta autorizada; tentativa de escolher conta igual à conta do caixa; tentativa de usar conta de tipo não permitido; troca de forma de pagamento no formulário sem perder coerência da conta selecionada.
- Cenários de erro: Falha do endpoint de sangria; saldo insuficiente com política de bloqueio/autorização; conta inexistente/inválida enviada no payload; indisponibilidade de impressão; inconsistência de configuração carregada no PDV.
- Cenários de regressão: Suprimento (uso de `conta_operacao`); fechamento de caixa (totais de sangria); cancelamento de sangria; sangria automática; trilha de auditoria e suporte (mensagens e logs operacionais).

### 6) Possíveis falhas por camada

- UI: Texto de ajuda induzindo comportamento incorreto; combo exibindo contas fora da autorização; pré-seleção automática mascarando escolha do operador; mensagens de erro pouco acionáveis.
- API: Aceite de `conta_operacao` fora da lista autorizada; ausência de rejeição consistente para conta inválida; retorno sem padronização para falha de regra de negócio.
- Banco/persistência: `id_conta_operacao` gravado incorretamente/nulo; inconsistência entre extrato e comprovante; perda de vínculo em sincronismo.
- Integrações: Impressão com conta destino divergente; sincronismo com retaguarda sem consistência de conta da operação.
- Permissões: Perfil sem permissão alterando configuração crítica; operador executando sangria fora da política definida.
- Dados: Configuração multi-conta mal serializada (lista/objeto), cache desatualizado no PDV e seleção indevida por dados antigos.

### 7) Plano de testes

- Unitário: Validar transformação de configuração em lista de contas válidas; validar regra de rejeição para conta igual à conta do caixa; validar ausência de default quando exigida seleção explícita.
- Integração: Validar `POST /pdv/vendarapida/api/sangriasuprimento` com `conta_operacao` autorizado vs não autorizado; validar mensagens de falha de negócio; validar persistência de `id_conta_operacao`.
- E2E: Configurar contas permitidas na retaguarda, abrir PDV, executar sangria com seleção de conta autorizada, confirmar efeitos em saldo/fechamento/comprovante.
- Manual exploratório: Navegação por atalho F3; troca de conta sob pressão operacional; falhas transitórias (rede/impressão); validação textual da ajuda e compreensão do operador.
- Onde testar (ambiente, tela, endpoint, processo, integração): Homologação com base realista; tela de Configuração de PDV (campo `pdv_vendarapida_sangria_conta_manual` e ajuda); tela PDV Sangria Manual (`conta_operacao`); endpoint `/pdv/vendarapida/api/sangriasuprimento`; validação de extrato (`id_conta_operacao`); processo de impressão/comprovante.
- Ordem de execução sugerida por risco: 1) Fluxo financeiro crítico com múltiplas contas (ALTO); 2) Regras de bloqueio e autorização (ALTO); 3) Persistência/auditoria (ALTO); 4) Erros controlados de API/impressão (MEDIO); 5) Regressões adjacentes (MEDIO); 6) Qualidade de texto/UX (BAIXO/MEDIO).

### 8) Matriz "o que testar x impacto esperado"

| Item a testar | Onde testar | Tipo de teste | Risco | Impacto esperado se falhar | Evidência |
| --- | --- | --- | --- | --- | --- |
| Lista de contas autorizadas aparece corretamente na Sangria Manual | PDV > Sangria (`conta_operacao`) | E2E/Manual | ALTO | Transferência para conta indevida e erro operacional | Vídeo curto + print da lista |
| Bloqueio de conta não autorizada via payload manipulado | API `/pdv/vendarapida/api/sangriasuprimento` | Integração | ALTO | Quebra de controle financeiro/auditoria | Payload + resposta + log |
| Validação de conta igual à conta do caixa | PDV + API | E2E/Integração | ALTO | Movimento inválido e inconsistência contábil | Print da mensagem + registro |
| Comportamento com zero contas autorizadas | Configuração + PDV | E2E/Manual | ALTO | Paralisação de operação ou abertura indevida de contas | Print da configuração + resultado no PDV |
| Texto de ajuda da configuração reflete seleção no caixa | Configuração > ajuda do campo | Manual | MEDIO | Erro recorrente de uso e aumento de chamados | Print antes/depois e checklist UX |
| Persistência correta de `id_conta_operacao` | Banco/Extrato | Integração | ALTO | Divergência financeira e de auditoria | Consulta + ID do comprovante |
| Impressão/comprovante com conta destino correta | Processo de impressão | E2E/Manual | MEDIO | Divergência documental para conferência | Comprovante gerado |
| Regra de sangria acima do saldo (bloqueio/autorização) preservada | PDV + Autenticador + API | E2E | ALTO | Risco financeiro direto | Evidência de fluxo bloqueado/autorizado |
| Regressão em Suprimento (`conta_operacao`) | PDV Suprimento + API | E2E/Integração | MEDIO | Regressão em fluxo adjacente crítico de caixa | Execução comparativa |
| Totais de fechamento de caixa após sangria | Fechamento de caixa | E2E | ALTO | Fechamento divergente e retrabalho fiscal/operacional | Print de totais + extrato |

### 9) Impactos potenciais

- Negócio: Alto risco de erro operacional em caixa e retrabalho em fechamento quando a conta destino é mal compreendida ou mal aplicada.
- Fiscal/contábil: Lançamentos em conta indevida afetam conciliação, trilha de auditoria e consistência de extrato.
- Segurança: Falta de validação forte de conta autorizada pode permitir desvio operacional por perfil com acesso indevido.
- Performance: Baixo impacto esperado; risco pontual de latência em carga de combo/lista de contas.
- Observabilidade: Necessidade de evidência rastreável entre configuração, ação no PDV, payload, persistência e comprovante.
- Suporte: Texto ambíguo aumenta chamados e erro de operação assistida.

### 10) Dependências e riscos

- Dependências: Ambiente de homologação com múltiplas contas por entidade; usuário operador e fiscal; acesso a logs de API; acesso a extrato/consulta de dados; impressora/rotina de comprovante ativa.
- Riscos: Regra de negócio para “zero contas autorizadas” não definida; desalinhamento frontend/backend sobre autorização de contas; cache de configuração no PDV sem refresh; falso positivo por dados de teste insuficientes.
- Mitigações recomendadas: Formalizar regra de zero contas antes do ciclo final; executar testes de injeção de payload fora da UI; limpar/recarregar sessão do PDV entre cenários; exigir evidência cruzada UI+API+dados para casos ALTO.

### 11) Critérios de aceite

- [ ] A configuração de Sangria Manual permite definir contas autorizadas e o PDV apresenta somente essas contas para seleção.
- [ ] A operação de sangria permite escolha da conta no momento do uso e registra corretamente a conta escolhida.
- [ ] A descrição/ajuda da configuração comunica claramente que a seleção ocorre no caixa entre contas autorizadas.
- [ ] Tentativas com conta inválida/não autorizada são bloqueadas com mensagem controlada e rastreável.
- [ ] Não há regressão crítica em suprimento, fechamento de caixa e impressão de comprovante.
- [ ] Evidências de teste crítico (UI, API, dados) estão completas e auditáveis.

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
