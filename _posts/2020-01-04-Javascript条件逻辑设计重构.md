---
layout: post
title: Javascript条件逻辑设计重构
image: /img/hello_world.jpeg
---

### 重构方法概要
-----------------
 * 分解条件表达式
 * 合并条件表达式
 * 以卫语句替代嵌套条件表达式
 * 以多态取代条件表达式
 * 引入特例
 * 引入断言



### 思考: 条件表达式易产生的问题
-----------------
* 复杂度极高: 表现是 if 嵌套两三层设置更多
* 大型函数可读性下降: 不知道为什么会发生这样事情
  
### 重构手法 1：分解条件表达式
-----------------
和任何大块头代码一样，我可以将它分解为多个独立的函数，根据每个小块代码的用途，为分解而得的新函数命名，并将原函数中对应的代码改为调用新函数，从而更清楚地表达自己的意图。对于条件逻辑，将每个分支条件分解成新函数还可以带来更多好处：可以突出条件逻辑，更清楚地表明每个分支的作用，并且突出每个分支的原因。本重构手法其实只是提炼函数的一个应用场景。

```
if (!aDate.isBefore(plan.summerStart) && !aDate.isAfter(plan.summerEnd))
　charge = quantity * plan.summerRate;
else
　charge = quantity * plan.regularRate + plan.regularServiceCharge;
```

进行重构后将条件提炼到一个独立的函数, 用三元运算符重新安排条件语句 :

```
charge = summer() ? summerCharge() : regularCharge();

function summer() {
　return !aDate.isBefore(plan.summerStart) && !aDate.isAfter(plan.summerEnd);
}
function summerCharge() {
　return quantity * plan.summerRate;
}
function regularCharge() {
　return quantity * plan.regularRate + plan.regularServiceCharge;
}
```

### 重构手法 2: 合并条件表达式
-----------------
检查条件各不相同，最终行为却一致。如果发现这种情况，就应该使用“逻辑或”和“逻辑与”将它们合并为一个条件表达式。
之所以要合并条件代码，有两个重要原因。首先，合并后的条件代码会表述“实际上只有一次条件检查，只不过有多个并列条件需要检查而已”，从而使这一次检查的用意更清晰。当然，合并前和合并后的代码有着相同的效果，但原先代码传达出的信息却是“这里有一些各自独立的条件测试，它们只是恰好同时发生”。其次，这项重构往往可以为使用提炼函数做好准备。将检查条件提炼成一个独立的函数对于厘清代码意义非常有用，因为它把描述“做什么”的语句换成了“为什么这样做”。

```
function disabilityAmount(anEmployee) {
　if ((anEmployee.seniority < 2)
　　　|| (anEmployee.monthsDisabled > 12)) return 0;
　if (anEmployee.isPartTime) return 0;
这句条件表达式使用提炼函数 ，使用逻辑与与逻辑或

function disabilityAmount(anEmployee) {
　if (isNotEligableForDisability()) return 0;
　// compute the disability amount

function isNotEligableForDisability() {
　return ((anEmployee.seniority < 2)
　　　　　|| (anEmployee.monthsDisabled > 12)
　　　　　|| (anEmployee.isPartTime));
}
```



### 重构手法 3：以卫语句替代嵌套条件表达式
-----------------
条件表达式通常有两种风格，一种是两个条件分支都属于正常行为，第二种风格是只有一个分支是属于正常行为，另一个分支则是异常行为。如果两个分支都属于正常行为，就应该使用例如 if else 条件表达式；如果某个条件极其罕见，就应该“就应该单独检查该条件，并在该条件为真时立刻从函数中返回。这样的单独检查常常被称为“卫语句"。

以卫语句取代嵌套条件表达式的精髓就是：给某一条分支以特别的重视。如果使用 if-then-else 结构，你对 if 分支和 else 分支的重视是同等的。这样的代码结构传递给阅读者的消息就是：各个分支有同样的重要性。卫语句就不同了，它告诉阅读者：“这种情况不是本函数的核心逻辑所关心的，如果它真发生了，请做一些必要的整理工作，然后退出。

每个函数只能有一个入口和一个出口”的观念，根深蒂固于某些程序员的脑海里。我发现，当我处理他们编写的代码时，经常需要使用以卫语句取代嵌套条件表达式。现今的编程语言都会强制保证每个函数只有一个入口，至于“单一出口”规则，其实不是那么有用。在我看来，保持代码清晰才是最关键的：如果单一出口能使这个函数更清楚易读，那么就使用单一出口；否则就不必这么做。

以下例子中的代码可能我们很多人都写过，我现在认为它是坏味道的代码:

```
function payAmount(employee) {
　let result;
　if(employee.isSeparated) {
　　result = {amount: 0, reasonCode:"SEP"};
　}
　else {
　　if (employee.isRetired) {
　　　result = {amount: 0, reasonCode: "RET"};
　　}
　　else {
　　　// logic to compute amount
　　　lorem.ipsum(dolor.sitAmet);1
　　　consectetur(adipiscing).elit();
　　　sed.do.eiusmod = tempor.incididunt.ut(labore) && dolore(magna.aliqua);
　　　ut.enim.ad(minim.veniam);
　　　result = someFinalComputation();
　　}
　}
　return result;
```

思考下位语句的概念，如果用位语句对其进行清晰的重构:

首先我们可以对 isDead 以及 isSeparated 进行改写位语句

```
function payAmount(employee) {
　let result;
　if (employee.isSeparated) return {amount: 0, reasonCode: "SEP"};
　if (employee.isRetired)   return {amount: 0, reasonCode: "RET"};
　xxxxxxxxxx
　return someFinalComputation();
}
```

有的时候不容易使用位语句，我们可以使用条件反转来实现以卫语句取代嵌套条件表达式

```
function adjustedCapital(anInstrument) {
　let result = 0;
　if (anInstrument.capital > 0) {
　　if (anInstrument.interestRate > 0 && anInstrument.duration > 0) {
　　　result = (anInstrument.income / anInstrument.duration) * anInstrument.adjustmentFactor;
　　}
　}
　return result;
}

```

位语句使用重点在与处理一条分支的正常情况的另一种特殊情况，即大多的少走的 else，所以这次想将 anInstrument.capital >0 用位语句表示出来：

```
function adjustedCapital(anInstrument) {
　let result = 0;
　if (anInstrument.capital <= 0) return result;
　if (anInstrument.interestRate > 0 && anInstrument.duration > 0) {
　　result = (anInstrument.income / anInstrument.duration) * anInstrument.adjustmentFactor;
　}
　return result;
}
```

下一步将 anInstrument.interestRate >0&& anInstrument.duration >0 进行反转

```
function adjustedCapital(anInstrument) {
　let result = 0;
　if (anInstrument.capital <= 0) return result;
　if (!(anInstrument.interestRate > 0 && anInstrument.duration > 0)) return result;
　result = (anInstrument.income / anInstrument.duration) * anInstrument.adjustmentFactor;
　return result;
}
```

    !(anInstrument.interestRate >0&& anInstrument.duration >0) 看起来并不好，进行改变“anInstrument.interestRate <= 0 || anInstrument.duration <= 0)并且使用重构方法 2 合并条件表达式进行。

```
function adjustedCapital(anInstrument) {
　let result = 0;
　if (   anInstrument.capital      <= 0
　　　|| anInstrument.interestRate <= 0
　　　|| anInstrument.duration     <= 0) return result;
　result = (anInstrument.income / anInstrument.duration) * anInstrument.adjustmentFactor;
　return result;
}
```

最使用去除多余变量进行将 result 去除。

```
function adjustedCapital(anInstrument) {
　if (   anInstrument.capital    　<= 0
　　　|| anInstrument.interestRate <= 0
　　　|| anInstrument.duration　　 <= 0) return 0;
　return (anInstrument.income / anInstrument.duration) * anInstrument.adjustmentFactor;
}
```

### 重构手法 4：以多态取代条件表达式
-----------------
以上介绍的重构的手法基本上是实用条件逻辑本身的结构就足以表达，但是我想寻求给条件逻辑添加结构，实用类和多态可以把逻辑的拆分表述的更加清晰，一个常见的场景是：我可以构造一组类型，每个类型处理各自的一种条件逻辑。例如，我会注意到，图书、音乐、食品的处理方式不同，这是因为它们分属不同类型的商品。最明显的征兆就是有好几个函数都有基于类型代码的 switch 语句。若果真如此，我就可以针对 switch 语句中的每种分支逻辑创建一个类，用多态来承载各个类型特有的行为，从而去除重复的分支逻辑。  

另一种情况是：有一个基础逻辑，在其上又有一些变体。基础逻辑可能是最常用的，也可能是最简单的。我可以把基础逻辑放进超类，这样我可以首先理解这部分逻辑，暂时不管各种变体，然后我可以把每种变体逻辑单独放进一个子类，其中的代码着重强调与基础逻辑的差异。

 
多态一直是解决复杂条件逻辑改善这种情况工具,并非所有条件逻辑都要用多态来取代，避免滥用

例子：

```
function plumages(birds) {
　return new Map(birds.map(b => [b.name, plumage(b)]));
}
function speeds(birds) {
　return new Map(birds.map(b => [b.name, airSpeedVelocity(b)]));
}

function plumage(bird) {
　switch (bird.type) {
　case 'EuropeanSwallow':
　　return "average";
　case 'AfricanSwallow':
　　return (bird.numberOfCoconuts > 2) ? "tired" : "average";
　case 'NorwegianBlueParrot':
　　return (bird.voltage > 100) ? "scorched" : "beautiful";
　default:
　　return "unknown";
　}
}

function airSpeedVelocity(bird) {
　switch (bird.type) {
　case 'EuropeanSwallow':
　　return 35;
　case 'AfricanSwallow':
　　return 40 - 2 * bird.numberOfCoconuts;
　case 'NorwegianBlueParrot':
　　return (bird.isNailed) ? 0 : 10 + bird.voltage / 10;
　default:
　　return null;
　}
}
```

使用多态进行重构的标志是，对于同一个对象或者属性进行判断逻辑操作，有两个不同的操作，其行为都随着“鸟的类型”发生变化，因此可以创建出对应的类，用多态来处理各类型特有的行为。

```
function plumages(birds) {
　return new Map(birds
　　　　　　　　 .map(b => createBird(b))
　　　　　　　　 .map(bird => [bird.name, bird.plumage]));
}
function speeds(birds) {
　return new Map(birds
　　　　　　　　 .map(b => createBird(b))
　　　　　　　　 .map(bird => [bird.name, bird.airSpeedVelocity]));
}

function createBird(bird) {
　switch (bird.type) {
　case 'EuropeanSwallow':
　　return new EuropeanSwallow(bird);
　case 'AfricanSwallow':
　　return new AfricanSwallow(bird);
　case 'NorwegianBlueParrot':
　　return new NorwegianBlueParrot(bird);
　default:
　　return new Bird(bird);
　}
}

class Bird {
　constructor(birdObject) {
　　Object.assign(this, birdObject);
　}
　get plumage() {
　　return "unknown";
　}
　get airSpeedVelocity() {
　　return null;
　}
}
class EuropeanSwallow extends Bird {
　get plumage() {
　　return "average";
　}
　get airSpeedVelocity() {
　　return 35;
　}
}
class AfricanSwallow extends Bird {
　get plumage() {
　　return (this.numberOfCoconuts > 2) ? "tired" : "average";
　}
　get airSpeedVelocity() {
　　return 40 - 2 * this.numberOfCoconuts;
　}
}
class NorwegianBlueParrot extends Bird {
　get plumage() {
　　return (this.voltage > 100) ? "scorched" : "beautiful";
　}
　get airSpeedVelocity() {
　　return (this.isNailed) ? 0 : 10 + this.voltage / 10;
　}
}
```

### 重构手法 5: 引入特例
-----------------

该手法主要是对一些特殊的值进行逻辑判断，之后才会进行处理，“一个数据结构的使用者都在检查某个特殊的值，并且当这个特殊值出现时所做的处理也都相同。如果我发现代码库中有多处以同样方式应对同一个特殊值，我就会想要把这个处理逻辑收拢到一处。处理这种情况的一个好办法是使用“特例”（Special Case）模式：创建一个特例元素，用以表达对这种特例的共用行为的处理。这样我就可以用一个函数调用取代大部分特例检查逻辑。


特例有几种表现形式。如果我只需要从这个对象读取数据，可以提供一个字面量对象（literal object），其中所有的值都是预先填充好的。如果除简单的数值之外还需要更多的行“为，就需要创建一个特殊对象，其中包含所有共用行为所对应的函数。特例对象可以由一个封装类来返回，也可以通过变换插入一个数据结构。一个通常需要特例处理的值就是 null，这也是这个模式常被叫作“Null 对象”（Null Object）模式的原因——Null 对象是特例的一种特例。


我们会首先创建一个关于判断特例对象为真的一个类，这个类具有这个特例为 null 的所有属性以及方法，我们将对即将要对某个对象的属性的获取判断，转化为代理这个值，如果存在就使用正常对象，如果不存在就要使用我们创建的那个特例对象的属性以及方法了。

```
Class Site {
  get customer() {return this._customer;}
}
class Customer{
  get name()           {...}
   get billingPlan()    {...}
   set billingPlan(arg) {...}
   get paymentHistory() {...}
}

client one

const aCustomer = site.customer;
// ... lots of intervening code ...
let customerName;
if (aCustomer === "unknown") customerName = "occupant";
else customerName = aCustomer.name;”

clint two

const plan = (aCustomer === "unknown") ?
      registry.billingPlans.basic
      : aCustomer.billingPlan;

clint three

const weeksDelinquent = isUnknown(aCustomer) ?
      0
      : aCustomer.paymentHistory.weeksDelinquentInLastYear;
改写

class Site {
  get customer() {
     return (this._customer === "unknown") ? createUnknownCustomer() : this._customer;
  }
}

function createUnknownCustomer() {
  return {
    isUnknown: true,
    name: "occupant",
    billingPlan: registry.billingPlans.basic,
    paymentHistory: {
      weeksDelinquentInLastYear: 0,
    }
  };
}

function isUnknown(arg) {
  return arg.isUnknown;
}

const customerName = aCustomer.name;

const plan = aCustomer.billingPlan;

const weeksDelinquent = aCustomer.paymentHistory.weeksDelinquentInLastYear;

```

### 重构手法 6：引入断言
----------
这样的假设通常并没有在代码中明确表现出来，你必须阅读整个算法才能看出。有时程序员会以注释写出这样的假设，而我要介绍的是一种更好的技术——使用断言明确标明这些假设。“断言是一个条件表达式，应该总是为真。如果它失败，表示程序员犯了错误。
断言的失败不应该被系统任何地方捕捉。整个程序的行为在有没有断言出现的时候都应该完全一样。实际上，有些编程语言中的断言可以在编译期用一个开关完全禁用掉。“常看见有人鼓励用断言来发现程序中的错误。这固然是一件好事，但却不是使用断言的唯一理由。断言是一种很有价值的交流形式——它们告诉阅读者，程序在执行到这一点时，对当前状态做了何种假设。另外断言对调试也很有帮助。而且，因为它们在交流上很有价值，即使解决了当下正在追踪的错误，我还是倾向于把断言留着。自测试的代码降低了断言在调试方面的价值，因为逐步逼近的单元测试通常能更好地帮助调试，但我仍然看重断言在交流方面的价值。
假设这扩率永远为整数

```
applyDiscount(aNumber) {
  return (this.discountRate)
    ? aNumber - (this.discountRate * aNumber)
    : aNumber;
}
```

先将三元改成 if-else 形式

```
applyDiscount(aNumber) {
  if (!this.discountRate) return aNumber;
  else return aNumber - (this.discountRate * aNumber);
}
```

加入断言

```
applyDiscount(aNumber) {
  if (!this.discountRate) return aNumber;
  else {
    assert(this.discountRate >= 0);
    return aNumber - (this.discountRate * aNumber);
  }
}
```

注意，不要滥用断言。我不会使用断言来检查所有“我认为应该为真”的条件，只用来检查“必须为真”的条件。滥用断言可能会造成代码重复，尤其是在处理上面这样的条件逻辑时。所以我发现，很有必要去掉条件逻辑中的重复，通常可以借助提炼函数手法。我只用断言预防程序员的错误。如果要从某个外部数据源读取数据，那么所有对输入值的检查都应该是程序的一等公民，而不能用断言实现——除非我对这个外部数据源有绝对的信心。断言是帮助我们跟踪 bug 的最后一招，所以，或许听来讽刺，只有当我认为断言绝对不会失败的时候，我才会使用断言。