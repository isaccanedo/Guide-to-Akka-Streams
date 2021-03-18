## Akka Streams

Guia para streams Akka

# 1. Visão Geral
Neste artigo, veremos a biblioteca de fluxos akka que é construída sobre a estrutura do ator Akka, que segue o manifesto de fluxos reativos. A API Akka Streams nos permite compor facilmente fluxos de transformação de dados a partir de etapas independentes.

Além disso, todo o processamento é feito de forma reativa, não bloqueante e assíncrona.

# 2. Dependências Maven
Para começar, precisamos adicionar as bibliotecas akka-stream e akka-stream-testkit em nosso pom.xml:

```
<dependency>
    <groupId>com.typesafe.akka</groupId>
    <artifactId>akka-stream_2.11</artifactId>
    <version>2.5.2</version>
</dependency>
<dependency>
    <groupId>com.typesafe.akka</groupId>
    <artifactId>akka-stream-testkit_2.11</artifactId>
    <version>2.5.2</version>
</dependency>
```

# 3. API Akka Streams
Para trabalhar com o Akka Streams, precisamos estar cientes dos principais conceitos da API:

- Source - o ponto de entrada para processamento na biblioteca akka-stream - podemos criar uma instância dessa classe a partir de várias fontes; por exemplo, podemos usar o método single () se quisermos criar um Source a partir de uma única String ou podemos criar um Source a partir de um Iterable de elementos;
- Flow - o bloco de construção principal de processamento - cada instância de Flow tem um valor de entrada e um valor de saída;
- Materializador - podemos usar um se quisermos que nosso Fluxo tenha alguns efeitos colaterais, como registrar ou salvar resultados; mais comumente, passaremos o alias NotUsed como um Materializador para denotar que nosso fluxo não deve ter nenhum efeito colateral;
- Operação Sink - quando estamos construindo um Fluxo, ele não é executado até que registremos uma operação Sink nele - é uma operação terminal que aciona todos os cálculos em todo o Fluxo.

# 4. Criação de fluxos em fluxos Akka
Vamos começar construindo um exemplo simples, onde mostraremos como criar e combinar vários fluxos - para processar um fluxo de inteiros e calcular a janela móvel média de pares inteiros do fluxo.

Analisaremos uma String delimitada por ponto-e-vírgula de inteiros como entrada para criar nossa fonte de fluxo akka para o exemplo.

### 4.1. Usando um fluxo para analisar a entrada
Primeiro, vamos criar uma classe DataImporter que terá uma instância do ActorSystem que usaremos mais tarde para criar nosso Fluxo:

```
public class DataImporter {
    private ActorSystem actorSystem;

    // standard constructors, getters...
}
```
A seguir, vamos criar um método parseLine que irá gerar uma Lista de Inteiros a partir de nossa String de entrada delimitada. Lembre-se de que estamos usando a Java Stream API aqui apenas para análise:

```
private List<Integer> parseLine(String line) {
    String[] fields = line.split(";");
    return Arrays.stream(fields)
      .map(Integer::parseInt)
      .collect(Collectors.toList());
}
```

Nosso fluxo inicial aplicará parseLine à nossa entrada para criar um fluxo com tipo de entrada String e tipo de saída Integer:

```
private Flow<String, Integer, NotUsed> parseContent() {
    return Flow.of(String.class)
      .mapConcat(this::parseLine);
}
```

Quando chamamos o método parseLine (), o compilador sabe que o argumento para essa função lambda será uma String - o mesmo que o tipo de entrada para nosso Fluxo.

Observe que estamos usando o método mapConcat () - equivalente ao método Java 8 flatMap () - porque queremos nivelar a Lista de Inteiro retornada por parseLine () em um Fluxo de Inteiro para que as etapas subsequentes em nosso processamento não precisem para lidar com a lista.

### 4.2. Usando um fluxo para realizar cálculos
Neste ponto, temos nosso fluxo de inteiros analisados. Agora, precisamos implementar a lógica que agrupará todos os elementos de entrada em pares e calculará uma média desses pares.

Agora, vamos criar um fluxo de inteiros e agrupá-los usando o método grouped ().

Em seguida, queremos calcular uma média.

Como não estamos interessados ​​na ordem em que essas médias serão processadas, podemos ter as médias calculadas em paralelo usando vários threads usando o método mapAsyncUnordered (), passando o número de threads como um argumento para esse método.

A ação que será passada como lambda para o Flow precisa retornar um CompletableFuture porque essa ação será calculada de forma assíncrona no thread separado:

```
private Flow<Integer, Double, NotUsed> computeAverage() {
    return Flow.of(Integer.class)
      .grouped(2)
      .mapAsyncUnordered(8, integers ->
        CompletableFuture.supplyAsync(() -> integers.stream()
          .mapToDouble(v -> v)
          .average()
          .orElse(-1.0)));
}
```

Estamos calculando médias em oito threads paralelos. Observe que estamos usando a API Java 8 Stream para calcular uma média.

### 4.3. Compondo vários fluxos em um único fluxo
A API Flow é uma abstração fluente que nos permite compor várias instâncias de Flow para atingir nosso objetivo final de processamento. Podemos ter fluxos granulares onde um, por exemplo, está analisando JSON, outro está fazendo alguma transformação e outro está coletando algumas estatísticas.

Essa granularidade nos ajudará a criar um código mais testável porque podemos testar cada etapa de processamento independentemente.

Criamos dois fluxos acima que podem funcionar independentemente um do outro. Agora, queremos compô-los juntos.

Primeiro, queremos analisar nossa String de entrada e, a seguir, queremos calcular uma média em um fluxo de elementos.

Podemos compor nossos fluxos usando o método via ():

```
Flow<String, Double, NotUsed> calculateAverage() {
    return Flow.of(String.class)
      .via(parseContent())
      .via(computeAverage());
}
```

Criamos um fluxo com o tipo de entrada String e dois outros fluxos depois dele. O fluxo parseContent () recebe uma entrada String e retorna um Integer como saída. O fluxo computeA Average () está pegando aquele Integer e calcula um Double retornando médio como o tipo de saída.

# 5. Adicionando dissipador ao fluxo
Como mencionamos, até este ponto todo o Fluxo ainda não foi executado porque é preguiçoso. Para iniciar a execução do Flow, precisamos definir um Sink. A operação Sink pode, por exemplo, salvar dados em um banco de dados ou enviar resultados para algum serviço da web externo.

Suponha que temos uma classe AverageRepository com o seguinte método save () que grava os resultados em nosso banco de dados:

```
CompletionStage<Double> save(Double average) {
    return CompletableFuture.supplyAsync(() -> {
        // write to database
        return average;
    });
}
```

Agora, queremos criar uma operação Sink que use este método para salvar os resultados do nosso processamento de fluxo. Para criar nosso coletor, primeiro precisamos criar um fluxo que leva o resultado de nosso processamento como o tipo de entrada. Em seguida, queremos salvar todos os nossos resultados no banco de dados.

Novamente, não nos importamos com a ordem dos elementos, portanto, podemos executar as operações save() em paralelo usando o método mapAsyncUnordered().

Para criar um Sink a partir do Flow, precisamos chamar toMat() com Sink.ignore() como primeiro argumento e Keep.right () como o segundo porque queremos retornar um status do processamento:

```
private Sink<Double, CompletionStage<Done>> storeAverages() {
    return Flow.of(Double.class)
      .mapAsyncUnordered(4, averageRepository::save)
      .toMat(Sink.ignore(), Keep.right());
}
```

# 6. Definindo uma fonte para o fluxo
A última coisa que precisamos fazer é criar uma fonte a partir da string de entrada. Podemos aplicar um fluxo CalculeA Average () a esta fonte usando o método via ().

Então, para adicionar o Sink ao processamento, precisamos chamar o método runWith() e passar o Sink storeAverages() que acabamos de criar:

```
CompletionStage<Done> calculateAverageForContent(String content) {
    return Source.single(content)
      .via(calculateAverage())
      .runWith(storeAverages(), ActorMaterializer.create(actorSystem))
      .whenComplete((d, e) -> {
          if (d != null) {
              System.out.println("Import finished ");
          } else {
              e.printStackTrace();
          }
      });
}
```

Observe que, quando o processamento é concluído, estamos adicionando o callback whenComplete (), no qual podemos executar alguma ação dependendo do resultado do processamento.

# 7. Testando Streams Akka
Podemos testar nosso processamento usando o akka-stream-testkit.

A melhor maneira de testar a lógica real do processamento é testar toda a lógica de fluxo e usar o TestSink para acionar o cálculo e confirmar os resultados.

Em nosso teste, estamos criando o fluxo que queremos testar e, a seguir, estamos criando uma fonte a partir do conteúdo de entrada de teste:

```
@Test
public void givenStreamOfIntegers_whenCalculateAverageOfPairs_thenShouldReturnProperResults() {
    // given
    Flow<String, Double, NotUsed> tested = new DataImporter(actorSystem).calculateAverage();
    String input = "1;9;11;0";

    // when
    Source<Double, NotUsed> flow = Source.single(input).via(tested);

    // then
    flow
      .runWith(TestSink.probe(actorSystem), ActorMaterializer.create(actorSystem))
      .request(4)
      .expectNextUnordered(5d, 5.5);
}
```

Estamos verificando se esperamos quatro argumentos de entrada, e dois resultados que são médias podem chegar em qualquer ordem porque nosso processamento é feito de forma assíncrona e paralela.

# 8. Conclusão
Neste artigo, examinamos a biblioteca akka-stream.
Definimos um processo que combina vários fluxos para calcular a média móvel dos elementos. Em seguida, definimos um Source que é um ponto de entrada do processamento do fluxo e um Sink que dispara o processamento real.

Finalmente, escrevemos um teste para nosso processamento usando o akka-stream-testkit.