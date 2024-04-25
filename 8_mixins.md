В Java дефолтные методы интерфейсов позволяют нам создавать миксины.
Дефолтные методы реализуются в самом интерфейсе и могут быть переопределены в классе-наследнике при необходимости.
Но достаточно реализовать интерфейс с дефолтным методом, чтобы добавить классу определенное поведение. При этом сам класс-наследник изменять не придется.
Т.к для интерфейсов поддерживается множественное наследование, то можно добвлять различное новое поведение, добавляя в объявление класса наименование очередного интерфейса-миксина.

```java
public class Person implements Mixin {
    private String name;
    private Integer age;

    Person(String name, int age) {
        this.age = age;
        this.name = name;
    }

    Integer getAge() {
        return this.age;
    }

    String getName() {
        return this.name;
    }

    public static void main(String[] args) {
        Person person = new Person("name", 3);
        
        person.operate();
    }

}

interface Mixin {
    default void operate() {
        System.out.println("This is mixin");
    }
}
```
