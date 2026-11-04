# Análise da Arquitetura de Microsserviços — StreamFlow

## Resumo

Neste trabalho, foi analisada a arquitetura de microsserviços do sistema StreamFlow, observando principalmente seus custos operacionais, a comunicação entre os serviços, a observabilidade, a consistência dos dados e a organização da arquitetura.

Para realizar a análise, o projeto foi executado utilizando Docker Compose. Também foram analisados alguns arquivos do código-fonte e realizados testes práticos envolvendo os serviços de Streaming, Catalog, Recommendation, Notification, Billing e Analytics.

Durante os testes, foram encontradas diferenças consideráveis nos tempos de resposta. A consulta ao catálogo apresentou mediana de aproximadamente 4,68 ms, enquanto a reprodução de um filme apresentou mediana de 206,08 ms. Também foram identificadas situações envolvendo falhas na comunicação entre serviços e o compartilhamento de banco de dados.

De modo geral, o sistema apresenta pontos positivos, como a divisão das responsabilidades entre os serviços e a utilização de logs estruturados. Porém, também foram encontrados problemas relacionados ao acoplamento, às chamadas síncronas e à perda de informações quando alguns serviços ficam indisponíveis.

A partir dessas observações, foram propostas algumas melhorias para tornar o sistema mais confiável e facilitar sua manutenção. Também foi analisado o caso do Prime Video, que demonstra como uma arquitetura mais simples pode ser uma alternativa interessante dependendo das necessidades da aplicação.

## 1. Custos operacionais ocultos e inventário dos serviços

Ao rodar o StreamFlow, vi que o projeto tem oito containers: sete serviços com as funcionalidades do sistema e um API Gateway, que recebe as requisições de fora e manda pro serviço certo.

### 1.1 Inventário dos serviços

| Serviço | Responsabilidade | Porta interna |
|---|---|---:|
| Gateway | Receber e encaminhar as requisições para os serviços internos | 8080 |
| Auth | Realizar a autenticação e gerar tokens | 3001 |
| Catalog | Gerenciar e consultar o catálogo de filmes | 3002 |
| Streaming | Gerenciar as sessões de reprodução | 3003 |
| Recommendation | Armazenar o histórico de visualizações e fornecer recomendações | 3004 |
| Notification | Enviar notificações | 3005 |
| Billing | Gerenciar assinaturas e faturas | 3006 |
| Analytics | Processar dados e gerar relatórios | 3007 |

No docker-compose.yml, percebi que o Gateway é o único com porta publicada pra acesso externo (8080). Os outros só são acessados pela rede interna do Docker, chamada streamflow-net.

### 1.2 Dependências e acoplamento

Apesar dos serviços serem separados, achei algumas dependências importantes:

• O Gateway recebe as requisições e manda pros serviços internos.
• O Streaming depende do Catalog pra ver se o filme tem licença pra ser reproduzido.
• Na reprodução, o Streaming também tenta registrar a visualização no Recommendation e mandar uma notificação pelo Notification.
• O Analytics acessa direto o banco de dados do Billing, usando o volume shared-billing-data.

O que mais me chamou atenção foi o banco compartilhado entre Billing e Analytics. Eles são serviços separados, mas dependem da mesma estrutura de dados. Se um dos dois mudar as tabelas, pode acabar quebrando o outro.

### 1.3 Custos operacionais ocultos

Dividir o sistema em microsserviços ajuda a organizar as responsabilidades, mas também traz dificuldades.
No StreamFlow, por exemplo, precisamos de oito containers rodando, e ainda tem que cuidar das configurações, da comunicação pela rede, dos logs, do monitoramento e do tratamento das falhas.
Outra coisa que notei é que a comunicação entre os serviços pode deixar a operação mais lenta, principalmente quando um serviço precisa esperar a resposta de outros antes de continuar.
Então, mesmo com as vantagens, dá mais trabalho manter toda essa estrutura funcionando. Esse é um custo que precisa ser levado em conta na hora de escolher microsserviços.

## 2. Latência e comunicação entre serviços

Pra entender como a comunicação entre os serviços afeta o desempenho, fiz testes comparando duas operações:
• GET /api/catalog: consulta os filmes disponíveis no catálogo.
• POST /api/streaming/play: começa a reprodução de um filme e envolve comunicação com outros serviços.
Pra cada operação, fiz uma chamada inicial de aquecimento (warm-up) e depois cinco medições.

### 2.1 Resultados dos testes

| Medição | Consulta ao catálogo (ms) | Reprodução (ms) |
|---|---:|---:|
| 1 | 5,776 | 206,080 |
| 2 | 4,678 | 213,661 |
| 3 | 4,460 | 161,259 |
| 4 | 4,550 | 292,716 |
| 5 | 6,009 | 183,447 |
| Med. | 4,678 | 206,080 |

Todas as consultas ao catálogo retornaram HTTP 200, e as reproduções retornaram HTTP 201.
Comparando as medianas, a reprodução demorou umas 44 vezes mais que a consulta ao catálogo.

### 2.2 Análise dos resultados

Essa diferença faz sentido por causa do que cada operação precisa fazer. A consulta ao catálogo é simples, já a reprodução faz outras verificações e conversa com vários serviços.
No código do Streaming, vi chamadas pro Catalog, Recommendation e Notification. Elas são síncronas, ou seja, o Streaming espera a resposta de cada uma antes de seguir.
Também percebi que nem todas essas dependências impedem a reprodução. Mesmo quando o Recommendation ou o Notification falham, o Streaming consegue terminar a operação.
É importante dizer que os tempos que medi são o tempo total de resposta da requisição. Então eles incluem o processamento e a comunicação entre os serviços, e não dá pra dizer que é só a latência da rede.

### 2.3 Evidências nos logs
Durante os testes, também olhei os logs do Gateway e do Streaming.
No Gateway, deu pra ver o encaminhamento das requisições, o serviço de destino, o código HTTP e o tempo de resposta.
Nos logs do Streaming, apareceram o início da reprodução, o id do filme, o usuário e o tempo que a operação levou pra terminar.
Esses registros me ajudaram a entender o caminho das requisições e como cada etapa de comunicação pesa no tempo de resposta.

## 3. Observabilidade: logs, métricas e rastreamento

Observabilidade serve pra entender o que está acontecendo dentro da aplicação, principalmente quando ela é dividida em vários serviços. No StreamFlow, olhei os logs gerados nos testes e alguns arquivos de configuração do projeto.

### 3.1 Logs

O sistema usa logs estruturados, com informações úteis pra acompanhar as requisições, como:
• Nível do registro, como informação (info) e aviso (warn).
• Identificador da requisição (reqId).
• Método HTTP e endereço acessado.
• Código de resposta HTTP.
• Tempo de processamento.
• Mensagens sobre as operações feitas.
Um exemplo foi o teste em que desliguei o Notification. O Streaming registrou um aviso dizendo que não conseguiu se comunicar com o serviço, mas a reprodução terminou normal, retornando HTTP 201.
Isso mostrou que os logs são importantes, porque dá pra descobrir problemas que nem sempre aparecem como erro na resposta que o usuário recebe.

### 3.2 Métricas e rastreamento distribuído

Também procurei nos arquivos JavaScript, JSON e YAML por referências a ferramentas como Prometheus, Grafana, Jaeger e OpenTelemetry.
Não achei nenhuma referência a elas nos arquivos que examinei. Também não vi configuração de monitoramento centralizado de métricas nem de rastreamento distribuído.
Os logs têm identificador de requisição, mas nos trechos que coletei não consegui ver um identificador de correlação compartilhado por todos os serviços de uma mesma operação.
Isso pode dificultar a investigação de problemas quando uma requisição passa por vários serviços.

### 3.3 Problema identificado no endpoint de saúde

Testando o endpoint /health, notei um comportamento estranho: o Gateway retornava HTTP 503 e o estado degraded, mesmo com todos os serviços internos respondendo normalmente.
Analisando o arquivo gateway/index.js, achei o problema na verificação dos estados. O código comparava o resultado com up, mas os serviços retornavam ok.
Pra corrigir, troquei a comparação pra ok e reconstruí o Gateway.
Esse problema mostrou que não basta ver se os containers estão rodando. Também precisa garantir que as verificações de saúde estão funcionando direito.

### 3.4 Propostas de melhoria

Com base no que observei, algumas melhorias possíveis são:
• Usar OpenTelemetry pra acompanhar uma requisição enquanto ela passa pelos serviços.
• Centralizar os logs pra facilitar a busca por erros.
• Adicionar métricas, como tempo de resposta, quantidade de requisições e taxa de erros.
• Usar ferramentas como Grafana e Prometheus pra acompanhar essas informações.
• Padronizar os identificadores de correlação entre os serviços.
Isso ajudaria a entender melhor o comportamento da aplicação e a achar a origem dos problemas mais fácil.

## 4. Consistência de dados entre Catalog e Recommendation

Consistência de dados é um ponto importante em sistemas distribuídos, principalmente quando serviços diferentes guardam informações relacionadas.
No StreamFlow, o Catalog cuida das informações dos filmes e das licenças, e o Recommendation guarda o histórico de visualizações e as preferências dos usuários.

### 4.1 Comunicação entre os serviços

No código, vi que quem faz a ligação entre esses dois serviços é o Streaming.
Na reprodução, ele consulta o Catalog pra verificar a licença do filme e depois manda uma requisição pro Recommendation registrar a visualização.
Essa comunicação é feita por chamadas HTTP síncronas. O Recommendation recebe os dados do usuário e do filme e salva no próprio banco de dados.

### 4.2 Experimento de indisponibilidade do Recommendation

Pra ver como o sistema se comportaria com o Recommendation fora do ar, fiz um teste prático.
Primeiro, iniciei a reprodução do filme de ID 3, que é o Parasita. A requisição retornou HTTP 201 e, consultando o banco do Recommendation, encontrei o registro da visualização na tabela view_history.
Depois desliguei o Recommendation e tentei reproduzir o mesmo filme de novo.
Mesmo com o serviço fora do ar, o Streaming retornou HTTP 201 e avisou que a reprodução tinha começado. Só que nos logs apareceu um erro dizendo que não foi possível encontrar o serviço Recommendation.
Quando liguei o Recommendation de novo, consultei a tabela view_history outra vez. Continuavam oito registros, sem o da segunda reprodução.
Ou seja, a reprodução aconteceu, mas a visualização não foi salva no histórico.

### 4.3 Análise da consistência

Esse teste mostrou que o StreamFlow continua funcionando mesmo com o serviço de recomendações fora do ar. Mas também mostrou que algumas informações podem ser perdidas durante essas falhas.
No código, não achei nenhum mecanismo de retentativa, fila de mensagens ou recuperação pra registrar depois as visualizações que não foram processadas.
Por isso, do jeito que está, não tem garantia de que essas informações vão ser recuperadas quando o serviço voltar. Na prática, o teste mostrou que o sistema não garante consistência eventual entre a reprodução feita e o histórico guardado pelo Recommendation.

### 4.4 Propostas de melhoria

Uma solução possível seria usar filas de mensagens ou eventos pra fazer essa comunicação de forma assíncrona.
Assim, o Streaming poderia registrar o evento de reprodução e deixar o Recommendation processar depois. Se o serviço estivesse fora do ar, a mensagem ficaria guardada até dar pra processar.
Também seria bom ter retentativas, evitar registros duplicados e acompanhar quais eventos ainda não foram processados.
Com isso, o sistema continuaria permitindo as reproduções durante uma falha, sem perder pra sempre as informações do histórico.

## 5. Lei de Conway e organização das equipes

A Lei de Conway diz que a estrutura de um sistema tende a refletir a forma como as pessoas e equipes que fazem ele se comunicam.
O StreamFlow está dividido em vários microsserviços, cada um com uma responsabilidade, como autenticação, catálogo, reprodução, recomendações, notificações, faturamento e análise de dados.
Não temos informação de como a equipe que desenvolveu o projeto está organizada de verdade. Então, nessa análise, eu pensei em como essa divisão poderia influenciar a organização de uma equipe de desenvolvimento.

### 5.1 Relação entre arquitetura e organização

Se cada serviço fosse mantido por uma equipe diferente, isso poderia ajudar na especialização dos desenvolvedores e dar mais autonomia pra cada equipe.
Por outro lado, olhando o funcionamento do StreamFlow, existem dependências entre os serviços, e isso exigiria comunicação entre as equipes.
Um exemplo é o fluxo de reprodução, em que o Streaming depende do Catalog, Recommendation e Notification. Outro é o banco compartilhado entre Billing e Analytics, que exige cuidado pra que uma mudança de um serviço não atrapalhe o outro.
Essas situações poderiam dificultar o trabalho independente das equipes e exigir mais alinhamento durante o desenvolvimento.

### 5.2 Benefícios e desafios

Organizar as equipes por serviço poderia trazer alguns benefícios:
• Desenvolvedores mais especializados.
• Responsabilidades melhor divididas.
• Possibilidade de desenvolver e publicar serviços separadamente.
• Mais facilidade pra mudar funcionalidades específicas.
Mas também tem desafios:
• Precisa de comunicação frequente entre as equipes.
• Dependência de contratos e interfaces entre os serviços.
• Possibilidade de conflito quando uma mudança afeta mais de um serviço.
• Mais difícil investigar problemas que passam por partes diferentes do sistema.

### 5.3 Recomendações

Uma alternativa seria organizar as equipes por domínio de negócio, deixando bem definido o que cada uma é responsável e como os serviços devem se comunicar.
Também seria importante evitar dependências desnecessárias, principalmente o compartilhamento direto de banco de dados entre serviços diferentes.
Além disso, documentar as APIs, usar integração contínua e ter ferramentas de monitoramento compartilhadas poderia ajudar na comunicação entre as equipes.
Assim, a divisão em microsserviços poderia dar mais autonomia pro desenvolvimento, desde que a organização das equipes acompanhe essa estrutura.

## 6. Estudo de caso: Prime Video

Em março de 2023, a equipe de tecnologia do Prime Video publicou um artigo explicando mudanças que fizeram na arquitetura do serviço de análise de qualidade de vídeo (Video Quality Analysis — VQA).
Esse serviço era usado pra monitorar milhares de transmissões ao vivo, identificando problemas de qualidade de áudio e vídeo, como imagens corrompidas e falta de sincronização entre os dois.

### 6.1 Problemas encontrados

No começo, a solução usava uma arquitetura distribuída, com vários componentes rodando separados e organizados por ferramentas como o AWS Step Functions.
Com o tempo, a equipe percebeu que essa estrutura gerava custos altos, principalmente por causa da transferência de dados entre os componentes e da necessidade de orquestrar as etapas do processamento.
Além disso, tinha limite de escalabilidade: a arquitetura chegou no limite com cerca de 5% da carga esperada.

### 6.2 Mudanças realizadas

Pra resolver, a equipe reorganizou a solução, colocando os componentes em um único processo e deixando os dados intermediários em memória.
Com isso, a comunicação ficou mais simples e diminuiu a necessidade de transferir dados entre componentes externos.
Eles também passaram a usar escalabilidade vertical e a distribuir o processamento entre várias instâncias, aumentando a capacidade conforme a demanda.

### 6.3 Resultados

Segundo o artigo da equipe do Prime Video, a mudança reduziu mais de 90% dos custos de infraestrutura e ainda melhorou a escalabilidade do serviço.
Esse caso mostra que microsserviços nem sempre são a melhor escolha. Dependendo da aplicação, uma arquitetura mais simples pode dar resultado melhor, tanto em desempenho quanto em custo.

### 6.4 Relação com o StreamFlow

O caso do Prime Video se relaciona com alguns problemas que encontrei no StreamFlow.
Nos testes, vi que o fluxo de reprodução faz várias chamadas síncronas entre serviços. Essas chamadas aumentam o tempo de resposta e criam dependências entre os componentes.
Assim como no Prime Video, vale avaliar se separar todas as funcionalidades em serviços independentes realmente compensa a complexidade que isso traz.
Isso não quer dizer que todos os microsserviços devem virar um monólito. A escolha depende do que cada sistema precisa, considerando escalabilidade, desempenho, custo de comunicação e facilidade de manutenção.
O principal aprendizado é que não existe uma arquitetura que seja a melhor pra todas as aplicações. Precisa analisar as necessidades do projeto antes de decidir.

## 7. Recomendações e decisões arquiteturais

Depois dos testes e da análise do código do StreamFlow, encontrei alguns pontos que poderiam melhorar. A tabela abaixo mostra as principais recomendações, ordenadas por prioridade.

| Prioridade | Problema identificado | Recomendação | Benefício esperado |
| Alta | Falha no registro de visualizações quando Recommendation está indisponível | Implementar filas de mensagens, retentativas e processamento assíncrono | Evitar perda de informações |
| Alta | Compartilhamento do banco entre Billing e Analytics | Separar os bancos e utilizar APIs ou eventos para disponibilizar os dados | Reduzir o acoplamento entre serviços |
| Alta | Ausência de rastreamento distribuído identificado na análise | Implementar OpenTelemetry e identificadores de correlação | Facilitar a investigação de falhas |
| Média | Chamadas síncronas entre Streaming e serviços auxiliares | Avaliar quais operações podem ser assíncronas | Reduzir dependências e o tempo de resposta |
| Média | Ausência de monitoramento centralizado identificado | Centralizar logs e métricas com ferramentas como Grafana e Prometheus | Melhorar o acompanhamento da aplicação |
| Média | Verificação de saúde com comparação incorreta de estados | Manter testes de saúde e validações automatizadas | Identificar problemas de disponibilidade corretamente |

### 7.1 Considerações finais

Na análise, deu pra ver que a arquitetura de microsserviços do StreamFlow tem vantagens, principalmente pela separação das responsabilidades e pela organização das funcionalidades em serviços diferentes.
Mas os testes também mostraram desafios. A comunicação entre vários serviços pode aumentar o tempo de resposta, o banco compartilhado cria dependências e a indisponibilidade de um serviço pode causar perda de informação.
Um dos pontos mais importantes que observei foi que o sistema continua fazendo reproduções mesmo com alguns serviços fora do ar, mas não tem mecanismos suficientes pra recuperar as informações que deixam de ser registradas durante as falhas.
Por isso, as principais melhorias que propus são diminuir o acoplamento entre os serviços, usar comunicação assíncrona quando for necessário e melhorar as ferramentas de observabilidade.
Por fim, o caso do Prime Video também mostrou que uma arquitetura mais distribuída nem sempre é a melhor solução. O importante é escolher a estrutura de acordo com o que o sistema precisa, buscando equilíbrio entre desempenho, confiabilidade, manutenção e custos.

## 8. Referências

BERNARDI, Elder. Guia completo de microsserviços. Disponível em: https://elderbernardiifsul.github.io/StreamFlow-Microsservices-madness/guia_completo_microsservicos.html. Acesso em: 4 nov. 2026.

BERNARDI, Elder. Vídeo de apoio sobre microsserviços. YouTube. Disponível em: https://www.youtube.com/watch?v=ILXcsNxbas4. Acesso em: 4 nov. 2026.

KOLNY, Marcin. Scaling up the Prime Video audio/video monitoring service and reducing costs by over 90%. Prime Video Tech, 22 mar. 2023. Disponível em: https://www.wudsn.com/productions/www/site/news/2023/2023-05-08-microservices-01.pdf. Acesso em: 4 nov. 2026.

STREAMFLOW. Código-fonte e documentação do projeto. Arquivos docker-compose.yml, gateway/index.js, services/streaming/index.js e services/analytics/index.js, entre outros arquivos analisados. Repositório utilizado na atividade.