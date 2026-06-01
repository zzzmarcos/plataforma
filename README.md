# Do ECS Fargate ao EKS: Como um time enxuto economizou R$ 200k/mês enfrentando os desafios da governança bancária

Migrar cargas de trabalho para o Kubernetes com a promessa de reduzir custos e ganhar escala é o sonho de qualquer liderança de tecnologia. Mas quem está na trincheira sabe a real: sem uma boa estratégia de plataforma, o Kubernetes joga uma complexidade brutal no colo dos desenvolvedores, gerando fricção, problemas de segurança e desperdício de nuvem.

**Nos últimos três anos**, a área de investimentos do maior banco da América Latina viveu intensamente esse desafio. Quando começamos a desenhar essa transição, tínhamos mais de 300 desenvolvedores cuidando de 160 serviços rodando em Amazon ECS Fargate. O cenário era o reflexo de muitas grandes empresas: custos subindo devido ao superdimensionamento (*overprovisioning*) crônico, problemas profundos para garantir a governança e lentidão no escalonamento de recursos.

A resposta óbvia do mercado seria: *"Migre para o EKS e adote GitOps tradicional"*. O problema é que, em um ambiente financeiro altamente regulado, você não muda as regras da casa da noite para o dia. Tínhamos amarras de governança muito fortes e imutáveis: 

1. Infraestrutura **apenas** via Terraform para provisionamento de recursos.
2. Pipelines engessados e trancados no GitHub Actions, com workflows prontos e sem possibilidade de customização.
3. Uso obrigatório do ArgoCD, mas **sem o modelo tradicional de GitOps** (sem repositórios dedicados por aplicação). O deploy precisava ser feito enviando o pacote do YAML direto para o Argo via API/CLI.

Com um time enxuto de 4 pessoas, mas com **apenas 2 engenheiros dedicados à frente da stack de Kubernetes durante todo esse ciclo**, nossa missão parecia impossível. Mas nós resolvemos o problema criando um operador Kubernetes customizado dentro de casa — e não usamos Go, usamos **Python (FastAPI) e Metacontroller**.

O resultado de três anos de evolução contínua? Uma economia de **75% nos custos de produção**, **90% em ambientes de desenvolvimento** e mais de **R$ 1 milhão por ano devolvidos ao negócio** em horas de engenharia que os times de produto gastavam brigando com a infraestrutura. No total, geramos uma eficiência combinada de **R$ 200.000 por mês**.

Neste artigo, vou te mostrar os bastidores dessa arquitetura, os principais desafios que enfrentamos na trincheira ao longo desses três anos e como você pode estender o Kubernetes para forçar padrões sem matar a velocidade dos desenvolvedores.

---

## O Cenário no ECS e o Peso da Governança

O ECS Fargate é uma tecnologia fantástica para começar, mas conforme os times de produto crescem, a falta de padronização e as travas corporativas cobram o seu preço. No nosso cenário original, os desenvolvedores tinham autonomia para criar suas *Task Definitions*. O resultado prático disso era um grande desafio de conformidade.

E para ser completamente honesto: **nós tínhamos um processo interno que eu odiava com todas as minhas forças, chamado "Marathon de FinOps".**

O Marathon de FinOps era uma reunião recorrente e extremamente desgastante. O objetivo era juntar o time de infraestrutura e os líderes de produto para ficar caçando recursos órfãos na AWS e implorando para que os times colocassem tags básicas nos recursos — coisas simples, como a tag da `squad`, o dono do serviço ou o centro de custo. Era um trabalho de herói, totalmente manual, reativo e ineficiente.

> **O grande impacto disso era a cegueira total de custos.** A falta dessas tags fazia com que não tivéssemos a menor visibilidade de para onde o dinheiro estava indo. Faturas massivas da AWS chegavam e nós simplesmente não sabíamos quem eram os donos daqueles recursos, o que eles faziam e, o pior de tudo, se aquele custo sequer se justificava para o negócio. Era impossível auditar e otimizar o que não conseguíamos enxergar.

Além dessa dor de cabeça com FinOps, o modelo descentralizado do ECS gerava um fardo imenso para os times de produto:

* **A Fadiga de Atualizações:** Os desenvolvedores precisavam estar constantemente de olho para ver se havia atualizações nos módulos de infraestrutura. Qualquer mudança institucional, por menor que fosse, se transformava em um pesadelo operacional: precisava ser aplicada **app por app, de forma isolada**.
* **Overprovisioning:** Na dúvida de quanta CPU ou Memória a aplicação precisava, o dev jogava o valor para o alto na Task Definition. Como o Fargate cobra exatamente pelo que está alocado (e não pelo que está sendo usado), a conta da AWS explodiu.
* **O Gargalo do Escalonamento:** Devido à lentidão do *scaling* no ECS — que precisava bater na API da AWS a cada nova réplica para buscar *secrets* e *parameters* via API —, nossa velocidade de resposta a picos de tráfego era severamente afetada.

Sabíamos que o Amazon EKS resolveria o problema do custo e da velocidade. Mas como fazer isso sem obrigar 300 desenvolvedores a aprenderem o manifesto complexo do Kubernetes (Deployments, Services, Ingress, HPA)? E pior: como garantir que a gente **matasse o Marathon de FinOps e a dependência de intervenções manuais app por app de uma vez por todas?**

Percebemos que a única saída era **interceptar o deploy dentro do próprio cluster**. Precisávamos de um operador que recebesse um YAML ultra simples do desenvolvedor e injetasse toda a complexidade e governança corporativa por baixo dos panos, de forma 100% automatizada.

---

## Desafio de Arquitetura: Por que Python e Metacontroller para um time de 2 pessoas?

Quando se fala em estender o Kubernetes e criar CRDs (*Custom Resource Definitions*), o padrão de mercado quase absoluto é usar Go com Kubebuilder ou Operator SDK. No entanto, precisávamos ser pragmáticos. Gastar semanas escrevendo o *boilerplate* massivo do Go e lidando com a complexidade do ciclo de vida de reconciliação tradicional nos faria estourar qualquer prazo.

Foi aí que tomamos uma decisão de arquitetura pouco ortodoxa para grandes corporações, mas extremamente eficiente: escolhemos o **Metacontroller** e escrevemos nossa lógica de negócios em **Python**.

> O Metacontroller funciona como um "facilitador". Ele roda como um operador dentro do cluster e cuida de toda a parte chata e complexa do Kubernetes: gerenciar os watchers, a fila de reconciliação e as chamadas de API. Tudo o que ele pede em troca é um Webhook HTTP.

Nossa arquitetura foi desenhada para ser simples:

1. O desenvolvedor submete um YAML minimalista (de no máximo 3 níveis de profundidade).
2. O pipeline fechado do GitHub Actions envia esse pacote diretamente para a API do ArgoCD.
3. O ArgoCD aplica esse recurso customizado no cluster.
4. O Metacontroller percebe a mudança e faz uma requisição HTTP POST para a nossa API em Python.
5. Nosso código recebe o estado atual, injeta os padrões do banco (tags de billing obrigatórias por squad, variáveis do Datadog, sidecars de segurança e *health checks*) e devolve a resposta.
6. O Metacontroller se encarrega de criar ou atualizar os Deployments, HPAs e Services gerados.

---

## Governança Centralizada na Prática: O Caso do Datadog e da Sanitização de Logs

Para entender o poder real desse modelo de plataforma, vale a pena comparar como duas grandes demandas institucionais do banco foram tratadas nos ambientes ECS Fargate e Amazon EKS.

### Exemplo 1: A Migração para o Datadog
Quando o banco definiu a mudança global da nossa ferramenta de observabilidade para o Datadog, a diferença de esforço foi gritante:
* **No ECS Fargate:** Foi um trabalho hercúleo. Os times de produto tiveram que entrar serviço por serviço, alterando cada Task Definition, configurando sidecars, variáveis de ambiente e atualizando seus pipelines manualmente. Foram meses de cobrança e fricção.
* **No nosso ecossistema EKS:** Quem já estava rodando na nossa plataforma baseada no operador **não precisou fazer absolutamente nada**. Nós atualizamos a lógica central do nosso webhook em Python. Na reconciliação seguinte do Metacontroller, todos os 160 serviços receberam as configurações do Datadog injetadas por baixo dos panos. Governança invisível.

### Exemplo 2: A Demanda de Sanitização de Logs
Recentemente, tivemos uma diretriz de segurança urgente para sanitizar dados sensíveis nos logs de todas as aplicações da divisão de investimentos.
* **No ECS Fargate:** Mais uma vez, o peso caiu sobre os desenvolvedores. Foi necessário acionar time por time para intervir código por código, aplicação por aplicação, para garantir que as strings sensíveis fossem mascaradas antes do envio.
* **No Amazon EKS:** Nós resolvemos o problema de forma global e centralizada na nossa camada de plataforma. Configuramos uma regra de filtro diretamente no **Fluent Bit** que rodava como DaemonSet no cluster. Em um único deploy da nossa equipe de SRE, garantimos a conformidade de 100% das aplicações rodando em Kubernetes, poupando milhares de horas de desenvolvimento e mitigando um risco de segurança crítico em minutos.

---

## A Evolução Técnico-Pragmática: Do Script Puro ao FastAPI

Quando olhamos para a arquitetura atual rodando em produção com 160 serviços, parece que tudo foi planejado em detalhes desde o primeiro dia. Mas a realidade da trincheira é bem diferente. Três anos atrás, quando começamos a desenhar o operador, **nós sabíamos apenas Python**. Nenhum dos dois engenheiros focados na stack de Kubernetes dominava Go ou tinha experiência prévia criando controllers complexos.

Em vez de travar o projeto para passar meses aprendendo uma nova linguagem, decidimos abraçar o pragmatismo: **vamos resolver o problema com o que sabemos hoje.**

A primeira versão do operador nasceu como um script em Python puro rodando dentro de um container, escutando os webhooks do Metacontroller. Era simples, direto e resolvia a dor imediata: manipulava os dicionários no código para injetar o necessário e devolvia a resposta.

### O Desafio da Escala e a Virada para o FastAPI

Conforme fomos migrando os serviços do ECS Fargate para o EKS e o volume de deploys começou a subir rumo aos 160 serviços atuais, o script em Python puro começou a sentir o peso da concorrência e das requisições simultâneas do Metacontroller. O throughput precisava melhorar.

Foi aí que fizemos a evolução natural para o **FastAPI**. A migração trouxe três grandes viradas de chave para o nosso time:

1. **Assincronismo nativo (async/await):** O FastAPI nos permitiu lidar com múltiplas requisições de reconciliação simultâneas do Metacontroller sem travar a execução do operador, reduzindo drasticamente o tempo de resposta dos webhooks.
2. **Validação com Pydantic:** Conseguimos tipar e validar estritamente o que vinha no YAML dos desenvolvedores logo na entrada da API. Se um dev tentasse passar algo fora do padrão de 3 níveis, o FastAPI barrava ali mesmo.
3. **Facilidade de Manutenção:** O ecossistema nos deu visibilidade, facilitando para que as outras duas pessoas do time de SRE — que não estavam no dia a dia do Kubernetes — conseguissem entender o código e dar suporte caso necessário.

Essa escolha provou que o melhor caminho para um time enxuto não é adotar a ferramenta mais complexa do mercado, mas sim evoluir a tecnologia de forma orgânica conforme o calo aperta na produção.

---

## O Pulo do Gato com Secrets e o CI/CD Engessado

Com a inteligência do operador em FastAPI rodando redonda, conseguimos atacar os dois últimos grandes problemas: a lentidão do escalonamento e as travas do pipeline sem GitOps tradicional. 

Para resolver o escalonamento, implementamos o **External Secrets**, que passou a montar os segredos diretamente como volumes nos pods do EKS. Isso eliminou completamente a necessidade de as aplicações fazerem chamadas de API externas na AWS em tempo de execução, tornando o *scaling* infinitamente mais rápido que no ECS Fargate.

Para contornar o pipeline imutável do banco, a esteira do GitHub Actions apenas empacotava esse YAML minimalista de 3 níveis e, usando comandos prontos de CLI/API, enviava o payload direto para o ArgoCD. O ArgoCD funcionava puramente como o motor de execução final, disparando o ciclo de vida dentro do cluster que o nosso operador interceptava e governava de forma invisível para o desenvolvedor.

---

## Conclusão: A Linha do Tempo de uma Jornada Viva

Olhando para trás, os desafios de sustentar e evoluir uma solução "feita em casa" ao longo desses três anos nos ensinaram que o sucesso de uma plataforma de engenharia corporativa depende de paciência, consistência e processos bem desenhados. Essa jornada de 36 meses dividiu-se em três grandes fases:

1. **Os Primeiros 6 Meses (Estudos e Provas de Conceito):** O período em que validamos o Metacontroller, rodamos os primeiros scripts em Python puro e provamos para a governança que conseguiríamos injetar os padrões institucionais de forma segura dentro do cluster.
2. **Os 2 Anos Seguintes (A Jornada de Migração):** O momento da trincheira. A migração dos serviços foi sendo feita de forma gradual e assistida pelos próprios times de produto. Famos refinando o operador (onde nasceu a virada para o FastAPI) e dando autonomia escalável para os desenvolvedores descerem do ECS.
3. **Os Últimos 6 Meses (A Consolidação do Valor):** Com boa parte dos serviços migrados e o ecossistema estabilizado, alcançamos a maturidade atual da plataforma. 

Hoje, colhemos uma **redução de 75% nos custos de infraestrutura em produção** e impressionantes **90% em desenvolvimento e homologação**. Matamos o fantasma do Marathon de FinOps, transformando a governança em um processo invisível e compulsório, devolvendo mais de R$ 1 milhão por ano em horas de trabalho para os times de produto. 

O projeto, porém, está longe de estar "concluído". Mesmo sem atingirmos 100% de migração total de toda a carga de trabalho histórica, a eficiência que já geramos nos abriu os olhos para novas frentes. Atualmente, estamos estudando e desenhando otimizações de arquitetura com a meta ousada de **trazer o custo total dessa infraestrutura para a casa dos R$ 8.000 mensais**. 

Em uma grande instituição financeira como a nossa, provamos que times enxutos usando tecnologia de forma pragmática conseguem sim mudar o rumo da governança, da experiência do desenvolvedor e do faturamento de nuvem. E o trabalho continua.
