# 1. Introdução
As anotações Java são um mecanismo para adicionar informações de metadados ao nosso código-fonte. Eles são uma parte poderosa do Java que foi adicionada ao JDK5. As anotações oferecem uma alternativa ao uso de descritores XML e interfaces de marcadores.

Embora possamos anexá-los a pacotes, classes, interfaces, métodos e campos, as anotações por si mesmas não têm efeito na execução de um programa.

Neste tutorial, vamos nos concentrar em como criar e processar anotações personalizadas. Podemos ler mais sobre anotações em nosso artigo sobre noções básicas de anotação.

# 2. Criação de anotações personalizadas
Vamos criar três anotações personalizadas com o objetivo de serializar um objeto em uma string JSON.

Usaremos o primeiro no nível da classe, para indicar ao compilador que nosso objeto pode ser serializado. Em seguida, aplicaremos o segundo aos campos que desejamos incluir na string JSON.

Finalmente, usaremos a terceira anotação no nível do método, para especificar o método que usaremos para inicializar nosso objeto.

### 2.1. Exemplo de anotação de nível de classe
A primeira etapa para criar uma anotação personalizada é declará-la usando a palavra-chave @interface:

```
public @interface JsonSerializable {
}
```

A próxima etapa é adicionar metanotações para especificar o escopo e o destino de nossa anotação personalizada:

```
@Retention(RetentionPolicy.RUNTIME)
@Target(ElementType.Type)
public @interface JsonSerializable {
}
```

Como podemos ver, nossa primeira anotação tem visibilidade de tempo de execução e podemos aplicá-la a tipos (classes). Além disso, ele não possui métodos e, portanto, serve como um marcador simples para marcar classes que podem ser serializadas em JSON.

### 2.2. Exemplo de anotação em nível de campo
Da mesma forma, criamos nossa segunda anotação para marcar os campos que vamos incluir no JSON gerado:

```
@Retention(RetentionPolicy.RUNTIME)
@Target(ElementType.FIELD)
public @interface JsonElement {
    public String key() default "";
}
```

A anotação declara um parâmetro String com o nome “chave” e uma string vazia como valor padrão.

Ao criar anotações personalizadas com métodos, devemos estar cientes de que esses métodos não devem ter parâmetros e não podem lançar uma exceção. Além disso, os tipos de retorno são restritos a primitivos, String, Class, enums, anotações e matrizes desses tipos, e o valor padrão não pode ser nulo.

### 2.3. Exemplo de anotação de nível de método
Vamos imaginar que antes de serializar um objeto em uma string JSON, queremos executar algum método para inicializar um objeto. Por esse motivo, vamos criar uma anotação para marcar este método:

```
@Retention(RetentionPolicy.RUNTIME)
@Target(ElementType.METHOD)
public @interface Init {
}
```

Declaramos uma anotação pública com visibilidade de tempo de execução que podemos aplicar aos métodos de nossas classes.

### 2.4. Aplicando Anotações
Agora vamos ver como podemos usar nossas anotações personalizadas. Por exemplo, vamos imaginar que temos um objeto do tipo Person que queremos serializar em uma string JSON. Este tipo possui um método que coloca em maiúscula a primeira letra do nome e do sobrenome. Queremos chamar este método antes de serializar o objeto:

```
@JsonSerializable
public class Person {

    @JsonElement
    private String firstName;

    @JsonElement
    private String lastName;

    @JsonElement(key = "personAge")
    private String age;

    private String address;

    @Init
    private void initNames() {
        this.firstName = this.firstName.substring(0, 1).toUpperCase() 
          + this.firstName.substring(1);
        this.lastName = this.lastName.substring(0, 1).toUpperCase() 
          + this.lastName.substring(1);
    }

    // Standard getters and setters
}
```

Usando nossas anotações personalizadas, estamos indicando que podemos serializar um objeto Person para uma string JSON. Além disso, a saída deve conter apenas os campos firstName, lastName e age desse objeto. Além disso, queremos que o método initNames () seja chamado antes da serialização.

Ao definir o parâmetro chave da anotação @JsonElement como “personAge”, estamos indicando que usaremos esse nome como identificador para o campo na saída JSON.

Para fins de demonstração, tornamos initNames () privado, portanto, não podemos inicializar nosso objeto chamando-o manualmente e nossos construtores também não o estão usando.

# 3. Processando anotações
Até agora, vimos como criar anotações personalizadas e como usá-las para decorar a classe Person. Agora veremos como tirar vantagem deles usando a API Reflection do Java.

A primeira etapa será verificar se nosso objeto é nulo ou não, bem como se seu tipo possui a anotação @JsonSerializable ou não:

```
private void checkIfSerializable(Object object) {
    if (Objects.isNull(object)) {
        throw new JsonSerializationException("The object to serialize is null");
    }
        
    Class<?> clazz = object.getClass();
    if (!clazz.isAnnotationPresent(JsonSerializable.class)) {
        throw new JsonSerializationException("The class " 
          + clazz.getSimpleName() 
          + " is not annotated with JsonSerializable");
    }
}
```

Em seguida, procuramos qualquer método com a anotação @Init e o executamos para inicializar os campos do nosso objeto:

```
private void initializeObject(Object object) throws Exception {
    Class<?> clazz = object.getClass();
    for (Method method : clazz.getDeclaredMethods()) {
        if (method.isAnnotationPresent(Init.class)) {
            method.setAccessible(true);
            method.invoke(object);
        }
    }
 }
```

A chamada de method.setAccessible (true) nos permite executar o método initNames () privado.

Após a inicialização, iteramos sobre os campos do nosso objeto, recuperamos a chave e o valor dos elementos JSON e os colocamos em um mapa. Em seguida, criamos a string JSON do mapa:

```
private String getJsonString(Object object) throws Exception {	
    Class<?> clazz = object.getClass();
    Map<String, String> jsonElementsMap = new HashMap<>();
    for (Field field : clazz.getDeclaredFields()) {
        field.setAccessible(true);
        if (field.isAnnotationPresent(JsonElement.class)) {
            jsonElementsMap.put(getKey(field), (String) field.get(object));
        }
    }		
     
    String jsonString = jsonElementsMap.entrySet()
        .stream()
        .map(entry -> "\"" + entry.getKey() + "\":\"" 
          + entry.getValue() + "\"")
        .collect(Collectors.joining(","));
    return "{" + jsonString + "}";
}
```

Novamente, usamos field.setAccessible (true) porque os campos do objeto Person são privados.

Nossa classe de serializador JSON combina todas as etapas acima:

```
public class ObjectToJsonConverter {
    public String convertToJson(Object object) throws JsonSerializationException {
        try {
            checkIfSerializable(object);
            initializeObject(object);
            return getJsonString(object);
        } catch (Exception e) {
            throw new JsonSerializationException(e.getMessage());
        }
    }
}
```

Finally, we run a unit test to validate that our object was serialized as defined by our custom annotations:

```
@Test
public void givenObjectSerializedThenTrueReturned() throws JsonSerializationException {
    Person person = new Person("soufiane", "cheouati", "34");
    ObjectToJsonConverter serializer = new ObjectToJsonConverter(); 
    String jsonString = serializer.convertToJson(person);
    assertEquals(
      "{\"personAge\":\"34\",\"firstName\":\"Soufiane\",\"lastName\":\"Cheouati\"}",
      jsonString);
}
```

# 4. Conclusão
Neste artigo, aprendemos como criar diferentes tipos de anotações personalizadas. Em seguida, discutimos como usá-los para decorar nossos objetos. Finalmente, vimos como processá-los usando a API Reflection do Java.