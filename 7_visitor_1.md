Пример неистинного наследования с переоопределением родительского метода в классах-наследниках.
Несмотря на перегруженные методы visit(), всегда будет вызываться один и тоот же метод visit(Shape figure) из-за позднего связывания.
```java
public class BadExtension {

    public static void main(String[] args) {
        printFigure(new Shape()); // this is shape
        printFigure(new Rectangle()); // this is shape
        printFigure(new Dot()); // this is shape
    }

    public static void printFigure(Shape shape) {
        Visitor visitor = new Visitor();
        visitor.visit(shape);
    }

}

class Visitor {
    public void visit(Shape figure) {
        System.out.println("this is shape");
    }

    public void visit(Dot figure) {
        System.out.println("this is dot");
    }

    public void visit(Rectangle figure) {
        System.out.println("this is rectangle");
    }
}

class Shape {
    public void print() {
        System.out.println("this is shape");
    }
}

class Dot extends Shape {
    @Override
    public void print() {
        System.out.println("this is dot");
    }
}

class Rectangle extends Shape {
    @Override
    public void print() {
        System.out.println("this is rectangle");
    }
}
```

Более точная реализация паттерна.
В данном случае мы сразу "сообщим" компилятору какой метод визитора нужно вызывать.

```java
package org.example.visitor;

public class VisitorPattern {
    public void visit(Dot shape) {
        System.out.println("this is Dot");
    }

    public void visit(Rectangle shape) {
        System.out.println("this is Rectangle");
    }

    public static void main(String[] args) {
        VisitorPattern visitor = new VisitorPattern();
        Dot dot = new Dot();
        Rectangle rectangle = new Rectangle();

        dot.accept(visitor); // this is Dot
        rectangle.accept(visitor); // his is Rectangle
    }

}

abstract class Shape {
    abstract void accept(VisitorPattern visitor);
}

class Dot extends Shape {

    @Override
    void accept(VisitorPattern visitor) {
        visitor.visit(this);
    }
}

class Rectangle extends Shape {

    @Override
    void accept(VisitorPattern visitor) {
        visitor.visit(this);
    }
}
```

Паттерн позволяет добавлять новые действия над фигурами, не изменяя классы самих фигур. Это плюс.
При этом сами эти действия нужно добавлять в существующий визитоор или создавать нового наследника.
Есть ощущение, что сам класс визитор можно не описывать, а само действие передавать через функциоональный интерфейс. 
То есть абстрактный метод accept доолжен выглядеть accept(T function). Тогда мы сможем передать реализацию нужного интерфейса.
И проблемы раннего связывания точно не возникнет.























