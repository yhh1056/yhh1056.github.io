---
title:  "[Java] 리플렉션 개념 이해하기 "
excerpt: ""
toc: true
toc_sticky: true

categories:
  - java
last_modified_at: 2022-01-29TO00:40:00+09:00
---

# 리플렉션

### 들어가며
자바를 처음 배우던 시절 생각해 보면 리플렉션이라는 단어조차 들어 본 적이 없었습니다. 자바를 점점 학습하면서 종종 들어봤지만 그때 당시 '리플렉션을 모르면 사용하지 말아라', '면접에서 안다고 얘기했다가 까였다.'
등의 글을 읽으며 핑계 삼아 공부를 미뤘습니다. 백기선님의 자바 라이브 스터디 시절 어노테이션을 공부하다가 리플렉션이라는 키워드를 다시 만나게 되었고 이때 간단하게 학습했습니다. 그리고 스프링 AOP를 공부하면서 다이나믹 프록시를 알게 되고 리플렉션에 대해 이해하게 되었습니다. 

저도 한 번에 이해하지 못 한 개념이고 어노테이션, aop 등 공부하다 보면 리플렉션에 대한 퍼즐 조각이 자연스럽게 맞춰질 수도 있습니다. (제가 그랬어요.)

자바의 리플렉션이 익숙하지 않다면 조금이나마 이 글을 읽고 도움이 되길 바랍니다.

## 리플렉션이란?

> Reflection is a feature in the Java programming language. It allows an executing Java program to examine or "introspect" upon itself, and manipulate internal properties of the program. For example, it's possible for a Java class to obtain the names of all its members and display them. 
> -- Oracle Technical Article

> In this article, we will be exploring Java reflection, which allows us to inspect or/and modify runtime attributes of classes, interfaces, fields, and methods. This particularly comes in handy when we don't know their names at compile time.
Additionally, we can instantiate new objects, invoke methods, and get or set field values using reflection.
> -- baeldung Guide


Oracle, Baeldung 문서에서 설명하는 리플렉션을 종합해 보면 런타임 단계에서 클래스, 인터페이스, 필드, 메서드를 검사하거나 수정할 수 있습니다. 또한 객체를 인스턴스화하고 메소드를 호출하고 필드 값을 가져오거나 설정할 수 있습니다. 컴파일 할 때 그들(클래스, 필드, 메서드...)의 이름을 모를 때 유용합니다. 리플렉션을 이해하고 있다면 해당 설명이 쉽게 이해가 되겠지만 리플렉션을 모르는 상태로 보면 생소한 말들입니다.

우선 리플렉션이 API가 제공하는 간단한 기능들을 사용해 보겠습니다. 필드에 접근하고, 인스턴스를 생성하고, 메소드를 사용해 보는 기능 정보만 사용해보겠습니다. 리플렉션에 대한 개념을 이해하기 위한 글이므로 더 다양한 기능들은 공식문서를 참고해주세요.

## 기능 사용해보기

### 🏷 필드 접근하기
리플렉션을 사용하여 어떤 필드들이 존재하는지 검사할 수 있습니다. 

#### Person

```java
public class Person {
    public String name;
    protected String address;
    private int age;
    String email;
}    
```
예제를 위해 pulbic, private 접근자를 사용해서 생성하였습니다. 

#### <code>getDeclaredFields()</code>
우선 클래스타입을 알아야 합니다. 그 다음 <code>Class&lt;T&gt;</code> 클래스의 <code>getgetFields()</code> 메서드를 사용하여 필드들을 조회 할 수 있습니다. <code>getgetFields()</code> 메서드는 pulblic으로 선언된 필드만 조회가 가능하기 때문에 <code>getDeclaredFields()</code> 메서드를 사용해보겠습니다.

```java
public static void main(String[] args) {
    Class<Person> clazz = Person.class;
    Field[] fields = clazz.getDeclaredFields();
    for (Field field : fields) {
        System.out.println(field.getName());
    }
}
```
#### 결과
```
name
address
age
email
```

### 🏷 인스턴스 생성하기
이번엔 생성자를 통해 인스턴스를 생성해 보겠습니다. 필드를 조회할 때는 필드를 조회하는 메서드를 사용했으니 이번엔 생성자를 가져오는 메서드를 사용하면 됩니다. 
#### Person
```java
public class Person {
    private String name;
    private int age;
    
    public Person() {}

    public Person(String name) {
        this.name = name;
    }

    public Person(int age) {
        this.age = age;
    }

    public Person(String name, int age) {
        this.name = name;
        this.age = age;
    }
}
```

#### <code>getConstructor()</code>
타입을 명시하여 원하는 생성자를 조회할 수 있습니다. 해당되는 생성자가 없을 수도 있기 때문에 <code>try-catch</code>문을 사용해야 합니다.

```java
Class<Person> clazz = Person.class;
try {
    Constructor<Person> constructor = clazz.getConstructor(String.class, int.class);
} catch (NoSuchMethodException e) {
    e.printStackTrace();
}
```

#### <code>newInstance()</code>
생성자를 가져왔다면 <code>newInstance()</code> 메서드를 사용하여 인스터를 생성할 수 있습니다.
```java
Class<Person> clazz = Person.class;
try {
    Constructor<Person> constructor = clazz.getConstructor(String.class, int.class);
    Person newPerson = constructor.newInstance("호호맨", 20);
} catch (NoSuchMethodException | IllegalAccessException | InstantiationException | InvocationTargetException e) {
    e.printStackTrace();
}
```

### 🏷 메소드 접근하기
필드 정보를 조회하고, 인스턴스도 생성했습니다. 마지막으로 메서드를 사용해보겠습니다.

#### Person
이전과 같이 인스턴스를 생성하고 <code>toString()</code> 메소드를 사용해 출력해보겠습니다.

```java
public class Person {
    private String name;
    private int age;

    public Person(String name, int age) {
        this.name = name;
        this.age = age;
    }
    
    @Override
    public String toString() {
        return "이름 : " + this.name + ", 나이 : " + age;
    }
}
```

#### getMethod()
<code>toString</code>은 <code>pulbic</code> 메소드이므로 <code>getDeclaredMethod()</code>메소드를 사용하지 않아도 됩니다.
```java
Class<Person> clazz = Person.class;
try {
    Constructor<Person> constructor = clazz.getConstructor(String.class, int.class);
    Person newPerson = constructor.newInstance("호호맨", 20);
    Method toString = newPerson.getClass().getMethod("toString");
} catch (NoSuchMethodException | IllegalAccessException | InstantiationException | InvocationTargetException e) {
    e.printStackTrace();
}
```
메소드를 조회하였고 이번엔 메소드를 실행해보겠습니다.


#### invoke()
<code>invoke()</code> 를 사용하여 조회해온 <code>toString()</code> 메서드를 실행시켜보겠습니다. 
```java
public static void main(String[] args) {
    Class<Person> clazz = Person.class;
    try {
        Constructor<Person> constructor = clazz.getConstructor(String.class, int.class);
        Person newPerson = constructor.newInstance("호호맨", 20);
        Method toString = newPerson.getClass().getDeclaredMethod("toString");
        String result = (String)toString.invoke(newPerson);
        System.out.println(result);
    } catch (NoSuchMethodException | IllegalAccessException | InstantiationException | InvocationTargetException e) {
        e.printStackTrace();
    }
}
```

#### 결과
```
이름 : 호호맨, 나이 : 20
```
<code>invoke(Object obj, Object... args)</code> 메서드는 인자로 해당 클래스 타입과 해당 메소드의 인자들을 인자로 받습니다. 
<code>invoke()</code>메소드는 라이브러리 내부를 디버깅할 때 정말 많이 보게 되는 메소드입니다. 스프링을 사용하면서 디버깅을 하다 보면 무조건 보게 될 메소드입니다. <code>invoke()</code> 메소드가 보인다면 대부분 리플렉션을 사용한 기능들일 것입니다. 


### 그냥 생성하면 되지 왜 어렵게 생성하나요?
맞습니다. Java를 사용한다면 <code>Person person = new person();</code> 으로 간단하게 생성하면 될 것을 어렵게 생성하였습니다. 스프링 프레임워크나 다른 라이브러리를 사용할 때는 구체적인 클래스를 알고 있기 때문에 리플렉션을 사용할 일이 드뭅니다. 하지만 본인이 라이브러리를 만들어본다고 생각해 봅시다. 라이브러리를 사용하는 사용자가 어떤 클래스를 만들지 예상할 수 있을까요? 예상할 수 없습니다. 또는 무엇인가 동적으로 만들어주는 프로그램을 만들고자 하면 어려움을 겪게 될 것입니다.

Jackson, hibernate는 리플렉션을 사용하고 있습니다. 
객체를 json 타입으로 변경해 주는 jackson 라이브러리를 잠시 살펴보겠습니다.

#### Person
```java
public class Person {
    private String name;
    private int age;

    public Person(String name, int age) {
        this.name = name;
        this.age = age;
    }

    public String getName() {
        return name;
    }

    public int getAge() {
        return age;
    }
}
```

#### objectMapper
```java
ObjectMapper objectMapper = new ObjectMapper();
try {
    String result = objectMapper.writeValueAsString(new Person("호호맨", 20));
    System.out.println(result);
} catch (JsonProcessingException e) {
    e.printStackTrace();
}
```

#### 결과
```
{"name":"호호맨","age":20}
```

라이브러리가 어떻게 <code>"호호맨", 20</code> 이라는 데이터를 가져올 수 있었을까요? Person 이라는 클래스를 사용자가 만들 거라 예상하고 getter를 사용해 데이터를 가져왔을까요?
우리가 좀 전에 예제로 살펴본 방식으로 리플렉션 api를 사용하여 Person이라는 클래스 정보를 받아옵니다. 그다음 'get'으로 시작하는 메소드를 찾아온 뒤 필드값에 접근하여 json 형식으로
데이터를 변환해 주기 때문에 우리는 간편하게 인스턴스만 넘겨주면 됩니다.


### 마무리
리플렉션이라는 개념이 조금 이해가 되셨나요? 제가 예제로 설명드린 메소드 말고 다른 메소드도 사용해 보면서 어떤 프로그램을 만들 수 있을까 고민해 본다면 더 빠른 습득이 가능하지 않을까 
생각해 봅니다. 그래서 저는 백준 문제를 풀 때 커스텀 어노테이션 기반으로 문제 번호와 난이도를 입력해 주면 동적으로 md 파일을 만들어주는 라이브러리를 만들어보았습니다. 실제로도 사용 중입니다.
자바가 제공하는 리플렉션 라이브러리는 반드시 클래스 타입을 넘겨주어야 합니다. 하지만 저는 해당 어노테이션이 존재하는 모든 클래스를 가져오길 바랐기 때문에 구글이 만든 리플렉션 라이브러리를 활용해서
구현했습니다.

왜 리플렉션은 클래스 타입을 넘겨주어야 하는지 다음에 다뤄보도록 하겠습니다.

&#160;
&#160;
&#160;

참고 : [https://www.oracle.com/technical-resources/articles/java/javareflection.html](https://www.oracle.com/technical-resources/articles/java/javareflection.html)  
참고 : [https://www.baeldung.com/java-reflection](https://www.baeldung.com/java-reflection)

<!-- 다라쓰 설치 코드 -->
<div id="darass" 
    data-project-key="CqpU2S8YaJ8DlOp565" 
    data-dark-mode="false"
    data-primary-color="#a7915e"
    data-show-sort-option="true"
    data-allow-social-login="true"
    data-show-logo="true"
    >
    <script type="text/javascript" defer>
        (function () {
        var $document = document;

        var $script = $document.createElement("script");
        $script.src = "https://deploy-script.darass.co.kr/embed.js";
        $script.defer = true;

        $document.head.appendChild($script);
        })();
    </script>
    <noscript>다라쓰 댓글 작성을 위해 JavaScript를 활성화 해주세요</noscript>
</div>
<!-- 다라쓰 설치 코드 끝 -->
