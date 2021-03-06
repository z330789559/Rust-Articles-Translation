* 原文： 
    * https://github.com/nrc/r4cppp/blob/master/destructuring.md
    * https://github.com/nrc/r4cppp/blob/master/destructuring%202.md
* 翻译者： Scott Huang
* 翻译日期： Sep 10, 2015 于厦门

# Destructuring 解构

上次我们看了Rust的数据类型。一旦你有了一些数据结构，你会想要从结构中取出数据。对于结构体，Rust就像C++那样，可以通过字段访问。对元组，元组结构和枚举，您必须使用解构（库里面有各种方便的函数，他们内部都使用解构）。解构数据结构不会在C++中发生，但如果使用过Python或各种函数式语言的话，你就会很熟悉它。这个想法是，正如你可以创建一个数据结构，用一堆本地变量填写它的字段，你也已可以从数据结构中赋值数据到一堆局部变量中去。从这个简单的开始，解构成为Rust的其中一个强大的功能。还有另外一种使用方法，解构结合模式匹配可以把匹配到内容分配到本地变量。

解构主要是通过let和match声明完成任务。当被解构的结构有不同的变体，比如枚举时，我们一般使用match声明。一个let表达式将变量拉入当前的范围，而匹配引入了一个新的范围。比较：

```rust
fn foo(pair: (int, int)) {
    let (x, y) = pair;
    // 我们现在可以在foo的任何地方使用 x 和 y

    match pair {
        (x, y) => {
            // x 和 y 只能用在这个范围。
        }
    }
}
```

模式的语法（在上面的例子中，用在`let`之后，以及`=>`之前）在这两种情况下是（基本）相同的。您还可以在函数声明中的参数位置使用这些模式：

```rust
fn foo((x, y): (int, int)) {
}
```

大多数的初始化表达式可以出现在解构模式，并且他们可以任意复杂。可以包括引用和简单字面量以及数据结构。例如，

```rust
struct St {
    f1: int,
    f2: f32
}

enum En {
    Var1,
    Var2,
    Var3(int),
    Var4(int, St, int)
}

fn foo(x: &En) {
    match x {
        &Var1 => println!("first variant"),
        &Var3(5) => println!("third variant with number 5"),
        &Var3(x) => println!("third variant with number {} (not 5)", x),
        &Var4(3, St { f1: 3, f2: x }, 45) => {
            println!("destructuring an embedded struct, found {} in f2", x)
        }
        &Var4(_, x, _) => {
            println!("Some other Var4 with {} in f1 and {} in f2", x.f1, x.f2)
        }
        _ => println!("other (Var2)")
    }
}
```

注意我们在模式如何通过引用`&`以及如何使用混合字面量（`5`，`3`，`St { ... }`），通配符（`_`），和变量（`×`）来解构。你可以使用`_`当你想要忽略模式中的一个项目，所以如果我们不在乎整数的话，我们可以用`&Var3（_）`。在第一个`Var4 `分支我们解构嵌入结构（嵌套模式），在第二个`Var4`分支将整个结构体赋予一个变量。
你也可以使用`..`在代替一个元组或结构的中的所有字段。所以如果你想为每个枚举变量做点啥，却不关心的变体的内容，你可以这么写：

```rust
fn foo(x: En) {
    match x {
        Var1 => println!("first variant"),
        Var2 => println!("second variant"),
        Var3(..) => println!("third variant"),
        Var4(..) => println!("fourth variant")
    }
}
```

当解构结构时，字段不需要排序，并且你你可以使用`..`来忽略其余字段。例如，

```rust
struct Big {
    field1: int,
    field2: int,
    field3: int,
    field4: int,
    field5: int,
    field6: int,
    field7: int,
    field8: int,
    field9: int,
}

fn foo(b: Big) {
    let Big { field6: x, field3: y, ..} = b;
    println!("pulled out {} and {}", x, y);
}
```

作为一个速记，你可以只使用字段名称创建一个本地变量名。在上面的例子中，创建了两个新的本地变量`x`和`y`。或者，你可以这么写：


```rust
fn foo(b: Big) {
    let Big { field6, field3, .. } = b;
    println!("pulled out {} and {}", field3, field6);
}
```

现在我们以字段相同的名称来创建本地变量，在这种情况下`field3`和`field6`被创建出来。

Rust的解构有几个技巧。让我们假设你想要一个指向一个模式中的变量的引用。你不能使用`&`，因为这匹配一个引用，而不是创建一个新的（从而导致解引用一个对象的效果）。例如，

```rust
struct Foo {
    field: &'static int
}

fn foo(x: Foo) {
    let Foo { field: &y } = x;
}
```

在这里，`y`有`int`类型，是x中的一个字段的副本。创建模式中的指向某些东西的指针，你要使用`ref`关键字。举例：

```rust
fn foo(b: Big) {
    let Big { field3: ref x, ref field6, ..} = b;
    println!("pulled out {} and {}", *x, *field6);
}
```

在这里，`x`和`field6`都有类型`&int`,并且是指向`b`中字段的引用。

最后一个技巧是，如果你是解构是一个复杂的对象，您可能想要为中间对象像单独的字段那样命名。回到前面的例子，我们的模式`&Var4（3，St { f1：3，f2：x }，45）`。在这种模式中，我们为一个结构的字段命名，但你也可能想命名整个结构对象。你可以写`&Var4（3,s,45）`，将绑定结构对象到`s`，但然后你不得不使用字段名来访问每个字段，或者如果你想匹配字段中特定的值，你必须使用一个嵌套匹配。那不好玩。Rust允许你用`@`来命名部分匹配。例如`& Var4（3，s @ St{f1：3，f2：x }，45）`可以让我们为`f2`命名`x`，同时为这个结构命名`s`。

这只不过涉及到了Rust模式匹配的部分内容。还有一些特性我还没有谈到，如匹配向量，但希望你知道如何使用`match`和`let`，并看到了你可以做一些强大的东西。

第二部分我会涉及匹配和借贷之间的一些微妙的相互作用，这曾经在我学习Rust时困扰过我好几次。

# Destructuring pt2 - match and borrowing  解构第二部分 - 匹配和借贷

当解构时你会发现一些借贷指针所导致的惊讶。希望，一旦你真正明白了借贷指针，就没有什么令人惊讶的，但值得讨论（我花了一段时间才弄明白，这是因为，当然事实上，自从我第一次把这篇博客的第一个版本搞砸后）。想象一下，你有一些`&Enum`变量`x`（其中`Enum`是一些枚举类型）。你有两种选择：你可以匹配`*×`然后列出所有变体（`variant1 => ...`等）或你可以匹配`x`，并列出引用到变量模型（`&Variant1 =>...`，等）。（作为一种风格的问题，可能的话更喜欢第一种形式，因为有更少的语法噪音）。`x`是一个借来的引用，如何解引用一个借贷指针是有严格的规定，这些与匹配表达式以令人惊讶的方式进行交互至少让我惊讶，尤其是当你以一个看似无害的方式修改现有的枚举而导致编译器在匹配的某个地方发生爆炸时。

在我们进入match表达式细节前，让我们回顾一下Rust的值传递规则。在C++中，当为一个变量赋值时，或传递它到一个函数时，有2个选择--通过值传递，或者通过引用传递。前者是默认的，也就是指一个值是使用拷贝构造函数或者按位拷贝进行复制。如果您用'&'注释函数传递目的地或者赋值，那么就是传引用了-只有一个指向值的指针被复制，当你操作新变量时，你是也在同时操作旧的值。

Rust有传引用选项，虽然在Rust里，源以及目的地必须注明`&`。至于Rust的值传递，有进一步的两种选择-复制或移动。一个复制和C++中的语义相同（除了Rust没有拷贝构造函数）。一个移动会复制值但同时释放旧的值 - Rust的类型系统确保您不能再访问旧值。作为例子，`int`具有复制语义而`Box<int>`具有移动语义：

```rust
    fn foo() {
    let x = 7i;
    let y = x;                // x 是复制的
    println!("x is {}", x);   // OK

    let x = box 7i;
    let y = x;                // x 是移动的
    //println!("x is {}", x); // 错误： 使用已经移动的值:`x`
}
```

Rust通过寻找析构函数来确定一个对象是否具有移动或复制的语义。析构函数可能需要单独的一篇的博客来解释，但是现在，一个对象在Rust里具有析构函数是看是否实现了`Drop`特质。正如C++，析构函数在对象被摧毁之前执行。如果一个对象有一个析构函数，那么它具有移动语义。如果它没有，那么所有的字段被检查，如果某个字段有析构函数，那么整个对象具有移动语义。整个对象结构都以此类推。如果在对象的任何地方都没有发现析构函数，那么它具有复制的语义。

现在，重要的是，一个借用的对象是不能移动的，否则你会有一个引用指向旧的不再有效的对象。这相当于
在超出范围后还持有一个引用指向一个已被摧毁的对象 - 这是一种悬空指针。如果你有一个指向对象的指针，可能还有其他的引用指向它。因此，如果一个对象具有移动语义且你有一个指向它的指针，那么该指针解引用是不安全的。（如有该对象是复制语义学，解引用将复制一份拷贝，旧的对象还存在，所以其他引用将没问题）。

好的，回到匹配表达式。正如我前面所说的，如果你想匹配一些具有`&T`类型的`x`，那么你可以在匹配条款时解引用一次或者在匹配分支的每个表达式中解引用。例如：

```rust
enum Enum1 {
    Var1,
    Var2,
    Var3
}

fn foo(x: &Enum1) {
    match *x {  // 选项1 : 在这里解引用.
        Var1 => {}
        Var2 => {}
        Var3 => {}
    }

    match x {
        // 选项2： 在每个分支解引用.
        &Var1 => {}
        &Var2 => {}
        &Var3 => {}
    }
}
```

在这种情况下，你可以尝试任何一种方法，因为`Enum1`有复制语义。让我们仔细看看每一个方法：在第一种方法中我们解引用`x`到一个类型为`Enum1`的临时变量（即复制`x`中的值）然后对三种`Enum1`做匹配。这是一个'一层'的匹配，因为我们没有深入到值的类型。第二种方法没有解引用。我们对一个类型为`&Enum1`的值来匹配每个变体的引用。这种match往下走了2层 - 它与类型匹配（通常是一个引用），并同时查看所引用的值（这是`Enum1`）。

无论哪种方式，我们必须确保编译器可以确保我们尊重Rust的不变量的移动和引用-如果有指针指向对象，那我们就不能移动对象的任何一部分。如果匹配的值有复制语义，那么就没有关系了。如果它有移动语义，那么我们必须确保移动不发生在任何match的分支。这是通过忽略会移动的数据，或者引用它（所以我们得用传引用的方式传参，而不是移动）。

```rust
enum Enum2 {
    // Box有析构器，所以Enum2有移动语义
    Var1(Box<int>),
    Var2,
    Var3
}

fn foo(x: &Enum2) {
    match *x {
        // 我们忽略了所包含的内容，所以这样没问题
        Var1(..) => {}
        // 其他分支没有变化
        Var2 => {}
        Var3 => {}
    }

    match x {
        // 我们忽略了所包含的内容，所以这样没问题
        &Var1(..) => {}
        // 其他分支没有变化
        &Var2 => {}
        &Var3 => {}
    }
}
```

在任何一种方法中，我们不涉及任何嵌套的数据，所以没有一项是移动的。在第一种方法中，即使`x`是被引用的，但我们没有触碰到解引用的范围的内核（即，匹配表达式）所以没有东西会被遗忘。我们也不把这个值（即，绑定的`*x`到一个变量），所以我们也不会移动对象整体。

我们可以在第二个match中取的任何一个变体的引用，但不能在解引用的版本。所以，在第二种方法中，用一个`a @ &Var2 = > {}`替换第二个分支是可以的，因为（`a`是一个引用），但是第一种方法下的我们不能写`a @ Var2 => {}`因为这将意味着移动`*×`到`a`。我们可以写`ref a @ Var2 => {}`（在这里`a`也是一个引用），虽然这不是一个你经常看到的结构。

但是，如果我们要使用嵌套在`Var1`中的数据该咋办？我们不能写：

```rust
match *x {
    Var1(y) => {}
    _ => {}
}
```

或者

```rust
match x {
    &Var1(y) => {}
    _ => {}
}
```

因为在这两种情况下，它都意味着移动`x`的一部分到`y`。我们可以用`ref`关键词来获得一个引用指向`Var1`中的数据：`&Var1（ref y） => {}`。这样可以，因为现在我们不在任何地方解引用，因此不移动`x`的任何部分。相反，我们正在创建一个指向`x`内部的指针。

或者，我们可以解构盒子（这种匹配将到达三级深度）：`&Var1（box y） => {} `。同样，这样可以，因为`int`具有复制语义，并且`y`是`Box`内的一个`int`的复制（这是`inside`借用参考）。因为`int`已经具有复制语义，我们不需要移动`x`的任何的一部分。我们还可以创建一个引用到int而不是复制：`&Var1（box ref y) => {}`。再次，这样可以，因为我们不做任何解引用，因此不需要移动`x`的任何部分。如果盒子内容里有移动语义，那么我么不能写`&Var1(box y) => {}`，我们将被迫使用引用版本。我们也可以使用类似技术与第一种方法来匹配，这看起来一样，但没有第一个`&`。例如，`Var1(box ref y) => {}`。

现在让我们变得更加复杂。假设你想匹配一对引用枚举值。现在我们不能用第一种方法：

```rust
fn bar(x: &Enum2, y: &Enum2) {
    // 错误，x和y正在被移动
    // match (*x, *y) {
    //     (Var2, _) => {}
    //     _ => {}
    // }

    // 可以
    match (x, y) {
        (&Var2, _) => {}
        _ => {}
    }
}
```

第一种方法是非法的，因为被匹配的值是通过对`x`和`y`解引用来创建的，然后移动他们到一个新的元组对象。所以在这种情况下，只有第二种方法适用。当然，你仍然必须遵循上述规则，以避免移动`x`和`y`的部分内容。

如果你最终只能得到一个数据的引用，但你需要值本身的话，除了复制这些数据外，你别无选择。通常利用`clone()`。如果数据没有实现克隆，你将需要进一步解构来手动复制或实现克隆自己。

如果我们没有一个指向移动语义的值的引用，只有值本身。那么移动是可以的，因为我们知道没有人有一个引用指向这个值（编译器确保如果他们这样做，我们就不能使用该值）。对于例子

```rust
fn baz(x: Enum2) {
    match x {
        Var1(y) => {}
        _ => {}
    }
}
```

还有一些事情要知道。首先，你只能移动一个地方。在上面的例子中，我们正在移动`x`的一部分到`y`中，我们会忘记其它部分。如果我们写`a @ Var1(y) => {}`我们会试图移动`x`所有的一切到`a`，并且部分`x`到`y`。这是不允许的，一个这样的分支是非法的。做一个`a`或`y`参考（使用`ref a`，等）也不是可选项，那么我们会有上面描述的问题，我们移动同时保持一个引用。我们可以对`a`和`y`做两个引用，那么这样我们可行 - 也没有移动，所以`x`仍然完整，并且我们有指针指向它的整体和它的一部分。

类似地（或更常见），如果我们有多个嵌套的变体数据，我们不能以一个数据为基准，然后移动另一个数据。例如，如果我们有一个`Var4`声明为`Var4(Box<int>, Box<int>)`我们可以有一个匹配的分支
引用两（`Var4(ref y, ref z) => {}`）或配合分支同时移动（`Var4（y，z） => {}`）但你不能有一个匹配的分支，部分移动而部分引用（`Var4（ref y，z） => {} `）。这是因为部分移动
仍然破坏了整个对象，所以引用将无效。