# APS 3 - Dossiê de Arquitetura Viva

AutoFlow Monitor: Monitoramento da produção

Aluno: Phillipe Mugnaini  
Curso: Engenharia de Software, 4º período - UniSenai  
Unidade curricular: Modelagem de Software  
Docente: Prof. Sidney L. da Silva  
Referência: RF09 e UC01  
Ator principal: Operador

## Funcionalidade escolhida

Permitir que o operador acompanhe a produção por meio de um dashboard, consultando a condição da CNC, os sensores, os últimos dados recebidos e os alertas existentes.

O AutoFlow Monitor pertence ao cenário de uma fábrica de carcaças de alumínio para bombas d'água automotivas. O dossiê representa a consulta das informações já recebidas pelo sistema. Sua entrega reúne os cinco artefatos sobre essa mesma funcionalidade.

A solução recebe telemetria em segundo plano. Para visualizar o monitoramento, o operador seleciona a CNC e o período, a interface consulta a API, o serviço verifica o acesso e reúne dados do PostgreSQL. A resposta indica informações utilizáveis, dados antigos, falhas de fonte e permissões.

## Fluxo e fronteiras

1. O operador abre o dashboard e seleciona os filtros.
2. O navegador inicia uma consulta identificada, sem bloquear a interação.
3. A API valida parâmetros e autorização sobre a CNC.
4. O serviço consulta condição da CNC, sensores/leituras e alertas em paralelo.
5. O join aguarda os três resultados; qualidade, idade e disponibilidade são avaliadas.
6. A interface aplica apenas a resposta correspondente à consulta atual.
7. Um novo ciclo periódico é proposto a cada 10 segundos, conforme a Sprint 1.

O painel mostra condição da CNC, saúde dos sensores e estados dos alertas como informações separadas. Leituras antigas ou uma fonte indisponível nunca são apresentadas como confirmação de normalidade.

## Entidade do ciclo de vida

ConsultaMonitoramento representa uma execução dessa funcionalidade, com consultaId, solicitante, filtros, instantes e resultado. É um refinamento de modelagem proposto para esta APS e não exige uma tabela adicional.

A máquina de estados acompanha a consulta até sua exibição, descarte, rejeição ou falha. Ela utiliza subestados para autorização, carregamento e qualificação. Cada atualização do painel cria outra consulta e outro identificador.

A condição da CNC continua sendo um dado monitorado. Concluir uma consulta não muda o estado físico do equipamento e não reconhece ou encerra alertas.

## Artefato 1 - Diagrama de sequência

Comunicação do operador, interface, controller, serviço e repositório durante a consulta.

Fonte: [sequencia.mmd](docs/diagrams/sequencia.mmd).

```mermaid
sequenceDiagram
    autonumber
    actor OP as Operador
    participant WEB as DashboardView
    participant API as MonitoramentoController
    participant SVC as MonitoramentoService
    participant REP as MonitoramentoRepository

    OP->>WEB: abrirDashboard(ParametrosPainelDTO)
    WEB-->>OP: EstadoPainelDTO

    loop enquanto o painel estiver aberto
        WEB->>WEB: iniciarConsulta(ParametrosPainelDTO)
        WEB-->>WEB: RequisicaoMonitoramentoDTO
        WEB-)API: consultarMonitoramento(RequisicaoMonitoramentoDTO)
        activate API
        Note over WEB,API: Fetch assíncrono. O navegador permanece interativo.
        alt parametrosValidos == false
            Note over API: Monta ResultadoConsultaDTO com status REJEITADA.
        else parametrosValidos == true
            Note over API,SVC: O contexto do usuário vem da sessão ou token verificado no servidor.
            API->>SVC: consultar(ConsultaMonitoramentoDTO)
            activate SVC
            SVC->>SVC: validarAcesso(ConsultaMonitoramentoDTO)
            SVC-->>SVC: AutorizacaoDTO
            alt acessoPermitido == false
                Note over SVC: Monta ResultadoConsultaDTO de acesso negado.
            else acessoPermitido == true
                par Condição da CNC
                    SVC->>REP: consultarCNC(FiltroMonitoramentoDTO)
                    REP-->>SVC: CNCResultadoDTO
                and Sensores e últimas leituras
                    SVC->>REP: consultarSensores(FiltroMonitoramentoDTO)
                    REP-->>SVC: SensoresResultadoDTO
                and Alertas existentes
                    SVC->>REP: consultarAlertas(FiltroMonitoramentoDTO)
                    REP-->>SVC: AlertasResultadoDTO
                end
                SVC->>SVC: qualificar(ResultadosFontesDTO)
                SVC-->>SVC: ResultadoConsultaDTO
                Note over SVC,REP: O join reúne sucesso ou falha de cada fonte antes de qualificar a resposta.
            end
            SVC-->>API: ResultadoConsultaDTO
            deactivate SVC
        end
        API-->>WEB: ResultadoConsultaDTO
        deactivate API

        alt respostaAtual == true
            WEB->>WEB: aplicarResultado(ResultadoConsultaDTO)
            WEB-->>WEB: EstadoPainelDTO
        else respostaAtual == false
            WEB->>WEB: descartarResposta(ResultadoConsultaDTO)
            WEB-->>WEB: EstadoPainelDTO
        end
        Note over WEB: Nova consulta após 10 segundos. Mudança de filtro inicia novo identificador.
    end
```


## Artefato 2 - Atividades com raias

Uma consulta, com autorização, três carregamentos concorrentes, join e apresentação qualificada.

Fonte: [atividades.puml](docs/diagrams/atividades.puml).

![Artefato 2 - Atividades com raias](docs/images/atividades.svg)

<details>
<summary>Fonte PlantUML</summary>

```plantuml
@startuml
title AutoFlow Monitor - Consulta do dashboard de produção
skinparam defaultFontName DejaVu Sans
skinparam defaultFontSize 14
skinparam shadowing false
skinparam activityBackgroundColor #EDF5FB
skinparam activityBorderColor #3A6F93
skinparam activityDiamondBackgroundColor #FFF4E6
skinparam activityDiamondBorderColor #BA7930

|Operador|
start
:Selecionar CNC e período;
:Abrir o dashboard;

|Dashboard - JavaScript|
:iniciarConsulta com identificador e filtros;
:Enviar requisição HTTP assíncrona;

|API - FastAPI|
:Validar parâmetros e contexto autenticado;
if (parametrosValidos?) then ([parametrosValidos])
    :Parâmetros aceitos;
else ([!parametrosValidos])
    :Retornar ResultadoConsultaDTO REJEITADA;
    |Dashboard - JavaScript|
    if (respostaAtual?) then ([respostaAtual])
        :aplicarResultado e exibir motivo;
    else ([!respostaAtual])
        :descartarResposta;
    endif
    stop
endif

|API - FastAPI|
:validarAcesso à CNC solicitada;
if (acessoPermitido?) then ([acessoPermitido])
    :Acesso autorizado;
else ([!acessoPermitido])
    :Retornar ResultadoConsultaDTO REJEITADA;
    |Dashboard - JavaScript|
    if (respostaAtual?) then ([respostaAtual])
        :aplicarResultado e remover dados protegidos;
    else ([!respostaAtual])
        :descartarResposta;
    endif
    stop
endif

|API - FastAPI|
fork
    |PostgreSQL|
    :consultarCNC;
    :Produzir CNCResultadoDTO\ncom sucesso ou falha;
fork again
    |PostgreSQL|
    :consultarSensores e últimas leituras;
    :Produzir SensoresResultadoDTO\ncom sucesso ou falha;
fork again
    |PostgreSQL|
    :consultarAlertas existentes;
    :Produzir AlertasResultadoDTO\ncom sucesso ou falha;
end fork

|API - FastAPI|
:Reunir todos os resultados no join;
:qualificar qualidade e idade dos dados;
if (fonteMinimaDisponivel?) then ([fonteMinimaDisponivel])
    if (resultadoCompleto?) then ([resultadoCompleto])
        :Preparar ResultadoConsultaDTO COMPLETA;
    else ([!resultadoCompleto])
        :Preparar ResultadoConsultaDTO PARCIAL\ncom limitações por fonte;
    endif
else ([!fonteMinimaDisponivel])
    :Preparar ResultadoConsultaDTO FALHOU;
endif
:Retornar ResultadoConsultaDTO;

|Dashboard - JavaScript|
if (respostaAtual?) then ([respostaAtual])
    :aplicarResultado;
    :Exibir dados qualificados, limitações\nou mensagem de erro correspondente;
else ([!respostaAtual])
    :descartarResposta sem substituir\no painel da consulta atual;
endif

|Operador|
:Consultar as informações ou o aviso apresentado;
stop

legend bottom
  As três consultas são concorrentes e usam conexões independentes.
  Cada ramo retorna sucesso ou falha após timeout: nenhum desvia antes do join.
  resultadoCompleto = todasFontesValidas && dadosRecentes.
  A atividade representa uma consulta, repetida periodicamente pelo painel.
endlegend
@enduml
```

</details>


## Artefato 3 - Máquina de estados

Ciclo de vida de ConsultaMonitoramento, incluindo rejeição, falha e descarte de resposta antiga.

Fonte: [estados.mmd](docs/diagrams/estados.mmd).

```mermaid
stateDiagram-v2
    direction TB
    [*] --> Criada

    Criada: entry / registrarIdentificador()
    Criada --> EmProcessamento: iniciarConsulta [parametrosValidos] / registrarInicio()
    Criada --> Pronta: validarParametros [!parametrosValidos] / prepararRejeicao()

    state EmProcessamento {
        direction TB
        [*] --> ValidandoAcesso
        ValidandoAcesso: do / verificarPermissao()
        ValidandoAcesso --> ConsultandoFontes: autorizar [acessoPermitido] / iniciarConsultasParalelas()
        ConsultandoFontes: do / aguardarTodosOsResultados()
        ConsultandoFontes --> QualificandoDados: concluirConsultas [todasFinalizadas] / reunirResultados()
        QualificandoDados: do / avaliarQualidadeEAtualidade()
    }

    EmProcessamento: entry / registrarInicioProcessamento()
    EmProcessamento: exit / registrarFimProcessamento()

    EmProcessamento --> Pronta: negarAcesso [emValidacaoAcesso && !acessoPermitido] / prepararRejeicao()
    EmProcessamento --> Pronta: prepararResposta [emQualificacao] / registrarResultadoQualificado()
    EmProcessamento --> Pronta: falharConsulta [falhaTecnica] / prepararFalha()

    Pronta: entry / registrarFimConsulta()
    Pronta: do / aguardarAplicacaoNoPainel()
    state AplicandoResultado <<choice>>
    Pronta --> AplicandoResultado: aplicarResposta [respostaAtual] / classificarResultado()
    Pronta --> Descartada: aplicarResposta [!respostaAtual] / ignorarResposta()
    AplicandoResultado --> Exibida: [resultadoDisponivel] / atualizarDados()
    AplicandoResultado --> Rejeitada: [resultadoRejeitado] / exibirMotivo()
    AplicandoResultado --> Falhou: [resultadoFalha] / exibirIndisponibilidade()

    Exibida --> [*]
    Descartada --> [*]
    Rejeitada --> [*]
    Falhou --> [*]
```


## Artefato 4 - Visão de contêineres C4 L2

Aplicações e armazenamento que sustentam o monitoramento, com tecnologias e protocolos.

Fonte: [workspace.puml](docs/architecture/workspace.puml).

![Artefato 4 - Visão de contêineres C4 L2](docs/images/arquitetura.svg)

<details>
<summary>Fonte PlantUML</summary>

```plantuml
@startuml
!include <C4/C4_Container>
LAYOUT_TOP_DOWN()
title AutoFlow Monitor - C4 Nível 2 - Monitoramento da produção

Person(operador, "Operador", "Consulta condição da CNC, sensores, leituras e alertas")
System_Ext(aquisicao, "Aquisição industrial", "Sensores e gateway da CNC; simulador Python no piloto")

System_Boundary(autoflow, "AutoFlow Monitor") {
    Container(web, "Dashboard web", "HTML, CSS, JavaScript e Chart.js", "Consulta assíncrona, filtros, avisos de qualidade e descarte de respostas antigas")
    Container(api, "API de monitoramento", "Python, FastAPI e Pydantic", "Controller, autorização, serviço agregador e repositórios SQLAlchemy")
    ContainerDb(db, "Banco do AutoFlow", "PostgreSQL", "Ativos, sensores, medições qualificadas, contexto e alertas")
    Container(ingestor, "Ingestor de telemetria", "Python e paho-mqtt", "Recebe leituras e mantém condição, qualidade e saúde do monitoramento")
    ContainerQueue(mqtt, "Broker MQTT", "Eclipse Mosquitto", "Transporte dos lotes de telemetria")
}

Rel(operador, web, "Abre o painel e seleciona filtros", "HTTPS")
Rel(web, api, "Consulta monitoramento a cada 10 s", "HTTPS / REST / JSON")
Rel(api, db, "Consulta por repositórios internos", "SQL / TLS")
Rel(aquisicao, mqtt, "Publica lotes de leituras", "MQTT / TLS")
Rel(mqtt, ingestor, "Entrega telemetria", "MQTT / TLS")
Rel(ingestor, db, "Grava medições, estado e alertas", "SQL / TLS")

Lay_D(operador, web)
Lay_D(web, api)
Lay_D(api, db)
Lay_R(api, ingestor)
Lay_U(ingestor, mqtt)
Lay_U(mqtt, aquisicao)
SHOW_LEGEND()
@enduml
```

</details>


## Artefato 5 - Registro de Decisão Arquitetural

Fonte: [adr-0001.md](docs/architecture/adr-0001.md).

### ADR 0001 - API agregadora com consultas concorrentes e resposta qualificada

#### Status

Proposto. Data: 10/11/2026. Projeto: AutoFlow Monitor. Funcionalidade: Monitoramento da produção, RF09 e UC01.

#### Contexto

O operador precisa acompanhar condição da CNC, sensores, últimos dados e alertas em um único painel. Essas informações possuem origens lógicas e horários distintos. Consultas lentas, falhas parciais e respostas fora de ordem podem fazer o painel exibir dados antigos como atuais ou indicar ausência de alertas quando a consulta falhou.

A Sprint 1 propõe HTML, CSS, JavaScript e Chart.js para o dashboard; Python/FastAPI, SQLAlchemy e PostgreSQL no backend. Também propõe atualização a cada 10 segundos e exige indicação da qualidade e idade dos dados.

#### Decisão

Adotar uma operação agregadora de consulta na API FastAPI. O navegador solicita os filtros e recebe ResultadoConsultaDTO com consultaId, ativoId, status, dados qualificados, consultadoEm e avisos.

O controller valida parâmetros e estabelece o contexto autenticado. O serviço verifica o acesso à CNC antes de disparar três consultas concorrentes: condição da CNC, sensores/leituras e alertas. Cada tarefa usa uma conexão própria e possui timeout. Falhas são capturadas e cada ramo retorna ao join.

O serviço qualifica o conjunto como COMPLETA, PARCIAL, REJEITADA ou FALHOU. O painel preserva horário e qualidade das informações e não confunde fonte indisponível com condição normal ou ausência de alertas.

A consulta é iniciada por fetch assíncrono. A interface verifica consultaId e ativoId antes de aplicar a resposta. A mudança de filtro invalida a consulta anterior. A atualização periódica é proposta a cada 10 segundos, evitando iniciar ciclos periódicos sobrepostos; a meta de até 15 segundos após o aceite no servidor permanece um critério a medir no piloto.

#### Alternativas consideradas

| Alternativa | Benefício | Limitação |
| --- | --- | --- |
| Três requisições independentes pelo navegador | Facilita separar os carregamentos. | Espalha autorização, tratamento de falhas e regras de qualidade na interface. |
| Uma operação com consultas sequenciais | Simplifica conexões e fluxo inicial. | Os tempos das três consultas se somam. |
| Uma operação com consultas concorrentes e qualificação | Centraliza o contrato, reúne os resultados e explicita falhas parciais. | Exige sessões independentes, timeouts, join e limite de concorrência. |
| Push por WebSocket | Pode reduzir espera por novos dados. | Acrescenta gestão de conexão e reconexão; o polling já consta da proposta do piloto. |

A concorrência é uma escolha de organização da consulta. Não há benchmark comprovando ganho de latência ou necessidade de escala para o piloto.

#### Consequências

##### Vantagens

- Contrato único para a visualização de monitoramento.
- Autorização centralizada antes da consulta dos dados.
- Tratamento explícito de informações ausentes, antigas ou indisponíveis.
- Respostas obsoletas não substituem a consulta atual.
- Fonte com falha não elimina automaticamente as informações úteis das demais.

##### Riscos e débitos aceitos

- Até três consultas usam o pool de conexões por solicitação.
- É necessário limitar concorrência e tempo de espera por tarefa.
- Resultados podem representar instantes diferentes; o DTO não é um snapshot transacional único.
- Polling tem atraso de atualização e gera carga mesmo sem mudança dos dados.
- O status PARCIAL exige apresentação clara das limitações.

##### Governança e mitigação

O frontend chama a API; controller e interface não executam SQL. Regras de qualificação pertencem ao serviço. Repositórios usam filtros parametrizados e escopo de autorização.

A UI aplica apenas a resposta atual, mostra o horário da última informação e diferencia zero alertas de falha ao consultar alertas. Após rejeição de acesso, elimina dados protegidos do painel.

Mudanças de protocolo, contrato de resposta ou significado dos estados devem atualizar diagramas, interfaces e esta ADR. Uma decisão substituta deve ser registrada se o projeto adotar push ou exigir snapshot transacional.

#### Critérios de validação futura da implementação

Testar usuário sem permissão, filtro inválido, atraso e falha de cada fonte, todos os dados indisponíveis, leituras antigas, retorno vazio de alertas, troca rápida de CNC, respostas fora de ordem e carga do piloto com atualização até 15 segundos após o aceite no servidor.

Esses são ensaios planejados da aplicação. A compilação dos modelos verifica sintaxe e consistência documental.

#### Referências

- [FastAPI: concorrência e async/await](https://fastapi.tiangolo.com/async/).
- [Python: tarefas assíncronas](https://docs.python.org/3/library/asyncio-task.html).
- [PostgreSQL: isolamento transacional](https://www.postgresql.org/docs/current/transaction-iso.html).
- Sprint 1 e Sprint 2 do AutoFlow Monitor.


## Interfaces e consistência entre modelos

As classes/interfaces públicas e os DTOs estão declarados em [interfaces.py](docs/contracts/interfaces.py). São contratos conceituais, sem implementação da aplicação. O arquivo [consistencia.md](docs/contracts/consistencia.md) define as guardas, os resultados e os pontos de integração.

| Participante da sequência | Interface pública | Papel |
| --- | --- | --- |
| WEB | DashboardView | Iniciar consulta e aplicar ou descartar resposta. |
| API | MonitoramentoController | Receber requisição e validar contrato/contexto autenticado. |
| SVC | MonitoramentoService | Autorizar, agregar e qualificar os resultados. |
| REP | MonitoramentoRepository | Consultar CNC, sensores/leituras e alertas. |

Todas as mensagens de chamada da sequência correspondem a operações públicas dessas interfaces. Os retornos usam DTOs. A seta -) indica o envio assíncrono por fetch; ->> indica chamada com espera lógica pelo resultado.

A sequência apresenta os ciclos periódicos do painel; a atividade detalha um ciclo; os estados acompanham a consulta desse ciclo; o C4 posiciona os participantes e os protocolos; a ADR explica as decisões comuns.

## Significado dos resultados

| Resultado | Efeito no painel |
| --- | --- |
| COMPLETA | Exibir os dados válidos e recentes das fontes. |
| PARCIAL | Exibir informações utilizáveis com avisos claros de ausência, atraso ou falha por fonte. |
| REJEITADA | Exibir o motivo apropriado e preservar o controle de acesso. |
| FALHOU | Informar que não foi possível obter o resultado mínimo. |

Uma lista vazia de alertas, obtida com sucesso, significa ausência de registros no filtro. Erro ao consultar alertas significa informação indisponível. Falha de consulta não comprova falha física do sensor.

## Atendimento à atividade

| Artefato | Evidência |
| --- | --- |
| Sequência | Cinco linhas de vida, chamadas síncronas e assíncronas, loop, alt/else, par e retornos DTO. |
| Atividades | Quatro raias, guardas booleanas opostas e barras UML de fork/join. |
| Estados | Estado inicial, subestados, quatro resultados terminais e transições com gatilho, guarda e efeito; entry, do e exit. |
| C4 L2 | Contêineres identificados com tecnologias e conectores com protocolos. |
| ADR 0001 | Título, status Proposto, contexto, decisão, alternativas e consequências. |

A escolha de PlantUML nas atividades permite representar raias e barras UML de fork/join diretamente. O arquivo de arquitetura usa a biblioteca C4-PlantUML. Essas alternativas são permitidas no enunciado; por isso as extensões correspondentes são .puml.

## Arquivos da entrega

| Caminho | Conteúdo |
| --- | --- |
| README.md | Documento principal com os cinco artefatos. |
| docs/diagrams/sequencia.mmd | Fonte Mermaid da sequência. |
| docs/diagrams/atividades.puml | Fonte PlantUML das atividades com raias. |
| docs/diagrams/estados.mmd | Fonte Mermaid da máquina de estados. |
| docs/architecture/workspace.puml | Fonte C4-PlantUML do nível 2. |
| docs/architecture/adr-0001.md | Registro de decisão arquitetural. |
| docs/contracts/ | Operações públicas, DTOs e regras de consistência. |
| docs/images/ | SVGs das fontes PlantUML, com identificação da versão da fonte. |
| docs/README.base.md | Texto-base usado para gerar este README. |
| scripts/ | Geração da documentação e verificações de consistência. |

## Visualização, atualização e validação

O GitHub renderiza os blocos Mermaid do README. As atividades e o C4 já possuem SVGs versionados, gerados das respectivas fontes PlantUML.

Para revisar o texto, editar docs/README.base.md. Para revisar os modelos, editar os arquivos de origem. Depois, gerar novamente a documentação e registrar as alterações reais no Git.

Requisitos para regeneração: Node.js 20 ou superior, Python 3.10 ou superior, Java 17 ou superior e Graphviz.

~~~bash
npm ci
npm run docs
npm run check
~~~

O gerador obtém PlantUML 1.2026.9 e verifica seu SHA-256. Também aceita o caminho de um JAR dessa versão na variável PLANTUML_JAR. O CI compila os modelos e verifica operações, DTOs, fontes e README.

O material é de modelagem. Não há aplicação, banco industrial ou integração MQTT executável neste pacote. A meta RNF01 de até 15 segundos e as demais condições operacionais continuam sendo critérios de teste da futura implementação.

## Publicação e autoria

Copiar esta pasta para o repositório individual no GitHub, revisar os arquivos e registrar sua evolução real em commits. Entregar o link do repositório no AVA. O pacote não contém um histórico de commits.

Material preparado com apoio de IA para organizar a documentação e elaborar os modelos. A revisão individual e a adaptação às decisões do projeto fazem parte do processo de autoria.

## Fontes

- Sprint 1(1).pdf: contexto, tecnologias, RF09 e RNF01/RNF04.
- Sprint 2 - Modelagem de Software.docx: escopo, atores, UC01 e entidades do AutoFlow.
- MODEL STORMING ESTRUTURAL - AutoFlow Monitor.pdf: separação entre condição, sensores e alertas.
- [Mermaid: Sequence diagrams](https://mermaid.js.org/syntax/sequenceDiagram.html).
- [Mermaid: State diagrams](https://mermaid.js.org/syntax/stateDiagram.html).
- [PlantUML: Activity diagram](https://plantuml.com/activity-diagram-beta).
- [C4 Model: Container diagram](https://c4model.com/diagrams/container).
- [GitHub: Creating diagrams](https://docs.github.com/en/get-started/writing-on-github/working-with-advanced-formatting/creating-diagrams).
