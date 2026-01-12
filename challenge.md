# Shipay Back-End Engineer Challenge

***Nota: Utilizaremos os seguintes critérios para a avaliação: Desempenho, Testes, Manutenabilidade e boas práticas de engenharia de software.***

1.- Sua squad irá desenvolver uma nova funcionalidade que irá prover um serviço de validação de cadastro de clientes, ao informar o CNPJ e o CEP do endereço do cliente iremos consultar duas APIs de terceiros, a primeira irá retornar informações da empresa de acordo com o CNPJ e a segunda API irá retornar detalhes de um endereço a partir de um CEP. Com os resultados das duas APIs iremos comparar o endereço do cadastro da empresa obtido pelo CNPJ com o endereço obtido através da consulta do CEP e verificar se as informações de unidade federativa, cidade e logradouro coincidem, e caso o endereço de uma consulta seja encontrada na outra retornaremos HTTP 200 e na negativa um HTTP 404.
Como este novo serviço deverá ser resiliente e essencial para os nossos cadastros, a solução proposta deverá permitir retentativas automáticas em casos de falhas e o chaveamento entre dois provedores de resolução do endereço pelo CEP, ou seja usaremos a API de um provedor como padrão e caso o serviço esteja fora do ar o serviço proposto deverá chamar o segundo provedor automaticamente após "N" tentativas.
Apesar de depender diretamente do consumo de múltiplas APIs de terceiros a resposta do serviço desenvolvido deverá ser síncrono.
Você pode verificar exemplos das APIs utilizadas em [requests_e_responses_apis_questao_1.json](https://github.com/shipay-pag/tech-challenges/blob/master/back_end/waimea/requests_e_responses_apis_questao_1.json).
Descreva e detalhe como você implementaria o referido serviço? Não é necessário desenvolver o código a menos que você julgue necessário. Sinta-se a vontade para utilizar diagramas, desenhos, descrição textual, arquitetura, design patterns, etc.

R - Apesar de possuir uma formulação conceitualmente simples, a abordabem adotada parte do princípio clássico da ciência da computação, difundido amplamente pelo Edsger Dijkstra: dividir um problema em partes e controláveis. 

A partir dessa premissa o assunto foi dividido em 3 camadas principais: 

Primeira camada: Define-se como a inserção dos dados no sistema. Normalização e validação são realizadas para garantir a qualidade das informações, impedindo a violação de qualquer diretriz da regra de negócio. 

Segunda camada: Tendo os valores normalizados, uma nova validação lógica é realizada na API de terceiros que fornecem informações de CNPJ e obtenção dos dados de endereço associados ao CEP informado, confirmando sua veracidade. 

E por último, a terceira camada: encontra-se a estrutura técnica responsável por garantir a confiabilidade e a resiliência do serviço. Essa camada incorpora mecanismos de tolerância a falhas, como retentativas automáticas, controle de tempo de resposta e chaveamento entre provedores externos. O serviço utiliza inicialmente um provedor principal para a resolução do CEP e, em caso de falhas sucessivas após um número configurável de tentativas, alterna automaticamente para um provedor secundário, assegurando a continuidade da operação mesmo diante de instabilidades externas.


Com isso montei esse diagrama: 

![Diagrama De Regra de Negócios](diagrams/diagram_for_business_questions.jpg)


A solução proposta possui algumas camadas de abstração, conforme demonstrado, exigindo uma arquitetura fragmentada, onde a regra de negócio não depende diretamente do framework para funcionar. 

Nesse contexto, a adoção do padrão Orchestrator me pareceu natural. Apesar do termo Orchestrator não apareça explicitamente na literatura clássica, ela é amplamente difundida e aplicada às arquiteturas modernas e pode ser identificada em obras como Arquitetura Limpa, do Robert C. Martin (Nosso Uncle Bob), sob a nomeclatura de UseCase.

A escolha se dá justamente pela possibilidade de contar uma “história” do negócio, promovendo uma separação lógica que identifica o que é SQL, consultas HTTP, FastAPI. Isso facilita em manutenabilidade e em troca de ferramentas, como banco de dados e frameworks. 

Além disso permite que as camadas sejam testadas isoladamente. Ou seja, serviços  pode ser validados sem a necessidade de um banco de dados ou infraestrutura externa, a não ser que a própria regra exija a execução em consultas reais.


Diagrama de exemplo: 

![Diagrama Técnico](diagrams/tech_diagram_for_first_tech.jpg)

2.- Foi nos solicitado a criação de um relatório que mostre a utilização do serviço de lançamentos de foguetes separados por cada um dos nossos clientes em um intervalo de 30 dias. A nossa proposta para o desenvolvimento deste relatório é o de tentar evitar ao máximo algum impacto no fluxo de execução deste endpoint/api (de lançamento de foguetes), uma vez que este é o principal produto da empresa. 
Com essas premissas em mente, o time propôs a utilização apenas das solicitações/requests em comum com o atual serviço e armazenar os dados necessários para o relatório utilizando uma base de dados paralela à base de dados do serviço de lançamentos.
Como você atenderia essa demanda? Lembre-se, caso o novo workflow proposto para o armazenamento dos dados dos relatórios falhe, ele não deve impactar no serviço de lançamentos. 
Descreva em detalhes como você implementaria a solução. Sinta-se a vontade para utilizar diagramas, desenhos, descrição textual, arquitetura, design patterns, etc.

```python
# Linguagem: Python

from uuid import uuid4
from dependency_injector.wiring import Provide, inject
from fastapi import APIRouter, Depends, FastAPI, HTTPException, Request
from shipay_auth_async.interfaces import AuthAdapter
from shipay.infrastructure.containers import Container
from shipay.endpoints.schemas import LaunchRequestBody
from shipay.launch.rules import RulesEngine
from shipay.services import LaunchService

router = APIRouter()
auth_client = shipay_auth_async.client()

@router.post('/v1/rocket/launch', tags=['rocket'])
@inject
@auth_client.required_authenticator([AuthAdapter(Container.claims_service(),
                                                 route='/v1/rocket/launch/post',
                                                 type_filter='resource'),
                                     AuthAdapter(Container.jwt_service())])
async def launch(request: Request, schema: LaunchRequestBody = None,
                 service: LaunchService = Depends(Provide[Container.launch_service])):
    try:
        trace_id = request.headers.get('trace_id', str(uuid4()))
        if not schema:
            raise HTTPException(status_code=400, detail='where is the request payload?')
        
        if not RulesEngine.is_launch_approved(schema):
            raise HTTPException(status_code=405, detail='your launch is not allowed.')
            
        pre_flight = await service.pre_flight_check()
        
        if not RulesEngine.is_pre_flight_status_equals_ok(pre_flight.status):
            raise HTTPException(status_code=500, detail='your launch is compromised, please abort.')
        
        countdown_status = service.countdown()

        return await service.launch(trace_id=trace_id,
                                    customer_id=schema.customer_id,
                                    countdown_status=countdown_status,
                                    pre_flight=pre_flight)

    except Exception as exception:
        raise HTTPException(status_code=500, detail=f'Error during launch...{exception.args[0]}')
```

R - Apesar de entender que o Kafka é difundido para esse tipo de problema, vou aplicar a solução utilizando Celery/Redis, ferramenta a qual tenho conhecimento pleno.

Diagrama: 

![Diagrama de Solução de Teste](diagrams/diagram_for_test_question.jpg)


Exemplo básico de implementação: 

```
# task do celery 

from celery import shared_task
from datetime import datetime
from loguru import logger
from shipay.database.reporting import save_launch_report # é uma função mockada, para simplificar a resposta

@shared_task(bind=True, autoretry_for=(Exception,), retry_backoff=5, retry_kwargs={"max_retries": 5})
def collect_launch_event(self, trace_id: str, customer_id: str, status: str):

    save_launch_report(
        trace_id=trace_id,
        customer_id=customer_id,
        status=status,
        created_at=datetime.utcnow()
    )
```


Já na camada mais alto-nível a task do celery tem um acoplamento bem mais simples: 

```
from shipay.tasks.reporting import collect_launch_event

@router.post('/v1/rocket/launch', tags=['rocket'])
@inject
@auth_client.required_authenticator([AuthAdapter(Container.claims_service(),
                                                 route='/v1/rocket/launch/post',
                                                 type_filter='resource'),
                                     AuthAdapter(Container.jwt_service())])
async def launch(request: Request, schema: LaunchRequestBody = None,
                 service: LaunchService = Depends(Provide[Container.launch_service])):

    trace_id = request.headers.get('trace_id', str(uuid4()))

    if not schema:
        raise HTTPException(status_code=400, detail='where is the request payload?')

    if not RulesEngine.is_launch_approved(schema):
        raise HTTPException(status_code=405, detail='your launch is not allowed.')

    pre_flight = await service.pre_flight_check()

    if not RulesEngine.is_pre_flight_status_equals_ok(pre_flight.status):
        raise HTTPException(status_code=500, detail='your launch is compromised, please abort.')

    countdown_status = service.countdown()

    result = await service.launch(
        trace_id=trace_id,
        customer_id=schema.customer_id,
        countdown_status=countdown_status,
        pre_flight=pre_flight
    )

    # Sem o .delay, o método vai ser executado como uma função, e caso dê erro, vai parar a funcionalidade. 
    collect_launch_event.delay(
        trace_id=trace_id,
        customer_id=schema.customer_id,
        status=result.status
    )

    return result

```

Em resumo: Após a conclusão de um lançamento com sucesso, um evento é disparado de forma assíncrona para uma task do Celery. Essa task é responsável por persistir as informações necessárias em uma base de dados dedicada a relatórios, completamente isolada da base principal do serviço de lançamentos.

3.- Para evitar sobrecargas em serviços de terceiros, nossa squad decidiu implementar um agendador de eventos para ser utilizado durante a verificação do status de execução de uma operação de reenderização de vídeos em um dos nossos workflows orquestrados utilizando kafka. Como o kafka não permite o agendamento de eventos, a squad acabou por desenvolver um agendador próprio que armazena o evento temporariamente em um banco de dados do tipo chave/valor em memória, bem como um processo executará consultas (em looping) por eventos enfileirados no banco chave/valor que estão com o agendamento para vencer. Ao encontrar um, este agendamento é transformado em um novo evento em um tópico do kafka para dar continuidade ao workflow temporariamente paralizado pelo agendamento e finalmente removido do banco de agendamentos. Confome ilustrado no diagrama [event_scheduler.png](https://github.com/shipay-pag/tech-challenges/blob/master/back_end/waimea/event_scheduler.png).
Como o referido workflow deverá ser resiliente e essencial para o nosso produto, a squad gostaria de garantir que o serviço conseguirá suportar 1.000 requesições por segundo com o P99 de 30ms de latencia nas requisições. Descreva detalhadamente quais testes você desenvolveria e executaria para garantir as premissas? Como você faria/executaria os testes propostos?

```python
# Linguagem: Python

from rq import Queue
from redis import Redis
from functions import publish_event

from fastapi import APIRouter, HTTPException, Request
from shipay_auth_async.interfaces import AuthAdapter
from shipay.infrastructure.containers import Container
from shipay.endpoints.schemas import RequestBody


router = APIRouter()
auth_client = shipay_auth_async.client()

@router.post('/v1/render/scheduler', tags=['render'])
@auth_client.required_authenticator([AuthAdapter(Container.claims_service(),
                                                 route='/v1/render/scheduler/post',
                                                 type_filter='resource'),
                                     AuthAdapter(Container.jwt_service())])
async def scheduler(request: Request, schema: RequestBody = None):
    try:

        if not schema:
            raise HTTPException(status_code=400, detail='where is the request payload?')
            
        queue = Queue(name='default', connection=Redis())

        return await queue.enqueue_at(schema.scheduler_datetime, publish_event, schema.event_content)

    except Exception as exception:
        raise HTTPException(status_code=500, detail=f'Error during scheduler event...{exception.args[0]}')
```

R- Para garantir que a solução atenda às premissas estabelecidas no enunciado (especialmente a capacidade de suportar 1.000 requisições por segundo com P99 de 30ms) foi definida uma estratégia de testes composta por múltiplas camadas de validação.

Como ferramenta principal de execução do testes, foi escolhido o K6, que se destaca atualmente no mercado por sua eficiência em testes de perfomance e pela excelente integração com o ecossistema Grafana, permitindo visualização clara de métricas e acompanhamento detalhado do resultados. 

A estratégia contemplate os seguintes tipos de teste:

> Testes de Carga (Load Testing): validação do comportamente do sistema sob volume esperado de requisições.

> Testes de Estresse (Stress Testing): identificação do limite máximo suportado antes da degradação do serviço.

> Testes de Resistência (Soak Testing): verificação da estabilidade da aplicação sob carga contínua por longos períodos. 

> Testes de Resiliência: simulação de falhas em dependências como Redis, workers e Kafka, avaliando a capacidade de recuperação. 

> Testes de Concorrência e Consistência: garantia de ausência de perda, duplicação ou corrupção de eventos sob alto paralelismo. 

Essa abordagem permite avaliar não apenas desempenho, mas também robustez, previsibilidade e confiabilidade do sistema em condições reais de produção. 

4.- Você ficou responsável por mentorar um novo membro do time que além de novo na empresa possui o perfil de nível junior. Ele está finalizando o desenvolvimento de um novo microserviço e está com dúvidas quanto a possíveis implementações de "anti-patterns" em seu código e gostaria da sua avaliação... Quantos anti-patterns você consegue identificar no código dele (se é que existe algum), e caso tenha encontrado por qual motivo você categorizou a implementação como sendo um anti-pattern? *** O código a ser avaliado está disponibilizado no diretório [anti_patterns](https://github.com/shipay-pag/tech-challenges/tree/master/back_end/waimea/anti_patterns).

R - De modo geral, o código apresentado demonstra boa organização e aplicação de princípios fundamentais de engenharia de software, especialmente os princípios do SOLID. Observa-se, por exemplo, a aplicação clara do Prinípio da Responsabilidade Única, evidenciada pela separação da camada de repositórios, evitando que outras partes do sistema lidem dieratamente com acesso a dados. Também é perceptível a aplicação adequada do Princípio da Substituição de Liskov (LSP) por meio da utilização de interfaces como IDatabase e IRepository, permitindo a troca de implementação sem impacto no restante do sistema.  

Apesar desses pontos positivos, alguns anti-patterns relevantes podem ser identificados.

O primeiro desles refere-se à organização da camada de serviços. A existência de arquivos genericos como service.py tende a gerar problemas de escalabilidade à medida que a regra de negócio cresce, concentrando múltimplas responsabilidades em um único arquivo. Uma organização mais adequada seria a divisão por contexto funcional, por exemplo:

- services/
    - crud.py
    - filters.py
    - handler.py 


Essa abordagem melhora significativamente e legibilidade, a testabilidade e a manutenção do código.

O segundo e mais crítico anti-pattern identificado é a existência do arquivo tools.py Esse arquivo concentra responsabilidades extremamente distintas, como:

> acesso a banco de dados (get_role_by_entity_type, get_secrets_by_id),

> validação e normalização de dados (cnpf_has_its_format_validated),

> Integração com serviços externos (send_instant_message)

Essa mistura de responsabilidades caracteriza o conhecido God Class, dificultando a manutenção, reutilização e os testes, além de violar diretamente o Princípio da Responsabilidade Única. 

Esse problema se agrava pelo fato de tools.py estar localizado dentro do diretório de middlewares, o que representa um erro conceitual importante. Middlewares devem conter apenas código relacionado ao pipeline de requisição e resposta (como loggin, autenticação e tratamento global de erros) e não lógica de negócio, acesso a banco ou integração externa.

Por fim, embora o Príncipio da Substituição de Liskov esteja corretamente aplicado na camada de infraestrutura, observa-se que sua utilização não é refletida de forma consistente nas camadas mais próximas do usuário, como service.py e controller.py, o que limita parte dos benefícios da abstração proposta.   

5.- ATENÇÃO: Caso você tenha escrito o código para responder a questão 1, por favor desconsiderar a questão 5 e nos encaminhe o código da questão 1 no lugar.
 Tomando como base a estrutura do banco de dados fornecida (conforme diagrama [ER_diagram.png](https://github.com/shipay-pag/tech-challenges/blob/master/back_end/waimea/ER_diagram.png) e/ou script DDL [1_create_database_ddl.sql](https://github.com/shipay-pag/tech-challenges/blob/master/back_end/waimea/1_create_database_ddl.sql), disponibilizados no repositório do github) construa uma API REST em Python que irá criar um usuário. Os campos obrigatórios serão nome, e-mail e papel do usuário. A senha será um campo opcional, caso o usuário não informe uma senha o serviço da API deverá gerar essa senha automaticamente.

R - Para esta API REST, optei por uma arquitetura simples e bem estruturada, adotando um padrão de projeto que permite alta reutilização e facilidade de manutenção. O padrão escolhido foi o Factory Method, pois ele facilita a padronização de comportamentos e a expansão do sistema sem impactar os módulos existentes.

Além disso, a solução foi desenvolvida seguindo princípios do SOLID e conceitos de Arquitetura Limpa, garantindo baixo acoplamento, alta coesão e maior testabilidade do código.

Todo o código-fonte está disponível no GitHub. Paralelamente, realizei o provisionamento manual de uma instância EC2 e um banco de dados RDS para viabilizar testes mais realistas da aplicação em ambiente de nuvem.

Devido ao limite de tempo, não foi possível implementar uma pipeline de CI/CD com GitHub Actions. No entanto, a aplicação encontra-se totalmente operacional em ambiente Linux, utilizando Docker Compose para orquestração dos serviços e Nginx como servidor de aplicação.

[Clique aqui para acessar a api](http://56.124.87.202/docs#/)

Usuário para login:

> email: ivancampelo@gmail.com

> password: A1b2c3d4@.

[Clique aqui para acessar o código no github](https://github.com/IvanCampelo22/shipay_rest_api)

Gostaria de destacar que existem alguns aprimoramentos planejados que, por limitações de tempo, não puderam ser implementados nesta versão da solução. Entre eles, a inclusão de testes unitários, com o objetivo de elevar o nível de confiabilidade e segurança do código. A implementação de um mecanismo de cache básico nos endpoints de filtros e listagens, reduzindo significativamente o tempo de resposta e a carga sobre o banco de dados. E a configuração de um domínio próprio com DNS, possibilitando a integração com serviços de segurança e performance como o Cloudflare, além de serviços de mensageria como o AWS SES para envio de notificações por e-mail.


6.- Ajude-nos fazendo o ‘Code Review’ do código de um robô/rotina que exporta os dados da tabela “users” de tempos em tempos. O código foi disponibilizado no mesmo repositório do git hub dentro da pasta [bot](https://github.com/shipay-pag/tech-challenges/tree/master/back_end/waimea/bot). ***ATENÇÃO: Não é necessário implementar as revisões, basta apenas anota-las em um arquivo texto ou em forma de comentários no código.***


R - A análise detalhada do código do robô pode ser encontrada no arquivo abaixo:

[Clique aqui para acessar o Code Review completo](code_review_bot.txt)

7.- Qual ou quais Padrões de Projeto/Design Patterns você utilizaria para normalizar serviços de terceiros (tornar múltiplas interfaces de diferentes fornecedores uniforme), por exemplo serviços de disparos de e-mails, ou então disparos de SMS. ***ATENÇÃO: Não é necessário implementar o Design Pattern, basta descrever qual você utilizaria e por quais motivos optou pelo mesmo.***

R - O padrão de projeto mais adequado para esse cenário é o Factory Method, combinado com o uso do Facade para a simplificação da API exposta à aplicação. 

O Factory Method é responsável por criar e normalizar as instâncias dos diferentes provedores, garantindo que funcionalidades comuns sejam acessadas de forma uniforme, mesmo quando implementadas por serviços distintos. Para que essa abordagem funcione corretamente, o Princípio da Segregação de Interface (ISP) deve ser aplicado de forma rigorsa, assegurando que cada módulo dependa apenas das interfaces estritamente necessárias, evitando acoplamentos desnecessários entre componentes.

O Facade entra como uma camada de abstração adicional, restringindo e organizando o acesso dos controllers aos serviços internos, fornecendo uma API mais simples, coesa e estável para o restante da aplicação.

Essa combinação de padrões promove desacoplamento, extensibilidade, manutenabilidade e clareza arquiteturial, além de facilitar a substituição ou inclusão de novos forncedores sem impacto significativo no código existente.

BOA SORTE!
