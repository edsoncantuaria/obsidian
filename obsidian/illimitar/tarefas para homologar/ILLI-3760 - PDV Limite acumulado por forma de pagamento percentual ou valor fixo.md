 
### 1) Resumo da demanda

- **Tipo: `BUG`**
- **Contexto funcional:** No fechamento de venda do PDV, há múltiplos lançamentos por forma de pagamento e totalizadores dinâmicos (total pago, saldo, troco). A demanda é garantir teto acumulado por forma, com configuração por percentual do total da venda ou valor fixo.

- **Objetivo da análise:** Definir validação orientada a risco para assegurar bloqueio consistente ao exceder limite acumulado por forma, com cobertura de UI e backend, preservando totalizadores e evitando bypass operacional/financeiro.

### 2) Escopo e fora de escopo

- **Escopo:** Validação do limite acumulado por forma no fluxo de pagamento PDV; cálculo por percentual ou valor fixo; bloqueio de novos lançamentos ao exceder teto; validação de configuração da forma/condição; validação de rejeição também em backend (critério obrigatório); regressão de total pago/saldo/troco, TEF, crediário, vale/troca e valor mínimo por parcela.

- **Fora de escopo:** Alterações de arquitetura; mudança de motor promocional; redesign de UX; regras fiscais de emissão além do impacto indireto de valores de pagamento.

### 3) Premissas e dúvidas

- **Premissas adotadas:** Acúmulo deve ser `somente por forma` (não segmentado por condição); bloqueio deve existir em `UI e backend`; cálculo percentual usa total da venda corrente; o campo de configuração aceita apenas um formato por vez (valor absoluto ou percentual).

- **Lacunas de contexto:** Base exata de cálculo quando há desconto/acréscimo/devolução; política de arredondamento oficial para limite e acumulado; comportamento esperado quando total da venda muda após pagamentos já lançados; código de erro/contrato padrão para rejeição no backend.


### 4) Comportamento atual vs esperado

- Atual: Pelo relato, não há bloqueio acumulado por forma. Na baseline inspecionada, já existe validação por `valor_limite_venda` no frontend PDV e exibição de “máximo permitido”, com bloqueio por mensagem; não há evidência de validação equivalente no backend de fechamento/persistência.

- Esperado: Limite acumulado por forma, configurável em percentual ou valor fixo, com bloqueio de novos lançamentos ao exceder teto, independentemente da quantidade de lançamentos, e com rejeição também no backend para impedir bypass.

- Diferença crítica: Risco de divergência entre ambiente relatado e baseline técnica; risco residual alto se bloqueio permanecer apenas no frontend.

### 5) Cenários de simulação

- Fluxo feliz: Venda R$ 100, forma Cartão com 20% => aceitar lançamentos até R$ 20 acumulados e bloquear excedente; Venda R$ 100 com limite fixo R$ 20 => aceitar até R$ 20 e bloquear excedente; múltiplos lançamentos fracionados na mesma forma devem respeitar teto acumulado.

- Cenários de borda: Limite exatamente igual ao acumulado (deve aceitar e depois bloquear novo lançamento); limite zero/nulo (sem bloqueio ou bloqueio total conforme regra oficial); mudança de total da venda após lançamentos (desconto/cancelamento item) com reavaliação do teto; centavos (R$ 19,99 + R$ 0,01).

- Cenários de erro: Configuração inválida (texto malformado); tentativa de lançar acima do limite com valor manual; tentativa de bypass via chamada direta ao backend com payload acima do teto; indisponibilidade parcial de serviço no momento da validação backend.

- Cenários de regressão: Troco/saldo/total pago; regras de valor mínimo de parcela; TEF/vale/troca/crediário; remoção de pagamento e liberação de limite; fechamento de venda e persistência financeira.

### 6) Possíveis falhas por camada

- UI: Exibir limite incorreto; não somar lançamentos anteriores da mesma forma; bloquear tarde (após inserir); mensagem ambígua; divergência entre formas com e sem condição.

- API: Endpoint de fechamento aceitar payload acima do limite; retorno sem código de erro consistente; ausência de rastreabilidade da rejeição.

- Banco/persistência: Salvar pagamentos excedentes por ausência de validação transacional; inconsistência entre total pago e distribuição por forma.

- Integrações: Fluxo TEF concluir transação externa e depois falhar por limite local sem tratamento operacional; sincronismo entre PDV local e servidor aceitar estados divergentes.

- Permissões: Perfis com permissão de override sem trilha de auditoria; operador comum contornar bloqueio.

- Dados: Configuração ambígua/inválida; arredondamento divergente entre telas; dados legados sem normalização.

### 7) Plano de testes

- Unitário: Cálculo de limite percentual/fixo; parser de configuração; comparação com acumulado; regra de arredondamento; agregação por forma independentemente de condição.

- Integração: Cadastro/alteração de configuração de forma e condição; leitura da configuração no PDV; rejeição no backend ao exceder limite; persistência íntegra quando dentro do limite.

- E2E: Fluxo completo de venda com múltiplos pagamentos na mesma forma; tentativa de exceder; ajuste por remoção de pagamento; alteração de total da venda com revalidação; fechamento concluído apenas quando aderente.

- Manual exploratório: Operação real de caixa com fracionamento intenso; combinações de forma (cartão+dinheiro+TEF); cenários de contingência e recuperação.

- Onde testar (ambiente, tela, endpoint, processo, integração): Homolog e pré-produção; cadastro financeiro de forma/condição de pagamento; janela de pagamento do PDV; endpoint de fechamento/persistência da venda no módulo `pdv/vendarapida`; integração TEF (se habilitada) e sincronismo.

- Ordem de execução sugerida por risco: 1) E2E de bloqueio financeiro e backend rejection, 2) consistência de totalizadores, 3) bordas de arredondamento e mudança de total, 4) regressões adjacentes (TEF/valor mínimo/troco), 5) observabilidade e auditoria.

### 8) Matriz "o que testar x impacto esperado"

| Item a testar | Onde testar | Tipo de teste | Risco | Impacto esperado se falhar | Evidência |
| --- | --- | --- | --- | --- | --- |
| Limite percentual acumulado por forma | PDV pagamento | E2E | ALTO | Recebimento acima da política comercial | Vídeo + print + logs |
| Limite fixo acumulado por forma | PDV pagamento | E2E | ALTO | Exposição financeira direta no caixa | Vídeo + print + logs |
| Bloqueio exato no limite (igual permitido) | PDV pagamento | Integração/E2E | MEDIO | Bloqueio indevido ou permissividade indevida | Cenário reproduzível + print |
| Acúmulo por forma independente de condição | PDV pagamento/condição | E2E | ALTO | Fragmentação indevida do controle por condição | Evidência de lançamentos cruzados |
| Rejeição backend de payload excedente | API de fechamento | Integração | ALTO | Bypass por automação/chamada direta | Request/response + log servidor |
| Reavaliação após alteração do total da venda | PDV itens/desconto + pagamento | E2E | ALTO | Inconsistência de limite após mudanças de venda | Vídeo passo a passo |
| Remoção/cancelamento de pagamento libera limite | PDV lista pagamentos | E2E | MEDIO | Bloqueio permanente incorreto | Antes/depois com totais |
| Regressão de troco/saldo/total pago | Painéis totalizadores PDV | E2E | ALTO | Diferença de caixa e retrabalho operacional | Print dos totalizadores |
| Compatibilidade com valor mínimo de parcela | Fluxo condição pagamento | Integração/E2E | MEDIO | Conflito de regras, bloqueios erráticos | Evidência de mensagens |
| Fluxo TEF com limite atingido | PDV + integração TEF | E2E/manual | ALTO | Transação externa sem conclusão local consistente | Log TEF + log PDV |
| Tratamento de configuração inválida | Cadastro prazo/condição | Unitário/Integração | MEDIO | Limite não aplicado ou aplicado incorretamente | Print cadastro + validações |
| Auditoria da rejeição (mensagem/código/trilha) | UI + backend logs | Manual/Integração | MEDIO | Suporte sem rastreabilidade | Log correlacionado |

### 9) Impactos potenciais

- Negócio: Descumprimento de política comercial e perda de governança de meios de pagamento.
- Fiscal/contábil: Divergência de composição de recebimento por forma, impactando conciliação e conferência.
- Segurança: Possibilidade de bypass se validação estiver apenas na UI.
- Performance: Revalidações frequentes no fluxo de pagamento podem degradar UX se não forem eficientes.
- Observabilidade: Sem códigos/mensagens padronizadas, investigação de incidentes fica lenta.
- Suporte: Aumento de chamados por bloqueio inesperado, diferença de centavos e necessidade de estorno manual.

### 10) Dependências e riscos

- Dependências: Ambiente com configurações de forma/condição; massa de dados com múltiplas formas; integração TEF disponível para cenários críticos; acesso a logs frontend/backend.
- Riscos: Divergência entre ambiente relatado e baseline; regra antiga de `pdvLimitacaoPagamento` interferindo em resultados; arredondamento inconsistente entre camadas; ausência de validação backend efetiva.
- Mitigações recomendadas: Executar smoke financeiro em UI e backend antes de regressão ampla; congelar massa de teste com casos de centavos; exigir evidência de rejeição backend; registrar matriz de compatibilidade com regras adjacentes (troco, valor mínimo, TEF).

### 11) Critérios de aceite

- [ ] Limite acumulado por forma respeita configuração percentual com cálculo sobre total da venda.
- [ ] Limite acumulado por forma respeita configuração fixa com bloqueio ao exceder teto.
- [ ] Acúmulo considera todos os lançamentos da mesma forma, independente de quantidade e condição.
- [ ] UI bloqueia tentativa excedente com mensagem clara e rastreável.
- [ ] Backend rejeita payload excedente com retorno consistente para impedir bypass.
- [ ] Totalizadores (total pago, saldo, troco) permanecem consistentes após bloqueios e ajustes.
- [ ] Regras adjacentes críticas (valor mínimo, TEF, troco) sem regressão crítica.

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
- [ ] Bloqueio funciona apenas na UI sem rejeição backend comprovada.

### 13) Checklist final de validação e evidências

- [ ] Evidência de fluxo feliz.
- [ ] Evidência de erro controlado.
- [ ] Evidência de regressão adjacente.
- [ ] Evidência de permissões/perfis.
- [ ] Evidência de integridade de dados.
- [ ] Evidência de impacto em integração externa (se aplicável).
- [ ] Rastreabilidade entre cenário, resultado e decisão.

