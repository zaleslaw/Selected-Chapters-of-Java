# Тайны самого простого типа в Java

На одном из собеседований мне задали вопрос: так, сколько байт занимает переменная типа boolean в памяти? А типа Boolean?

Я, не долго думая, выпалил, что, мол о каких байтах речь, наверняка boolean в Java занимает 1 БИТ, а восемь флажков так и вообще 1 БАЙТ.

Мне сказали, что я не прав, в Java все иначе и вообще, идите, учите матчасть. 

Давайте разбираться.

#### Типы в Java и их размеры

Я думаю вы часто встречали что-то вроде такого:

| Type | Size in Bytes | Range |
| :--- | :--- | :--- |
| byte | 1 byte | -128 to 127 |
| short | 2 bytes | -32,768 to 32,767 |
| int | 4 bytes | -2,147,483,648 to 2,147,483, 647 |
| long | 8 bytes | -9,223,372,036,854,775,808 to 9,223,372,036,854,775,807 |
| float | 4 bytes | approximately ±3.40282347E+38F \(6-7 significant decimal digits\) Java implements IEEE 754 standard |
| double | 8 bytes | approximately ±1.79769313486231570E+308 \(15 significant decimal digits\) |
| char | 2 byte | 0 to 65,536 \(unsigned\) |
| boolean | not precisely defined\* | true or false |

Табличка, где boolean стыдливо обходится стороной и замалчивается как бедный родственник, отсидевший в тюрьме за кражу поддельного айфона из перехода.

Согласно официальному [туториалу](http://docs.oracle.com/javase/tutorial/java/nutsandbolts/datatypes.html "туториалу") от Sun/Oracle мы видим следующую картину для народных масс: boolean представляет 1 бит информации, но размер остается на совести того, кто воплощает [спеку JVM.](https://docs.oracle.com/javase/specs/jvms/se8/jvms8.pdf "спеку JVM")

#### **Заглядывая в спеку JVM**

Собственно в спеке, на одной из первых страниц \[стр.20\], мы видим приписку, что, мол boolean в ранних спеках и за тип не считался, настолько он специфический. Впрочем, параграф 2.3.4, приоткрывает завесу тайны над идеями имплементации boolean на конкретной виртуальное машине.

> There are no Java Virtual Machine instructions solely dedicated to operations on boolean values. Instead, expressions in the Java programming language that operate on boolean values are compiled to use values of the Java Virtual Machine int data type.

Т.е. нам ясно говорят, что в целом boolean внутренне - это типичный 4-байтовый int. Соответственно, переменная типа boolean, скорее всего будет занимать 4 байта \(**в 32 раза больше**, чем само значение, которое она презентует\).

#### Проверяя bytecode

Напишем простой пример

```java
public class Sample {
    public static void main(String[] args) {
        boolean flag = true;
        flag = false;
    }
}
```

и поглядим в его bytecode

```java
// class version 52.0 (52)
// access flags 0x21
public class experiment/Sample {

  // compiled from: Sample.java

  // access flags 0x1
  public <init>()V
   L0
    LINENUMBER 3 L0
    ALOAD 0
    INVOKESPECIAL java/lang/Object.<init> ()V
    RETURN
   L1
    LOCALVARIABLE this Lexperiment/Sample; L0 L1 0
    MAXSTACK = 1
    MAXLOCALS = 1

  // access flags 0x9
  public static main([Ljava/lang/String;)V
   L0
    LINENUMBER 5 L0
    ICONST_1
    ISTORE 1
   L1
    LINENUMBER 6 L1
    ICONST_0
    ISTORE 1
   L2
    LINENUMBER 7 L2
    RETURN
   L3
    LOCALVARIABLE args [Ljava/lang/String; L0 L3 0
    LOCALVARIABLE flag Z L1 L3 1
    MAXSTACK = 1
    MAXLOCALS = 2
}
```

Мы видим тут замечательные ICONST\_1/ICONST\_0 - это специальные инструкции для JVM, чтобы положить на стек 1 и 0, соответственно. Т.е. good-old true/false превращаются в 1 и 0. А нам еще запрещают в java писать выражения  'true + 1'

INT? Ну серьезно? Почему не short или byte? Почему огромный многословный int? Это вызывает вопросы, на которые я попробую ответить ближе к концу статьи.

#### А массивы?

Кажется, что должно быть все грустно. Массив из десятка boolean будет занимать столько же места, что и массив из десятка int-ов. Как бы не так!

В той же спеке, в параграфе 2.3.4, говорится

> The Java Virtual Machine does directly support boolean arrays. Its newarray instruction \(§newarray\) enables creation of boolean arrays. Arrays of type boolean are accessed and modified using the byte array instructions baload and bastore \(§baload, §bastore\).

Поглядим байткод для одного такого массива с парочкой изменяемых элементов.

```java
   boolean[] flags = new boolean[100000];
   flags[99999] = false;
   flags[88888] = true && false;
```

```java
// class version 52.0 (52)
// access flags 0x21
public class experiment/Sample {

  // compiled from: Sample.java

  // access flags 0x1
  public <init>()V
   L0
    LINENUMBER 5 L0
    ALOAD 0
    INVOKESPECIAL java/lang/Object.<init> ()V
    RETURN
   L1
    LOCALVARIABLE this Lexperiment/Sample; L0 L1 0
    MAXSTACK = 1
    MAXLOCALS = 1

  // access flags 0x9
  public static main([Ljava/lang/String;)V
   L0
    LINENUMBER 9 L0
    LDC 100000
    NEWARRAY T_BOOLEAN
    ASTORE 1
   L1
    LINENUMBER 10 L1
    ALOAD 1
    LDC 99999
    ICONST_0
    BASTORE
   L2
    LINENUMBER 11 L2
    ALOAD 1
    LDC 88888
    ICONST_0
    BASTORE
   L3
    LINENUMBER 14 L3
    RETURN
   L4
    LOCALVARIABLE args [Ljava/lang/String; L0 L4 0
    LOCALVARIABLE flags [Z L1 L4 1
    MAXSTACK = 3
    MAXLOCALS = 2
}
```

Ура, новые операции видны. Впрочем наши любимые ICONST\_0 никто не отменял.



Там же идет мелким шрифтом, как часть контракта, которую никто не читает

> In Oracle’s Java Virtual Machine implementation, boolean arrays in the Java programming language are encoded as Java Virtual Machine byte arrays, using 8 bits per boolean element.

Грубо говоря, парни, на правильных JVM все не так плохо и мы можем сэкономить на ~~счетах за электричество.~~

Уже британскими учеными разработаны новейшие инструкции для загрузки и выгрузки значений в такой массив, да и сам массив будет задействовать только по 1 байту на элемент.

Ну что, неплохо, но надо проверять.

Я давно не доверяю методам замера памяти а-ля Runtime.getRuntime\(\).freeMemory\(\), поэтому я воспользовался библиотечкой JOL из состава OpenJDK.

```XML
       <!--Maven import--->
        
        <dependency>
            <groupId>org.openjdk.jol</groupId>
            <artifactId>jol-core</artifactId>
            <version>0.8</version>
        </dependency>
```

Там есть простая возможность узнать размеры элемента массива для вашей конкретной JVM

```java
   System.out.println(VM.current().details());
```

Результат оказался ожидаемым, наш элемент и вправду занимает 1 байт на моей машине вместе с Oracle JDK 1.8.66

```
# Running 64-bit HotSpot VM.
# Using compressed oop with 3-bit shift.
# Using compressed klass with 3-bit shift.
# Objects are 8 bytes aligned.
# Field sizes by type: 4, 1, 1, 2, 2, 4, 4, 8, 8 [bytes]
# Array element sizes: 4, 1, 1, 2, 2, 4, 4, 8, 8 [bytes]
```

#### 

#### **А что насчет типа Boolean?**

Вот уж кто, наверное, жрет памяти за обе щеки.

А вот и нет, вполне себе скромный честный труженик с тратой памяти на заголовок и выравнивание

```
header:   8 bytes 
value:    1 byte 
padding:  7 bytes
------------------
sum:      16 bytes
```

#### Что же делать?

Ну если вам нужен именно тип boolean, он у вас во всех сигнатурах и т.д. - то ничего не делать, сидеть на попе ровно и ждать 50 релиза Java, может там престарелый Леша Шиппелoff научится распихивать свои стринги по карманам boolean.

Если дело только в эффективных структурах данных, то даже в самой Java есть кое-что на закуску. Это [https://docs.oracle.com/javase/8/docs/api/java/util/BitSet.html](https://docs.oracle.com/javase/8/docs/api/java/util/BitSet.html "BitSet ")

Это структура данных, умеющая оперировать набором логических значений как набором бит. Впрочем с boolean она несовместима. Ее можно использовать для эффективного представления в памяти достаточно больших наборов.

```java
import java.util.BitSet;
public class BitSetDemo {

  public static void main(String args[]) {
      BitSet bits1 = new BitSet(16);
      BitSet bits2 = new BitSet(16);
      
      // set some bits
      for(int i = 0; i < 16; i++) {
         if((i % 2) == 0) bits1.set(i);
         if((i % 5) != 0) bits2.set(i);
      }
     
      System.out.println("Initial pattern in bits1: ");
      System.out.println(bits1);
      System.out.println("\nInitial pattern in bits2: ");
      System.out.println(bits2);

      // AND bits
      bits2.and(bits1);
      System.out.println("\nbits2 AND bits1: ");
      System.out.println(bits2);

      // OR bits
      bits2.or(bits1);
      System.out.println("\nbits2 OR bits1: ");
      System.out.println(bits2);

      // XOR bits
      bits2.xor(bits1);
      System.out.println("\nbits2 XOR bits1: ");
      System.out.println(bits2);
   }
}
```

Вывод

```bash
Initial pattern in bits1:
{0, 2, 4, 6, 8, 10, 12, 14}

Initial pattern in bits2:
{1, 2, 3, 4, 6, 7, 8, 9, 11, 12, 13, 14}

bits2 AND bits1:
{2, 4, 6, 8, 12, 14}

bits2 OR bits1:
{0, 2, 4, 6, 8, 10, 12, 14}

bits2 XOR bits1:
{}
```

#### Почему сразу не сделали хорошо?

Во-первых, я думаю дело в том, что адресоваться к битам неудобно и сложно. Все заточено под байты - адресная арифметика, всякие машинные команды и прочее. Очень сложно подойти к биту и попытаться выполнить на нем какое-то адресное смещение. Да и вряд ли такая операция будет дешевой - мы все равно будем тратиться на обращение к байту и поиск в нем искомого бита., чтобы установить его или сбросить.

Во-вторых, и byte и short не являются полноценными численными типами, над ними довлеет проклятие int и его целочисленных операций, вряд ли бы мы смогли существенно сэкономить, перегоняя boolean-&gt;byte-&gt;int







