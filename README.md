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

### Detalhamento dos Eventos
##### LancamentoRegistrado
>  Json
{
  "id": "uuid",
  "valor": 150.00,
  "tipo": "CREDITO",
  "dataHora": "2025-03-31T10:30:00Z",
  "descricao": "Venda de produtos",
  "categoria": "VENDAS",
  "usuarioId": "uuid",
  "estabelecimento": "Loja Matriz",
  "occurredAt": "2025-03-31T10:30:00Z"
}
>

#### SaldoDiarioAtualizado

>  Json
{
  "data": "2025-03-31",
  "totalCreditos": 5000.00,
  "totalDebitos": 3200.00,
  "saldo": 1800.00,
  "quantidadeTransacoes": 15,
  "atualizadoEm": "2025-03-31T10:30:05Z"
}
>

#### LancamentoCancelado
> Json
{
  "id": "uuid",
  "usuarioId": "uuid",
  "valor": 150.00,
  "dataHora": "2025-03-31T10:30:00Z",
  "canceladoEm": "2025-03-31T11:00:00Z"
}

>

#### SaldoBaixoAlertado
> Json
{
  "usuarioId": "uuid",
  "data": "2025-03-31",
  "saldoAtual": -250.00,
  "limiteAlerta": 0,
  "emailUsuario": "comerciante@exemplo.com"
}
>

#### RelatorioPeriodoGerado
> Json
{
  "periodoInicio": "2025-03-01",
  "periodoFim": "2025-03-31",
  "formato": "PDF",
  "usuarioId": "uuid",
  "geradoEm": "2025-03-31T23:59:59Z",
  "urlDownload": "/relatorios/2025-03-relatorio.pdf"
}
>

### Agregados
Agregados são raízes de consistência transacional que garantem a 

integridade dos dados.

|Agregado	|Raiz|	Atributos Principais|	Comportamentos|
| --- | --- |---|---|
|Lancamento|	LancamentoId|	valor, tipo, dataHora, descrição, categoria, status|	registrar(), cancelar(), confirmar()|
|SaldoDiario|	Data|	totalCreditos, totalDebitos, saldo, quantidadeTransacoes|	atualizar(), recalcular(), verificarLimite()|
|SaldoMensal|	AnoMes|	saldoInicial, totalEntradas, totalSaidas, saldoFinal|	consolidarMensal(), projetar()|
|RelatorioPeriodo|	Periodo|	lista de lançamentos, resumoFinanceiro|	gerar(), exportar()

### Detalhamento dos Agregados
#### Lancamento
- Regras de consistência:
  - Valor deve ser maior que zero
  - Tipo deve ser DEBITO ou CREDITO
  - DataHora não pode ser futura
  - Status só pode ser alterado de CONFIRMADO para CANCELADO
- Eventos gerados:
  - LancamentoRegistrado (ao criar)
  - LancamentoCancelado (ao cancelar)
- SaldoDiario
  - Regras de consistência:
  - Data é única (uma entrada por dia)
  - Saldo = totalCreditos - totalDebitos
  - QuantidadeTransacoes é incrementada a cada lançamento
- Eventos gerados:
   - SaldoDiarioAtualizado (a cada atualização)
   - SaldoBaixoAlertado (quando saldo < limite)
#### SaldoMensal
- Regras de consistência:
    - Consolidado a partir dos SaldoDiario do mês
    - SaldoFinal do mês = SaldoInicial do próximo mês
### RelatorioPeriodo
- Regras de consistência:
  - Período válido (início <= fim)
  - Relatório pode ser cacheado por 1 hora

### 6. Políticas / Regras de Negócio
Políticas definem comportamentos automáticos em resposta a eventos.

|Política|	Gatilho|	Ação|	Descrição|
|--|--|---|---|
|Validar |Lançamento|	LancamentoRegistrado|	Validar dados	Verifica se valor > 0, tipo válido, etc.|
|Calcular Saldo Diário|	LancamentoRegistrado|	Atualizar Saldo Diario|	Soma créditos e débitos do dia|
Verificar Limite de Alerta|	SaldoDiarioAtualizado|	Disparar alerta|	Se saldo < 0, notificar comerciante|
|Atualizar Cache (Redis)|	SaldoDiarioAtualizado|	Atualizar Redis|	Manter cache quente para consultas|
Registrar Auditoria|	Todos os eventos|	Salvar no MongoDB|	Log de todas as operações|

### Regras de Negócio Detalhadas
#### 1. Validação de Lançamento



SE valor <= 0 ENTÃO
    REJEITAR com erro "Valor deve ser maior que zero"
    
SE tipo NÃO ESTIVER EM ['DEBITO', 'CREDITO'] ENTÃO
    REJEITAR com erro "Tipo inválido"
    
SE dataHora > DataHoraAtual ENTÃO
    REJEITAR com erro "Data não pode ser futura"
    
SE descricao VAZIA ENTÃO
    REJEITAR com erro "Descrição obrigatória"


#### 2. Cálculo de Saldo Diário
RECUPERAR SaldoDiario da data
SE NÃO EXISTE ENTÃO
    CRIAR SaldoDiario com saldo = 0
    
SE tipo = 'CREDITO' ENTÃO
    saldo = saldo + valor
    totalCreditos = totalCreditos + valor
SENÃO
    saldo = saldo - valor
    totalDebitos = totalDebitos + valor
    
quantidadeTransacoes = quantidadeTransacoes + 1
ultimaAtualizacao = DataHoraAtual

#### 3. Alerta de Saldo Baixo
RECUPERAR limiteAlerta do usuário (padrão = 0)
SE saldo < limiteAlerta ENTÃO
    DISPARAR evento SaldoBaixoAlertado
    ENVIAR e-mail/Slack para comerciante

#### 4. Atualização de Cache (Redis)
CHAVE = "saldo:{data:yyyy-MM-dd}"
TTL = 24 horas
SE cache EXISTE ENTÃO
    ATUALIZAR valor do cache
SENÃO
    CRIAR cache a partir do Read Database

#### Auditoria

PARA CADA evento recebido:
    CRIAR registro com:
        - id = UUID
        - tipo = nome do evento
        - dados = JSON do evento
        - usuarioId = extraído do evento
        - ocorridoEm = timestamp
        - origem = tópico/fonte
    SALVAR no MongoDB



### 7. Leituras / Consultas (CQRS)
No padrão CQRS, as operações de leitura são separadas das operações de escrita para otimização de performance.


|Consulta|	Descrição|	Fonte de Dados|	Cache|
|--|--|--|--|
|Consultar Saldo Diário|	Obter saldo de uma data específica|	Redis (cache) ou PostgreSQL|	✅ 24h|
Obter Extrato Período|	Listar todos lançamentos de um intervalo|	PostgreSQL|	❌|
|Relatório Gerencial|	Relatório consolidado com análises|	PostgreSQL (views materializadas)|	✅ 1h|
