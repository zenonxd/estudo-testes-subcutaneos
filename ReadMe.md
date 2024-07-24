# Estudo Testes Subcutâneos

## Referência: Testes automatizados na prática com Spring Boot
[Compre aqui](https://www.udemy.com/course/testes-automatizados-na-pratica-com-spring-boot/)
<hr>

## Dica para leitura:
Durante o estudo, os códigos mudam pois geralmente é ensinado algo básico onde depois iremos implementar o que de fato
é utilizado no mercado de trabalho. Tome cuidado ao considerar códigos do início do estudo, se atente ao código final.

## Tópicos




<hr>


# Introdução

Aqui trabalharemos com teste de componente e end-to-end.

![img.png](img.png)

As vantagens destes testes é trazer mais segurança, porque ele testa tudo junto! A integração de controller, service e
repository.

Aqui não teremos Mocks, então veremos de fato como ele funciona na vida real (banco, servidor de aplicação).

Não teremos também, vários cenários de teste. Somente testes que nos permitem ver o cenário principal da aplicação. Se
o essencial está funcionando.

**Para estes testes, usaremos um cenário real: servidor tomcat + banco de dados mySQL.**

<hr>

## Configuração do servidor de aplicação

Atualmente os nossos testes rodam em um servidor mockado. Agora, criaremos um teste que utilize um servidor real.

1. Criaremos uma classe chamada PlanetIT dento fora dos pacotes. 

Destacando que, se tivéssemos outros controllers/services seria interessante separar essa classe por domínios. Como
aqui só trabalhamos com Planetas, podemos fazer fora dos pacotes dessa forma.
<hr>

2. Essa classe irá separar entre os testes mais baratos de se executar dos testes mais caros. Agora, faremos os mais
caros, então não seria interessante executar sempre.
<hr>

3. Como esses testes são pesados, trabalharemos somente com cenários de sucesso.


4. Passaremos a anotação @SpringBootTest para subir o servidor de aplicação.

Essa anotação não só sobe o servidor, mas também cria o contexto de aplicação de Spring
e colocar todos os Beans nele. Com isso podemos testar, por exemplo, se existe algum
problema de configuração nos Beans do nosso projeto.


5. Criaremos o método contextLoads(). Caso ocorra algum erro, o teste irá falhar.

PS: Não fica NADA dentro dele, é só criação do método com o @Test. O Spring por sí só
consegue testar o carregamento da aplicação (graças a @SpringBootTest).

6. Como usar um servidor real Tomcat?.

A anotação @SpringBootTest recebe alguns parâmetros, como, por exemplo, o ambiente web
a ser utilizado juntamente com a porta utilizada. Geralmente é a 8080, mas como é teste
e para não dar conflito, usaremos uma aleatoria, veja:
``@SpringBootTest(webEnvironment = WebEnvironment.RANDOM_PORT)``.
<hr>

## Configuração do banco MySQL: profiles de teste

Faremos agora essa classe utilizar o banco MySQL e não o H2.

Trabalharemos com um conceito novo, chamado Profiles.

Na nossa aplicação, nós temos diferentes tipos de perfil para teste. Exemplo: Temos os nossos
testes mais leves, que são os de unidade que ficam no banco H2.

Mas nosso teste de integração (componente e end-to-end) será executado no banco MySQL. Então temos duas necessidades
diferentes para os nossos testes, pois são de tipos diferentes. Por isso nesses cenários utilizamos profiles, perfis
diferentes. Faremos o seguinte:

1. Selecionaremos um perfil para este teste, usando @ActiveProfiles(""it"), dando o nome a ele de IT, sufixo da nossa
classe (integration test).
<hr>

2. Precisamos fazer um application.properties para esse teste de integração. Ela se chamará ``application-it.properties``.
```properties
# Schema Initialization
# Vai criar a tabela planet no banco assim que iniciada, pois ainda não
# existe
string.jpa.hibernate.dll-auto=update

# Database
spring.datasource.url=${MYSQL_HOST:jdbc:mysql://localhost/starwars?useSSL=false}
spring.datasource.username=${MYSQL_USERNAME:user}
spring.datasource.password=${MYSQL_PASSWORD:senha}
```
<hr>

## Testando cadastro de usuário com sucesso

Agora, implementaremos os testes subcutâneos. Como são testes do topo da piramide, só trabalharemos com cenários de 
sucesso.

### Primeiro caso de uso - Criação de planeta
Como é um teste end-to-end, chamaremos o controller/serviço web que estará rodando e teremos a resposta, conferindo se é
o que esperávamos.

1. Para invocar esse service, usaremos o componente RestTemplate. Mas usaremos o específico para testes, chamado
TestRestTemplate.

Como o SpringBootTest configura e inicia nossos Beans, usaremos @AutoWired no restTemplate.
<hr>

2. Criaremos o método ``createPlanet_ReturnsCreated``.
<hr>

3. Usaremos o restTemplate para chamar o service de criação de planeta.

Ele tem um método http postForEntity, que iremos utilizar. o forEntity traduz a resposta (que é um JSON) e serializamos 
em uma entidade automaticamente (que iremos informar).

Esse método possui 3 parâmetros (url, request(o que estamos criando), responseType(tipo da resposta, para que tenha
serialização)). Colocaremos o retorno disso dentro de um sut do tipo ResponseEntity<Planet>.

**Com isso, podemos verificar o status da requisição, o retorno, então faremos as Assertions.**

4. Usaremos o assertThat para verificar se o status do sut é igual a created.
<hr>

Uma outra coisa, é que como são testes end-to-end, agora teremos ID's que serão geradas automaticamente pelo banco.

Então podemos verificar:

1. se o id não é vazio:

```java
assertThat(sut.getBody().getId()).isNotNull();
```
<hr>

2. O nome do planeta, clima e terreno:

```java
assertThat(sut.getBody().getName()).isEqualTo(PLANET.getName());
assertThat(sut.getBody().getClimate()).isEqualTo(PLANET.getClimate());
assertThat(sut.getBody().getTerrain()).isEqualTo(PLANET.getTerrain());
```

Código final:
O codigo abaixo possui um erro, veja na [sessão abaixo](#configurando-rollback-para-os-testes)

```java

@ActiveProfiles("it")
@SpringBootTest
public class PlanetIT {

    @Autowired
    TestRestTemplate restTemplate;

    @Test
    public void createPlanet_ReturnsCreated() {
        ResponseEntity<Planet> sut = restTemplate.postForEntity("/planets", PLANET, Planet.class);

        assertThat(sut.getStatusCode()).isEqualTo(HttpStatus.CREATED);
        assertThat(sut.getBody().getId()).isNotNull();
        assertThat(sut.getBody().getName()).isEqualTo(PLANET.getName());
        assertThat(sut.getBody().getClimate()).isEqualTo(PLANET.getClimate());
        assertThat(sut.getBody().getTerrain()).isEqualTo(PLANET.getTerrain());
    }
}
```
<hr>

## Configurando rollback para os testes

Se rodarmos o código acima duas vezes, irá dar um erro. Porque o planeta que queremos já foi criado.

A gente não pode persistir dados de forma fixa no banco. Isso deixa os testes frágeis.

### Como limpar depois de cada teste o que foi inserido no banco de dados?

Usaremos um script de dados, que irá limpar (truncar) a tabela de planetas após cada execução de testes.

1. Criar no resources (de teste), um arquivo chamado remove_planets.sql.

Dentro dele, passar : `TRUNCATE TABLE  planets;`
<hr>

2. No código de teste, dizer para ele rodar esse script depois da execução de cada método.

Usaremos o @SQL, que já vimos previamente. ``@Sql(scripts = "/remove_planets.sql", executionPhase = Sql.ExecutionPhase.AFTER_TEST_METHOD)``.

O executionPhase é a fase de execução que rodaremos esse Script, usaremos ExecutionPhase.AFTER_TEST_METHOD.

Poderia ser antes de cada teste ou depois também. Depois de criar o planeta, ele será removido, podendo usar esse teste
quantas vezes quisermos.
<hr>

## Testando consulta de planeta com sucesso

Aqui teremos uma consulta de planeta por ID. Será semelhante ao teste acima, porém com umas coisas a mais.

1. Criaremos o método: ``public void getPlanet_ReturnsPlanet()``.
<hr>

2. Faremos a requisição com o restTemplate, mas agora será o GET. 

Como estamos trabalhando com requisição de dados, precisamos fazer uma carga de dados para ser consultado.
Para não inserir isso manualmente, usaremos o script SQL.
<hr>

3. Podemos criar vários @SQL em cima do método, escolhendo quando inciá-los. Usaremos aquele import_planets,
mas dessa vez usando o ".BEFORE_TEST_METHOD".
<hr>

4. No sut, podemos passar o número "1", na url pois é uma ID existente no script SQL.
<hr>

5. Fazer a asserção dos dados com o getStatusCode, vendo se é igual a OK. 
<hr>

6. Podemos checar o corpo da requisição usando as [Contrants](https://github.com/zenonxd/estudo-testes-subcutaneos/blob/main/src/test/java/com/demo/swplanetapi/common/PlanetConstrants.java) 
criadas no outro arquivo.

Podemos ver então se o body do sut é igual a variável que estamos testando, neste caso, TATOOINE.
<hr>


# Exercícios

## Exercício - Testando consulta por nome, listagem e remoção de planeta


# Resumo

Chama o restTemplate, obtém o resultado da requisição e depois verifica o que você quer (seja status de resposta ou corpo).


<hr>

## Fim

