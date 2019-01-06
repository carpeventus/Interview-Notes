# 秋招面试题总结

## Java基础

#### 1、Hashmap是怎么实现的，底层原理？

HashMap的底层使用数组+链表/红黑树实现。

`transient Node<K,V>[] table;`这表示HashMap是Node数组构成，其中Node类的实现如下，可以看出这其实就是个链表，链表的每个结点是一个<K,V>映射。

```java
static class Node<K,V> implements Map.Entry<K,V> {
    final int hash;
    final K key;
    V value;
    Node<K,V> next;
}
```

HashMap的每个下标都存放了一条链表。

**常量/变量定义**

```java
/* 常量定义 */

// 初始容量为16
static final int DEFAULT_INITIAL_CAPACITY = 1 << 4;
// 最大容量
static final int MAXIMUM_CAPACITY = 1 << 30;
// 负载因子，当键值对个数达到DEFAULT_INITIAL_CAPACITY * DEFAULT_LOAD_FACTOR会触发resize扩容 
static final float DEFAULT_LOAD_FACTOR = 0.75f;
// 当链表长度大于8，且数组长度大于MIN_TREEIFY_CAPACITY，就会转为红黑树
static final int TREEIFY_THRESHOLD = 8;
// 当resize时候发现链表长度小于6时，从红黑树退化为链表
static final int UNTREEIFY_THRESHOLD = 6;
// 在要将链表转为红黑树之前，再进行一次判断，若数组容量小于该值，则用resize扩容，放弃转为红黑树
// 主要是为了在建立Map的初期，放置过多键值对进入同一个数组下标中，而导致不必要的链表->红黑树的转化，此时扩容即可，可有效减少冲突
static final int MIN_TREEIFY_CAPACITY = 64;

/* 变量定义 */

// 键值对的个数
transient int size;
// 键值对的个数大于该值时候，会触发扩容
int threshold;
// 非线程安全的集合类中几乎都有这个变量的影子，每次结构被修改都会更新该值，表示被修改的次数
transient int modCount;
```

关于modCount的作用见[这篇blog](https://blog.csdn.net/u012926924/article/details/50452411)

> 在一个迭代器初始的时候会赋予它调用这个迭代器的对象的modCount，如果在迭代器遍历的过程中，一旦发现这个对象的modCount和迭代器中存储的modCount不一样那就抛异常。
> **Fail-Fast机制**：java.util.HashMap不是线程安全的，因此如果在使用迭代器的过程中有其他线程修改了map，那么将抛出ConcurrentModificationException，这就是所谓fail-fast策略。这一策略在源码中的实现是通过modCount域，modCount顾名思义就是修改次数，对HashMap内容的修改都将增加这个值，那么在迭代器初始化过程中会将这个值赋给迭代器的expectedModCount。在迭代过程中，判断modCount跟expectedModCount是否相等，如果不相等就表示已经有其他线程修改了Map。

**注意初始容量和扩容后的容量都必须是2的次幂**，为什么呢?

**hash方法**

先看散列方法

```java
static final int hash(Object key) {
    int h;
    return (key == null) ? 0 : (h = key.hashCode()) ^ (h >>> 16);
}
```

HashMap的散列方法如上，其实就是将hash值的高16位和低16位异或，我们将马上看到hash在与n - 1相与的时候，高位的信息也被考虑了，能使碰撞的概率减小，散列得更均匀。

在JDK 8中，HashMap的putVal方法中有这么一句

```java
if ((p = tab[i = (n - 1) & hash]) == null)
    tab[i] = newNode(hash, key, value, null);
```

关键就是这句`(n - 1) & hash`，这行代码是把待插入的结点散列到数组中某个下标中，其中hash就是通过上面的方法的得到的，为待插入Node的key的hash值，n是table的容量即`table.length`，2的次幂用二进制表示的话，只有最高位为1，其余为都是0。减去1，刚好就反了过来。比如16的二进制表示为10000，减去1后的二进制表示为01111，除了最高位其余各位都是1，保证了在相与时，可以使得散列值分布得更均匀（因为如果某位为0比如1011，那么结点永远不会被散列到1111这个位置），且当n为2的次幂时候有`(n - 1) & hash == hash % n`, 举个例子，比如hash等于6时候，01111和00110相与就是00110，hash等于16时，相与就等于0了，多举几个例子便可以验证这一结论。最后来回答为什么HashMap的容量要始终保持2的次幂

- **使散列值分布均匀**
- **位运算的效率比取余的效率高**

注意table.length是数组的容量，而`transient int size`表示存入Map中的键值对数。

`int threshold`表示临界值，当键值对的个数大于临界值，就会扩容。threshold的更新是由下面的方法完成的。

```java
static final int tableSizeFor(int cap) {
    int n = cap - 1;
    n |= n >>> 1;
    n |= n >>> 2;
    n |= n >>> 4;
    n |= n >>> 8;
    n |= n >>> 16;
    return (n < 0) ? 1 : (n >= MAXIMUM_CAPACITY) ? MAXIMUM_CAPACITY : n + 1;
}

```

该方法返回大于等于cap的最小的二次幂数值。比如cap为16，就返回16，cap为17就返回32。

**put方法**

put方法主要由putVal方法实现：

- 如果没有产生hash冲突，直接在数组`tab[i = (n - 1) & hash]`处新建一个结点；
- 否则，发生了hash冲突，此时key如果和头结点的key相同，找到要更新的结点，直接跳到最后去更新值
- 否则，如果数组下标中的类型是TreeNode，就插入到红黑树中
- 如果只是普通的链表，就在链表中查找，找到key相同的结点就跳出，到最后去更新值；到链表尾也没有找到就在尾部插入一个新结点。接着判断此时链表长度若大于8的话，还需要将链表转为红黑树（注意在要将链表转为红黑树之前，再进行一次判断，若数组容量小于64，则用resize扩容，放弃转为红黑树）

**get方法**

get方法由getNode方法实现：

- 如果在数组下标的链表头就找到key相同的，那么返回链表头的值
- 否则如果数组下标处的类型是TreeNode，就在红黑树中查找
- 否则就是在普通链表中查找了
- 都找不到就返回null

remove方法的流程大致和get方法类似。

**HashMap的扩容，resize()过程？**

``` java
newCap = oldCap << 1
```

resize方法中有这么一句，说明是扩容后数组大小是原数组的两倍。

```java
Node<K,V>[] newTab = (Node<K,V>[])new Node[newCap];
        table = newTab;
        if (oldTab != null) {
            for (int j = 0; j < oldCap; ++j) {
                Node<K,V> e;
                if ((e = oldTab[j]) != null) {
                    oldTab[j] = null;
                    // 如果数组中只有一个元素，即只有一个头结点，重新哈希到新数组的某个下标
                    if (e.next == null)
                        newTab[e.hash & (newCap - 1)] = e;
                    else if (e instanceof TreeNode)
                        ((TreeNode<K,V>)e).split(this, newTab, j, oldCap);
                    else { // preserve order
                        // 数组下标处的链表长度大于1，非红黑树的情况
                        Node<K,V> loHead = null, loTail = null;
                        Node<K,V> hiHead = null, hiTail = null;
                        Node<K,V> next;
                        do {
                            next = e.next;
                            // oldCap是2的次幂，最高位是1，其余为是0，哈希值和其相与，根据哈希值的最高位是1还是0，链表被拆分成两条，哈希值最高位是0分到loHead。
                            if ((e.hash & oldCap) == 0) {
                                if (loTail == null)
                                    loHead = e;
                                else
                                    loTail.next = e;
                                loTail = e;
                            }
                            // 哈希值最高位是1分到hiHead
                            else {
                                if (hiTail == null)
                                    hiHead = e;
                                else
                                    hiTail.next = e;
                                hiTail = e;
                            }
                        } while ((e = next) != null);
                        if (loTail != null) {
                            loTail.next = null;
                            // loHead挂到新数组[原下标]处；
                            newTab[j] = loHead;
                        }
                        if (hiTail != null) {
                            hiTail.next = null;
                            // hiHead挂到新数组中[原下标+oldCap]处
                            newTab[j + oldCap] = hiHead;
                        }
                    }
                }
            }
        }
        return newTab;
```

举个例子，比如oldCap是16，二进制表示是10000，hash值的后五位和oldCap相与，因为oldCap的最高位（从右往左数的第5位）是1其余位是0，因此hash值的该位是0的所有元素被分到一条链表，挂到新数组中原下标处，hash值该位为1的被分到另外一条链表，挂到新数组中原下标+oldCap处。举个例子：桶0中的元素其hash值后五位是0XXXX的就被分到桶0种，其hash值后五位是1XXXX就被分到桶4中。

#### 2、Java中的错误和异常？

Java中的所有异常都是Throwable的子类对象，Error类和Exception类是Throwable类的两个直接子类。

Error：包括一些严重的、程序不能处理的系统错误类。这些错误一般不是程序造成的，比如StackOverflowError和OutOfMemoryError。

Exception：异常分为运行时异常和检查型异常。

- 检查型异常要求必须对异常进行处理，要么往上抛，要么try-catch捕获，不然不能通过编译。这类异常比较常见的是IOException。
- 运行时异常，可处理可不处理，在编译时可以通过，异常在运行时才暴露。比如数组下标越界，除0异常等。

#### 3、Java的集合类框架介绍一下？

首先接口Collection和Map是平级的，Map没有实现Collection。

Map的实现类常见有HashMap、TreeMap、LinkedHashMap和HashTable等。其中HashMap使用散列法实现，低层是数组，采用链地址法解决哈希冲突，每个数组的下标都是一条链表，当长度超过8时，转换成红黑树。TreeMap使用红黑树实现，可以按照键进行排序。LinkedHashMap的实现综合了HashMap和双向链表，可保证以插入时的顺序（或访问顺序，LRU的实现）进行迭代。HashTable和HashMap比，前者是线程安全的，后者不是线程安全的。HashTable的键或者值不允许null，HashMap允许。

Collection的实现类常见的有List、Set和Queue。List的实现类有ArrayList和LinkedList以及Vector等，ArrayList就是一个可扩容的对象数组，LinkedList是一个双向链表。Vector是线程安全的（ArrayList不是线程安全的）。Set的里的元素不可重复，实现类常见的有HashSet、TreeSet、LinkedHashSet等，HashSet的实现基于HashMap，实际上就是HashMap中的Key，同样TreeSet低层由TreeMap实现，LinkedHashSet低层由LinkedHashMap实现。Queue的实现类有LinkedList，可以用作栈、队列和双向队列，另外还有PriorityQueue是基于堆的优先队列。

#### 4、Java反射是什么？为什么要用反射，有什么好处，哪些地方用到了反射？

反射：允许任意一个类在运行时获取自身的类信息，并且可以操作这个类的方法和属性。这种动态获取类信息和动态调用对象方法的功能称为Java的反射机制。

反射的核心是JVM在运行时才动态加载类或调用方法/访问属性。它不需要事先（写代码的时候或编译期）知道运行对象是谁，如`Class.ForName()`根本就没有指定某个特定的类，完全由你传入的类全限定名决定，而通过new的方式你是知道运行时对象是哪个类的。 反射避免了将程序“写死”。

反射可以降低程序耦合性，提高程序的灵活性。new是造成紧耦合的一大原因。比如下面的工厂方法中，根据水果类型决定返回哪一个类。

```java
public class FruitFactory {
    public Fruit getFruit(String type) {
        Fruit fruit = null;
        if ("Apple".equals(type)) {
            fruit = new Apple();
        } else if ("Banana".equals(type)) {
            fruit = new Banana();
        } else if ("Orange".equals(type)) {
            fruit = new Orange();
        }
        return fruit;
    }
}

class Fruit {}
class Banana extends Fruit {}
class Orange extends Fruit {}
class Apple extends Fruit {}
```

但是我们事先并不知道之后会有哪些类，比如新增了Mango，就需要在if-else中新增；如果以后不需要Banana了就需要从if-else中删除。这就是说只要子类变动了，我们必须在工厂类进行修改，然后再编译。如果用反射呢？

```java
public class FruitFactory {
    public Fruit getFruit(String type) {
        Fruit fruit = null;
        try {
            fruit = (Fruit) Class.forName(type).newInstance();
        } catch (Exception e) {
            e.printStackTrace();
        }
        return fruit;
    }
}

class Fruit {}
class Banana extends Fruit {}
class Orange extends Fruit {}
class Apple extends Fruit {}
```

如果再将子类的全限定名存放在配置文件中。

```properties
class-type=com.fruit.Apple
```

那么不管新增多少子类，根据不同的场景只需修改文件就好了，上面的代码无需修改代码、重新编译，就能正确运行。

哪些地方用到了反射？举几个例子

- 加载数据库驱动时
- Spring的IOC容器，根据XML配置文件中的类全限定名动态加载类
- 工厂方法模式中（如上）

#### 5、说说你对面向对象、封装、继承、多态的理解？

- 封装：隐藏实现细节，明确标识出允许外部使用的所有成员函数和数据项。 防止代码或数据被破坏。
- 继承：子类继承父类，拥有父类的所有功能，并且可以在父类基础上进行扩展。实现了代码重用。子类和父类是兼容的，外部调用者无需关注两者的区别。
- 多态：一个接口有多个子类或实现类，在运行期间（而非编译期间）才决定所引用的对象的实际类型，再根据其实际的类型调用其对应的方法，也就是“动态绑定”。

Java实现多态有三个必要条件**：继承、重写、向上转型。**

- 继承：子类继承或者实行父类
- 重写：在子类里面重写从父类继承下来的方法
- 向上转型：父类引用指向子类对象

```java
public class OOP {
    public static void main(String[] args) {
        /*
         * 1. Cat继承了Animal
         * 2. Cat重写了Animal的eat方法
         * 3. 父类Animal的引用指向了子类Cat。
         * 在编译期间其静态类型为Animal;在运行期间其实际类型为Cat，因此animal.eat()将选择Cat的eat方法而不是其他子类的eat方法
         */
        Animal animal = new Cat();
        printEating(animal);
    }

    public static void printEating(Animal animal) {
        animal.eat();
    }
}

abstract class Animal {
    abstract void eat();
}
class Cat extends Animal {
    @Override
    void eat() {
        System.out.println("Cat eating...");
    }
}
class Dog extends Animal {
    @Override
    void eat() {
        System.out.println("Dog eating...");
    }
}
```

#### 6、实现不可变对象的策略？比如JDK中的String类。

- 不提供setter方法（包括修改字段、字段引用到的的对象等方法）
- 将所有字段设置为final、private
- 将类修饰为final，不允许子类继承、重写方法。可以将构造函数设为private，通过工厂方法创建。
- 如果类的字段是对可变对象的引用，不允许修改被引用对象。 1）不提供修改可变对象的方法；2）不共享对可变对象的引用。对于外部传入的可变对象，不保存该引用。如要保存可以保存其复制后的副本；对于内部可变对象，不要返回对象本身，而是返回其复制后的副本。

#### 7、Java序列话中如果有些字段不想进行序列化，怎么办？

对于不想进行序列化的变量，使用transient关键字修饰。功能是：阻止实例中那些用此关键字修饰的的变量序列化；当对象被反序列化时，被transient修饰的变量值不会被持久化和恢复。transient只能修饰变量，不能修饰类和方法。

#### 8、==和equals的区别？

== 对于基本类型，比较值是否相等，对于对象，比较的是两个对象的地址是否相同，即是否是指相同一个对象。

equals的默认实现实际上使用了==来比较两个对象是否相等，但是像Integer、String这些类对equals方法进行了重写，比较的是两个对象的内容是否相等。

对于Integer，如果依然坚持使用==来比较，有一些要注意的地方。对于[-128,127]区间里的数，有一个缓存。因此

```java
Integer a = 127;
Integer b = 127;
System.out.println(a == b); // true

Integer a = 128;
Integer b = 128;
System.out.println(a == b); // false

// 不过采用new的方式，a在堆中，这里打印false
Integer a = new Integer(127);
Integer b = 127;
System.out.println(a == b);
```

对于String，因为它有一个常量池。所以

```java
String a = "gg" + "rr";
String b = "ggrr";
System.out.println(a == b); // true

// 当然牵涉到new的话，该对象就在堆上创建了，所以这里打印false
String a = "gg" + "rr";
String b = new String("ggrr");
System.out.println(a == b);
```

#### 9、接口和抽象类的区别？

- Java不能多继承，一个类只能继承一个抽象类；但是可以实现多个接口。
- 继承抽象类是一种IS-A的关系，实现接口是一种LIKE-A的关系。
- 继承抽象类可以实现对父类代码的复用，也可以重写抽象方法实现子类特有的功能。实现接口可以为类新增额外的功能。
- 抽象类定义基本的共性内容，接口是定义额外的功能。
- 调用者使用动机不同, 实现接口是为了使用他规范的某一个行为；继承抽象类是为了使用这个类属性和行为.

#### 10、给你一个Person对象p，如何将该对象变成JSON表示？

本质是考察Java反射，因为要实现一个通用的程序。实现可能根本不知道该类有哪些字段，所以不能通过get和set等方法来获取键-值。使用反射的getDeclaredFields()可以获得其声明的字段。如果字段是private的，需要调用该字段的`f.setAccessible(true);`，才能读取和修改该字段。

```java
import java.lang.reflect.Field;
import java.util.HashMap;

public class Object2Json {
    public static class Person {
        private int age;
        private String name;

        public Person(int age, String name) {
            this.age = age;
            this.name = name;
        }
    }

    public static void main(String[] args) throws IllegalAccessException {
        Person p = new Person(18, "Bob");
        Class<?> classPerson = p.getClass();
        Field[] fields = classPerson.getDeclaredFields();
        HashMap<String, String> map = new HashMap<>();
        for (Field f: fields) {
            // 对于private字段要先设置accessible为true
            f.setAccessible(true);
            map.put(String.valueOf(f.getName()), String.valueOf(f.get(p)));
        }
        System.out.println(map);
    }
}

```

得到了map，再弄成JSON标准格式就好了。

#### 11、JDBC中sql查询的完整过程？操作事务呢？

```java
@Test
public void fun2() throws SQLException, ClassNotFoundException {
    // 1. 注册驱动
    Class.forName("com.mysql.jdbc.Driver");
    String url = "jdbc:mysql://localhost:3306/xxx?useUnicode=true&characterEncoding=utf-8";
    // 2.建立连接
    Connection connection = DriverManager.getConnection(url, "root", "admin");
    // 3. 执行sql语句使用的Statement或者PreparedStatment
    Statement statement = connection.createStatement();
    String sql = "select * from stu;";
    ResultSet resultSet = statement.executeQuery(sql);

    while (resultSet.next()) {
        // 第一列是id，所以从第二行开始
        String name = resultSet.getString(2); // 可以传入列的索引，1代表第一行，索引不是从0开始
        int age = resultSet.getInt(3);
        String gender = resultSet.getString(4);
        System.out.println("学生姓名：" + name + " | 年龄：" + age + " | 性别：" + gender);
    }
    // 关闭结果集
    resultSet.close();
    // 关闭statemenet
    statement.close();
    // 关闭连接
    connection.close();
}
```

**ResultSet维持一个指向当前行记录的cursor（游标）指针**

- 注册驱动
- 建立连接
- 准备sql语句
- 执行sql语句得到结果集
- 对结果集进行遍历
- 关闭结果集（ResultSet）
- 关闭statement
- 关闭连接（connection）

由于JDBC默认自动提交事务，每执行一个update ,delete或者insert的时候都会自动提交到数据库，无法回滚事务。所以若需要实现事务的回滚，要指定`setAutoCommit(false)`。

- `true`：sql命令的提交（commit）由驱动程序负责
- `false`：sql命令的提交由应用程序负责，程序必须调用commit或者rollback方法

JDBC操作事务的格式如下，在捕获异常中进行事务的回滚。

```java
try {
  con.setAutoCommit(false);//开启事务…
  ….
  …
  con.commit();//try的最后提交事务
} catch() {
  con.rollback();//回滚事务
}
```

#### 12、实现单例，有哪些要注意的地方？

就普通的实现方法来看。

- 不允许在其他类中直接new出对象，故构造方法私有化
- 在本类中创建唯一一个static实例对象
- 定义一个public static方法，返回该实例

```java
public class SingletonImp {
    // 饿汉模式
    private static SingletonImp singletonImp = new SingletonImp();
    // 私有化（private）该类的构造函数
    private SingletonImp() {
    }

    public static SingletonImp getInstance() {
        return singletonImp;
    }
}
```

饿汉模式：线程安全，不能延迟加载。

```java
public class SingletonImp4 {
    private static volatile SingletonImp4 singletonImp4;

    private SingletonImp4() {}

    public static SingletonImp4 getInstance() {
        if (singletonImp4 == null) {
            synchronized (SingletonImp4.class) {
                if (singletonImp4 == null) {
                    singletonImp4 = new SingletonImp4();
                }
            }
        }

        return singletonImp4;
    }
}
```

双重检测锁+volatile禁止语义重排。因为`singletonImp4 = new SingletonImp4();`不是原子操作。

```java
public class SingletonImp6 {
    private SingletonImp6() {}

    // 专门用于创建Singleton的静态类
    private static class Nested {
        private static SingletonImp6 singletonImp6 = new SingletonImp6();
    }

    public static SingletonImp6 getInstance() {
        return Nested.singletonImp6;
    }
}
```

静态内部类，可以实现延迟加载。

最推荐的是单一元素枚举实现单例。

- 写法简单
- 枚举实例的创建默认就是线程安全的
- 提供了自由的序列化机制。面对复杂的序列或反射攻击，也能保证是单例

```java
public enum Singleton {
    INSTANCE;
    public void anyOtherMethod() {}
}
```

## 数据结构与算法

#### 1、二叉树的遍历方式？它们属于深搜还是广搜？

- 先序遍历。父结点 -> 左子结点 -> 右子结点
- 中序遍历。左子结点 -> 父结点 -> 右子结点
- 后序遍历。左子结点 -> 右子结点 -> 父结点
- 层序遍历。一层一层自上而下，从左往右访问。

其中，先序、中序、后序遍历属于深度优先搜索（DFS），层序遍历属于广度优先搜索（BFS）

#### 2、什么是平衡二叉树，它的好处是什么？被应用在哪些场景中？

平衡二叉树首先是一棵二叉查找树，其次它需要满足其左右两棵子树的高度之差不超过1，且子树也必须是平衡二叉树，换句话说对于平衡二叉树的每个结点，要求其左右子树高度之差都不超过1。

二叉查找树在最坏情况下，退化成链表，查找时间从平均O(lg n)降到O(n),平衡二叉树使树的结构更加平衡，提高了查找的效率；但是由于插入和删除后需要重新恢复树的平衡，所以插入和删除会慢一些。

应用场景比如在HashMap中用到了红黑树（平衡二叉树的特例），数据库索引中的B+树等。

#### 3、数组和链表的区别?

- 数组是一段连续的内存空间，所以支持随机访问，可以通过下标以O(1)的时间进行查找。链表中的元素在内存中不是连续的，可以分散在内存中各个地方。因此它不支持随机访问，查找效率是O(n)
- 数组的插入和删除元素时间是O(n)，插入和删除后要移动元素；而链表只需断开结点链接再重新连上，所以链表的删除和插入时间是O(1)
- 数组必须指定初始大小，达到最大容量如果不扩容就不能再存入新的元素；而链表没有容量的限制

应用场景：数组适合读多写少、事先知道元素大概个数的情况；链表适合写多读少的情况。

#### 4、冒泡和快排的区别，最差和平均的时间复杂度？

- 冒泡：相邻元素进行两两比较，将最大值推动到最右边。重复以上过程。时间复杂度平均`O(N^2)`最差`O(N^2)`，空间复杂度`O(1)`
- 快排：选择数组中第一个元素作为基准，从左边向右找到第一个大于等于基准的元素，从右边向左找到第一个小于等于基准的元素，交换这两个元素，最后基准左边的元素都小于等于基准，基准右边的元素都大于等于基准。然后固定基准元素，对其左边和右边采取同样的做法。典型的分治思想。时间复杂度平均`O(N lgN)`最差`O(N^2)`，基于递归的实现由于用到了系统栈，所以平均情况下空间复杂度为`O(lgN)`
- 冒泡排序是稳定的排序算法，快速排序不是稳定的排序算法。

排序中所说的稳定是指，对于两个相同的元素，在排序后其相对位置没有发生变化。

常见的稳定排序有，冒泡、插入、归并、基数排序。选择、希尔、快排、堆排序都不是稳定的。

#### 5、说说常用的散列方法？解决哈希冲突的方法有哪些？

- 最常用的除留余数法（取余），大小为M的数组，key的哈希值为k，那么k % M的值一定是落在0-M-1之间的。
- 直接定址法。用一个函数对key作映射得到哈希值。如线性函数：`hash(key) = a * key + b`
- 其他（略）

解决哈希冲突的方法：

- 开放定址法：采用M大小的数组存放N个键值对，其中M > N。开放定址中最简单的是线性探测法，当发生碰撞时，直接检查散列表中的下一个位置。如果到了数组末尾，折回到索引0处继续查找。
- 链地址法：采用链表数组实现，当发生哈希冲突时，将冲突键值以头插或者尾插的方式插入数组下标所在的链表，HashMap中正是使用了这种方法。
- 再哈希法：当发生哈希冲突时，换一个散列函数重新计算哈希值。
- 公共溢出区法：建立一个基本表和溢出表，所有冲突的键值都存放到溢出表中。在查找时，先在基本表中查，相等，查找成功，如不相等则去溢出表中进行顺序查找。

#### 6、堆排序的过程说一下？

堆排序使用了最大堆/最小堆，拿数组升序排序来说，需要建立一个最大堆，基于数组实现的二叉堆可以看作一棵完全二叉树，其满足堆中每个父结点它左右两个结点值都大，且堆顶的元素最大。

- 将堆顶元素和数组最后一个元素交换，最大元素被交换数组最后一个位置，同时从堆中删除原来处于堆顶的最大元素
- 被交换到堆顶的元素一般会打破堆结构的定义，因此需要进行堆的调整（下沉）。将堆顶的元素和其左右两个结点比较，将三者中的最大的交换到堆顶，然后继续跟踪此结点，循环上述过程，直到他比左右两个结点都大时停止调整，此时堆调整完毕，再次满足堆结构的定义
- 重复以上两个过程。直到堆中只剩一个元素，此时排序完成

每次调整堆的平均时间为O(lg N)，因此对大小为N的数组排序，时间复杂度最差和平均都 O(N lg N).

#### 7、堆排序和快排应用场景的？时间复杂度和空间复杂度各是多少？

快排序在平均情况下，比绝大多数排序算法都快些。很多编程语言的sort默认使用快排，像Java的Array.sort()就采用了双轴快速排序 。堆排序使用堆实现，空间复杂度只有O(1)。堆排序使用堆的结构，能以O(1)的时间获得最大/最小值，在处理TOP K问题时很方便，另外堆还可以实现优先队列。

时间复杂度：

- 快排，平均O(N lg N) ,最差O(N^2)，这种情况发生在数组本身就有序，这样每次切分元素都是数组中的最小值，切分得就极为不平衡。
- 堆排序，平均和最差都是O(N lgN)。因为每次调整堆结构都是O(lg N)，因此对大小为N的数组排序，时间复杂度最差和平均都 O(N lg N).

空间复杂度：

- 快排，一般基于递归实现，需要使用系统栈O(lg N)
- 堆排序，额外空间复杂度O(1）

放一张神图

![](http://picmeup.oss-cn-hangzhou.aliyuncs.com/coding/975503-20170214211234550-1109833343.png)

#### 8、双向链表，给你Node a，在其后插入Node c？

```java
c.next = a.next;
a.next = c;
c.prev = a;
// 如果a不是最后一个结点，就有下面一句
c.next.prev = c;
```

#### 9、写并查集？

```java
public class UnionFind {
    // id相同的分量是连通的
    private int[] id;
    //连通分量的个数
    private int count;

    public UnionFind(int n) {
        count = n;
        id = new int[n];
        for (int i = 0; i < n; i++) {
            id[i] = i;
        }
    }
	// 所属连通分量的id
    public int find(int p) {
        return id[p];
    }
	// 将和p同一个连通分量的结点全部归到和q一个分量中，即将p所在连通分量与q所在连通分量合并。
	// 反过来也可以
	// if (id[i] == qID) {
	// id[i] = pID;
	// }
    public void union(int p, int q) {
        int pId = find(p);
        int qId = find(q);
        if (pId == qId) return;
        for (int i = 0; i < id.length; i++) {
            if (id[i] == pId) {
                id[i] = qId;
            }
        }
        // 合并后，连通分量减少1
        count--;
    }

    public boolean connected(int p, int q) {
        return find(p) == find(q);
    }
}
```

还可以有优化的写法，一个连通分量看成是一棵树。同一个连通分量其树的根结点相同。

```java
public class UnionFind {
    private int[] parentTo;
    private int count;

    public UnionFind(int n) {
        count = n;
        parentTo = new int[n];
        for (int i = 0; i < n; i++) {
            parentTo[i] = i;
        }
    }

    public int find(int p) {
        // 向上一直到根结点
        // p = parentTo[p]说明到达树的根结点，返回根结点
        while (p != parentTo[p]) {
            p = parentTo[p];
        }
        return p;
    }
	// 这行的意思就是q所在连通分量和q所在连通分量合并
	// 从树的角度来看，p树的根结点成为了q树根结点的孩子结点
	// 反过来也可以，parentTo[qRoot] = pRoot;
    public void union(int p, int q) {
        int pRoot = find(p);
        int qRoot = find(q);
        if (pRoot == qRoot) return;
        parentTo[pRoot] = qRoot;
        count--;
    }

    public boolean connected(int p, int q) {
        return find(p) == find(q);
    }
}
```

#### 10、HashMap存了若干(name, age)这样的键值对，现在想按照年龄排序，打印出姓名？

因为人类的年龄在一个固定范围内，假设0~100吧。可以设置101个桶，每个桶中放的是该年龄的所有用户名。

```java
    public static void printNameOrderByAge(Map<String, Integer> map) {
        LinkedList<String>[] bucket = new LinkedList[101];
        for (String name : map.keySet()) {
            int age = map.get(name);
            if (bucket[age] == null) {
                bucket[age] = new LinkedList<>();
            }
            bucket[age].add(name);
        }

        for (int i = 0;i < bucket.length;i++) {
            if (bucket[i] != null) {
                System.out.print(i + " : ");
                System.out.println(bucket[i]);
            }
        }
    }

    public static void main(String[] args) throws InterruptedException {
        Map<String, Integer> map = new HashMap<>();
        map.put("Alice", 20);
        map.put("Bob", 20);
        map.put("Tim", 19);
        map.put("Joker", 19);
        map.put("Carlos", 18);
        map.put("XiaoMing", 22);
        printNameOrderByAge(map);
    }
```

打印如下内容

```
18 : [Carlos]
19 : [Joker, Tim]
20 : [Bob, Alice]
22 : [XiaoMing]
```

## 计算机网络

#### 1、HTTP有哪些请求方法？它们的作用或者说应用场景？

- GET: 请求指定的页面信息，并返回实体主体。
- HEAD: 和GET类似，只不过不返回报文主体，只返回响应首部。可用于确认URI的有效性及资源更新的日期时间；
- POST: 向指定资源提交数据进行处理请求（例如提交表单或者上传文件）。数据被包含在请求体中。POST请求可能会导致新的资源的建立和/或已有资源的修改。
- PUT: 用来传输文件，要求请求报文的主体中包含文件内容，然后保存到请求URI指定的位置。
- DELETE: 和PUT相反，按请求URI删除指定的资源。
- OPTIONS: 用来查询针对请求URI指定的资源支持的方法。如果请求成功，会有一个Allow的头包含类似“GET,POST”这样的信息
- TRACE: 让服务端将之前的请求通信返回给客户端的方法（因此客户端可以得知请求是怎么一步步到服务端的）。主要用于测试或诊断。
- CONNECT: 使用 SSL（Secure Sockets Layer，安全套接层）和 TLS（Transport Layer Security，传输层安全）协议把通信内容加密后经网络隧道传输。

#### 2、GET和POST的对比，或者说区别？

- GET用于获取数据，POST用于提交数据;
- GET的参数长度有限制（不同的浏览器和服务器限制不同），POST没有限制
- GET把参数包含在URL中，POST通过封装参数到请求体中发送；
- GET请求只能进行url编码，而POST支持多种编码方式。
- GET可以发送的参数只能是ASCII类型，POST没有限制，甚至可以传输二进制。
- GET比POST更不安全，因为参数直接暴露在URL上，所以不能用来传递敏感信息;
- GET刷新无害，而POST会再次提交数据
- 还有GET请求会被保存浏览器历史记录，可以被收藏为书签，而POST请求不能等等
  

GET和POST本质都是TCP连接。不过GET产生一个TCP数据包；POST产生两个TCP数据包。

对于GET方式的请求，浏览器会把http header和data一并发送出去，服务器响应200 OK（返回数据）；

而对于POST，浏览器先发送header，服务器响应100 continue，浏览器再发送data，服务器响应200 OK（返回数据）。

#### 3、TCP三次握手？四次挥手？

**三次握手**

- 请求先由客户端发起。客户端发送SYN = 1和客户端序号c给服务端，同时进入SYN-SENT状态
- 服务端收到SYN后需要做出确认，于是发送ACK = 1，同时自己也发送SYN = 1、服务端序号s，还有确认号c + 1，表示想收到的下一个序号。此时服务端进入SYN-RCVD状态
- 客户端收到服务端的SYN和ACK，做出确认，发送ACK = 1，以及序号c +１，同时发送确认号s + 1，表示客户端想收到下一个序号。此时客户端和服务端进入ESTABLISHED状态，连接已建立！

![f8ef1381-713f-4fbe-83d7-b13a1c4b6bc9](http://picmeup.oss-cn-hangzhou.aliyuncs.com/coding/f8ef1381-713f-4fbe-83d7-b13a1c4b6bc9.jpg)

**四次挥手**

- 关闭连接也是先有客户端发起。客户端发送FIN = 1和序号c向服务端请求断开连接。此时客户端进入FIN-WAIT-1状态
- 服务端收到FIN后做出确认，发送ACK = 1和服务端序号，还有确认号c + 1表示想要收到的下一个序号。服务端此时还可以向客户端发送数据。此时服务端进入CLOSE-WAIT状态，客户端进入FIN-WAIT-2状态
- 服务端没有数据发送时，它向客户端发送FIN= 1、ACK = 1请求断开连接，同时发送服务端序号s以及确认号c + 1。此时服务端进入LAST-ACK状态
- 客户端收到后进行确认，发送ACK = 1，以及需要c + 1和确认号s + 1。此时客户端进入TIME-WAIT状态。客户端需要等待2MSL，确保服务端收到了ACK，若这期间客户端没有收到服务端的消息，便可认为服务端收到了确认，此时可以断开连接。客户端和服务端进入CLOSED状态。

![8e9432d9-e6ca-4227-a8b6-fd958df00b6b](http://picmeup.oss-cn-hangzhou.aliyuncs.com/coding/8e9432d9-e6ca-4227-a8b6-fd958df00b6b.jpg)

#### 4、TCP为什么需要三次握手？两次不行吗？

两次握手的话，只要服务端发出确认就建立连接了。有一种情况是客户端发出了两次连接请求，但由于某种原因，使得第一次请求被滞留了。第二次请求先到达后建立连接成功，此后第一次请求终于到达，这是一个失效的请求了，服务端以为这是一个新的请求于是同意建立连接，但是此时客户端不搭理服务端，服务端一直处于等待状态，这样就浪费了资源。假设采用三次握手，由于服务端还需要等待客户端的确认，若客户端没有确认，服务端就可以认为客户端没有想要建立连接的意思，于是这次连接不会生效。

#### 5、四次挥手，为什么客户端发送确认后还需要等待2MSL?

因为第四次挥手客户端发送ACK确认后，有可能丢包了，导致服务端没有收到，服务端就会再次发送FIN = 1，如果客户端不等待立即CLOSED，客户端就不能对服务端的FIN = 1进行确认。等待的目的就是为了能在服务端再次发送FIN = 1时候能进行确认。如果在2MSL内客户端都没有收到服务端的任何消息，便认为服务端收到了确认。此时可以结束TCP连接。

#### 6、cookie和session区别和联系？

- Session是在服务端保存的一个数据结构，用来跟踪用户的状态，这个数据可以保存在集群、数据库、文件中；
- Cookie是客户端保存用户信息的一种机制，用来记录用户的一些信息，也是实现Session的一种方式。
- session的运行依赖session id，而session id是存在cookie中的，也就是说，如果浏览器禁用了cookie ，同时session也会失效（但是可以通过url重写，即在url中传递session_id）

#### 7、为何使用session？session的原理？

比如网上购物，每个用户有自己的购物车，当点击下单时，由于HTTP协议无状态，并不知道是哪个用户操作的，所以服务端要为特定的用户创建特定的Session，用于标识这个用户，并且跟踪用户。

Session原理：浏览器第一次访问服务器时，服务器会响应一个cookie给浏览器。这个cookie记录的就是sessionId，之后每次访问携带着这个sessionId，服务器里查询该sessionId，便可以识别并跟踪特定的用户了。

Cookie原理：第一次访问服务器，服务器响应时，要求浏览器记住一个信息。之后浏览器每次访问服务器时候，携带第一次记住的信息访问。相当于服务器识别客户端的一个通行证。Cookie不可跨域，浏览览器判断一个网站是否能操作另一个网站Cookie的依据是域名。Google与Baidu的域名不一样，因此Google不能操作Baidu的Cookie，换句话说Google只能操作Google的Cookie。

#### 8、网络的7层模型了解吗？

即OSI参考模型。

- 应用层。针对特定应用的协议，为应用程序提供服务。如电子邮件、远程登录、文件传输等协议。
- 表示层。主要负责数据格式的转换，把不同表现形式的信息转换成适合网络传输的格式。
- 会话层。通信管理，负责建立和断开通信连接。即何时建立连接、何时断开连接以及保持多久的连接。
- 传输层。在两个通信结点之间负责数据的传输，起着可靠传输的作用。
- 网络层。路由选择。在多个网络之间转发数据包，负责将数据包传送到目标地址。
- 数据链路层。负责物理层面上互联设备之间的通信传输。例如与一个以太网相连的两个节点之间的通信。是数据帧与1、0比特流之间的转换。
- 物理层。主要是1、0比特流与电子信号的高低电平之间的转换。

还有一种TCP/IP五层模型，就是把应用层、表示层、会话层统一归到应用层。借用一张图。

![20170905102825355](http://picmeup.oss-cn-hangzhou.aliyuncs.com/coding/20170905102825355.jpg)

#### 9、有了传输层为什么还需要网络层？或者说网络层和传输层是如何协作的？

网络层是针对主机与主机之间的服务。而传输层针对的是不同主机进程之间的通信。传输层协议将应用进程的消息传送到网络层，但是它并不涉及消息是怎么在网络层之间传送（这部分是由网络层的路由选择完成的）。网络层真正负责将数据包从源IP地址转发到目标IP地址，而传输层负责将数据包再递交给主机中对应端口的进程。

![e24d3846-c2d5-4714-afed-106c7c8096bf](http://picmeup.oss-cn-hangzhou.aliyuncs.com/coding/e24d3846-c2d5-4714-afed-106c7c8096bf.jpg)

打个比方。房子A中的人要向房子B中的人写信。房子中都有专门负责将主人写好的信投递到邮箱，以及从邮箱接收信件后交到主人手中的管家。那么：

- 房子 = 主机
- 信的内容 = 应用程序消息
- 信封 = 数据包，带有源端口、目的端口、源IP地址、目的IP地址。
- 邮递员 = 网络层协议，知道信从哪个房子开始发的，以及最后要送到哪个具体的房子。
- 管家 = 传输层协议，负责将信投入到信箱中、以及从信箱中接收信件。知道这封信是谁写的以及要送到谁手上（具体端口号）

以上只是个人理解，如有误请联系更正。

#### 10、TCP和UDP的区别？

- TCP面向连接，传输数据之前要需要建立会话。UDP是无连接的。
- TCP提供可靠传输，保证数据不丢包、不重复且按顺序到达；UDP只尽努力交付，不保证可靠交付
- TCP提供了拥塞控制；UDP不提供
- TCP是面向字节流的；UDP面向报文。
- TCP只支持点到点通信；UDP支持一对一、一对多、多对多的交互通信。
- TCP首部开销大20字节，UDP首部开销小8字节。

#### 11、传输层的可靠传输指的是什么？是如何实现的？

可靠传输是指

- 传输的信道不产生差错
- 保证传输数据的正确性，无差错、不丢失、不重复且按顺序到达。

TCP如何实现可靠传输：

- 应用数据被分割成TCP认为最适合发送的块进行传输
- 超时重传，TCP发出一个分组后，它启动一个定时器，等接收方确认收到这个分组。如果发送方不能及时收到一个确认，将重传给接收方。
- 序号，用于检测丢失的分组和冗余的分组。
- 确认，告知对方已经正确收到的分组以及期望的下一个分组
- 校验和，校验数据在传输过程中是否发生改变，如校验有错则丢弃分组；
- 流量控制，使用滑动窗口，发送窗口的大小由接收窗口和拥塞窗口的的大小决定（取两者中小的那个），当接收方来不及处理发送方的数据，能提示发送方降低发送的速率，防止包丢失。
- 拥塞控制：当网络拥塞时，减少数据的发送。

#### 12、主机A向主机B发送数据，在这个过程中，传输层和网络层做了什么？

当TCP连接建立之后，应用程序就可使用该连接进行数据收发。应用程序将数据提交给TCP，TCP将数据放入自己的缓存，数据会被当做字节流并进行分段，然后加上TCP头部并提交给网络层。再加上IP头后被网络层提交给到目的主机，目的主机的IP层会将分组提交给TCP，TCP根据报文段的头部信息找到相应的socket，并将报文段提交给该socket，socket是和应用关联的，于是数据就提交给了应用。

对于UDP会简单些，UDP面向报文段。传输层加上UDP头部递交给网络层，再加上IP头部经路由转发到目的主机，目的主机将分组提交给UDP，UDP根据头部信息找到相应的socket，并将报文段提交给该socket，socket是和应用关联的，于是数据就提交给了应用。

#### 13、TCP序号的作用？怎么保证可靠传输的？ 

序号和确认号是实现可靠传输的关键。

- 序号：当前数据包的首个字节的顺序号。
- 确认号：表示下一个想要接收的字节序号，同时确认号表示对发送方的一个确认回应，表示已经正确收到确认号之前的字节了。

通信双方通过序号和确认号，来判断数据是否丢失、是否按顺序到达、是否冗余等，以此决定要不要进行重传丢失的分组或丢弃冗余的分组。换句话说，因为有了序号、确认号和重传机制，保证了数据不丢失、不重复、有序到达。

#### 14、浏览器发起HTTP请求后发生了什么？越详细越好。

当在浏览器输入网址www.baidu.com并敲下回车后：

- DNS域名解析，将域名www.baidu.com解析成IP地址
- 发起TCP三次握手，建立TCP连接。浏览器以一个随机端口（1024~65535）向服务器的80端口发起TCP连接。
- TCP连接建立后，发起HTTP请求。
- 服务端响应HTTP请求，将html代码返回给浏览器。
- 浏览器解析html代码，请求html中的资源
- 浏览器对页面进行渲染呈现给用户
  

![964016-20160830113547246-672458721](http://picmeup.oss-cn-hangzhou.aliyuncs.com/coding/964016-20160830113547246-672458721.png)

#### 15、DNS域名解析的请求过程？

- 先在浏览器自身的DNS缓存中搜索
- 如上述步骤未找到，浏览器搜索操作系统本身的DNS缓存
- 如果在系统DNS缓存中未找到，则尝试读取hosts文件，寻找有没有该域名对应的IP
- 如果hosts文件中没找到，浏览器会向本地配置的首选DNS服务器发起域名解析请求 。运营商的DNS服务器首先查找自身的缓存，若找到对应的条目且没有过期，则解析成功。如果没有找到，运营商的DNS代我们的浏览器，以根域名->顶级域名->二级域名->三级域名这样的顺序发起迭代DNS解析请求。

![964016-20160830113557949-272537363](http://picmeup.oss-cn-hangzhou.aliyuncs.com/coding/964016-20160830113557949-272537363.png)

#### 16、HTTP是基于TCP还是UDP的？

HTTP协议是基于TCP协议的，客户端向服务端发送一个HTTP请求时，需要先与服务端建立TCP连接（三次握手），握手成功以后才能进行数据交互。

#### 17、HTTP请求和响应的报文结构（格式）？

HTTP请求的报文格式：

- 请求行：包括请求方法、URL、HTTP协议版本号
- 请求头：若干键值对组成
- 请求空行：告诉服务器请求头的键值对已经发送完毕
- 请求主体

![20170330192653242](http://picmeup.oss-cn-hangzhou.aliyuncs.com/coding/20170330192653242.png)

HTTP响应的报文格式：

- 响应行：HTTP协议版本号、状态码、状态码描述
- 响应头：若干键值对表示
- 响应空行：标识响应头的结束
- 响应主体

![20170330192754102](http://picmeup.oss-cn-hangzhou.aliyuncs.com/coding/20170330192754102.png)
#### 18、HTTP常见的状态码？

- 1XX：信息性状态码，表示接收的请求正在处理
- 2XX：成功状态码，表示请求正常处理完毕
- 3XX：重定向状态码，表示需要进行附加操作以完成请求
- 4XX：客户端错误状态码，表示服务器无法处理请求
- 5XX：服务端错误状态码，表示服务器处理请求出错

常见的状态码有：

- 200 OK，请求被正常处理
- 301 Move Permanently，永久性重定向
- 302 Found，临时性重定向
- 400 Bad Request，请求报文中存在语法错误
- 403 Forbidden，对请求资源的访问被服务器拒绝
- 404 Not Found，在服务器上不能找到请求的资源
- 500 Internal Server Error，服务器内部错误

## 并发/多线程

#### 1、两个线程对可以同一个ArrayList进行add操作吗？会出现什么结果？

```java
import java.util.ArrayList;
import java.util.List;

public class A {
     static List<Integer> list = new ArrayList<>();
     static class BB implements Runnable {
        @Override
        public void run() {
            for (int j = 0; j < 100; j++) {
                list.add(j);
            }
        }
    }
    public static void main(String[] args) throws InterruptedException {
        BB b = new BB();
        Thread t1 = new Thread(b);
        Thread t2 = new Thread(b);
        t1.start();
        t2.start();
        t1.join();
        t2.join();
        System.out.println(list.size());
    }
}


```

比如上面的例子，打印的结果不一定是200.

因为ArrayList不是线程安全的，问题出在add方法

```java
public boolean add(E e) {
    ensureCapacityInternal(size + 1);  // Increments modCount!!
    elementData[size++] = e;
    return true;
}
```

上面的程序，可能有三种情况发生：

- **数组下标越界**。首先要检查容量，必要时进行扩容。每当在数组边界处，如果A线程和B线程同时进入并检查容量，也就是它们都执行完ensureCapacityInternal方法，因为还有一个空间，所以不进行扩容，此时如果A暂停下来，B成功自增；然后接着A从`elementData[size++] = e`开始执行，由于A之前已经检查过没有扩容，而B成功自增使得现在没有空余空间了，此时A就会发生数组下标越界。
- **小于200**。size++可以看成是`size = size + 1`，这一行代码包括三个步骤，先读取size，然后将size加1，最后将这个新值写回到size。此时若A和B线程同时读取到size假设为10，B先自增成功size变11，然后回来A因为它读到的size也是10，所以自增后写入size被更新成11，也就是说两次自增，实际上size只增大了1。因此最后的size会小于200。
- 200。运气很好，没有发生以上的情况。

#### 2、volatile和synchronized讲一下？

synchronized保证了当有多个线程同时操作共享数据时，任何时刻只有一个线程能进入临界区操作共享数据，其他线程必须等待。因此它可以保证操作的原子性。synchronized通过同步锁保证线程安全，进入临界区前必须获得对象的锁，其他没有获得锁的线程不可进入。当临界区中的线程操作完毕后，它会释放锁，此时其他线程可以竞争锁，得到锁的那个线程便可以进入临界区。

synchronized还可以保证可见性。因为对一个变量的unlock操作之前，必须先把次变量同步回主内存中。它还可以保证有序性，因为一个变量在任何时刻只能有一个线程对其进行lock操作（也就是任何时刻只有一个线程可以获得该锁对象），这决定了持有同一把锁的两个同步块只能串行进入。

volatile是一个关键字，用于修饰变量。被其修饰的变量具有可见性和有序性。

可见性，当一条线程修改了这个变量的值，新值能被其他线程立刻观察到。具体来说，volatile的作用是：在本CPU对变量的修改直接写入主内存中，同时这个写操作使得其他CPU中对应变量的缓存行无效，这样其他线程在读取这个变量时候必须从主内存中读取，所以读取到的是最新的，这就是上面说得能被立即“看到”。

有序性，volatile可以禁止指令重排。volatile在其汇编代码中有一个lock操作，这个操作相当于一个内存屏障，指令重排不能越过内存屏障。具体来说在执行到volatile变量时，内存屏障之前的语句一定被执行过了且结果对后面是已知的，而内存屏障后面的语句一定还没执行到；在volatile变量之前的语句不能被重排后其之后，相反其后的语句也不能被重排到之前。

#### 3、synchronized和重入锁的区别？

synchronized是JVM的内置锁，而重入锁是Java代码实现的。重入锁是synchronized的扩展，可以完全代替后者。重入锁可以重入，允许同一个线程连续多次获得同一把锁。其次，重入锁独有的功能有：

- 可以相应中断，synchronized要么获得锁执行，要么保持等待。而重入锁可以响应中断，使得线程在迟迟得不到锁的情况下，可以不再等待。主要由`lockInterruptibly()`实现，这是一个可以对中断进行响应的锁申请动作，锁中断可以避免死锁。
- 锁的申请可以有等待时限，用`tryLock()`可以实现限时等待，如果超时还未获得锁会返回false，也防止了线程迟迟得不到锁时一直等待，可避免死锁。
- 公平锁，即锁的获得按照线程先来后到的顺序依次获得，不会产生饥饿现象。synchronized的锁默认是不公平的，重入锁可通过传入构造方法的参数实现公平锁。
- 重入锁可以绑定多个Condition条件，这些condition通过调用await/singal实现线程间通信。

#### 4、synchronized作了哪些优化？

synchronized对内置锁引入了偏向锁、轻量级锁、自旋锁、锁消除等优化。使得性能和重入锁差不多了。

- 偏向锁：偏向锁会偏向第一个获得它的线程，如果在接下来的执行过程中，该锁没有被其他线程获取，则持有偏向锁的线程永远也不需要再进行同步。偏向锁是在无竞争的情况下把整个同步都消除掉，CAS操作也没有了。适合于同一个线程请求同一个锁，不适用于不同线程请求同一个锁，此时会造成偏向锁失效。
- 轻量级锁：如果偏向锁失效，虚拟机不会立即挂起线程，会使用一种称为轻量级锁的优化手段，轻量级锁的加锁和解锁都是通过CAS操作完成的。如果线程获得轻量级锁成功，则可以顺利进入临界区。如果轻量级锁加锁失败，表示其他线程抢先得到了锁，轻量级锁将膨胀为重量级锁。
- 自旋锁：锁膨胀后，虚拟机为了避免线程真实地在操作系统层面挂起，虚拟机还会做最后的努力--自旋锁。如果共享数据的锁定状态只有很短的一段时间，为了这段时间去挂起和恢复线程（都需要转入内核态）并不值得，所以此时让后面请求锁的那个线程稍微等待以下，但不放弃处理器的执行时间。这里的等待其实就是执行了一个忙循环，这就是所谓的自旋。虚拟机会让当前线程做几个循环，若干次循环后如果得到了锁，就顺利进入临界区；如果还是没得到，这才将线程在操作系统层面挂起。
- 锁消除：虚拟机即时编译时，对一些代码上要求同步，但被检测到不可能存在共享数据竞争的锁进行消除。锁消除的依据来源于“逃逸分析”技术。堆上的所有数据都不会逃逸出去被其他线程访问到，就可以把它们当栈上的数据对待，认为它们是线程私有的，同步加锁就是没有必要的。

#### 5、Java中线程的创建方式有哪些？

- 继承Thread并重写run方法
- 实现Runnable并重写run方法，然后作为参数传入Thread
- 实现Callable，并重写call()，call方法有返回值。使用FutureTask包装Callable实现类，其中FutureTask实现了Runnable和Future接口，最后将FutureTask作为参数传入Thread中
- 由线程池创建并管理线程。

#### 6、Java中线程池怎么实现的，核心参数讲一讲？

Executors是线程池的工厂类，通过调用它的静态方法如

```java
Executors.newCachedThreadPool();
Executors.newFixedThreadPool(n);
```

可返回一个线程池。这些静态方法统一返回一个`ThreadPoolExecutor`，只是参数不同而已。

```java
public ThreadPoolExecutor(int corePoolSize,
                          int maximumPoolSize,
                          long keepAliveTime,
                          TimeUnit unit,
                          BlockingQueue<Runnable> workQueue,
                          ThreadFactory threadFactory,
                          RejectedExecutionHandler handler) {}
```

包括以上几个参数，其中：

- corePoolSize：指定了线程池中线程的数量；
- maximumPoolSize：线程池中的最大线程数量；
- keepAliveTime：当线程池中线程数量超过corePoolSize时，多余的空闲线程的存活时间；
- unit：上一个参数keepAliveTime的单位
- 任务队列，被提交但还未被执行额任务
- threadFactory：线程工厂，用于创建线程，一般用默认工厂即可。
- handler：拒绝策略。当任务太多来不及处理的时候，采用什么方法拒绝任务。

最重要的是任务队列和拒绝策略。

任务队列主要有ArrayBlockingQueue有界队列、LinkedBlockingQueue无界队列、SynchronousQueue直接提交队列。

使用ArrayBlockingQueue，当线程池中实际线程数小于核心线程数时，直接创建线程执行任务；当大于核心线程数而小于最大线程数时，提交到任务队列中；因为这个队列是有界的，当队列满时，在不大于最大线程的前提下，创建线程执行任务；若大于最大线程数，执行拒绝策略。

使用LinkedBlockingQueue时，当线程池中实际线程数小于核心线程数时，直接创建线程执行任务；当大于核心线程数而小于最大线程数时，提交到任务队列中；因为这个队列是有无界的，所以之后提交的任务都会进入任务队列中。newFixedThreadPool就采用了无界队列，同时指定核心线程和最大线程数一样。

使用SynchronousQueue时，该队列没有容量，对提交任务的不做保存，直接增加新线程来执行任务。newCachedThreadPool使用的是直接提交队列，核心线程数是0，最大线程数是整型的最大值，keepAliveTime是60s，因此当新任务提交时，若没有空闲线程都是新增线程来执行任务，不过由于核心线程数是0，当60s就会回收空闲线程。

当线程池中的线程达到最大线程数时，就要开始执行拒绝策略了。有如下几种

- 直接抛出异常
- 在调用者的线程中，运行当前任务
- 丢弃最老的一个请求，也就是将队列头的任务poll出去
- 默默丢弃无法处理的任务，不做任何处理

#### 7、BIO、NIO、AIO的区别？

首先要搞明白在I/O中的同步、异步、阻塞、非阻塞是什么意思。

- 同步I/O。由用户进程自己处理I/O的读写，处理过程中不能做其他事。需要主动去询问I/O状态。

- 异步I/O。由系统内核完成I/O操作，完成后系统会通知用户进程。

- 阻塞。I/O请求操作需要的条件不满足，请求操作一直等待，直到条件满足。

- 非阻塞。 I/O请求操作需要的条件不满足，会立即返回一个标志，而不会一直等待。

现在来看BIO、NIO、AIO的区别。

**BIO**：同步并阻塞。用户进程在发起一个I/O请求后，必须等待I/O准备就绪，I/O操作也由自己来处理，在IO操作未完成之前，用户进程必须等待。 

**NIO**：同步非阻塞。用户进程发起一个I/O请求后可立即返回去做其他任务，当I/O准备就绪时它会收到通知。接着由这个线程自行进行I/O操作，I/O操作本身还是同步的。

**AIO**：异步非阻塞。用户进程发起一个I/O操作以后可立即返回去做其他任务，真正的I/O操作由内核完成后通知用户进程。

NIO和AIO的不同：NIO是操作系统通知用户进程I/O已经准备就绪，由用户进程自行完成I/O操作；AIO是操作系统完成I/O后通知用户进程。 

BIO是为每一个客户端连接开启一个线程，简单说就是一个连接一个线程。

NIO主要组件有Seletor、Channel、Buffer，数据需要通过BUffer包装后才能使用Channel进行读取和写入。一个Selector可以由一个线程管理，每一个Channel可看作一个客户端连接。一个Selector可以监听多个Channel，即使用一个或极少数的线程来管理大量的客户端连接。当与客户端连接的数据没有准备好时，Selector处于等待状态，一旦某个Channel的准备好了数据，Selector就能立即得到通知。

#### 8、两个线程交替打印奇数和偶数？

先使用synchronized实现。PrintOdd用于打印奇数；PrintEven用于打印偶数。核心就是判断当前count如果是奇数，就让PrintEven阻塞，PrintOdd打印后唤醒在lock对象上等待的PrintEven并且释放锁。此时PrintEven获得锁打印偶数再唤醒PrintOdd，两个线程如此交替唤醒对方就实现了交替打印奇偶数。

```java
public class PrintOddEven {
    private static final Object lock = new Object();
    private static int count = 1;

    static class PrintOdd implements Runnable {
        @Override
        public void run() {
            for (int i = 0; i < 10; i++) {
                synchronized (lock) {
                    try {
                        while ((count & 1) != 1) {
                            lock.wait();
                        }
                        System.out.println(Thread.currentThread().getName() + " " +count);
                        count++;
                        lock.notify();
                    } catch (InterruptedException e) {
                        e.printStackTrace();
                    }
                }
            }
        }
    }

    static class PrintEven implements Runnable {
        @Override
        public void run() {
            for (int i = 0; i < 10; i++) {
                synchronized (lock) {
                    try {
                        while ((count & 1) != 0) {
                            lock.wait();
                        }
                        System.out.println(Thread.currentThread().getName() + " " +count);
                        count++;
                        lock.notify();
                    } catch (InterruptedException e) {
                        e.printStackTrace();
                    }
                }
            }
        }
    }

    public static void main(String[] args) {
        new Thread(new PrintOdd()).start();
        new Thread(new PrintEven()).start();
    }
}
```

如果要实现3个线程交替打印ABC呢？这次打算使用重入锁，和上面没差多少，但是由于现在有三个线程了，在打印完后需要唤醒其他线程，注意不可使用`sigal()`，因为唤醒的线程是随机的，不能保证打印顺序不说，还会造成死循环。一定要使用`sigalAll()`唤醒所有线程。

```java
import java.util.concurrent.locks.Condition;
import java.util.concurrent.locks.ReentrantLock;

public class ThreeThreadPrintABC {
    private static ReentrantLock lock = new ReentrantLock();
    private static Condition wait = lock.newCondition();
    // 用来控制该打印的线程
    private static int count = 0;

    public static void main(String[] args) {
        Thread printA = new Thread(new PrintA());
        Thread printB = new Thread(new PrintB());
        Thread printC = new Thread(new PrintC());
        printA.start();
        printB.start();
        printC.start();

    }

    static class PrintA implements Runnable {
        @Override
        public void run() {
            for (int i = 0; i < 10; i++) {
                lock.lock();
                try {
                    while ((count % 3) != 0) {
                        wait.await();
                    }
                    System.out.println(Thread.currentThread().getName() + " A");
                    count++;
                    wait.signalAll();
                } catch (InterruptedException e) {
                    e.printStackTrace();
                } finally {
                    lock.unlock();
                }
            }
        }
    }

    static class PrintB implements Runnable {
        @Override
        public void run() {
            for (int i = 0; i < 10; i++) {
                lock.lock();
                try {
                    while ((count % 3) != 1) {
                        wait.await();
                    }
                    System.out.println(Thread.currentThread().getName() + " B");
                    count++;
                    wait.signalAll();
                } catch (InterruptedException e) {
                    e.printStackTrace();
                } finally {
                    lock.unlock();
                }
            }
        }
    }

    static class PrintC implements Runnable {
        @Override
        public void run() {
            for (int i = 0; i < 10; i++) {
                lock.lock();
                try {
                    while ((count % 3) != 2) {
                        wait.await();
                    }
                    System.out.println(Thread.currentThread().getName() + " C");
                    count++;
                    wait.signalAll();
                } catch (InterruptedException e) {
                    e.printStackTrace();
                } finally {
                    lock.unlock();
                }
            }
        }
    }
}

```

如果觉得不好理解，重入锁是可以绑定多个条件的。创建3个Condition分别让三个打印线程在上面等待。A打印完了，唤醒等待在waitB对象上的PrintB；B打印完了唤醒在waitC对象上的PrintC；C打印完了，唤醒在waitA对象上等待的PrintA，如此循环地唤醒对方即可。

```java
import java.util.concurrent.locks.Condition;
import java.util.concurrent.locks.ReentrantLock;

public class ThreeThreadPrintABC {
    private static ReentrantLock lock = new ReentrantLock();
    private static Condition waitA = lock.newCondition();
    private static Condition waitB = lock.newCondition();
    private static Condition waitC = lock.newCondition();
    // 用来控制该打印的线程
    private static int count = 0;

    public static void main(String[] args) {
        Thread printA = new Thread(new PrintA());
        Thread printB = new Thread(new PrintB());
        Thread printC = new Thread(new PrintC());
        printA.start();
        printB.start();
        printC.start();

    }

    static class PrintA implements Runnable {
        @Override
        public void run() {
            for (int i = 0; i < 10; i++) {
                lock.lock();
                try {
                    while ((count % 3) != 0) {
                        waitA.await();
                    }
                    System.out.println(Thread.currentThread().getName() + " A");
                    count++;
                    waitB.signal();
                } catch (InterruptedException e) {
                    e.printStackTrace();
                } finally {
                    lock.unlock();
                }
            }
        }
    }

    static class PrintB implements Runnable {
        @Override
        public void run() {
            for (int i = 0; i < 10; i++) {
                lock.lock();
                try {
                    while ((count % 3) != 1) {
                        waitB.await();
                    }
                    System.out.println(Thread.currentThread().getName() + " B");
                    count++;
                    waitC.signal();
                } catch (InterruptedException e) {
                    e.printStackTrace();
                } finally {
                    lock.unlock();
                }
            }
        }
    }

    static class PrintC implements Runnable {
        @Override
        public void run() {
            for (int i = 0; i < 10; i++) {
                lock.lock();
                try {
                    while ((count % 3) != 2) {
                        waitC.await();
                    }
                    System.out.println(Thread.currentThread().getName() + " C");
                    count++;
                    waitA.signal();
                } catch (InterruptedException e) {
                    e.printStackTrace();
                } finally {
                    lock.unlock();
                }
            }
        }
    }
}


```

#### 9、进程间通信的方式？线程间通信的方式？

进程间通信的方式[推荐阅读这篇博客](https://www.cnblogs.com/liugh-wait/p/8533003.html)

- 管道。分为几种管道。普通管道PIPE：单工，单向传输，只能在父子或者兄弟进程间使用；流管道，半双工，可双向传输，只能在父子或兄弟进程间使用；命名管道：可以在许多并不相关的进程之间进行通讯。
- 消息队列。消息队列是由消息的链表，存放在内核中并由消息队列标识符标识。消息队列克服了信号传递信息少、管道只能承载无格式字节流以及缓冲区大小受限等缺点。
- 信号。用于通知接收进程某个事件已经发生 
- 信号量。信号量是一个计数器，可以用来控制多个进程对共享资源的访问。它常作为一种锁机制，防止某进程正在访问共享资源时，其他进程也访问该资源。因此，主要作为进程间以及同一进程内不同线程之间的同步手段。
- 共享内存。共享内存就是映射一段能被其他进程所访问的内存，这段共享内存由一个进程创建，但多个进程都可以访问。共享内存是最快的 IPC（进程间通信） 方式，它往往与其他通信机制如信号量配合使用，来实现进程间的同步和通信。
- 套接字。可用于不同机器间的进程通信。

线程间通信的方式：

[推荐阅读这篇博客](https://www.cnblogs.com/xh0102/p/5710074.html)

- 锁机制。包括互斥锁、条件变量、读写锁。互斥锁以排他方式方式数据被并发修改；读写锁允许多个线程同时读取，对写操作互斥；条件变量可以以原子的方式阻塞进程，直到某个特定条件为真为止。对条件的测试是在互斥锁的保护下进行的。条件变量始终与互斥锁一起使用。 
- 信号量（Semaphore) 机制。包括无名线程信号量和命名线程信号量 
- 信号（Signal）机制。类似进程间的信号处理 

#### 10、原子类比如AtomicInteger为什么能保证原子性？

JDK并发包下有一个atomic包，里面实现了一些直接使用CAS操作的线程安全的类型。AtomicInteger就是其中之一。得益于CAS操作，因此保证了原子性。CAS操作具体说一说是什么？

CAS(Compare And Swap)，即“比较并交换”。CAS基于乐观的态度，是无锁操作，它操作包含三个参数，当前要更新的变量、期望值、新值，仅当：当前值和预期值一样时，才会将当前值设置为新值；如果当前值和预期值不一样，说明这个变量已经被其他线程修改过了。如果有多个线程同时使用CAS操作一个变量时，只有一个会胜出，并成功更新。其他线程允许放弃操作，也允许再次尝试，直到修改成功为止。CAS操作是由硬件支持的，现在的处理器基本支持原子化的CAS指令。

CAS由什么缺点？如何解决？

可能引发"ABA"问题，即一个变量原来是A，先被修改成B后又修改回了A，由于CAS操作只是比较当前值和预期值是否一样（只比较结果，不在乎过程中状态的变化），在其他线程来看，该变量就好像没有发生过变化。

可以为数据添加时间戳，每次成功修改数据时，不仅更新数据的值，同时要更新时间戳的值。CAS操作时，不仅要比较当前值和预期值，还要比较当前时间戳和预期时间戳。两者都必须满足预期值才能修改成功。

#### 11、实现一个简单的线程池？

实现一个类似于`Executors.newFixedThreadPool(n)`的固定大小线程池，当小于corePoolSize时候，优先创建线程去执行该任务；当超过该值时，将任务提交到任务队列中，然后各个线程从任务队列中取任务来执行。

```java
import java.util.HashSet;
import java.util.Set;
import java.util.concurrent.BlockingQueue;
import java.util.concurrent.Executors;
import java.util.concurrent.LinkedBlockingQueue;

public class MyThreadPool {
    private int workerCount;
    private int corePoolSize;
    private BlockingQueue<Runnable> workQueue;
    private Set<Worker> workers;
    private volatile boolean RUNNING = true;
    public MyThreadPool(int corePoolSize) {
        this.corePoolSize = corePoolSize;
        workQueue = new LinkedBlockingQueue<>();
        workers = new HashSet<>();
    }

    public void execute(Runnable r) {
        if (workerCount < corePoolSize) {
            addWorker(r);
        } else {
            try {
                workQueue.put(r);
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
        }
    }

    private void addWorker(Runnable r) {
        workerCount++;
        Worker worker = new Worker(r);
        Thread t = worker.thread;
        workers.add(worker);
        t.start();
    }

    class Worker implements Runnable {
        Runnable task;
        Thread thread;

        public Worker(Runnable task) {
            this.task = task;
            this.thread = new Thread(this);
        }

        @Override
        public void run() {
            while (RUNNING) {
                Runnable task = this.task;
                // 执行当前的任务，所以把这个任务置空，以免造成死循环
                this.task = null;
                if (task != null || (task = getTask()) != null) {
                    task.run();
                }
            }
        }
    }

    private Runnable getTask() {
        Runnable r = null;
        try {
            r = workQueue.take();
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
        return r;
    }


    public static void main(String[] args) {
        MyThreadPool threadPool = new MyThreadPool(5);
        Runnable r = new Writer();
        for (int i = 0; i < 10; i++) {
            threadPool.execute(r);
        }
    }


}

class Writer implements Runnable {

    @Override
    public void run() {
        System.out.println(Thread.currentThread().getName() + " ");
    }
}

```

Worker实现了Runnale，是真正执行任务的类。当线程池中工作线程小于核心线程时候，调用addWorker直接start线程执行它的第一个任务。否则，将任务放入任务队列中，等线程来执行它们。Worker中的run方法是一个死循环，执行第一个任务（addWorker时调用start方法执行的那个任务），或者通过getTask方法不断从任务队列中取得任务来执行。正是getTask方法实现了线程的复用，即一个线程虽然只能调用一次start方法，但是后续的任务可以在Worker的run方法里直接调用任务的run方法得以执行。简单来说就是在Worker的run里调用任务的run方法。

任务全部执行完毕后，线程池需要被关闭，否则程序一直死循环。上述代码中并没有实现`shutdown()`方法。

#### 12、实现生产者-消费者模型？

可以有几种方式实现生产者-消费者模型：

- wait()/notify()

- await()/signal()

- BlockingQueue

生产者-消费者问题的关键在于：

- 没有“产品”时，消费者不能消费
- “产品”线满时，生产者不能生产

如果用队列来存放“产品”：

- 队列为空时，消费者需要一直等待，不为空时消费者才能取走。
- 队列为满时，生产者需要一直等待，不为满时生产者才能进行生产。

等待和唤醒可以使用wait()/notify()实现。Java中的阻塞队列BlockingQueue，其`take()`和和`put()`方法就是阻塞的，内部其实就是await()/signal()方法的配合使用，非常适合作为数据传输的通道。

以ArrayBlockingQueue来说，同步是重入锁保证的。和该lock绑定了两个Condition，一个是notEmpty一个是notFull。简单说明下take和put方法。注意这并不是源码，只是方便理解把核心部分抽取出来。

```java
public E take() {
    lock.lock();
    try {
        // 当队列为空，不能取，必须等待
        while (count==0) {
            notEmpty.await();
        }
        // 不再阻塞说明队列有元素了，直接删除并返回
        return dequeue();
    } finally ｛
        lock.unlock();
	}
}

private void enqueue(E x) {
    // ...insert element
    // 因为插入了元素，说明队列不为空，唤醒在notEmpty上等待的线程
    notEmpty.signal();
}

public void put(E e) {
    lock.lock();
    try {
        // 队列满了，不能放入，必须等待
        while (count == items.length) {
            notFull.await();
        }
        // 此时队列不为满，可以放入
        enqueue(e);
    } finally {
        lock.unlock();
    }
}

private E dequeue() {
    // ...delete element
    // 移除了元素，因而队列不为满，唤醒在notFull上等待的线程
    E x = (E) items[takeIndex];
    notFull.signal();
    return x;
}
```

了解了原理，现在用阻塞队列实现生产者-消费者模型

```java
package easy;

import java.util.Random;

import java.util.concurrent.BlockingQueue;
import java.util.concurrent.ExecutorService;
import java.util.concurrent.Executors;
import java.util.concurrent.LinkedBlockingQueue;

public class ProducerConsumer {
    public static class Producer implements Runnable {
        private BlockingQueue<Integer> blockingQueue;
        private Random random = new Random();

        public Producer(BlockingQueue<Integer> blockingQueue) {
            this.blockingQueue = blockingQueue;
        }

        @Override
        public void run() {
            while (true) {
                Integer a = makeProduct();
                try {
                    blockingQueue.put(a);
                    System.out.println(Thread.currentThread().getName() + "生产了" + a + " 队列大小" + blockingQueue.size());
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
            }
        }

        public Integer makeProduct() {
            try {
                Thread.sleep(1000);
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
            return random.nextInt(10);
        }
    }

    public static class Consumer implements Runnable {
        private BlockingQueue<Integer> blockingQueue;

        public Consumer(BlockingQueue<Integer> blockingQueue) {
            this.blockingQueue = blockingQueue;
        }


        @Override
        public void run() {
            while (true) {
                Integer a = useProduct();
                System.out.println(Thread.currentThread().getName() + "消费产品" + a + " 队列大小" + blockingQueue.size());
            }
        }

        public Integer useProduct() {
            Integer a = null;
            try {
                a = blockingQueue.take();
                Thread.sleep(1000);
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
            return a;
        }
    }

    public static void main(String[] args) {
        BlockingQueue<Integer> blockingQueue = new LinkedBlockingQueue<>(5);
        ExecutorService exec = Executors.newCachedThreadPool();
        int producerNum = 3;
        int consumerNum = 5;

        for (int i = 0; i < producerNum; i++) {
            exec.submit(new Producer(blockingQueue));
        }
        for (int i = 0; i < consumerNum; i++) {
            exec.submit(new Consumer(blockingQueue));
        }

    }
}
```

#### 13、介绍下J.U.C.下的类？

- java.util.concurrent.locks包下的ReentrantLock，即重入锁。可以实现同步，是synchronized的扩展。还有ReadWriteLock即读写锁，读-读非阻塞，读和读之间可并行，因此适合读多写少的场合。
- ConcurrentHashMap，线程安全的HashMap；
- CopyOnWriteArrayList，核心是在写入操作时，先用一个副本复制原数组，然后新值写入到副本中，写入完成后再将修改完的副本替换掉原来的数组。这种实现使得写入操作也不会阻塞读操作了，只有写-写会同步等待。适合读多写少的场合。
- 阻塞队列BlockingQueue，有基于数组实现的有界队列ArrayBlockingQueue，还有基于链表实现的无界队列LinkedBlockingQueue。
- ConcurrentLinkedQueue，高效地并发队列。
- 信号量Semaphore，允许多个线程同时访问，synchronized和重入锁都只允许在一个时刻只有一个线程可以进入临界区访问共享资源，而信号量允许多个线程同时访问某一个资源。
- 倒计时器CountDownLatch，给倒计时器设定一个计数个数，每完成一个任务计数减1，某一个线程等待在倒计时器上，当计数完毕后才能继续该线程的执行。强调一个线程等待其他线程执行完成后才能继续执行。
- 循环栅栏CyclicBarrier，和CountDownLatch比较类似，可传入一个Runnable，该计数器可循环使用，每次计数完成会执行该Runnable。更强调线程之间的互相等待，必须所有线程等准备完毕（一次计数完成），才能执行某个任务。
- 跳表ConcurrentSkipListMap，跳表是一个多层的链表结构，最下层拥有全部的键值数据，越往上越少；查找时从顶层开始查找，在本层没找到转到下一层接着查找，用于快速查找，而且跳表中的数据是已排序的。
- 线程池，Executor、Executors、EexcutorService、ThreadPoolExecutor等。Executors是一个线程池工厂，其静态方法通过返回new ThreadPool，以EexcutorService接收，而EexcutorService和ThreadPoolExecutor都是Executor的实现类。
- Fork/Join框架，如ForkJoinPool线程池
- atomic包下的各种原子类，如AtomicInteger，主要使用了CAS操作实现无锁
- LockSupport类，线程阻塞工具类，主要有park和unpark方法表示线程的挂起和唤醒，

#### 14、读写锁用过没？

ReadWriteLock即读写锁，它有两个方法如下，分别返回一个读锁和写锁，即读写锁分离。

```java
ReadWriteLock readWriteLock = new ReentrantReadWriteLock();
Lock readLock = readWriteLock.readLock();
Lock writeLock = readWriteLock.writeLock();
```

在读时使用readLock进行加锁，在写时使用writeLock进行加锁。使得读-读不阻塞，读线程完全并行，适合读多写少的场合。

#### 15、自旋锁是什么，为什么要用自旋锁？自选锁的缺点？

指当一个线程在获取锁的时候，锁已经被其它线程获取，但是有可能锁的状态只会持续很短的一段时间，为此将线程挂起、恢复并不值得，因为线程的挂起和恢复都需要转入到内核态。系统假定未请求到锁的线程在不久之后就能获得这锁，于是让后面请求锁的那个线程执行一个忙循环，然后不断的判断锁是否能够被成功获取，直到获取到锁才会退出循环 。这就是自旋锁。

自旋锁的好处：不会使线程状态发生改变，即一直处于用户态，不会转入内核态（用户态和内核态的切换系统开销很大）。不会使线程进入阻塞状态，减少了不必要的上下文切换，执行速度快。

自旋锁的缺点：一直占用CPU时间，如果锁被占用时间很短，自旋等待效果就很好，如果锁占用时间太长，自旋的线程只会白白消耗CPU资源。

后来引入了自适应自旋锁，自旋时间不再是固定的了，由上一次在同一个锁上的自旋时间和锁的拥有者的状态决定。如果在同一个锁对象上，自旋等待刚刚成功获得过锁，且持有锁的线程正在运行中，虚拟机认为这次自旋也会成功，而且它将允许自旋等待持续更长的时间；相反，如果对于某个锁，自旋很少成功获得过，在以后要获取这个锁时可能就会省略自旋过程。

#### 16、进程和线程的区别？

进程是资源分配的最小单位，线程是程序执行的最小单位。 进程是线程的容器，即进程里面可以容纳多个线程，多个线程之间可以共享数据。

#### 17、线程的死锁指什么？如何检测死锁？如何解决死锁？

是指两个或两个以上的线程在执行过程中，互相占用着对方想要的资源但都不释放，造成了互相等待，结果线程都无法向前推进。

死锁的检测：可以采用等待图（wait-for gragh）。采用深度优先搜索的算法实现，如果图中有环路就说明存在死锁。

解决死锁：

- 破环锁的四个必要条件之一，可以预防死锁。
- 加锁顺序保持一致。不同的加锁顺序很可能导致死锁，比如哲学家问题：A先申请筷子1在申请筷子2，而B先申请筷子2在申请筷子1，最后谁也得不到一双筷子（同时拥有筷子1和筷子2）
- 撤消或挂起进程，剥夺资源。终止参与死锁的进程，收回它们占有的资源，从而解除死锁。 

#### 18、CPU线程调度？

- **协同式线程调度**：线程的执行时间以及线程的切换都是由线程本身来控制，线程把自己的任务执行完后，主动通知系统切换到另一个线程。优点是没有线程安全的问题，缺点是线程执行的时间不可控，可能因为某一个线程不让出CPU，而导致整个程序被阻塞。

- **抢占式调度模式**：线程的执行时间和切换都是由系统来分配和控制的。不过可以通过设置线程优先级，让优先级高的线程优先占用CPU。 

  Java虚拟机默认采用抢占式调度模型。

#### 19、HashMap在多线程下有可能出现什么问题？

- JDK8之前，并发put下可能造成死循环。原因是多线程下单链表的数据结构被破环，指向混乱，造成了链表成环。JDK 8中对HashMap做了大量优化，已经不存在这个问题。
- 并发put，有可能造成键值对的丢失，如果两个线程同时读取到当前node，在链表尾部插入，先插入的线程是无效的，会被后面的线程覆盖掉。

#### 20、ConcurrentHashMap是如何保证线程安全的？

JDK 7中使用的是分段锁，内部分成了16个Segment即分段，每个分段可以看作是一个小型的HashMap，每次put只会锁定一个分段，降低了锁的粒度：

- 首先根据key计算出一个hash值，找到对应的Segment
- 调用Segment的lock方法（Segment继承了重入锁），锁住该段内的数据，所以并没有锁住ConcurrentHashMap的全部数据
- 根据key计算出hash值，找到Segment中数组中对应下标的链表，并将该数据放置到该链表中
- 判断当前Segment包含元素的数量大于阈值，则Segment进行扩容（Segment的个数是不能扩容的，但是单个Segment里面的数组是可以扩容的）

多线程put的时候，只要被加入的键值不属于 同一个分段，就可以做到真正的并行put。**对不同的Segment则无需考虑线程同步，对于同一个Segment的操作才需考虑。**

JDK 8中使用了CAS+synchronized保证线程安全，也采取了数组+链表/红黑树的结构。 

put时使用synchronized锁住了桶中链表的头结点。

数组的扩容，被问到了我在看吧.....我只知道多个线程可以协助数据的迁移。

有这么一个问题，ConcurrentHashMap，有三个线程，A先put触发了扩容，扩容时间很长，此时B也put会怎么样？此时C调用get方法会怎么样？C读取到的元素是旧桶中的元素还是新桶中的

A先触发扩容，ConcurrentHashMap迁移是在锁定旧桶的前提下进行迁移的，并没有去锁定新桶。

- 在某个桶的迁移过程中，别的线程想要对该桶进行put操作怎么办？一旦某个桶在迁移过程中了，必然要获取该桶的锁，所以其他线程的put操作要被阻塞。**因此B被阻塞**。
- 某个桶已经迁移完成（其他桶还未完成），别的线程想要对该桶进行put操作怎么办？该线程会首先检查是否还有未分配的迁移任务，如果有则先去执行迁移任务，如果没有即全部任务已经分发出去了，那么此时该线程可以直接对新的桶进行插入操作（映射到的新桶必然已经完成了迁移，所以可以放心执行操作）

ConcurrentHashMap的get操作没有加锁，所以可以读取到值，不过是旧桶中的值。

```java
if (finishing) {
    nextTable = null;
    table = nextTab;
    sizeCtl = (n << 1) - (n >>> 1);
    return;
}
```

从table = nextTable可以看出，当所有数据迁移完成时，才将用nextTab新数组去覆盖旧数组table。所以在A扩容过程中，**C读取到的是旧数组中的元素**。

#### 21、ThreadLocal的作用和实现原理？

对于共享变量，一般采取同步的方式保证线程安全。而ThreadLocal是为每一个线程都提供了一个线程内的局部变量，每个线程只能访问到属于它的副本。

实现原理，下面是set和get的实现

 ```java
// set方法
public void set(T value) {
    Thread t = Thread.currentThread();
    ThreadLocalMap map = getMap(t);
    if (map != null)
        map.set(this, value);
    else
        createMap(t, value);
}

// 上面的getMap方法
ThreadLocalMap getMap(Thread t) {
    return t.threadLocals;
}

// get方法
public T get() {
    Thread t = Thread.currentThread();
    ThreadLocalMap map = getMap(t);
    if (map != null) {
        ThreadLocalMap.Entry e = map.getEntry(this);
        if (e != null) {
            @SuppressWarnings("unchecked")
            T result = (T)e.value;
            return result;
        }
    }
    return setInitialValue();
}
 ```

从源码中可以看出：每一个线程拥有一个ThreadLocalMap，这个map存储了该线程拥有的所有局部变量。

set时先通过Thread.currentThread()获取当前线程，进而获取到当前线程的ThreadLocalMap，然后以ThreadLocal自己为key，要存储的对象为值，存到当前线程的ThreadLocalMap中。

get时也是先获得当前线程的ThreadLocalMap，以ThreadLocal自己为key，取出和该线程的局部变量。

题话外，一个线程内可以设置多个ThreadLocal，这样该线程就拥有了多个局部变量。比如当前线程为t1，在t1内创建了两个ThreadLocal分别是tl1和tl2，那么t1的ThreadLocalMap就有两个键值对。

```java
t1.threadLocals.set(tl1, obj1) // 等价于在t1线程中调用tl1.set(obj1)
t1.threadLocals.set(tl2, obj2) // 等价于在t1线程中调用tl2.set(obj1)

t1.threadLocals.getEntry(tl1) // 等价于在t1线程中调用tl1.get()获得obj1
t1.threadLocals.getEntry(tl2) // 等价于在t1线程中调用tl2.get()获得obj2
```

#### 22、ArrayBlockingQueue和LinkedBlockingQueue的区别？

- ArrayBlockingQueue基于数组，是有界的阻塞队列，初始化时需要指定容量且不可扩容；LinkedBlockingQueue基于链表，是无界的阻塞队列，容量无限制。
- ArrayBlockingQueue读写共用一把锁，因此put和take是互相阻塞的；而LinkedBlockingQueue使用了两把锁，一把putLock和一把takeLock，实现了锁分离，使得put和take写数据和读数据可以并发的进行。

#### 23、CountDownLatch和CyclicBarrier的区别？

- CountDownLatch强调一个线程等待其他所有线程，通过cdl.await()让当前线程等待在倒计数器上，每有一个线程执行完，cdl.countDown()，将计数减1，减到0时通知当前线程执行。简单的说就是一个线程等待，直到他所等待的其他线程都执行完成，当前线程才可以继续执行。
- cyclicBarrier强调线程之间互相等待，只要有一个线程还没到来，所有线程会一起等待。可以传入一个Runnable作为计数完成要执行的任务。每有一个线程调用cyc.await()计数减1，减到0时会执行一次该Runnable。简单地说就是线程之间互相等待，等所有线程都准备好，即调用await()方法之后，执行一次Runnable，此时所有线程开始同时执行！
- CountDownLatch的计数器只能使用一次。而CyclicBarrier的计数器可以使用reset() 方法重置。

#### 24、synchronized内部实现原理？

推荐阅读下面两篇博客。

[倔强中的小白](https://blog.csdn.net/tryingpfq/article/details/82115612)

[zejian_的博客](https://blog.csdn.net/javazejian/article/details/72828483)

对同步方法，JVM采用`ACC_SYNCHRONIZED`标记符来实现同步。 对于同步代码块。JVM采用`monitorenter`、`monitorexit`两个指令来实现同步。

同步方法通过`ACC_SYNCHRONIZED`关键字隐式的对方法进行加锁。当线程要执行的方法被标注上`ACC_SYNCHRONIZED`时，需要先获得锁才能执行该方法。

同步代码块通过`monitorenter`和`monitorexit`执行来进行加锁。当线程执行到`monitorenter`的时候要先获得所锁，才能执行后面的方法。当线程执行到`monitorexit`的时候则要释放锁。

每个对象自身维护这一个被加锁次数的计数器，当计数器数字为0时表示可以被任意线程获得锁。当计数器不为0时，只有获得锁的线程才能再次获得锁，即可重入锁。换句话说，一个线程获取到锁之后可以无限次地进入该临界区。

Synchronized原子性

原子性是指一个操作是不可中断的，要全部执行完成，要不就都不执行。

在Java中，为了保证原子性，提供了两个高级的字节码指令`monitorenter`和`monitorexit`。而这两个字节码指令，在Java中对应的关键字就是`synchronized`。通过`monitorenter`和`monitorexit`指令，可以保证被`synchronized`修饰的代码在同一时间只能被一个线程访问，在锁未释放之前，无法被其他线程访问到。因此，在Java中可以使用`synchronized`来保证方法和代码块内的操作是原子性的。

Synchronized可见性

可见性是指当多个线程访问同一个变量时，一个线程修改了这个变量的值，其他线程能够立即看得到修改的值。`synchronized`修饰的代码，在开始执行时会加锁，执行完成后会进行解锁。而为了保证可见性，有一条规则是这样的：对一个变量解锁之前，必须先把此变量同步回主存中。这样解锁后，后续线程就可以访问到被修改后的值。

Synchronized有序性

有序性即程序执行的顺序按照代码的先后顺序执行。由于`synchronized`修饰的代码，同一时间只能被同一线程访问。那么也就是单线程执行的。所以，可以保证其有序性。

#### 25、什么叫做锁的可重入？

同一个线程可以多次获得同一个锁，即一个线程获取到锁之后可以无限次地进入该临界区 (对于ReentrantLock来说，通过调用`lock.lock()`)；当然锁的释放也需要相同次数的unlock()操作。注意：除了ReentrantLock，synchronized的锁也是可重入的。

#### 26、Java线程生命周期的状态？

```java
public enum State {
    NEW,
    RUNNABLE,
    BLOCKED,
    WAITING,
    TIMED_WATING,
    TERMINATED;
}
```

线程的所有状态都在枚举类State中定义了，其中：

NEW表示刚刚创建的线程，此时线程还没有开始执行。调用`start()`后线程进入RUNNABLE状态，线程在执行过程中遇到synchronized同步块，就进入BLOCKED阻塞状态，此时线程暂停执行，直到获得请求的锁。WAITING和TIMED_WAITING都表示等待，区别是前者是无时间限制的等待，后者是有时限的等待。等待可以是执行`wait()`方法后等待`notify()`方法将其唤醒，也可以是通过`join()`方法等待的线程等待目标线程的执行结束。一旦等待了期望事件，线程再次执行，从等待状态变成RUNNABLE状态。线程执行结束后，进入TERMINATED状态。

#### 27、被notify()唤醒的线程可以立即得到执行吗？

被notify唤醒的线程不是立刻可以得到执行的，因为`notify()`不会立刻释放锁，`wait()`状态的线程也不能立刻获得锁；等到执行`notify()`的线程退出同步块后，才释放锁，此时其他处于`wait()`状态的线程才能获得该锁。

#### 28、ThreadLocal的应用场景？

数据库连接

```java
private static ThreadLocal<Connection> connectionHolder = new ThreadLocal<Connection>() {  
    public Connection initialValue() {  
        return DriverManager.getConnection(DB_URL);  
    }  
};  
  
public static Connection getConnection() {  
    return connectionHolder.get();  
}  
```

Session管理

```java
private static final ThreadLocal threadSession = new ThreadLocal();  
  
public static Session getSession() throws InfrastructureException {  
    Session s = (Session) threadSession.get();  
    try {  
        if (s == null) {  
            s = getSessionFactory().openSession();  
            threadSession.set(s);  
        }  
    } catch (HibernateException ex) {  
        throw new InfrastructureException(ex);  
    }  
    return s;  
}  
```

#### 29、sleep、wait、yield的区别和联系？

sleep() 允许指定以毫秒为单位的一段时间作为参数，它使得线程在指定的时间内进入阻塞状态，不能得到CPU 时间，指定的时间一过，线程重新进入可执行状态。调用sleep后不会释放锁。

yield() 使得线程放弃CPU执行时间，但是不使线程阻塞，线程从运行状态进入就绪状态，随时可能再次分得 CPU 时间。有可能当某个线程调用了yield()方法暂停之后进入就绪状态，它又马上抢占了CPU的执行权，继续执行。

wait()是Object的方法，会使线程进入阻塞状态，和sleep不同，wait会同时释放锁。wait/notify在调用之前必须先获得对象的锁。

#### 30、Thread类中的start和run方法区别？

run方法只是一个普通方法调用，还是在调用它的线程里执行。

start才是开启线程的方法，run方法里面的逻辑会在新开的线程中执行。

## JVM

#### 1、Java内存区域（注意不是Java内存模型JMM）的划分？

- 程序计数器。
- 虚拟机栈。
- 本地方法栈。
- Java堆。
- 方法区。

前三个是线程私有的，后两个是线程共享的。

字节码解释器通过改变程序计数器的值来决定下一条要执行的指令，为了在线程切换后每条线程都能正确回到上次执行的位置，因为每条线程都有自己的程序计数器。

虚拟机栈是存放Java方法内存模型，每个方法在执行时都会创建一个栈帧，用于存储局部变量表、操作数栈、动态链接、方法返回地址等信息。方法的开始调用对应着栈帧的进栈，方法执行完成对应这栈帧的出栈。位于栈顶被称为“当前方法”。

本地方法栈和虚拟机栈类似，不过虚拟机栈针对Java方法，而本地方法栈针对Native方法。

Java堆。对象实例被分配内存的地方，也是垃圾回收的主要区域。

方法区。存放被虚拟机加载的**类信息、常量（final）、静态变量（static）、即时编译期编译后的代码**。方法区是用永久代实现的，这个区域的内存回收目标主要是针对常量池的回收和类型的卸载。**运行时常量池是方法区的一部分**，运行时常量池是Class文件中的一项信息，存放编译期生成的各种字面量和符号引用。

#### 2、新生代和老年代。对象如何进入老年代，新生代怎么变成老年代？

Java堆分为新生代和老年代。在新生代又被划分为Eden区，From Sruvivor和To Survivor区，比例是8:1:1，所以新生代可用空间其实只有其容量的90%。对象优先被分配在Eden区。

- 不过大对象比如长字符串、数组由于需要大量连续的内存空间，所以直接进入老年代。这是对象进入老年代的一种方式，
- 还有就是长期存活的对象会进入老年代。在Eden区出生的对象经过一次Minor GC会若存活，且Survivor区容纳得下，就会进入Survivor区且对象年龄加1，当对象年龄达到一定的值，就会进入老年代。
- 在上述情况中，若Survivor区不能容纳存活的对象，则会通过分配担保机制转移到老年代。
- 同年龄的对象达到suivivor空间的一半，大于等于该年龄的对象会直接进入老年代。

#### 3、新生代的GC和老年代的GC？

发生在新生代的GC称为Minor GC，当Eden区被占满了而又需要分配内存时，会发生一次Minor GC，一般使用复制算法，将Eden和From Survivor区中还存活的对象一起复制到To Survivor区中，然后一次性清理掉Eden和From Survivor中的内存，使用复制算法不会产生碎片。

老年代的GC称为Full GC或者Major GC：

- 当老年代的内存占满而又需要分配内存时，会发起Full GC
- 调用System.gc()时，可能会发生Full GC，并不保证一定会执行。
- 在Minor GC后survivor区放不下，通过担保机制进入老年代的对象比老年代的内存空间还大，会发生Full GC；
- 在发生Minor GC之前，会先比较历次晋升到老年代的对象平均年龄，如果大于老年代的内存，也会触发Full GC。如果不允许担保失败，直接Full GC。

#### 4、对象在什么时候可以被回收，调用finalize方法后一定会被回收吗？

在经过可达性分析后，到GC Roots不可达的对象可以被回收（但并不是一定会被回收，至少要经过两次标记），此时对象被第一次标记，并进行一次判断：

- 如果该对象没有调用过或者没有重写finalize()方法，那么在第二次标记后可以被回收了；
- 否则，该对象会进入一个FQueue中，稍后由JVM建立的一个Finalizer线程中去执行回收，此时若对象中finalize中“自救”，即和引用链上的任意一个对象建立引用关系，到GC Roots又可达了，在第二次标记时它会被移除“即将回收”的集合；如果finalize中没有逃脱，那就面临被回收。

因此finalize方法被调用后，对象不一定会被回收。

#### 5、哪些对象可以作为GC Roots？

- 虚拟机栈中引用的对象
- 方法区中类静态属性引用的对象（static）
- 方法区中常量引用的对象（final）
- 本地方法栈中引用的对象

#### 6、讲一讲垃圾回收算法？

- 复制算法，一般用于新生代的垃圾回收
- 标记清除， 一般用于老年代的垃圾回收
- 标记整理，一般用于老年代的垃圾回收
- 分代收集：根据对象存活周期的不同把Java堆分为新生代和老年代。新生代中又分为Eden区、from survivor区和to survivor区，默认8:1:1，对象默认创建在Eden区，每次垃圾收集时新生代都会有大量对象死亡。此时利用复制算法将Eden区和from survivor区还存活的对象一并复制到tosurvivor区。老年代的对象存活率高，没有额外空间进行分配担保，因此采用标记-清除或者标记-整理的算法进行回收。前者会产生空间碎片，而后者不会。

#### 7、介绍下类加载器和类加载过程？

**先说类加载器**。

在Java中，系统提供了三种类加载器。

- 启动类加载器（Bootstrap ClassLoader），启动类加载器无法被Java程序直接引用，用户在编写自定义类加载器时，如果需要委派给启动类加载器，直接使用null。
- 扩展类加载器（Extension ClassLoader）
- 应用程序类加载器（Application ClassLoader），负责加载用户类路径（ClassPath）上锁指定的类库。是程序中默认的类加载器。

当然用户也可以自定义类加载器。

**再说类加载的过程**。

主要是以下几个过程：

```html
加载 -> 验证 -> 准备 -> 解析 -> 初始化 -> 使用 -> 卸载
```

**加载**

- 通过一个类的全限定名获取定义该类的二进制字节流
- 将字节流表示的静态存储结构转化为方法区的运行时数据结构
- 在内存中生成这个类的Class对象，作为方法区这个类的各种数据的访问入口

**验证**

- 文件格式验证：比如检查是否以魔数0xCAFEBABE开头
- 元数据验证：对类的元数据信息进行语义校验，保证不存在不符合Java语言规范的元数据信息。比如检查该类是否继承了被final修饰的类。
- 字节码验证，通过数据流和控制流的分析，验证程序语义是合法的、符合逻辑的。

**准备**。
为类变量（static）分配内存并设置默认值。比如static int a = 123在准备阶段的默认值是0，但是如果有final修饰，在准备阶段就会被赋值为123了。

**解析**。

将常量池中的符号引用替换成直接引用的过程。包括类或接口、字段、类方法、接口方法的解析。

**初始化**。

按照程序员的计划初始化类变量。如static int a = 123，在准备阶段a的值被设置为默认的0，而到了初始化阶段其值被设置为123。

#### 8、什么是双亲委派模型，有什么好处？如何打破双亲委派模型？

类加载器之间满足双亲委派模型，即：除了顶层的启动类加载器外，其他所有类加载器都必须要自己的父类加载器。当一个类加载器收到类加载请求时，自己首先不会去加载这个类，而是不断把这个请求委派给父类加载器完成，因此所有的加载请求最终都传递给了顶层的启动类加载器。只有当父类无法完成这个加载请求时，子类加载器才会尝试自己去加载。

双亲委派模型的好处？使得Java的类随着它的类加载器一起具备了一种带有优先级的层次关系。Java的Object类是所有类的父类，因此无论哪个类加载器都会加载这个类，因为双亲委派模型，所有的加载请求都委派给了顶层的启动类加载器进行加载。所以Object类在任何类加载器环境中都是同一个类。

如何打破双亲委派模型？使用OSGi可以打破。*OSGI*(Open Services Gateway Initiative)，或者通俗点说JAVA动态模块系统。可以实现代码热替换、模块热部署。在OSGi环境下，类加载器不再是双亲委派模型中的树状结构，而是进一步发展为更加复杂的网状结构。

#### 9、说一说CMS和G1垃圾收集器？各有什么特点。

CMS(Concurrent Mark Sweep) 从名字可以看出是可以进行并发标记-清除的垃圾收集器。针对老年代的垃圾收集器，目的是尽可能地减少用户线程的停顿时间。

收集过程有如下几个步骤：

- 初始标记：标记从GC Roots能直接关联到的对象，会暂停用户线程
- 并发标记：即在堆中堆对象进行可达性分析，从GC Roots开始找出存活的对象，可以和用户线程一起进行
- 重新标记：修正并发标记期间因用户程序继续运作导致标记产生变动的对象的标记记录
- 并发清除：并发清除标记阶段中确定为不可达的对象

CMS的缺点：

- 由于是基于标记-清除算法，所以会产生空间碎片
- 无法处理浮动垃圾，即在清理期间由于用户线程还在运行，还会持续产生垃圾，而这部分垃圾还没有被标记，在本次无法进行回收。
- 对CPU资源敏感

CMS比较类似适合用户交互的场景，可以获得较小的响应时间。

G1(Garbage First)，有如下特点：

- 并行与并发
- 分代收集
- 空间整合 ：整体上看是“标记-整理”算法，局部（两个Region之间 ）看是复制算法。确保其不会产生空间碎片。（这是和CMS的区别之一）
- 可预测的停顿：G1除了追求低停顿外，还能建立可预测的时间模型，主要原因是它可以有计划地避免在整个Java堆中进行全区域的垃圾收集。

在使用G1收集器时，Java堆的内存划分为多个大小相等的独立区域，新生代和老年代不再是物理隔离。G1跟踪各个区域的垃圾堆积的价值大小，在后台维护一个优先列表，每次根据允许的收集时间，优先回收价值最大的区域。

G1的收集过程和CMS有些类似：

- 初始标记：标记与GC Roots直接关联的对象，会暂停用户线程（Stop the World）
- 并发标记：并发从GC Roots开始找出存活的对象，可以和用户线程一起进行
- 最终标记：修正并发标记期间因用户程序继续运作导致标记产生变动的对象的标记记录
- 筛选回收：清除标记阶段中确定为不可达的对象，具体来说对各个区域的回收价值和成本进行排序，根据用户所期望的GC停顿时间来制定回收计划。

G1的优势：可预测的停顿；实时性较强，大幅减少了长时间的gc；一定程度的高吞吐量。

#### 10、CMS和G1的区别？

由上一个问题可总结出CMS和G1的区别：

- G1堆的内存布局和其他垃圾收集器不同，它将整个Java堆划分成多个大小相等的独立区域(Region)。G1依然保留了分代收集，但是新生代和老年代不再是物理隔离的，它们都属于一部分Region的集合，因此仅使用G1就可以管理整个堆。
- CMS基于标记-清除，会产生空间碎片；G1从整体看是标记-整理，从局部（两个Region之间）看是复制算法，不会产生空间碎片。
- G1能实现可预测的停顿。

#### 11、GC一定会导致停顿吗，为什么一定要停顿？任意时候都可以GC吗还是在特定的时候？

GC进行时必须暂停所有Java执行线程，这被称为Stop The World。为什么要停顿呢？因为可达性分析过程中不允许对象的引用关系还在变化，否则可达性分析的准确性就无法得到保证。所以需要STW以保证可达性分析的正确性。

程序执行时并非在所有地方都能停顿下来开始GC，只有在“安全点”才能暂停。安全点指的是：HotSpot没有为每一条指令都生成OopMap（Ordinary Object Pointer），而是在一些特定的位置记录了这些信息。这些位置就叫安全点。

## 数据库

#### 1、数据库设计的三大范式？

- 第一范式1NF: 数据表中的每一列(字段)，必须是不可拆分的最小单元，也就是确保每一列的原子性。如订单信息列为orderInfo = "DD1024 2018.5.18"，必须拆分为orderId和orderTime。
- 第二范式2NF: 在满足第一范式的基础上，表中的所有列都必需依赖于主键（和主键有关系），其他和主键没有关系的列可以拆分出去。通俗点说就是：一个表只描述一件事情。比如order表中有orderId、orderTime、userId和userName，只有前两列依赖于订单表，后两列需要拆分到user表中。
- 第三范式3NF: 在满足第二范式的基础上，要求数据不能有传递关系。表中的每一列都要与主键直接相关，而不是间接相关（表中的每一列只能依赖于主键）。比如order表中有orderId、orderTime、userId和userName，根据orderId可以查出userId，根据userId又可以查出userName，这就是数据的传递性，完全可以只留下userId这一列。

#### 2、MySql的事务隔离级别？推荐使用哪种？

- 读未提交
- 读已提交
- 可重复读
- 串行化

在具体解释上面的四个隔离级别前。有必要了解事务的**四大特性（ACID）**

[推荐阅读这篇博客](https://www.cnblogs.com/huanongying/p/7021555.html)

- 原子性（Atomicity）：事务是一个不可分割的工作单位，事务中的操作要么都发生，要么都不发生。
- 一致性（Consistency）：事务开始前和结束后，数据的完整性约束没有被破环。比如A向B转了钱，转账前后钱的总数不变。
- 隔离性（Isolation）：多个用户并发访问数据数据库时，一个用户的事务不能被其他用户的事务所干扰，多个并发事务之间的数据相互隔离。比如事务A和事务B都修改同一条记录，这条记录就会被重复修改或者后者会覆盖前者的修改记录。
- 持久性（Durability）：事务完成后，事务对数据库的更新被保存到数据库，其结果是永久的。

事务并发可能产生的问题：
脏数据：事务对缓冲池中的行记录进行修改，但是还没有被提交。

- 脏读：事务A读取到了事务B修改但未提交的数据。如果此时B回滚到修改之前的状态，A就读到了脏数据。
- 不可重复读：事务A多次读取同一个数据，此时事务B在A读取过程中对数据修改并提交了，导致事务A在同一个事务中多次读取同一数据而结果不同。
- 幻读：事务A对表进行修改，这个修改涉及到表中所有的行，但此时事务B新插入了一条数据，事务A就会发现居然还有数据没有被修改，就好像发生幻觉一样。

脏读是读取到事务未提交的数据，不可重复度读读取到的是提交提交后的数据，只不过在一次事务中读取结果不一样。

不可重复读侧重于修改，幻读侧重于新增或删除。解决不可重复读的问题只需锁住满足条件的行，解决幻读需要锁表。

![821187-20160811171241606-133220585](http://picmeup.oss-cn-hangzhou.aliyuncs.com/coding/821187-20160811171241606-133220585.jpg)

一般来说，数据库隔离级别不一样，可能出现的并发问题也不同。级别最高的是串行化，所有问题都不会出现。但是在并发下性能极低，可重复读会只会导致幻读。

所以一般使用MySQL默认的可重复读即可。MVCC（多版本并发控制）使用undo_log使得事务可以读取到数据的快照（某个历史版本），从而实现了可重复读。MySQL采用Next-Key Lock算法，对于索引的扫描不仅是锁住扫描到的索引，还锁住了这些索引覆盖的范围，避免了不可重复读和幻读的产生。

#### 3、MySql数据库在什么情况下出现死锁？产生死锁的四个必要条件？如何解决死锁？

死锁是指两个或两个以上的事务在执行过程中，因争夺锁资源而造成的一种互相等待的现象，若无外力作用两个事务都无法推进，这样就产生了死锁。下去 死锁的四个必要条件：

- 互斥条件：即任何时刻，一个资源只能被一个进程使用。其他进程必须等待。
- 请求和保持条件：即当资源请求者在请求其他的资源的同时保持对原有资源的占有且不释放。 
- 不剥夺条件：资源请求者不能强制从资源占有者手中夺取资源，资源只能由资源占有者主动释放。 
- 环路等待条件：比如A占有B在等待的资源（B等待A释放），B占有A在等待的资源（A等待B释放）。多个进程循环等待着相邻进程占用着的资源。

避免死锁可以通过破环四个必要条件之一。

解决死锁的方法：

- 加锁顺序保持一致。不同的加锁顺序很可能导致死锁，比如哲学家问题：A先申请筷子1在申请筷子2，而B先申请筷子2在申请筷子1，最后谁也得不到一双筷子（同时拥有筷子1和筷子2）
- 超时，为其中一个事务设置等待时间，若超过这个阈值事务就回滚，另一个等待的事务就能得以继续执行。
- 及时检测出死锁，回滚undo量最小的事务。一般是采用等待图（wait-for gragh）。采用深度优先搜索的算法实现，如果图中有环路就说明存在死锁。

#### 4、现在发现sql查询很慢，如何分析哪里出了问题，应该如何优化？

开启慢查询，查找哪些sql语句执行得慢。使用explain查看语句的执行计划，比如有没有使用到索引，是否启用了全表扫描等。查询慢，很大可能是因为没有使用索引或者索引没有被命中。还有其他的原因，比如发生了死锁，硬件、网速等原因。

优化手段：为相关列添加索引，并且确保索引可以被命中。优化sql语句的编写。

#### 5、索引的好处？

索引是对数据库表中一个或多个列的值进行排序的结构。MySql中索引是B+树，在查找时可以利用二分查找等高效率的查找方式，以O(lg n)的时间找到。因此索引可以加快查询速度。

#### 6、哪些情况需要建立索引？

- 在经常要搜索的列上
- 经常出现在where后面的列上
- 在作为主键的列上
- 作为外键的列上
- 经常需要排序、分组和联合操作的字段建立索引

哪些情况不适合建立索引？

- 查询中很少使用的字段
- 数值太少的字段
- 唯一性不太差的字段
- 更新频繁的字段
- 不会出现在where后的字段
- 索引适合建立在小字段上，text和blob等大字段不适合建立索引

#### 7、索引的最左匹配原则了解吗？

建了一个(a,b,c)的联合索引，那么实际等于建了(a),(a,b),(a,b,c)三个索引，但是有时在条件查询时只会匹配到a或者(a, b)而不会匹配到(a, b, c)。下面的例子

```sql
SELECT * FROM table WHERE a = 1 AND c = 3; // 使用了索引a，c不走索引
SELECT * FROM table WHERE a = 1 AND b < 2 AND c = 3; // 使用到了索引(a,b)，c不走索引
```

建立联合索引(a, b ,c)，所以索引是按照a -> b -> c的顺序进行排序的。a-b-c这样的索引是先找a，然后在范围里面找b，再在范围内找c。 所以上面的语句里的c 会分散在很多个b里面且不是排序的，所以没办法走索引。

举个例子比如(a, b)联合索引，先按a排序再按b排序，得到

```sql
(1,1)->(1, 2)->(2, 1)  (2, 4)->(3, 1)->(3, 2)
```

如果执行`select a from table where b=2`，就没有使用到(a, b)这个联合索引，因为b的值1,2,1,4,1,2显然不是排序的。

具体来说：MySQL会从左开始一直向右匹配直到遇到范围查询（>,<,BETWEEN,LIKE）就停止匹配，比如： a = 1 AND b = 2 AND c > 3 AND d = 4，如果建立 （a,b,c,d）顺序的索引，使用了索引(a, b, c)，但是d是没有走索引的，如果建立（a,b,d,c）的索引，则可以命中索引(a, b, c, d)，其中a,b,d的顺序可以任意调整。

等于（=）和in 可以乱序。比如，a = 1 AND b = 2 AND c = 3 建立（a,b,c）索引可以任意顺序。

#### 8、如何建立复合索引，可以使sql语句能尽可能匹配到索引？

- 等于条件的索引放在前面（最左），范围查询放在后面。` a = 1 AND b = 2 AND c > 3 AND d = 4`，建立（a, b, d, c）就是不错的选择；
- 先过滤后排序（ORDER BY）如`SELECT * FROM t WHERE c = 100 and d = 'xyz' ORDER BY b`建立(c, d, b)联合索引就是不错的选择
- 对于索引列的查询，一般不建议使用LIKE操作，像`LIKE '%abc'`这样的不能命中索引；不过`LIKE 'abc%'`可以命中索引。

#### 9、建立了索引，索引就一定会被命中吗？或者说索引什么时候失效

- 使用了`not in, <>,!=`则不会命中索引。注：`<>`是不等号
- innoDB引擎下，若使用OR，只有前后两个列都有索引才能命中（执行查询计划，type是index_merge），否则不会使用索引。
- 模糊查询中，通配符在最前面时，即`LIKE '%abc'`这样不能命中索引
- 对列进行函数运算的情况（如 where md5(password) = "xxxx"）
- 联合索引中，遇到范围查询时，其后的索引不会被命中
- 存了数字的char或varchar类型，常见的如用字符串表示的手机号，在查询时不加引号，则不会命中（如where phone=‘13340456789’能命中，where phone=13340456789不能命中）
- 当数据量小时，MySQL发现全表扫描反而比使用索引查询更快时不会使用索引。

#### 10、为什么要使用联合索引？

> MySQL5.0之前，一个表一次只能使用一个索引，无法同时使用多个索引分别进行条件扫描。但是从5.1开始，引入了 index merge 优化技术，对同一个表可以使用多个索引分别进行条件扫描。

[推荐阅读这篇博客](https://www.jianshu.com/p/XgXfhf)

- **减少开销**。建了一个(a,b,c)的联合索引，相当于建了(a),(a,b),(a,b,c)三个索引 
- **覆盖索引**。减少了随机IO操作。同样的有复合索引（a,b,c），如果有如下的sql: select a,b,c from table where a=1 and b = 1。那么MySQL可以直接通过遍历索引取得数据，而无需回表，这减少了很多的随机io操作
- **效率高**。索引列越多，通过索引筛选出的数据越少。比如有1000W条数据的表，有如下sql:select * from table where a = 1 and b =2 and c = 3,假设假设每个条件可以筛选出10%的数据，如果只有单值索引，那么通过该索引能筛选出1000W*10%=100w 条数据，然后再回表从100w条数据中找到符合b=2 and c= 3的数据，然后再排序，再分页；如果是复合索引，通过索引筛选出1000w *10% *10% *10%=1w，然后再排序、分页。

#### 11、既然索引可以加快查询速度，索引越多越好是吗？

[推荐阅读这篇优博客](https://www.jb51.net/article/81875.htm)

大多数情况下索引能大幅度提高查询效率，但数据的变更（增删改）都需要维护索引，因此更多的索引意味着更多的维护成本和更多的空间 （一本100页的书，却有50页目录？）而且过小的表，建立索引可能会更慢（读个2页的宣传手册，你还先去找目录？）

#### 12、主键和唯一索引的区别？

- 主键是一种约束，唯一索引是索引，一种数据结构。
- 主键一定是唯一索引，唯一索引不一定是主键。
- 一个表中可以有多个唯一索引，但只能有一个主键。
- 主键不允许空值，唯一索引允许。
- 主键可以做为[外键](https://www.baidu.com/s?wd=%E5%A4%96%E9%94%AE&tn=SE_PcZhidaonwhc_ngpagmjz&rsv_dl=gh_pc_zhidao)，唯一索引不行； 

#### 13、B+树和B-树的区别？

B-树是一种平衡的多路查找树。2-3树和2-3-4树都是B-树的特例。一棵M阶的B-树，除了根结点外的其他非叶子结点，最多含有M-1对键和链接，最少含有M/2对键和链接。根结点可以少于M/2，但是也不能少于2对。

- 关键字集合分布在整颗树中
- 每个元素在该树中只出现一次，可能在叶子结点上，也可能在非叶子结点上。
- 搜索有可能在非叶子结点结束。
- 所有叶子结点位于同一层

B+树是B-树的变体，也是一种多路查找树。

- 非叶子结点值可以看作索引，仅含有其子树中的最大（或）最小关键字。
- 叶子结点保存了所有关键字，且叶子结点按照从小到大的顺序排列，是一个双向链表结构。
- 只能在叶子节点命中搜索

B+ 树更适合用于数据库和操作系统的文件系统中。 

假设一个结点就是一个页面，B树遍历所有记录，通过中序遍历的方式，要多次返回到父结点，同一个结点多次访问了，增加了磁盘I/O操作的次数。B+因为在叶子结点存放了所有的记录，而且是双向链表的结构，只需在叶子节点这一层就能遍历所有记录，大大减少了磁盘I/O操作，所以数据库索引用B+树结构更好。

#### 14、聚集索引与非聚集索引的区别？

- 对于聚集索引，表记录的排列顺序和与索引的排列顺序是一致的；非聚集索引不是
- 聚集索引就是按每张表的主键构造一棵B+树，每张表只能拥有一个聚集索引；一张表可以有多个非聚集索引
- 聚集索引的叶子结点存放的是整张表的行记录数据；非聚集索引的叶子结点并不包含行记录的全部数据，除了包含键值还包含一个书签——即相应行数据的聚集索引键。因此通过非聚集索引查找时，先根据叶子结点的指针获得指向主键索引的主键，然后再通过主键索引来找到一个完整的行记录。

#### 15、InnoDB和MyISAM引擎的区别？

- InnoDB支持事务，MyISAM不支持
- InnoDB是行锁设计，MyISAM是表锁设计
- InnoDB支持外键，MyISAM不支持
- InnoDB采用聚集的方式，每张表按照主键的顺序进行存放。如果没有主键，InnoDB会为每一行生成一个6字节的ROWID并以此为主键；MyISAM可以不指定主键和索引
- InnoDB没有保存表的总行数，因此查询行数时会遍历整表；而MyISAM有一个变量存储可表的总行数，查询时可以直接取出该值
- InnoDB适合联机事务处理(OLTP)，MyISAM适合联机分析处理(OLAP)

#### 16、`COUNT(*)`和`COUNT(1)`的区别？`COUNT(列名)`和`COUNT(*)`的区别？

`COUNT(*)`和`COUNT(1)`没区别。`COUNT(列名)`和`COUNT(*)`区别在于前者不会统计列为NULL的数据，后者会统计。 

#### 17、数据库中悲观锁和乐观锁讲一讲？

悲观锁：总是假设在并发下会出现问题，即假设多个事务对同一个数据的访问会产生冲突。当其他事务想要访问数据时，会在临界区提前加锁，需要将其阻塞挂起。比如MySQL中的排他锁（X锁）、和共享锁（S锁）

乐观锁： 总是假设任务在并发下是安全的，即假设多个事务对同一个数据的访问不会发生冲突，因此不会加锁，就对数据进行修改。当遇到冲突时，采用CAS或者版本号、时间戳的方式来解决冲突。数据库中使用的乐观锁是版本号或时间戳。乐观并发控制（OCC）是一种用来解决写-写冲突的无锁并发控制，不用加锁就尝试对数据进行修改，在修改之前先检查一下版本号，真正提交事务时，再检查版本号有，如果不相同说明已经被其他事务修改了，可以选择回滚当前事务或者重试；如果版本号相同，则可以修改。

 提一下乐观锁和MVCC的区别，其实MVCC也利用了版本号，和乐观锁还是能扯上些关系。

MVCC主要解决了读-写的阻塞，因为读只能读到数据的历史版本（快照）；OCC主要解决了写-写的阻塞，多个事务对数据进行修改而不加锁，更新失败的事务可以选择回滚或者重试。

>  当多个用户/进程/线程同时对数据库进行操作时，会出现3种冲突情形：读-读，不存在任何问题；读-写，有隔离性问题，可能遇到脏读、不可重复读 、幻读等。写-写，可能丢失更新。多版本并发控制（MVCC）是一种用来解决读-写冲突的无锁并发控制，读操作只读该事务开始前的数据库的快照，实现了一致性非锁定读。 这样在读操作不用阻塞写操作，写操作不用阻塞读操作的同时，避免了脏读和不可重复读。乐观并发控制（OCC）是一种用来解决写-写冲突的无锁并发控制，不用加锁就尝试对数据进行修改，在修改之前先检查一下版本号，真正提交事务时，再检查版本号有，如果不相同说明已经被其他事务修改了，可以选择回滚当前事务或者重试；如果版本号相同，则可以修改。 

#### 18、MySQL的可重复读是如何实现的？

MVCC（多版本并发控制）使用undo_log使得事务可以读取到数据的快照（某个历史版本），从而实现了可重复读。MySQL采用Next-Key Lock算法，对于索引的扫描不仅是锁住扫描到的索引，还锁住了这些索引覆盖的范围，避免了不可重复读和幻读的产生。

具体来说：

在可重复读下： select....from会采用MVCC实现的一致性非锁定读，读取的是事务开始的快照，避免了不可重复读。select .....from .... for update会采用 Next-Key Locking来保证可重复读和幻读。

在读已提交下: select....from 会采用快照，读取的是最新一份的快照数据，不能够保证不可重复读和幻读；select .....from .... for update会采用Record Lock，不能够保证不可重复读/幻读。 

#### 19、覆盖索引是什么？

如果一个索引包含(或覆盖)所有需要查询的字段的值，即只需扫描索引而无须回表，这称为“覆盖索引”。InnoDB的辅助索引在叶子节点中保存了部分键值信息以及指向聚集索引键的指针，如果辅助索引叶子结点中的键值信息已经覆盖了要查询的字段，就没有必要利用指向主键索引的主键，然后再通过主键索引来找到一个完整的行记录了。

#### 20、MySQL中JOIN和UNION什么区别？

UNION 操作符用于合并两个或多个 SELECT 语句的结果集。UNION 内部的 SELECT 语句必须拥有相同数量的列。列也必须拥有相同的数据类型。同时，每条 SELECT 语句中的列的顺序必须相同。默认情况下，UNION会过滤掉重复的值。使用 UNION ALL则会包含重复的值。

JOIN用于连接两个有关联的表，筛选两个表中满足条件（ON后的条件）的行记录得到一个结果集。从结果集中SELECT的字段可以是表A或者表B中的任意列。

JOIN常用的有LEFT JOIN、RIGHT JOIN、INNER JOIN。

- LEFT JOIN会以左表为基础，包含左表的所有记录，以及右表中匹配ON条件的记录，对于未匹配的列，会以NULL表示。
- LEFT JOIN会以右表为基础，包含左表的所有记录，以及右表匹配ON条件的记录，对于未匹配的列，会以NULL表示。
- INNER JOIN，产生两个表的交集（只包含满足ON条件的记录）

![231634008872011](http://picmeup.oss-cn-hangzhou.aliyuncs.com/coding/231634008872011.png)

INNER JOIN

![231634016379381](http://picmeup.oss-cn-hangzhou.aliyuncs.com/coding/231634016379381.png)

FULL OUTER JOIN

![231634027315181](http://picmeup.oss-cn-hangzhou.aliyuncs.com/coding/231634027315181.png)

![231634045743650](http://picmeup.oss-cn-hangzhou.aliyuncs.com/coding/231634045743650.png)

LEFT JOIN

![231634072774691](http://picmeup.oss-cn-hangzhou.aliyuncs.com/coding/231634072774691.png)

RIGHT JOIN和LEFT JOIN类似。

![241947220904425](http://picmeup.oss-cn-hangzhou.aliyuncs.com/coding/241947220904425.jpg)

#### 21、WHERE和HAVING的区别？

- WHERE过滤的是行，HAVING过滤分组。
- WHERE能完成的，都可以用HAVING(只是有时候没必要)
- WHERE在分组前对数据进行过滤，HAVING在分组后对数据进行过滤
- WHERE后不能接聚合函数，HAVING后面通常都有聚合函数

#### 22、SQL注入是什么，如何防止？

所谓SQL注入式攻击，就是攻击者把SQL命令插入到Web表单的输入域或页面请求的查询字符串，欺骗服务器执行恶意的SQL命令。

比如在登录界面，如果用户名填入`'xxx' OR 1=1 --`就能构造下面的SQL语句，因为OR 1=1，password被注释掉，因此无论name和password填入什么都能登录成功。

```sql
SELECT * FROM USER WHERE NAME='xxx' OR 1=1 -- and password='xxx';
```

使用PrepareStatement，可以防止sql注入攻击，sql的执行需要编译，注入问题之所以出现，是因为用户填写 sql语句参与了编译。使用PrepareStatement对象在执行sql语句时，会分为两步，第一步将sql语句 "运送" 到mysql上预编译，再回到java端拿到参数运送到mysql端。**预先编译好，也就是SQL引擎会预先进行语法分析，产生语法树，生成执行计划，也就是说，后面你输入的参数，无论你输入的是什么，都不会影响该sql语句的语法结构了。用户填写的 sql语句，就不会参与编译，只会当做参数来看。从而避免了sql注入问题。**

## SSM

#### 1、Spring有什么好处（特性），怎么管理对象的？

- IOC：Spring的IOC容器，将对象之间的创建和依赖关系交给Spring，降低组件之间的耦合性。即Spring来控制对象的整个生命周期。其实就是平常说的DI或者IOC。
- AOP：面向切面编程。可以将应用各处的功能分离出来形成可重用的组件。核心业务逻辑与安全、事务、日志等这些非核心业务逻辑分离，使得业务逻辑更简洁清晰。

- 使用模板消除了样板式的代码。比如使用JDBC访问数据库。
- 提供了对像关系映射（ORM）、事务管理、远程调用和Web应用的支持。

Spring使用IOC容器创建和管理对象，比如在XML中配置了类的全限定名，然后Spring使用反射+工厂来创建Bean。BeanFactory是最简单的容器，只提供了基本的DI支持，ApplicationContext基于BeanFactory创建，提供了完整的框架级的服务，因此一般使用应用上下文。

#### 2、什么是IOC？

IOC(Inverse of Control)即控制反转。可以理解为控制权的转移。传统的实现中，对象的创建和依赖关系都是在程序进行控制的。而现在由Spring容器来统一管理、对象的创建和依赖关系，控制权转移到了Spring容器，这就是控制反转。

#### 3、什么是DI？DI的好处是什么？

DI(Dependency Injection)依赖注入。对象的依赖关系由负责协调各个对象的第三方组件在创建对象的时候进行设定，对象无需自行创建或管理它们的依赖关系。通俗点说就是Spring容器为对象注入外部资源，设置属性值。DI的好处是使得各个组件之间松耦合，一个对象如果只用接口来表明依赖关系，这种依赖可以在对象毫不知情的情况下，用不同的具体类进行替换。

**IOC和DI其实是对同一种的不同表述**。

#### 4、什么是AOP，AOP的好处？

AOP(Aspect-Orientid Programming)面向切面编程，可以将遍布在应用程序各个地方的功能分离出来，形成可重用的功能组件。系统的各个功能会重复出现在多个组件中，各个组件存在于核心业务中会使得代码变得混乱。使用AOP可以将这些多处出现的功能分离出来，不仅可以在任何需要的地方实现重用，还可以使得核心业务变得简单，实现了将核心业务与日志、安全、事务等功能的分离。

具体来说，散布于应用中多处的功能被称为*横切关注点*，这些横切关注点从概念上与应用的业务逻辑是相分离的，但是又常常会直接嵌入到应用的业务逻辑中，AOP把这些横切关注点从业务逻辑中分离出来。安全、事务、日志这些功能都可以被认为是应用中的横切关注点。

通常要重用功能，可以使用继承或者委托的方式。但是继承往往导致一个脆弱的对像体系；委托带来了复杂的调用。面向切面编程仍然可以在一个地方定义通用的功能，但是可以用声明的方法定义这个功能要在何处出现，而无需修改受到影响的类。**横切关注点可以被模块化为特殊的类，这些类被称为切面（Aspect）**。好处在于：

- 每个关注点都集中在一个地方，而非分散在多处代码中；
- 使得业务逻辑更简洁清晰，因为这样可以只关注核心业务，次要的业务被分离成关注点转移到切面中了。

AOP术语介绍

**通知**：切面所做的工作称为通知。通知定义了切面是什么，以及在何时使用。Spring切面可以应用5种类型的通知

- 前置通知（Before）：在目标方法被调用之前调用通知功能；
- 后置通知（After）：在目标方法被调用或者抛出异常之后都会调用通知功能；
- 返回通知（After-returning）：在目标方法成功执行之后调用通知；
- 异常通知（After-throwing）：在目标方法抛出异常之后调用通知；
- 环绕通知（Around）：通知包裹了被通知的方法，在目标方法被调用之前和调用之后执行自定义的行为。
  

**连接点**：可以被通知的方法

**切点**：实际被通知的方法

**切面**：即通知和切点的结合，它是什么，在何时何处完成其功能。

**引入**：允许向现有的类添加新方法或属性，从而可以在无需修改这些现有的类情况下，让它们具有新的行为和状态。

**织入**：把切面应用到目标对象并创建新的代理对象的过程。切面在指定的连接点被织入到目标对象中。在目标对象的生命周期里有多个点可以进行织入：

- 编译期，切面在目标类编译时织入。
- 类加载期，切面在目标类加载到JVM时被织入。
- 运行期，切面在应用运行的某个时刻被织入，**在织入切面时，AOP容器会为目标对象动态地创建一个代理对象。Spring AOP就是以这种方式织入切面的**。

Spring AOP构建在动态代理基础之上，所以Spring对AOP的支持仅限于方法拦截。

Spring的切面是由包裹了目标对象的代理类实现的。代理类封装了目标类，并拦截被通知方法的调用，当代理拦截到方法调用时，在调用目标bean方法之前，会执行切面逻辑。**其实切面只是实现了它们所包装bean相同接口的代理**。

#### 5、AOP的实现原理：Spring AOP使用的动态代理。

Spring AOP中的动态代理主要有两种方式，JDK动态代理和CGLIB动态代理。JDK动态代理通过反射来接收被代理的类，并且要求被代理的类必须实现一个接口。JDK动态代理的核心是InvocationHandler接口和Proxy类。

如果目标类没有实现接口，那么Spring AOP会选择使用CGLIB来动态代理目标类。CGLIB（Code Generation Library），是一个代码生成的类库，可以在运行时动态的生成某个类的子类，注意，CGLIB是通过继承的方式做的动态代理，因此如果某个类被标记为final，那么它是无法使用CGLIB做动态代理的。

Spring使用动态代理，代理类封装了目标类，当代理拦截到方法调用时，在调用目标bean的方法之前，会执行切面逻辑。

#### 6、Spring的生命周期？

Spring创建、管理对象。Spring容器负责创建对象，装配它们，配置它们并管理它们的整个生命周期。

- 实例化：Spring对bean进行实例化
- 填充属性：Spring将值和bean的引用注入到bean对应的属性中
- 调用BeanNameAware的setBeanName()方法：若bean实现了BeanNameAware接口，Spring将bean的id传递给setBeanName方法
- 调用BeanFactoryAware的setBeanFactory()方法：若bean实现了BeanFactoryAware接口，Spring调用setBeanFactory方法将BeanFactory容器实例传入
- 调用ApplicationContextAware的setApplicationContext方法：如果bean实现了ApplicationContextAware接口，Spring将调用setApplicationContext方法将bean所在的应用上下文传入
- 调用BeanPostProcessor的预初始化方法：如果bean实现了BeanPostProcessor，Spring将调用它们的叛postProcessBeforeInitialization方法
- 调用InitalizingBean的afterPropertiesSet方法：如果bean实现了InitializingBean接口，Spring将调用它们的afterPropertiesSet方法
- 如果bean实现了BeanPostProcessor接口，Spring将调用它们的postProcessAfterInitialzation方法
- 此时bean已经准备就绪，可以被应用程序使用，它们将一直驻留在应用杀死那个下文中，直到该应用的上下文被销毁。
- 如果bean实现了DisposableBean接口，Spring将调用它的destroy方法。

####  7、Spring的配置方式，如何装配bean？bean的注入方法有哪些？

- XML配置，如`<bean id="">`
- Java配置即JavaConfig，使用`@Bean`注解
- 自动装配，组件扫描（component scanning）和自动装配（autowiring），`@ComponentScan`和`@AutoWired`注解

bean的注入方式有：

- 构造器注入
- 属性的setter方法注入
  

推荐对于强依赖使用构造器注入，对于弱依赖使用属性注入。

#### 8、bean的作用域？

- 单例（Singleton）：在整个应用中，只创建bean一个实例。
- 原型（Prototype）：每次注入或通过Spring应用上下文获取时，都会创建一个新的bean实例。
- 会话（Session）：在Web应用中，为每个会话创建一个bean实例。
- 请求（Request）：在Web应用中，为每个请求创建一个bean实例。

默认情况下Spring中的bean都是单例的。

#### 9、Spring中涉及到哪些设计模式？

- 工厂方法模式。在各种BeanFactory以及ApplicationContext创建中都用到了；
- 单例模式。在创建bean时用到，Spring默认创建的bean是单例的；
- 代理模式。在AOP中使用Java的动态代理；
- 策略模式。比如有关资源访问的Resource类
- 模板方法。比如使用JDBC访问数据库，JdbcTemplate。
- 观察者模式。Spring中的各种Listener，如ApplicationListener
- 装饰者模式。在Spring中的各种Wrapper和Decorator
- 适配器模式。Spring中的各种Adapter，如在AOP中的通知适配器AdvisorAdapter

#### 10、MyBatis和Hibernate的区别和应用场景？

Hibernate :是一个标准的ORM(对象关系映射) 框架； SQL语句是自己生成的，程序员不用自己写SQL语句。因此要对SQL语句进行优化和修改比较困难。适用于中小型项目。

MyBatis： 程序员自己编写SQL， SQL修改和优化比较自由。 MyBatis更容易掌握，上手更容易。主要应用于需求变化较多的项目，如互联网项目等。

## 海量数据处理

首先要了解几种数据结构和算法：

- HashMap，记住对于同一个键，哈希出来的值一定是一样的，不同的键哈希出来也可能一样，这就是发生了冲突（或碰撞）。
- BitMap，可以看成是bit数组，数组的每个位置只有0或1两种状态。Java中可以使用int数组表示位图，arr[0]是一个int，一个int是32位，故可以表示0-31的数，同理arr[1]可表示32-63...实际上就是用一个32位整型表示了32个数。
- 大/小根堆，O(1)时间可在堆顶得到最大值/最小值。利用小根堆可用于求Top K问题。
- 布隆过滤器。使用长度为m的bit数组和k个Hash函数，某个键经过k个哈希函数得到k个下标，将k个下标在bit数组中对应的位置设置为1。对于每个键都重复上述过程，得到最终设置好的布隆过滤器。对于新来的键，使用同样的过程，得到k个下标，判断k个下标在bit数组中的值是否为1，若有一个不为1，说明这个键一定不在集合中。若全为1，也可能不在集合中。就是说：查询某个键，判断不属于该集合是绝对正确的；判断属于该集合是低概率错误的。因为多个位置的1可能是由不同的键散列得到。

对上亿个无重复数字的排序，或者找到没有出现过数字，注意因为无重复数字，而BitMap的0和1正好可以表示该数字有没有出现过。如果要求更小的内存，可以先分出区间，对落入区间的进行计数。必然有的区间数量未满，再遍历一次数组，只看该区间上的数字，使用BitMap，遍历完成后该区间中必然有没被设置成0的的地方，这些地方就是没出现的数。

数据在小范围内波动，比如人类年龄，而且数据允许重复，可用计数排序处理数值排序或查找没有出现过的值，计数的桶中频次为0的就是没有出现过的数。

数据是数字，要找最大的Top K，直接用大小为K的小根堆，不断淘汰最小元素即可。

数据是数字或非数字，要找频次最高的Top K。可使用HashMap统计频次，统计出频次最大的前K个即可。统计频次选出前K的过程可以用小根堆。还可以用Hash分流的方法，即用一个合适的hash函数将数据分到不同的机器或者文件中，因为对于同样的数据，由于hash函数的性质，必然被分配到同一个文件中，因此不存在相同的数据分布在不同的文件这种情况。对每个文件采用HashMap统计频次，用小根堆选出Top K，然后汇总全部文件，从所有部分结果的Top K中再利用小根堆得到最终的Top K。

查找数值的排名，比如找到中位数。比如将数划分区间，对落入每个区间的数进行计数。然后可以得知中位数落在哪个区间，再遍历所有数，这次只关心落在该区间的数，不划分区间的对其进行计数，就可以找出中位数。

## 操作系统

#### 1、Linux中怎么查看CPU使用率、内存、磁盘、进程？

使用`top`命令，如下第三行显示了CPU使用率。

```shell
top - 17:35:12 up 5 min,  1 user,  load average: 0.05, 0.16, 0.09
Tasks: 141 total,   1 running, 105 sleeping,   0 stopped,   0 zombie
%Cpu(s):  3.5 us,  1.7 sy,  0.0 ni, 94.8 id,  0.0 wa,  0.0 hi,  0.0 si,  0.0 st
KiB Mem :  3073524 total,  1547424 free,   528024 used,   998076 buff/cache
KiB Swap:  4194304 total,  4194304 free,        0 used.  2316492 avail Mem

PID USER      PR  NI    VIRT    RES    SHR S %CPU %MEM     TIME+  
585 root      20   0  978032  91332  44924 S  2.0  3.0   0:03.97 Xorg       
2129 sun       20   0  533920  39272  30000 S  1.7  1.3   0:01.95 deepin-termin+ 
  911 sun       20   0  796128  24480  19976 S  1.0  0.8   0:00.94 dde-session-i+ 
  330 root      20   0  629996  17244  13288 S  0.7  0.6   0:00.38 dde-system-da+ 
  751 sun       20   0  118800   2184   1796 S  0.3  0.1   0:00.90 VBoxClient      
```

%CPU：进程占用CPU的使用

%MEM：进程使用的物理内存和总内存的百分

还可以使用`free`命令，简单查看内存状况

```shell
              total        used        free      shared  buff/cache   available
Mem:        3073524      536724     1603808       75300      932992     2302016
Swap:       4194304           0     4194304
```

使用`df -h`查看磁盘使用情况。

```shell
文件系统        容量  已用  可用 已用% 挂载点
udev            1.5G     0  1.5G    0% /dev
tmpfs           301M  1.2M  299M    1% /run
/dev/sda1        30G   20G  8.2G   71% /
tmpfs           1.5G     0  1.5G    0% /dev/shm
tmpfs           5.0M  4.0K  5.0M    1% /run/lock
tmpfs           1.5G     0  1.5G    0% /sys/fs/cgroup
vmshare         238G  193G   46G   81% /mnt/work
tmpfs           301M   12K  301M    1% /run/user/1000
/dev/sr0         56M   56M     0  100% /media/sun/VBox_GAs_5.2.12

```

使用`ps –ef` 查看进程的状态。

```shell
UID        PID  PPID  C STIME TTY          TIME CMD
root         1     0  0 15:56 ?        00:00:01 /sbin/init splash
root         2     0  0 15:56 ?        00:00:00 [kthreadd]
root         4     2  0 15:56 ?        00:00:00 [kworker/0:0H]
root         6     2  0 15:56 ?        00:00:00 [mm_percpu_wq]
root         7     2  0 15:56 ?        00:00:00 [ksoftirqd/0]
root         8     2  0 15:56 ?        00:00:00 [rcu_sched]

```

#### 2、写一个LRU？

Least Recently Used(LRU)，即最近最少使用页面置换算法。**选择最长时间没有被引用的页面进行置换**，思想是：如果一个页面很久没有被引用到，那么可以认为在将来该页面也很少被访问到。

当发生缺页（CPU要访问的页不在内存中），计算内存中每个页上一次被访问的时间，置换上次使用到当前时间最长的一个页面。

如何实现？可以使用双向链表+哈希表的方式

维护一个按最近一次访问时间排序的页面链表。

- 链表头结点是最近刚刚访问过的页面
- 链表尾结点是最久未被访问的页面

HashMap主要是为了判断是否命中缓存。

访问内存时，若命中缓存，找到响应的页面，将其移动到链表头部，表示该页面是最近刚刚访问的。缺页时，将链表尾部的页面移除，同时新页面放到链表头。

```java
import java.util.HashMap;
import java.util.LinkedList;

public class LRUCache2<K,V> {
    private final int cacheSize;
    private LinkedList<K> cacheList = new LinkedList<>();
    private HashMap<K,V> map = new HashMap<>();

    public LRUCache2(int cacheSize) {
        this.cacheSize = cacheSize;
    }

    public synchronized void put(K key, V val) {
        if (!map.containsKey(key)) {
            if (map.size() >= cacheSize) {
                removeLastElement();
            }
            cacheList.addFirst(key);
            map.put(key, val);
        } else {
            moveToFirst(key);
        }
    }

    public V get(K key) {
        if (!map.containsKey(key)) {
            return null;
        }
        moveToFirst(key);
        return map.get(key);
    }

    private synchronized void moveToFirst(K key) {
        cacheList.remove(key);
        cacheList.addFirst(key);
    }

    private synchronized void removeLastElement() {
        K key = cacheList.removeLast();
        map.remove(key);
    }

    @Override
    public String toString() {
        return cacheList.toString();
    }

    public static void main(String[] args) {
        LRUCache2<String,String> lru = new LRUCache2<>(4);
        lru.put("C", null);
        lru.put("A", null);
        lru.put("D", null);
        lru.put("B", null);
        lru.put("E", null);
        lru.put("B", null);
        lru.put("A", null);
        lru.put("B", null);
        lru.put("C", null);
        lru.put("D", null);

        System.out.println(lru);
    }
}

/* out:
[D, C, B, A] 
*/
```

Java的LinkedHashMap是双向链表和HashMap的结合。可以直接用其搞定，构造方法设置参数`accessOrder=true`即可按访问顺序排序。而`removeEldestEntry`就是用来删除最久未被使用的结点。

```java
import java.util.LinkedHashMap;
import java.util.Map;

public class LRUCache<K,V> extends LinkedHashMap<K, V> {
    static final float LOAD_FACTOR = 0.75f;
    static final boolean ACCESS_ORDER = true;
    private int cacheSize;
    public LRUCache(int initialCapacity, int cacheSize) {
        super(initialCapacity, LOAD_FACTOR, ACCESS_ORDER);
        this.cacheSize = cacheSize;
    }

    @Override
    protected synchronized boolean removeEldestEntry(Map.Entry eldest) {
        return size() > cacheSize;
    }

    @Override
    public V get(Object key) {
        return super.get(key);
    }

    @Override
    public synchronized V put(K key, V value) {
        return super.put(key, value);
    }

    public static void main(String[] args) {
        LRUCache<String, String> lru = new LRUCache<>(4, 4);
        lru.put("C", null);
        lru.put("A", null);
        lru.put("D", null);
        lru.put("B", null);
        lru.put("E", null);
        lru.put("B", null);
        lru.put("A", null);
        lru.put("B", null);
        lru.put("C", null);
        lru.put("D", null);

        System.out.println(lru);
    }
}

```

上面打印是{A, B, C, D}这样的顺序，倒过来就行了（我也不知道为啥要这样）。

#### 3、地址总线和数据总线分别影响了计算机的什么？

- 地址总线决定了计算机的寻址能力（范围），比如32位的寻址能力就是4GB
- 数据总线决定每次传输数据的大小

## 其他

#### 1、你说你用到了线性回归，怎么理解回归的意思？

线性回归的英文是Regression toward the mean，直译就是"回归到均值"。各种数据都有其平均值，比如人类的身高。高的人大概率孩子也高，矮的人大概率孩子也矮，但是高的人身高每增加一个单位，其孩子可能只能增加半个单位；矮的人身高每减少一个单位，其孩子可能只会减少半个单位。也就是子代的平均高度会向着一个中心趋势靠拢，这就是回归到均值。

回归问题等价于函数拟合，线性回归也就是用线性的函数去拟合所有观测数据，利用最小二乘法使得误差（预测值和真实值差值的平方和）最小化。对于新的观测数据，使用该函数计算得到一个预测值。

#### 2、一个c/c++程序是怎么从代码到可执行文件的？

额，面试官是C++的吧，然而我面Java。

[推荐这篇博客](https://blog.csdn.net/f905699146/article/details/72877413)

主要经历以下步骤：

- 预处理
- 编译
- 汇编
- 链接

![20170606130136651](http://picmeup.oss-cn-hangzhou.aliyuncs.com/coding/20170606130136651.jpg)

编译的过程又分为以下几步：词法分析，语法分析，语义分析，源代码优化，代码生成和目标代码优化

![20170606133523278](http://picmeup.oss-cn-hangzhou.aliyuncs.com/coding/20170606133523278.png)

然后链接的过程如下

![20170606134035683](http://picmeup.oss-cn-hangzhou.aliyuncs.com/coding/20170606134035683.jpg)

#### 3、一致性哈希了解吗？

[推荐阅读这篇博客](http://www.zsythink.net/archives/1182)

这是一种分布式数据缓存的方案，常用于负载均衡。解决将许多缓存均匀分布到各个服务器上的问题。

- 首先求出服务器（节点）的哈希值，一致性哈希算法将哈希值对`2^32`取模，将服务器配置到0～2^32的圆上。

- 然后采用同样的方法求出存储数据的键的哈希值，并映射到相同的圆上。

- 然后从数据映射到的位置开始顺时针查找，将数据保存到找到的第一个服务器上。如果超过2^32仍然找不到服务器，就会保存到第一台服务器上。

![Snipaste_2018-09-02_23-22-06](http://picmeup.oss-cn-hangzhou.aliyuncs.com/coding/Snipaste_2018-09-02_23-22-06.png)

优点：

- 在删除服务器结点后，失效的缓存只是一部分，不会影响到所有缓存
- 新增服务器结点，缓存迁移的代价很小

但是有时会出现服务器hash得不均匀的情况，这样有大量的缓存都存在一个服务器中。

![Snipaste_2018-09-02_23-27-46](http://picmeup.oss-cn-hangzhou.aliyuncs.com/coding/Snipaste_2018-09-02_23-27-46.png)

这种情况可以通过设置虚拟节点，如图中淡蓝色中的A、B、C就是虚拟结点。添加虚拟结点后，1、3被分到A中，4、5分到B中，2、6被分到C中，分布就很均匀了。

![Snipaste_2018-09-02_23-29-43](http://picmeup.oss-cn-hangzhou.aliyuncs.com/coding/Snipaste_2018-09-02_23-29-43.png)

#### 4、虚拟机VM和Docker的区别？

Docker容器不是虚拟机。

容器和 VM（虚拟机）的主要区别是，容器提供了**基于进程的隔离**，而虚拟机提供了资源的完全隔离。

虚拟机的启动比容器慢很多，虚拟机可能要一分钟来启动，但是容器只需要几秒甚至不到一秒。

容器使用宿主操作系统的内核，而虚拟机使用独立的内核。Docker使用的是Linux容器（LinuX Container, 简称LXC），跟其宿主运行同样的操作系统。多个容器之间是共用同一套操作系统资源的。

#### 5、写一个观察者模式？

![0df5d84c-e7ca-4e3a-a688-bb8e68894467](http://picmeup.oss-cn-hangzhou.aliyuncs.com/coding/0df5d84c-e7ca-4e3a-a688-bb8e68894467.png)

先定义主题和观察者的接口。

```java
public interface Subject {
    void resisterObserver(Observer o);

    void removeObserver(Observer o);

    void notifyObserver();
}

public interface Observer {
    void update(float temp, float humidity, float pressure);
}
```

然后是实现类，这里以Head First里经典的气象站为例。

```java
public class WeatherData implements Subject {
    private List<Observer> observers;
    private float temperature;
    private float humidity;
    private float pressure;

    public WeatherData() {
        observers = new ArrayList<>();
    }

    public void setMeasurements(float temperature, float humidity, float pressure) {
        this.temperature = temperature;
        this.humidity = humidity;
        this.pressure = pressure;
        notifyObserver();
    }

    @Override
    public void resisterObserver(Observer o) {
        observers.add(o);
    }

    @Override
    public void removeObserver(Observer o) {
        int i = observers.indexOf(o);
        if (i >= 0) {
            observers.remove(i);
        }
    }

    @Override
    public void notifyObserver() {
        for (Observer o : observers) {
            o.update(temperature, humidity, pressure);
        }
    }
}
```

```java
/* 观察者1 */
public class StatisticsDisplay implements Observer {

    public StatisticsDisplay(Subject weatherData) {
        weatherData.resisterObserver(this);
    }

    @Override
    public void update(float temp, float humidity, float pressure) {
        System.out.println("StatisticsDisplay.update: " + temp + " " + humidity + " " + pressure);
    }
}
```

```java
/* 观察者2 */
public class CurrentConditionsDisplay implements Observer {

    public CurrentConditionsDisplay(Subject weatherData) {
        weatherData.resisterObserver(this);
    }

    @Override
    public void update(float temp, float humidity, float pressure) {
        System.out.println("CurrentConditionsDisplay.update: " + temp + " " + humidity + " " + pressure);
    }
}
```

```java
/* 测试类 */
public class WeatherStation {
    public static void main(String[] args) {
        WeatherData weatherData = new WeatherData();
        CurrentConditionsDisplay currentConditionsDisplay = new CurrentConditionsDisplay(weatherData);
        StatisticsDisplay statisticsDisplay = new StatisticsDisplay(weatherData);

        weatherData.setMeasurements(0, 0, 0);
        weatherData.setMeasurements(1, 1, 1);
    }
}
```

 