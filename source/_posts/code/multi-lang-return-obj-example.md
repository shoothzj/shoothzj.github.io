---
title: 多语言编程 返回多个不同类型的方法样例
date: 2023-09-18 21:10:15
tags:
  - Code
  - 多语言编程
---
<!-- toc -->

# 背景

你可能会在一些场景下碰到需要返回多个不同类型的方法。比如协议解析读取报文时，更具体地像kubernetes在开始解析Yaml的时候，怎么知道这个类型是属于Deployment还是Service？

# C

C语言通常通过使用Struct（结构体）和Union（联合体）的方式来实现这个功能，如下文例子

```c
#include <stdio.h>
#include <stdlib.h>
#include <string.h>

typedef enum {
    MONKEY,
    COW,
    UNKNOWN
} AnimalType;

typedef struct {
    char* description;
} Monkey;

typedef struct {
    char* description;
} Cow;

typedef struct {
    AnimalType type;
    union {
        Monkey monkey;
        Cow cow;
    };
} Animal;

Animal createAnimal(const char* animalType) {
    Animal animal;
    if (strcmp(animalType, "Monkey") == 0) {
        animal.type = MONKEY;
        animal.monkey.description = "I am a monkey!";
    } else if (strcmp(animalType, "Cow") == 0) {
        animal.type = COW;
        animal.cow.description = "I am a cow!";
    } else {
        animal.type = UNKNOWN;
    }
    return animal;
}

int main() {
    Animal animal1 = createAnimal("Monkey");
    if (animal1.type == MONKEY) {
        printf("%s\n", animal1.monkey.description);
    }

    Animal animal2 = createAnimal("Cow");
    if (animal2.type == COW) {
        printf("%s\n", animal2.cow.description);
    }

    Animal animal3 = createAnimal("Dog");
    if (animal3.type == UNKNOWN) {
        printf("Unknown animal type\n");
    }

    return 0;
}
```

# C++

在C++中，我们可以使用基类指针来指向派生类的对象。可以使用动态类型识别（RTTI）来在运行时确定对象的类型

```cpp
#include <iostream>
#include <stdexcept>

class Animal {
public:
    virtual std::string toString() const = 0;
};

class Monkey : public Animal {
public:
    std::string toString() const override {
        return "I am a monkey!";
    }
};

class Cow : public Animal {
public:
    std::string toString() const override {
        return "I am a cow!";
    }
};

Animal* createAnimal(const std::string& animalType) {
    if (animalType == "Monkey") {
        return new Monkey();
    }
    if (animalType == "Cow") {
        return new Cow();
    }
    throw std::runtime_error("Unknown animal type: " + animalType);
}

int main() {
    try {
        Animal* animal1 = createAnimal("Monkey");

        if (Monkey* monkey = dynamic_cast<Monkey*>(animal1)) {
            std::cout << monkey->toString() << std::endl;
        }
        delete animal1;

        Animal* animal2 = createAnimal("Cow");

        if (Cow* cow = dynamic_cast<Cow*>(animal2)) {
            std::cout << cow->toString() << std::endl;
        }
        delete animal2;
    }
    catch (const std::runtime_error& e) {
        std::cerr << e.what() << std::endl;
    }

    return 0;
}
```

# Go

Go的常见处理方式，是返回一个接口或者**interface{}**类型。调用者使用Go语言类型断言来检查具体的类型

```Go
package main

import (
	"fmt"
)

type Animal interface {
	String() string
}

type Monkey struct{}

func (m Monkey) String() string {
	return "I am a monkey!"
}

type Cow struct{}

func (c Cow) String() string {
	return "I am a cow!"
}

func createAnimal(typeName string) (Animal, error) {
	switch typeName {
	case "Monkey":
		return Monkey{}, nil
	case "Cow":
		return Cow{}, nil
	default:
		return nil, fmt.Errorf("Unknown animal type: %s", typeName)
	}
}

func main() {
	animal1, err := createAnimal("Monkey")
	if err != nil {
		fmt.Println(err)
		return
	}

	if monkey, ok := animal1.(Monkey); ok {
		fmt.Println(monkey)
	}

	animal2, err := createAnimal("Cow")
	if err != nil {
		fmt.Println(err)
		return
	}

	if cow, ok := animal2.(Cow); ok {
		fmt.Println(cow)
	}
}
```

# Java

Java语言的常见处理方式，是返回Object类型或者一个基础类型。然后由调用方在进行instance of判断。或者Java17之后，可以使用模式匹配的方式来简化转型

```java
public class MultiTypeReturnExample {
    static class Monkey {
        @Override
        public String toString() {
            return "I am a monkey!";
        }
    }

    static class Cow {
        @Override
        public String toString() {
            return "I am a cow!";
        }
    }

    public static Object createAnimal(String type) throws IllegalArgumentException {
        switch (type) {
            case "Monkey":
                return new Monkey();
            case "Cow":
                return new Cow();
            default:
                throw new IllegalArgumentException("Unknown animal type: " + type);
        }
    }

    public static void main(String[] args) throws Exception {
        Object animal1 = createAnimal("Monkey");

        // java8 写法，后面如果明确用做精确的类型，需要强制转换

        if (animal1 instanceof Monkey) {
            System.out.println(animal1);
        }

        Object animal2 = createAnimal("Cow");
        if (animal2 instanceof Cow) {
            System.out.println(animal2);
        }

        // java17 写法，不需要强制转换
        if (createAnimal("Monkey") instanceof Monkey animal3) {
            System.out.println(animal3);
        }

        if (createAnimal("Cow") instanceof Cow animal4) {
            System.out.println(animal4);
        }
    }
}
```

# Javascript

动态类型语言，使用instanceof运算符判断

```jsx
class Animal {
    toString() {
        return 'I am an animal';
    }
}

class Monkey extends Animal {
    toString() {
        return 'I am a monkey';
    }
}

class Cow extends Animal {
    toString() {
        return 'I am a cow';
    }
}

function createAnimal(animalType) {
    switch (animalType) {
        case 'Monkey':
            return new Monkey();
        case 'Cow':
            return new Cow();
        default:
            throw new Error(`Unknown animal type: ${animalType}`);
    }
}

try {
    const animal1 = createAnimal('Monkey');
    if (animal1 instanceof Monkey) {
        console.log(animal1.toString());
    }

    const animal2 = createAnimal('Cow');
    if (animal2 instanceof Cow) {
        console.log(animal2.toString());
    }

    const animal3 = createAnimal('Dog');
} catch (error) {
    console.error(error.message);
}
```

# Kotlin

Kotlin可以使用Sealed Class(密封类)和Any类型两种方式。使用Any的场景，与Java返回Object类似。Sealed Class更加安全、更方便一些。

## 使用Any类型

```kotlin
open class Animal

class Monkey: Animal() {
    override fun toString(): String {
        return "I am a monkey!"
    }
}

class Cow: Animal() {
    override fun toString(): String {
        return "I am a cow!"
    }
}

fun createAnimal(type: String): Any {
    return when (type) {
        "Monkey" -> Monkey()
        "Cow" -> Cow()
        else -> throw IllegalArgumentException("Unknown animal type: $type")
    }
}

fun main() {
    val animal1 = createAnimal("Monkey")
    when (animal1) {
        is Monkey -> println(animal1)
        is Cow -> println(animal1)
    }

    val animal2 = createAnimal("Cow")
    when (animal2) {
        is Monkey -> println(animal2)
        is Cow -> println(animal2)
    }
}
```

## 使用SealedClass

```kotlin
sealed class Animal {
    data class Monkey(val info: String = "I am a monkey!") : Animal()
    data class Cow(val info: String = "I am a cow!") : Animal()
}

fun createAnimal(type: String): Animal {
    return when (type) {
        "Monkey" -> Animal.Monkey()
        "Cow" -> Animal.Cow()
        else -> throw IllegalArgumentException("Unknown animal type: $type")
    }
}

fun main() {
    val animal1 = createAnimal("Monkey")
    when (animal1) {
        is Animal.Monkey -> println(animal1.info)
        is Animal.Cow -> println(animal1.info)
    }

    val animal2 = createAnimal("Cow")
    when (animal2) {
        is Animal.Monkey -> println(animal2.info)
        is Animal.Cow -> println(animal2.info)
    }
}
```

# Python

Python是动态类型的语言，可以简单基于一些条件返回不同类型的对象，然后在接收到返回值之后使用type()函数或isinstance()函数来确定其类型

```python
class Animal:
    def __str__(self):
        return "I am an animal"

class Monkey(Animal):
    def __str__(self):
        return "I am a monkey"

class Cow(Animal):
    def __str__(self):
        return "I am a cow"

def create_animal(animal_type):
    if animal_type == "Monkey":
        return Monkey()
    elif animal_type == "Cow":
        return Cow()
    else:
        raise ValueError(f"Unknown animal type: {animal_type}")

def main():
    animal1 = create_animal("Monkey")
    if isinstance(animal1, Monkey):
        print(animal1)

    animal2 = create_animal("Cow")
    if isinstance(animal2, Cow):
        print(animal2)

if __name__ == "__main__":
    main()
```

# Ruby

Ruby也较为简单，在方法内部直接返回不同类型的对象。然后，可以使用**is_a**方法或**class**方法来确定返回对象的实际类型。

```ruby
class Animal
  def to_s
    "I am an animal"
  end
end

class Monkey < Animal
  def to_s
    "I am a monkey"
  end
end

class Cow < Animal
  def to_s
    "I am a cow"
  end
end

def create_animal(animal_type)
  case animal_type
  when "Monkey"
    Monkey.new
  when "Cow"
    Cow.new
  else
    raise "Unknown animal type: #{animal_type}"
  end
end

begin
  animal1 = create_animal("Monkey")
  if animal1.is_a? Monkey
    puts animal1
  end

  animal2 = create_animal("Cow")
  if animal2.is_a? Cow
    puts animal2
  end
end
```

# Rust

在Rust中，可以使用enum（枚举）来创建一个持有多种不同类型的数据结构。然后使用match语句来做模式匹配。

```rust
use std::fmt;

enum Animal {
    Monkey,
    Cow,
}

impl fmt::Display for Animal {
    fn fmt(&self, f: &mut fmt::Formatter<'_>) -> fmt::Result {
        match self {
            Animal::Monkey => write!(f, "I am a monkey!"),
            Animal::Cow => write!(f, "I am a cow!"),
        }
    }
}

fn create_animal(animal_type: &str) -> Result<Animal, String> {
    match animal_type {
        "Monkey" => Ok(Animal::Monkey),
        "Cow" => Ok(Animal::Cow),
        _ => Err(format!("Unknown animal type: {}", animal_type)),
    }
}

fn main() {
    match create_animal("Monkey") {
        Ok(animal) => match animal {
            Animal::Monkey => println!("{}", animal),
            _ => (),
        },
        Err(e) => println!("{}", e),
    }

    match create_animal("Cow") {
        Ok(animal) => match animal {
            Animal::Cow => println!("{}", animal),
            _ => (),
        },
        Err(e) => println!("{}", e),
    }

    match create_animal("Dog") {
        Ok(_) => (),
        Err(e) => println!("{}", e),
    }
}
```

# Scala

scala中，可以使用sealed trait和case class来创建一个能够返回多种不同类型的方法。Sealed trait可以定义一个有限的子类集合，可以确保类型安全

```scala
sealed trait Animal {
  def info: String
}

case class Monkey() extends Animal {
  val info: String = "I am a monkey!"
}

case class Cow() extends Animal {
  val info: String = "I am a cow!"
}

object MultiTypeReturnExample {
  def createAnimal(animalType: String): Animal = {
    animalType match {
      case "Monkey" => Monkey()
      case "Cow" => Cow()
      case _ => throw new IllegalArgumentException(s"Unknown animal type: $animalType")
    }
  }

  def main(args: Array[String]): Unit = {
    try {
      val animal1 = createAnimal("Monkey")
      animal1 match {
        case Monkey() => println(animal1.info)
        case _ =>
      }

      val animal2 = createAnimal("Cow")
      animal2 match {
        case Cow() => println(animal2.info)
        case _ =>
      }
    } catch {
      case e: IllegalArgumentException => println(e.getMessage)
    }
  }
}
```

# TypeScript

总得来说，和javascript区别不大

```tsx
abstract class Animal {
  abstract toString(): string;
}

class Monkey extends Animal {
  toString(): string {
    return 'I am a monkey';
  }
}

class Cow extends Animal {
  toString(): string {
    return 'I am a cow';
  }
}

function createAnimal(animalType: string): Animal {
  switch (animalType) {
    case 'Monkey':
      return new Monkey();
    case 'Cow':
      return new Cow();
    default:
      throw new Error(`Unknown animal type: ${animalType}`);
  }
}

try {
  const animal1 = createAnimal('Monkey');
  if (animal1 instanceof Monkey) {
    console.log(animal1.toString());
  }

  const animal2 = createAnimal('Cow');
  if (animal2 instanceof Cow) {
    console.log(animal2.toString());
  }

  const animal3 = createAnimal('Dog');
} catch (error) {
  console.error(error.message);
}
```
