# snapshot_to_s3
Quick step by step on how to create a snapshot of elasticsearch indexes and send them to an s3 bucket


# # Artigo : ElasticSearch - Snapshot de índices para um Bucket-s3

Este artigo descreve um passo a passo de como realizar backup dos indices do elasticsearch para um bucket na cloud da Amazon através de uma API, repository-s3.

Em ambientes pequenos, geralmente não damos importância para a quantidade e tamanho de armazenamento que os índices do elasticsearch consomem. Porém, como armazenamento é limitado, e dependendo do tipo de ambiente, a quantidade de dados pode passar de gigas para tera ou até mesmo petabytes em questão de horas ou dias, o gasto com armazenamento se torna um fator exponencial. Se não houver uma política de controle de rotação para dados antigos, os valores podem ser exorbitantes. O elasticsearch provê algumas configurações de rotação e deleção por datas e tamanho dos índices, e ajuda muito na liberação de espaço de armazenamento. Mas e quando não podemos perder esses dados? Então, devemos procurar alguma solução para armazenar esses índices de forma menos custosa, aí que entra o tema deste artigo, felizmente a AWS possui algumas ferramentas que auxiliam no armazenamento de informações, de acordo com nível de acessibilidade e "urgência" em caso de necessiadade de resgate dessas informações, que variam de alguns minutos/segundos, horas ou até mesmo dia. Nesse ponto, quanto menor for a necessidade de obter essas informações salvas de forma rápida, menor será o gasto com o armazenamento. Realocar  índices antigos para dispositivos/tecnologias que diminuem o impacto financeiro pode ser uma mão na roda para amenizar os custos com a infraestrutura. 

Existem algumas  ferramentas e produtos dentro da Aws para armazenamento de informações, neste caso iremos utilizar o Amazon S3. Abaixo uma pequena descrição das soluções :

#### Amazon S3

Ou Amazon Simple Storage Service (S3) - 

> "É um serviço de armazenamento de objetos que oferece escalabilidade
> líder do setor, disponibilidade de dados, segurança e performance.
> Isso significa que clientes de todos os tamanhos e setores podem
> usá-lo para armazenar qualquer volume de dados em uma grande variedade
> de casos de uso, como sites, aplicações para dispositivos móveis,
> backup e restauração, arquivamento, aplicações empresariais,
> dispositivos IoT e análises de big data. O Amazon S3 fornece recursos
> de gerenciamento fáceis de usar, de maneira que você possa organizar
> os dados e configurar os controles de acesso refinados para atender a
> requisitos específicos comerciais, organizacionais e de conformidade.
> O Amazon S3 foi projetado para 99,999999999% (11 9s) de durabilidade e
> armazena dados para milhões de aplicativos para empresas de todo o
> mundo."

https://aws.amazon.com/pt/s3/

Conforme dito no início do artigo, o nível de urgência que se tem caso haja necessidade de resgatar esses dados vai definir a "Classe de Armazenamento" que precisaremos adotar no s3.

Abaixo uma pequena descrição de cada Classe de Armazenamento que a Amazon disponibiliza no serviço:

**Amazon S3 Standard (S3 Standard)** -  Uso mais comum, esta é a classe padrão que é adotada quando não especifica nenhuma outra. 

 - Baixa latência e alto throughput
 - Projetada para fornecer 99,999999999% de resiliência de objetos em várias zonas de disponibilidade
 - Resiliência contra eventos que causam impactos em uma zona de disponibilidade inteira
 - performance para dados acessados **com frequência**

**Amazon S3 Intelligent-Tiering** - projetada para otimizar os custos movendo automaticamente os dados para o nível de acesso mais econômico, sem impacto na performance ou sobrecarga operacional. Monitora objetos e os move entre classe que são, acessados com mais frequência e acessados com menos frequência.**Por uma pequena taxa mensal de automação e monitoramento por objeto**, o Amazon S3 monitora os padrões de acesso dos objetos no S3 Intelligent-Tiering e move aqueles que não foram acessados por 30 dias consecutivos para o nível de acesso infrequente.

 - A mesma performance de baixa latência e alto throughput da categoria S3 Standard
 - Pequena taxa mensal de monitoramento e divisão automática em níveis
 - Projetada para fornecer 99,999999999% de resiliência de objetos em várias zonas de disponibilidade
 - Resiliência contra eventos que causam impactos em uma zona de disponibilidade inteira
 - um nível é otimizado para acesso frequente e outro nível de custo mais baixo, que é otimizado para acesso infrequente.
 
 **Amazon S3 Standard-Infrequent Access** - é indicado para dados acessados com menos frequência, mas que **exigem acesso rápido quando necessários**.

 - A mesma performance de baixa latência e alto throughput da categoria S3 Standard
 - Projetada para fornecer 99,999999999% de resiliência de objetos em várias zonas de disponibilidade
 - com taxas reduzidas por GB de armazenamento e GB de recuperação
 
 **Amazon S3 One Zone-Infrequent Access** - Ao contrário de outras classes de armazenamento do S3, que armazenam dados em no mínimo três Zonas de disponibilidade (AZs), a S3 One Zone-IA armazena dados em uma única AZ, com um custo 20% inferior ao S3 Standard-IA. Ideal para clientes que querem uma opção de menor custo para dados acessados com pouca frequência, mas não precisam da disponibilidade e da resiliência S3 Standard ou S3 Standard-IA.

 - A mesma performance de baixa latência e alto throughput da categoria S3 Standard
 - Projetada para fornecer 99,999999999% de resiliência de objetos em **uma única zona de disponibilidade†**.
 - † Como a categoria S3 One Zone – IA armazena dados em uma única zona de disponibilidade da AWS, os dados armazenados nessa categoria de armazenamento serão perdidos em caso de destruição da zona de disponibilidade.
 
 **Amazon S3 Glacier** - é uma classe de armazenamento segura, durável e de baixo custo para arquivamento de dados. Você pode armazenar com confiabilidade qualquer volume de dados a um custo competitivo ou inferior ao custo de soluções no local. Para manter os custos baixos, mas com condições de suprir necessidades variáveis, o S3 Glacier disponibiliza três opções de recuperação, que podem levar de alguns minutos a várias horas.

 - Projetada para fornecer 99,999999999% de resiliência de objetos em várias zonas de disponibilidade
 - O design de custo baixo é ideal para arquivos de longa duração
 - **Tempos de recuperação configuráveis que variam de minutos a horas**
 - -   API PUT do S3 para uploads diretos ao S3 Glacier e gerenciamento de ciclo de vida do S3 para migração automática de objetos
 
 **Amazon S3 Glacier Deep Archive** - é a classe de armazenamento mais barata do Amazon S3 e oferece suporte à retenção e preservação digitais de longo prazo para dados que podem ser acessados uma ou duas vezes por ano. Essa classe é projetada para clientes que mantêm conjuntos de dados por 7 a 10 anos ou mais para cumprir requisitos de conformidade normativa, especialmente em setores altamente regulados como serviços financeiros, saúde e setores públicos. O S3 Glacier Deep Archive também pode ser usado para casos de uso de backup e recuperação de desastres, além de ser uma alternativa mais barata e fácil de gerenciar em comparação aos sistemas de fita magnética como bibliotecas locais ou serviços externos.

 - Projetada para fornecer 99,999999999% de resiliência de objetos em várias zonas de disponibilidade
 - A classe de armazenamento com custo mais baixo projetada para retenção de dados em longo prazo que serão mantidos por 7 a 10 anos.
 - Tempo de recuperação de até 12 horas
 - API PUT do S3 para uploads diretos ao S3 Glacier Deep Archive e gerenciamento de ciclo de vida do S3 para migração automática de objetos


Como podemos ver, as Classes de Armazenamento possuem características importantes que devemos levar em consideração e por na ponta do lápis afim de reduzir os custos de armazenamento. Neste artigo iremos realizar o snapshot dos índices em um Bucket S3 Standard, porém futuramente iremos migra-los para uma outra classe com intuito de diminuir nossos custos com armazenamento.

# Etapas

No nosso ambiente temos 4 servidores com Elasticsearch, um deles é escalado como node Master. Utilizamos o Kibana, que faz parte do conjunto de ferramentas que formam a ELK Stack (Elasticsearch, Kibana e Logstash), para leitura e analise das informações contidas no Elasticsearch, de quebra, o Kibana nos dá um console de desenvolvimento, onde podemos brincar e interagir com os índices, sem necessariamente acessarmos um terminal onde o elasticsearch está instalado.

### Env

**Cluster**: Elasticsearch formado por 4 Servidores

**Kibana**: deve estar conectado a este cluster

**Plugin** : fundamental para fazer interface com o serviço da Aws S3

A versão do Elasticsearch e kibana que estamos utilizando no momento deste tutorial é a 6.5.1.

### Passos

Instalação do plugin repository-s3, deve ser realizado no node Master do elasticsearch, esse plugin é responsável por fazer a interface com o serviço do Amazon S3.

    sudo bin/elasticsearch-plugin install repository-s3

Note1: A versão do plugin instalada segue a a mesma versão do Elasticsearch.
Note2: Após a instalação do plugin deve-se reiniciar o serviço do Elasticsearch.

    sudo systemctl restart elasticsearch

Verifique se o plugin foi instalado :

    curl -X GET "localhost:9200/_cat/plugins"
 ou ainda...
 

    /usr/share/elasticsearch/bin/plugin list
  
  Lembrando que o procedimento pode ser realizado dentro do console do Kibana.

    GET _cat/plugins
   
   Output : ip-xx.xx.xx.xx repository-s3 6.5.1
 
 Existem algumas formas de fazer com que um usuário ou servidor se conecte com recursos da aws, no nosso exemplo, o elasticsearch deve pode escrever (armazenar) seus indices no bucket que iremos criar mais lá na frente. Listarei duas delas:

IAM role : Uma IAM role atrelada a Ec2 no momento de sua criação.
User : Usuário com role/policy e credentials para acessar recursos.

No nosso caso, optamos por usar um usuário, visto que nosso cluster **já está em funcionamento**, caso você esteja criando um cluster do zero, a opção de IAM Role é a que faz mais sentido. 

### Criação do usuário para acesso ao BucketS3

No console da Amazon siga os passos a seguir :

 - Serviços -> IAM -> Usuários -> Adicionar Usuário ->
 - Insira um nome para o usuário
 - Marque a caixa "Tipo de acesso Programático"
	 - Essa opção nos dará acesso as credentials do usuário (access e secret key), faça o download do arquivo .csv e guarde em um local seguro, iremos precisar dele em breve.

### Criação de grupo com policy aplicada

Ainda na aba lateral do IAM

 - Acesse a aba grupos
 - Insira um nome adequado ao grupo ex: (elastics3bucket)
 -  Procure pela Policy (AmazonS3FullAccess), isso dará aos usuário pertencentes ao grupo acesso completo ao AmazonS3.
 - Conclua o processo e adicione o usuário ao grupo recém criado.

Outra alternativa seria adicionar uma politica diretamente ao usuário :

    {
    "Version": "2012-10-17",
    "Statement": [
        {
            "Action": [
                "s3:ListBucket"
            ],
            "Effect": "Allow",
            "Resource": [
                "arn:aws:s3:::nome_do_meu_bucket"
            ]
        },
        {
            "Action": [
                "s3:GetObject",
                "s3:PutObject",
                "s3:DeleteObject",
                "iam:PassRole"
            ],
            "Effect": "Allow",
            "Resource": [
                "arn:aws:s3:::nome_do_meu_bucket/*"
            ]
        }
    ]


### Criação do Bucket

Acesse o painel de serviços da Amazon e siga os passos:

 - Amazon S3 -> buckets -> Criar Bucket
 - Basicamente next e next, aqui seria a hora de alterar a classe de armazenamento, porém no nosso cenário, iremos deixar tudo no padrão.
 - Em region, escolhemos colocar na mesma região no qual se encontra nosso cluster, mas isso não é obrigatório.
 
 Note: O nome do bucket é global e único, escolha um nome que melhor se adeque as suas necessidade.

## Configurando e registrando nosso snapshot

Agora devemos realizar as configurações de acesso no nosso nó master, para isso faremos o uso do utilitário do próprio Elasticsearch :

    elasticsearch-keystore

Na máquina utilize o seguinte comando e insira o value da access-key que fizemos o download quando criamos nosso usuário :
  

    $cd /usr/share/elasticsearch/bin/

    ./elasticsearch-keystore add s3.client.default.access_key
Insira o valor no prompt que irá se abrir.

Agora faremos o mesmo para o secret-key

    ./elasticsearch-keystore add s3.client.default.access_key
Insira o valor da secret-key no prompt que irá se abrir.

Feito isso, iremos registrar nosso repositorio de snapshot. Vamos para o console do Kibana :

   

     PUT _snapshot/s3_repository?verify=false&pretty
    {"type":"s3","settings":{"bucket":"nome_do_seu_bucket","region":"us-west-2"}}

Caso queira fazer direto do console do nó master :

   

         curl -XPUT 'http://localhost:9200/_snapshot/s3_repository?verify=false&pretty' -d'
       {
         "type": "s3",
         "settings": {
          "bucket": "nome_do_seu_bucket",
          "region": "eu-west-2"
        }
  

Pronto, agora que registramos nosso repositório de snapshot, podemos iniciar um snapshot de apenas um índice, ou todos os índices que existem no cluster.

**Os passos abaixo realizam um snapshot completo dos indices do cluster.**


Pelo kibana :

    PUT _snapshot/s3_repository/nome_do_snapshot?pretty?wait_for_completion=true"

Note: wait_for_completion - Quando true, informa ao comando para aguardar até a conclusão do snapshot para informar o status da execução.

Pela command line do cluster :

    curl -XPUT "http://localhost:9200/_snapshot/s3_repository/nome_do_snapshot?pretty?wait_for_completion=true"


Os passos abaixo realizam snapshot de um ou mais indices do cluster.

Pelo kibana :

 

       PUT _snapshot/s3_repository/snapshot1
    {
      "indices": "indice_1, indice_2...",
      "ignore_unavailable**": true,  
      "include_global_state**": false
    }

Pelo command line :

        curl -XPUT "http://localhost:9200/_snapshot/s3_repository/snap1?pretty?wait_for_completion**=true" -d'  
    {  
    "indices": "products, index_1, index_2",  
    "ignore_unavailable": true,  
    "include_global_state": false  
    }
    

Atributos:

**ignore_unavailable** -  Se true, se um índice não existir, ele skipa para o proximo sem quebrar a cadeia.
**include_global_state** - Importante atributo, quando setado como false, possibilita recupear o snapshot em um outro cluster com configurações diferentes, ja que o state guarda informações do cluster no qual foi feito o snapshot.


## Final
Após realizado o snapshot, podemos verificar o status do andamento com o comando abaixo, existem outro, porem esse detalhar bem o que está sendo realizado.

    GET _snapshot/s3_repository/snapshot1?pretty

No final do output teremos uma mensagem similar a esta :

    "include_global_state" : true,
      "state" : "SUCCESS",
      "start_time" : "2020-05-07T00:05:56.739Z",
      "start_time_in_millis" : 1588809956739,
      "end_time" : "2020-05-07T07:18:42.661Z",
      "end_time_in_millis" : 1588835922661,
      "duration_in_millis" : 25965922,
      "failures" : [ ],
      "shards" : {
        "total" : 6060,
        "failed" : 0,
        "successful" : 6060
      }
    }
  ]
}

Outros comandos :
Informa de forma simples com poucos detalhes

    GET _snapshot/s3_repository/_status

Informa todos os snapshots realizados com status final

    GET _cat/snapshots/s3_repository?v
