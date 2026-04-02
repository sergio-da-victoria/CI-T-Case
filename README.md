# CI&T Case - Fluxo de Caixa
####  Controle de fluxo de caixa diário com os lançamentos(débitos e créditos), também precisa de um relatório que disponibilize o saldo diário consolidado.

![Texto](/ddd-case.jpg)


### Descrição do Event Storming - Fluxo de Caixa

![Texto](/event-storming.jpg)


#### 1. Visão Geral
O Event Storming é uma técnica de modelagem colaborativa que visa mapear todo o fluxo de negócio do sistema de Controle de Fluxo de Caixa, identificando comandos, eventos, agregados, atores, políticas, leituras e sistemas externos envolvidos no processo de registro de lançamentos e consolidação diária de saldos.

| Categoria  | Descrição |
| --- | ---  |
|Comandos |	Ações iniciadas pelo usuário ou sistema|
|Eventos de Domínio |	Ocorrências importantes no negócio |
|Agregados|	Entidades que garantem consistência transacional|
|Atores / Usuários|	Quem interage com o sistema|
|Políticas / Regras de Negócio|	Regras e validações do domínio|
|Leituras (CQRS)|	Consultas otimizadas separadas dos comandos|
|Sistemas Externos|	Infraestrutura e serviços de terceiros|



### 2. Atores / Usuários

| Atores  | Descrição | Responsabilidades |
| --- | --- |---|
| Comerciante | Usuário final do sitema| Registrar lançamentos, consultar saldos, cancelar lançamentos, exportar extratos |
|Administrador|	Gestor do sistema|	Gerar relatórios gerenciais, configurar limites de alerta, auditar operações|
|Sistema (Automático)|	Processos automatizados|	Disparar alertas de saldo baixo, atualizar caches, consolidar saldos|

###  3. Comandos
Comandos representam ações ou intenções iniciadas por um ator.

|Comando|	Descrição|	Origem|
| --- | --- |---|
|Registrar Lançamento |	Criar um novo lançamento (débito ou crédito) |Comerciante|
|Consultar Saldo Diário|Buscar o saldo consolidado de uma data específica|Comerciante|
|Cancelar Lançamento|	Estornar um lançamento previamente registrado	|Comerciante|
|Gerar Relatório Período|	Produzir relatório financeiro de um intervalo	|Administrador|
|Exportar Extrato|	Gerar arquivo (PDF/Excel) com movimentações	|Comerciante|

### Detalhamento dos Comandos

#### Registrar Lançamento
- Entrada: Valor, tipo (débito/crédito), descrição, categoria, estabelecimento
- Pré-condições: Usuário autenticado, valor > 0
- Pós-condições: Lançamento persistido, evento publicado
#### Consultar Saldo Diário
- Entrada: Data (YYYY-MM-DD)
- Pré-condições: Usuário autenticado
- Pós-condições: Retorno do saldo (via cache ou banco)
### Cancelar Lançamento
- Entrada: ID do lançamento
- Pré-condições: Lançamento existe e não está cancelado
- Pós-condições: Status alterado para CANCELADO, saldo recalculado
#### Gerar Relatório Período
- Entrada: Data inicial, data final, formato (PDF/Excel)
- Pré-condições: Usuário com permissão de administrador
- Pós-condições: Arquivo gerado e disponibilizado para download
#### Exportar Extrato
- Entrada: Período, formato
- Pré-condições: Usuário autenticado
- Pós-condições: Arquivo exportado

### 4. Eventos de Domínio
Eventos representam fatos que ocorreram no sistema e são relevantes para o negócio.

|Evento|	Descrição|	Disparado por|
| --- | --- |---|
|LancamentoRegistrado|	Um novo lançamento foi criado|	Comando Registrar Lançamento|
|SaldoDiarioAtualizado|	O saldo de uma data foi recalculado|	Política de Cálculo de Saldo|
|LancamentoCancelado|	Um lançamento foi cancelado|	Comando Cancelar Lançamento|
|SaldoBaixoAlertado|	O saldo diário atingiu um limite crítico|	Política de Verificação de Limite|
|RelatorioPeriodoGerado|	Um relatório foi gerado com sucesso|	Comando Gerar Relatório|

