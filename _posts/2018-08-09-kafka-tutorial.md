---
title: A Kafka tutorial
tags: tutorial, kafka
---

Kafka is a system for message exchanging based on publisher/subscriber paradigm.
As such, an emitter publishes a information within a context without indicating
directly which will be its receivers. These receivers subscribe themselves to
these contexts to receive messages published through them. It is very common to
use a centralized element that deals with context management, published messages
and receiver subscriptions.

In Kafka, the emitters are called "producers", receivers are called "consumers"
and context are called "topics" which can be subdivided into one or more
"partitions". To wrap things up: producers publish messages through a particular
topic. This message will be added to a partition associated to this topic. Consumers
that are subscribed to this topic that are listening to this partition will get the message.
This description is very shallow and omits a lot of important details, which will be explained
in the following details.



Elementos principais
A configuração e funcionamento do Kafka usam os conceitos listados a seguir. Cada um dos itens será detalhado em momento oportuno neste tutorial - a lista será mais útil se considerada como um resumo de tudo o que acontece durante o uso do Kafka.
Instância do Kafka: uma instância de software que desempenhará o papel de broker de mensagens;
Líder: instância do Kafka responsável por eleger líderes de partições;
Lista de brokers de bootstrap: lista de instâncias do Kafka que servirá para informar um cliente da situação atual do cluster;
Cluster: grupo coordenado de instâncias do Kafka que atendem um grupo de clientes;
Cliente: uma instância de software que envia requisições a uma instância do Kafka para obtenção ou transmissão de informações;
Produtor: um cliente que desempenha o papel de emissor de mensagens;
Consumidor: um cliente que desempenha o papel de receptor de mensagens;
Partição: um registro de mensagens associado a um tópico e gerenciado por uma instância do Kafka que mantém um subconjunto de todas as mensagens publicadas neste tópico;
Líder da partição: instância do Kafka responsável pelo gerenciamento de uma partição;
Partição original: uma partição que serve de fonte de informações para a geração de registros em partições de réplica;
Réplica: partições que são cópias idênticas de uma partição original. A instância do Kafka que gerencia as réplicas são diferentes do líder da partição de origem;
ISR (In-sync replica): Partições que estão em sincronia com a partição original;
Grupo de consumidores: conjunto de consumidores que atuam de forma coordenada na recepção de mensagens provenientes de diferentes partições de um tópico;
Líder do grupo de consumidores: instância de um consumidor que atua como gerenciador durante a associação entre consumidores e partições de um tópico;
Controlador: instância do Kafka que atua como seletor de qual consumidor é o líder do grupo de consumidores.
Este tutorial seguirá o fluxo normal de uso do Kafka, detalhando cada fase conforme necessário. A primeira fase é a de inicialização de uma instância do Kafka, passando para conexões de clientes e criação de tópicos. Em seguida, a publicação de mensagens será explicada, assim como seu consumo (incluindo detalhes de como consumidores se organizam para receber mensagens).
Configuração inicial
Ao iniciar uma instância do Kafka, o administrador pode configurar os seguintes parâmetros:
broker.id: identificador da instância caso seja utilizada em um cluster. Este valor deve ser um inteiro maior do que zero e pode ser arbitrário embora deva ser único dentro do cluster;
port: porta de rede através da qual a instância ouvirá mensagens. A porta padrão é a 9092. Não há restrições (além das impostas pelo sistema operacional) caso seja necessário configurar outra porta;
zookeeper.connect: localização do Zookeeper. Esta conexão será utilizada para armazenamento dos metadados do broker para serem consumidos por outras instâncias do Kafka. Este atributo deve conter o endereço e porta do Zookeeper, além do caminho do Znode, como “localhost:2181/path”;
log.dirs: diretório utilizado para armazenamento dos logs do Kafka. As informações salvas neste diretório são relacionadas com as mensagens recebidas pelo broker, não com informações textuais sobre a operação do software. Novos logs para partições são armazenadas nos diretórios que são menos utilizados (não os que possuem menor espaço ocupado em disco);
num.recovery.threads.per.data.dir: número de threads a serem utilizadas para a recuperação de logs. Estas threads serão utilizadas quando: (1) Kafka for iniciado normalmente, (2) após uma falha a fim de verificar a situação de cada segmento de log de uma partição e (3) durante o encerramento do processo para fechar corretamente os segmentos de log;
auto.create.topics.enable: esta configuração habilita ou desabilita a criação automática de um tópico quando: (1) um produtor envia mensagens para o tópico, (2) um consumidor inicia a leitura de mensagens deste tópico ou (3) quando um cliente pede metadados referente a este tópico. Esta configuração deve ser utilizada com cuidado pois, caso esteja habilitada, não haverá como identificar se um tópico não existe, uma vez que o envio deste questionamento ao broker irá criá-lo.
Existem várias outras configurações para um broker, que podem ser conferidas no repositório do projeto. A título de informação, são 182 itens de configuração. Elas podem ser divididas nas seguintes classes:
Zookeeper: configurações relacionadas a como se conectar no Zookeeper;
Identificador do broker: como gerar o identificador do broker sendo iniciado;
Threads de rede, I/O e geração de réplicas, filas de processamento;
Portas e nomes de hosts associados ao broker;
Configuração de conexões de rede (tamanhos de buffer, número de conexões);
Identificador do rack onde o servidor está instalado;
Configuração dos logs (número de partições, tamanho dos segmentos, tempos de retenção, limpeza e liberação, timestamps);
Políticas de gerenciamento de tópico (criação, modificação);
Políticas de gerenciamento de réplicas;
Políticas de encerramento do processo;
Configurações para coordenação de consumidores;
Configurações para gerenciamento de offsets;
Configurações para gerenciamento de transações;
Configurações para gerenciamento de cotas;
Configurações de geração de métricas;
Configurações de segurança (SSL, Sasl, delegação e gerenciamento de senhas).
Após a inicialização, a instância do Kafka se registrará no Zookeeper (especificado no zookeeper.connect) e tentará assumir o papel de líder do seu cluster. Isto é feito através da criação de um nó efêmero “/controller” e, caso não receba um erro de nó existente, esta instância do Kafka torna-se a líder e passa a ter a responsabilidade extra de eleger líderes de partições. Caso o nó “/controller” já exista, nada acontece e a instância torna-se um broker comum.
Todas as instâncias comuns (não líder) monitoram a presença do nó efêmero /controller. Caso ele deixe de existir, o que indica uma falha do atual líder, todas as instâncias são notificadas e a primeira que for capaz de recriar este nó torna-se a nova líder. Para evitar problemas com falhas de conexão entre instâncias e o Zookeeper, o nó /controller possui associado um valor de época: toda vez que este nó é recriado, sua época é incrementada. Assim, caso uma instância receba uma mensagem com época anterior à atual, é seguro ignorá-la.
Após a seleção do líder, a instância torna-se apta a receber requisições de clientes (produtores e consumidores).
Clientes
Como dito anteriormente, um cliente é uma instância de software que envia requisições a uma instância do Kafka para obtenção ou transmissão de informações. Para tal, deve se conectar ao Kafka para obter informações gerais de como estão organizadas as informações dentro do cluster antes de tomar qualquer ação independente do seu papel (produtor ou consumidor). As informações recebidas são:

TopicMetadataRequest
Lista de instâncias do Kafka (incluindo identificador do nó, configurado através do parâmetro broker.id, endereço do servidor e porta de rede);
Lista de informações sobre todos os tópicos, a qual inclui:
Nome do tópico;
Lista de informações sobre suas partições: identificador da partição, líder da partição (identificador da instância), identificador das réplicas (instâncias), lista de ISRs.

Estas informações podem ser obtidas enviando uma requisição a qualquer instância dentro do cluster. Esta instância é uma definidas na lista de brokers de bootstrap.
Criação de tópicos e réplicas
Antes de enviar ou receber mensagens, um cliente deve criar um tópico (caso a opção auto.create.topics esteja desabilitada na configuração da instância do Kafka). Esta operação é feita através de uma mensagem enviada a uma instância do Kafka, a qual contém os seguintes parâmetros:

Create Topics Request
topic: nome do tópico
num_partitions: número de partições que formarão o tópico
replication_factor: número de réplicas para cada partição
replica_assignment: associação {partição, [réplicas]} que indica quais instâncias do Kafka (indicadas pela lista de identificadores de réplicas) serão responsáveis por quais partições deste tópico. O primeiro item desta lista é o líder preferencial.
timeout: tempo em milissegundos esperado para que o tópico seja criado no lado do controlador.

É importante notar que o cliente é responsável pela indicação da distribuição das réplicas entre as instâncias do cluster. No entanto, não cabe a ele escolher qual instância será a líder de cada partição do tópico - esta tarefa é de responsabilidade da instância que receberá esta requisição. O mecanismo de seleção é relativamente simples: ao ser feita uma adição de um registro em uma partição, o líder daquela partição espera todas as réplicas dentro da lista de ISR terminarem o processo de escrita antes de reconhecer o processo de registro da mensagem como terminado. Portanto, quando o líder da partição fica indisponível, qualquer uma das ISRs pode assumir o seu papel. Como esta lista é armazenada no Zookeeper, é relativamente simples fazer esta escolha. Mais detalhes sobre o mecanismo de seleção é relativamente extenso e é detalhado nesta página.
Produtores
A produção de uma mensagem a ser publicada em um tópico é relativamente simples. Tudo o que o produtor necessita enviar é uma mensagem com os seguintes atributos:

Produce Request
RequiredAcks: indicando se reconhecimento de recebimento deve ser necessário. Possíveis valores são 0 (modo fire-and-forget, ou seja, sem confirmação de recebimento), 1 (confirmação do recebimento apenas da instância líder daquela partição, sem confirmação da escrita nas réplicas) e -1 (com confirmação de todas as escritas, na instância líder da partição e suas réplicas.
Timeout: tempo em milissegundos que o produtor irá esperar por todos os RequiredAcks (caso necessário). No entanto, este tempo poderá não ser respeitado completamente: as instâncias do Kafka não interromperão as escritas caso elas levem mais tempo do que o definido neste atributo. Se for necessário um tempo máximo estrito, deve-se utilizar as configurações de socket fornecido pelo sistema operacional.
TopicName: o nome do tópico utilizado para a publicação da mensagem
Partition: identificador da partição do tópico utilizado para publicação da mensagem
MessageSetSize: tamanho do conjunto de mensagens a serem publicadas.
MessageSet: o conjunto de mensagens a serem publicadas.

Estes atributos são os presentes na requisição enviada à instância do Kafka. Como é possível perceber, é necessário enviar previamente qual partição aquela mensagem será publicada. Portanto é necessário que o cliente faça tome esta decisão previamente. A título de exemplo, o cliente padrão fornecido pelo projeto Kafka executa os seguintes procedimentos durante a publicação de uma mensagem:
O emissor da mensagem cria uma estrutura chamada “ProducerRecord” que contém os seguintes atributos:
Topic: nome do tópico usado para publicação;
Partition: parâmetro opcional que define previamente qual a partição a ser utilizada para publicação do conjunto de mensagens;
Key: parâmetro também opcional que serve como entrada para a seleção de uma mesma partição. Publicações que sejam feitas com o mesmo parâmetro “Key” serão associadas ao mesmo tópico;
Value: Mensagem a ser enviada.
Esta estrutura é utilizada por um elemento de software chamado “Partitioner”. Ele leva em consideração quais são as partições criadas nas diversas instâncias para selecionar qual deve ser a partição de destino, caso seja permitido a isso. Estas configurações e mecanismos tendem a ser seguidas por outras implementações de clientes, mas isto não é garantido.
As mensagens completas (com partições já associadas) são armazenadas em conjuntos que serão enviadas à instância líder daquela partição assim que possível.
É importante lembrar que estes procedimentos são executados pelo cliente distribuído pelo projeto Kafka. Outras implementações podem seguir ou não estas recomendações.
Consumidores
Antes de detalhar como é feito a recepção de mensagens por consumidores, é importante entender como um conjunto de consumidores é coordenado para que consigam processar mensagens de forma escalável. Para alcançar tal objetivo, consumidores podem ser associados a grupos: um elemento pertencente a um grupo receberá mensagens armazenadas em um ou mais partições sendo que nenhuma partição será utilizada por mais de um consumidor dentro de um mesmo grupo. A maior vantagem desta abordagem é a distribuição uniforme da carga de processamento de mensagens entre os vários consumidores. A decisão de como será feita a distribuição da carga fica sob responsabilidade do líder do grupo.
Eleição do líder do grupo
Para a criação de um novo grupo, inicialmente um consumidor deve localizar o coordenador, que é uma instância do Kafka que será responsável por gerenciar as informações daquele grupo. Isto é feito através do envio de uma mensagem a um dos bootstrap servers contendo apenas o identificador do grupo a ser criado. A resposta enviada pelo servidor de bootstrap contém os seguintes parâmetros:

Group Coordinator Request
CoordinatorId: identificador da instância que servirá como coordenadora;
CoordinatorHost: endereço onde esta instância está sendo executada;
CoordinatorPort: porta de rede usada por esta instâncias.

Com estas informações, o consumidor pode enviar uma outra mensagem relacionada a sua necessidade de fazer parte de um grupo. Esta mensagem contém os seguintes atributos:

Join Group Request
GroupId: identificador do grupo a ser criado;
SessionTimeout: usado para indicar que um cliente está ativo;
RebalanceTimeout: valor usado durante rebalanceamento para indicar se cliente está ainda ativo;
MemberId: sugestão de identificador de membro do grupo;
ProtocolType: recomenda-se ser sempre “consumer”;
GroupProtocols: uma estrutura contendo dois sub-parâmetros:
ProtocolName: um esquema de associação – a ser utilizado com UserData;
ProtocolMetadata: uma sub-estrutura contendo:
Version;
Subscription: lista de tópicos;
UserData: sequência de bytes.

O coordenador recebe todas as mensagens e envia respostas a todos os clientes. Somente aquele que ele decide ser o líder do grupo, ele envia informações sobre os membros do grupo. Esta mensagem de resposta contém as seguintes informações:

Join Group Response
GenerationId: identificador daquele conjunto de consumidores dentro do grupo. Qualquer alteração do grupo (saída ou entrada de nós) alterará este atributo;
GroupProtocol: mesmo valor da requisição;
LeaderId: identificador do líder
MemberId: identificador do membro adicionado ao grupo;
Members: lista de estruturas {MemberId, MemberMetadata} contendo todos os identificadores dos consumidores dentro do grupo. Como comentado anteriormente, esta lista é recebida exclusivamente pelo líder do grupo selecionado pelo coordenador.

Esta mensagem confirma que o consumidor entrou no grupo requisitado. Para saber qual partição deverá ser utilizada pelo consumidor dentro do grupo, uma terceira mensagem deve ser enviada ao coordenador contendo as seguintes informações:

Sync Group Request
GroupId: identificador do grupo ao qual este consumidor faz parte
MemberId: identificador de membro associado a este consumidores
GroupAssignment: informações de associação entre membros do grupo, tópicos e partições. Contém os atributos:
MemberId: identificador do consumidor que faz parte do grupo
MemberAssignment: uma estutura representando a associação do membro do grupo ao tópico e a uma partição:
Version: versão do protocolo utilizadas
PartitionAssignment: uma lista de pares contendo o tópico inscrito e uma lista de identificadores de partição.
UserData: dados de usuário relacionado a esta mensagem.

Todos os consumidores devem enviar uma mensagem destas ao coordenador, mas apenas o líder deve preencher o atributo GroupAssignment. Assim todos os consumidores têm o conhecimento de qual tópico em qual instância utilizando qual partição deve ser empregada para a recepção de mensagens.
Caso novos consumidores entrem ou deixem o grupo após a sua formação inicial (ou, por extensão, caso o líder do grupo torne-se indisponível), todos consumidores devem deixar o grupo para, em seguida, iniciar a sua nova formação. Enquanto o processo de rebalanceamento está sendo executado, nenhuma mensagem é consumida de partição alguma. Mais informações sobre o processo de formação de um grupo de consumidores podem ser encontradas nesta página.
Consumo de mensagens
Após a recepção da mensagem contendo as informações da instância, tópico e partição, o consumidor pode começar a receber mensagens do Kafka. Isto é feito através do envio periódico de uma mensagem com os seguintes parâmetros:

Fetch Request
ReplicaId: identificador da instância que cuida da réplica a ser atualizada. Em caso de um consumidor comum, este identificador é -1;
MaxWaitTime: tempo máximo (em milissegundos) de espera por informações;
MinBytes: mínimo número de bytes que devem estar disponíveis para formar uma resposta;
TopicName: nome do tópico
Partition: identificador da partição lida
FetchOffset: offset do início da leitura desta requisição
MaxBytes: número máximo de bytes para incluir na mensagem desta requisição.
A resposta é uma mensagem contendo os seguintes atributos:
ThrottleTime: duração de um atraso proposital devido a violação de cotas;
TopicName: nome do tópico lido;
Partition: identificador da partição lida;
HighwaterMarkOffset: offset do fim do log para esta partição;
MessageSetSize: tamanho da lista de mensagens retornada;
MessageSet: lista de mensagens.

