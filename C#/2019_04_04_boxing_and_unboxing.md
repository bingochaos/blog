# C# 中的装箱（boxing）以及拆箱（unboxing）
## 装箱
装箱：装箱其实是一个值类型的数据转换成一个 object 类型或者任意引用该值类型的 interface type。 CLR 在对值类型进行装箱的过程中，其实就是将他包装成一个 Object 类型的变量，并存储在托管堆（managed heap）中。


``` C#
int i = 1;
// 这里将 i 进行了装箱操作
object o = i;
```
i 是存储在栈上的变量，在上述装箱操作中，对 i 的值进行了 copy，然后转化为对应的引用类型并存储于托管堆中，o 则在栈中，指向新装箱的对象。在这个过程中需要新建一个对象，并完成值的拷贝。

![装箱操作](images/2019_04_04_boxing_and_unboxing/boxing-operation-i-o-variables.png "boxing-operation-i-o-variables.png")
## 拆箱
拆箱：拆箱是与装箱相反的操作，即将一个已装箱的值类型，提取出来，从引用类型转化为值类型。
```
o = 1;
i = (int)o; // unboxing
```
拆箱一般是通过类型的强制转化来完成的，这里一共需要完成两个步骤：
1. 检查 o 的类型是否为需要拆箱的类型所对应的装箱对象。
2. 将 o 所指向的值类型数据拷贝到值类型变量上（栈中）

在拆箱过程中，如果对应的对象为 null 则会抛出 `NullRefreenceException` 异常，若为非指定的值类型，则会抛出 `InvalidCastException` 异常。

![拆箱操作](images/2019_04_04_boxing_and_unboxing/unboxing-conversion-operation.png "unboxing-conversion-operation.png")

> wrapping 和 unwrapping ： 将 T 的实例转换成 Nullable<T> 的实例的过程在 C# 语言规范中称为包装（wrapping)，相反的过程则为拆包。但是 Nullable<T> 是一个值类型！它和装箱拆箱不是一回事。如果要将 Nullable<T> 转换成引用类型，还需要经过装箱。
	
## 装箱在 C# 中的应用
```
// String.Concat example
// 其中 42 和 true，都发生了装箱操作
Console.WriteLine(String.Concat("Answer", 42, true));

// List example
// 泛型相关使用
List<object> mixedList = new List<object>();

// 添加一个 string
mixedList.Add("First Group:");

// 添加 int ，会对每一个 int 发生装箱操作成为 object 的子类
for (int j = 1; j < 5; j++)
{
    mixedList.Add(j);
}

// 拆箱计算
var sum = 0;
for (var j = 1; j< 5; j++)
{
    sum += (int)mixedList[j] * (int)mixedList[j];
}
```
> 拆箱不是直接将装箱的过程倒过来。拆箱的代价要比装箱低的多。拆箱其实就是获取指针的过程，该指针指向包含在一个对象中的原始值类型。
## 如何避免装箱拆箱
装箱和拆箱虽然大部分都是自动进行的，但是还是消耗较大的操作，因此在使用的过程中应该尽量避免频繁的装箱拆箱。
1. 使用重载方法。例如 `Console.WriteLine(3);`这个方法提供了大量的重载方法，目的就是为了减少值类型装箱的次数。
2. 使用泛型。因为装箱和拆箱的性能问题，所以在 .NET 2.0 中引入了泛型，他的主要目的就是避免值类型和引用类型之间的装箱和拆箱。比如 ArrayList 对应的List<T>，Hashtable 对应的 Dictionary<TKey, TValue>
3. 如果提前知道某个值类型需要进行装箱后来操作，那就提前进行装箱，并以装箱后的变量进行操作。
```
int j = 3;
object o = j;
ArrayList a = new ArrayList();
for (int i = 0; i < 100; i++)
    a.Add(o);
```
4. ToString 方法。值类型一般都重写了 ToString 方法（基类不是 System.Object ，而是 System.ValueType）。在调用时并不会进行装箱。所以在处理 String.Format 问题时，可以对所有参数调用一次 ToString。（目前版本不确定是否需要，4.7.2已有字符串模板可以使用）

> 虽然未装箱值类型没有类型对象指针，但仍可调用由类型继承或者重写的虚方法（比如 Equals, GetHashCode 或者 ToString)。如果值类型重写了其中任何虚方法，那么 CLR 可以非虚地调用该方法，因为值类型隐式密封，不可能有类型从它们派生，而且调用虚方法的值类型实例没有装箱。

> 误区：引用类型保存在堆上，值类型保存在栈上。
前半句是正确的，引用类型的实例总是在堆上创建的。但第二部分并不是十分正确。变量的值是在它声明的位置存储的。所以假定一个类中有一个 int 类型的实例变量，那么在这个类的任何对象中，该变量的值总是和对象中的其他数据在一起的。只有局部变量（方法内部声明的变量）和方法参数在栈上。而对于 C# 2 及更高版本，很多局部变量并不完全存放在栈中（匿名方法）。

## 不要使用接口更改已装箱值类型中的字段
```
    internal struct Point
    {
        private Int32 m_x, m_y;

        public Point(Int32 x, Int32 y)
        {
            m_x = x;
            m_y = y;
        }

        public void Change(Int32 x, Int32 y)
        {
            m_x = x;
            m_y = y;
        }

        public override String ToString()
        {
            return String.Format("({0}, {1})", m_x.ToString(), m_y.ToString());
        }
    }
    class Program
    {
        static void Main(string[] args)
        {
            Point p = new Point(1, 1);
            Console.WriteLine(p);//(1,1)
            p.Change(2, 2);
            Console.WriteLine(p);//(2,2)
            Object o = p;
            Console.WriteLine(p);//(2,2)
            ((Point)o).Change(3, 3);
            Console.WriteLine(o);//(2,2)  临时对象的拷贝，只有值的存在不能当引用对象使用
            p.Change(3,3);
            Console.WriteLine(p);//(3,3)
            Console.ReadLine();
        }
    }
```

从上面例子可以看出 struct 作为值类型也会参与到装箱拆箱中来，它与类的行为类似，但是在发生装箱时在修改，并不能反映到原来的值中，应该装箱产生的是一个副本。所以像这种值类型，应该标记为 readonly 的，尽量避免修改器实例字段成员，避免出现以上由于装箱拆箱导致的问题。

## 为啥会设计 value type 和 ref type
value type ： value type 变量声明后，不管是否已经赋值，编译器为其分配内存，在线程 Stack 上分配（静态分配）。所有的 value type 均隐式派生于 System.ValueType。
ref type : ref type在声明时， 只在 Stack 中分配一小片内存用于容纳一个地址， 而此时并没有为其分配 heap 上的空间， 当使用 new 创建一个实例时，分配 heap 上的空间，并把 heap 上空间的地址保存到 stack 上分配的空间中。总是在进程 heap 中分配（动态分配）
value type 只需分配一次即可，适用于主要职责是数据存储、共有借口完全由一些数据成员存取属性定义、永远不可能有子类、不具有多态行为的类型。而 ref type 消耗更多的时间，造成更多的内存碎片，适用于定义应用程序的行为。
简而言之：value type 更多是临时使用，快速创建与销毁，ref type 代表程序语义，可能会跟随程序生命周期。

## C++ 中的 unboxing 和 boxing
C++ 中并不存在自动的装箱，但是也存在需要用 object 作为参数而实际却是使用 int 类型的情况，一般操作方式如下：
```
// The box type
class IntBox
{
	// The boxed value
	private int val;
 
	// Create the box with a boxed value
	public IntBox(int val)
	{
		this.val = val;
	}
 
	// Unbox by getting the boxed value back out
	int Unbox()
	{
		return val;
	}
}

// Create a value to box
int val = 123;
 
// Box the value
// Store it as just an 'object'
object boxed = new IntBox(val);
 
// Unbox the value
int unboxed = ((IntBox)boxed).Unbox();
```
而 C# 中的自动装箱其实是为了减少上述代码手写的语法糖而已。

## JS 有 Value type 吗？
有，而且存储位置分为栈和堆，基本与 C# 保持一致
1. 值类型(基本类型)：数值(number)、布尔值(boolean)、null、undefined、string(在赋值传递中会以引用类型的方式来处理)、Symbol

2. 引用类型：对象、数组、函数。


> JavaScript 为基本类型提供了对象包装器，被称为原生类型（String、Number、Boolean 等等）。这些对象包装器使这些值可以访问每种对象子类型的恰当行为（String#trim() 和 Array#concat(..)）。
如果你有一个像 "abc" 这样的简单基本类型标量，而且你想要访问它的 length 属性或某些 String.prototype 方法，JS 会自动地“封箱”这个值（用它所对应种类的对象包装器把它包起来），以满足这样的属性/方法访问。
