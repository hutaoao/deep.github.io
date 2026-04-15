---
title: "JavaScript继承的六种实现方式深度解析"
date: 2026-04-15 08:00:00 +0800
categories: [前端核心, JavaScript]
tags: [JavaScript, 继承, 原型链, 面试题, 寄生组合继承]
description: "系统梳理 JavaScript 继承的六种实现方式，从原型链继承到寄生组合继承，深入分析各方案的优缺点与适用场景。"
---

## 一句话概括

JavaScript 继承本质上是**原型链的复用**，从最原始的原型链继承到最优雅的寄生组合继承，每一种方案都是对前一种缺陷的修补，理解这条演进路径是掌握 JS 面向对象的关键。

---

## 背景

在 ES6 `class` 语法出现之前，JavaScript 没有原生的类继承机制。开发者需要通过各种技巧来模拟继承行为。即便是今天，ES6 的 `class extends` 底层依然是原型链，面试中也常常要求手写各种继承方式。

理解六种继承方式的演进，不仅能应对面试，更能深刻理解 JavaScript 的对象模型与原型机制。

---

## 概念与定义

**继承**：子类能够使用父类的属性和方法，并可以扩展自己的特性。

JavaScript 实现继承的核心手段：
1. **原型链**：将父类实例作为子类原型
2. **构造函数借用**：在子类构造函数中调用父类构造函数
3. **两者结合**：取长补短，形成最优方案

---

## 最小示例

```js
// 父类定义
function Animal(name) {
  this.name = name;
  this.colors = ['black', 'white'];
}
Animal.prototype.sayName = function () {
  console.log(this.name);
};

// 子类定义
function Dog(name, breed) {
  Animal.call(this, name); // 借用构造函数
  this.breed = breed;
}

// 寄生组合继承（最优方案）
Dog.prototype = Object.create(Animal.prototype);
Dog.prototype.constructor = Dog;

Dog.prototype.bark = function () {
  console.log('Woof!');
};

const d = new Dog('Rex', 'Husky');
d.sayName(); // Rex
d.bark();    // Woof!
console.log(d instanceof Dog);    // true
console.log(d instanceof Animal); // true
```

---

## 核心知识点拆解

### 方式一：原型链继承

**原理**：将父类的实例赋值给子类的 `prototype`。

```js
function Animal(name) {
  this.name = name;
  this.colors = ['black', 'white'];
}
Animal.prototype.sayName = function () {
  console.log(this.name);
};

function Dog() {}
Dog.prototype = new Animal('动物'); // 核心：父类实例作为子类原型

const d1 = new Dog();
const d2 = new Dog();

d1.colors.push('brown');
console.log(d2.colors); // ['black', 'white', 'brown'] ← 引用类型被共享！
```

**优点**：
- 实现简单，子类可访问父类原型上的方法

**缺点**：
1. **引用类型属性被所有实例共享**（最致命的问题）
2. 创建子类实例时，无法向父类构造函数传参
3. 子类原型上会存在父类实例属性（冗余）

---

### 方式二：构造函数继承（借用构造函数）

**原理**：在子类构造函数中通过 `call/apply` 调用父类构造函数。

```js
function Animal(name) {
  this.name = name;
  this.colors = ['black', 'white'];
}
Animal.prototype.sayName = function () {
  console.log(this.name);
};

function Dog(name, breed) {
  Animal.call(this, name); // 借用父类构造函数
  this.breed = breed;
}

const d1 = new Dog('Rex', 'Husky');
const d2 = new Dog('Max', 'Poodle');

d1.colors.push('brown');
console.log(d2.colors); // ['black', 'white'] ← 不再共享！

// d1.sayName(); // ❌ 报错！无法访问父类原型方法
```

**优点**：
1. 解决了引用类型共享问题
2. 可以向父类构造函数传参

**缺点**：
1. **无法继承父类原型上的方法**（`sayName` 不可用）
2. 每次创建实例都会重新创建父类中的方法，无法复用

---

### 方式三：组合继承（最常用的传统方式）

**原理**：原型链继承 + 构造函数继承的组合。

```js
function Animal(name) {
  this.name = name;
  this.colors = ['black', 'white'];
}
Animal.prototype.sayName = function () {
  console.log(this.name);
};

function Dog(name, breed) {
  Animal.call(this, name); // 第一次调用父类构造函数
  this.breed = breed;
}

Dog.prototype = new Animal(); // 第二次调用父类构造函数
Dog.prototype.constructor = Dog;

const d1 = new Dog('Rex', 'Husky');
const d2 = new Dog('Max', 'Poodle');

d1.colors.push('brown');
console.log(d2.colors); // ['black', 'white'] ✅
d1.sayName();           // Rex ✅
```

**优点**：
1. 解决了引用类型共享问题
2. 可以向父类传参
3. 可以继承父类原型方法

**缺点**：
- **父类构造函数被调用了两次**，子类原型上会存在多余的父类实例属性（被子类实例属性遮蔽，但仍占内存）

---

### 方式四：原型式继承

**原理**：基于已有对象创建新对象，不需要构造函数。ES5 的 `Object.create()` 就是这种思想的规范化。

```js
function createObject(proto) {
  function F() {}
  F.prototype = proto;
  return new F();
}

// 等价于 Object.create(proto)

const animal = {
  name: '动物',
  colors: ['black', 'white'],
  sayName() {
    console.log(this.name);
  }
};

const dog = createObject(animal);
dog.name = 'Rex';
dog.colors.push('brown');

const cat = createObject(animal);
console.log(cat.colors); // ['black', 'white', 'brown'] ← 引用类型仍然共享！
```

**优点**：
- 不需要构造函数，适合对象之间的简单继承

**缺点**：
- 引用类型属性仍然被共享（与原型链继承相同的问题）

---

### 方式五：寄生式继承

**原理**：在原型式继承的基础上，增强对象（添加方法），然后返回。

```js
function createDog(original) {
  const clone = Object.create(original); // 原型式继承
  // 增强对象
  clone.bark = function () {
    console.log('Woof!');
  };
  return clone;
}

const animal = { name: '动物', colors: ['black', 'white'] };
const dog = createDog(animal);
dog.bark(); // Woof!
```

**优点**：
- 可以在继承的基础上添加新方法

**缺点**：
- 方法在工厂函数中定义，无法复用（每次都创建新函数）
- 引用类型共享问题依然存在

---

### 方式六：寄生组合继承（最优方案 ⭐）

**原理**：用 `Object.create()` 替代 `new Parent()` 来设置子类原型，避免二次调用父类构造函数。

```js
function Animal(name) {
  this.name = name;
  this.colors = ['black', 'white'];
}
Animal.prototype.sayName = function () {
  console.log(this.name);
};

function Dog(name, breed) {
  Animal.call(this, name); // 只调用一次父类构造函数
  this.breed = breed;
}

// 核心：用 Object.create 创建原型，不调用父类构造函数
Dog.prototype = Object.create(Animal.prototype);
Dog.prototype.constructor = Dog; // 修正 constructor 指向

Dog.prototype.bark = function () {
  console.log('Woof!');
};

const d1 = new Dog('Rex', 'Husky');
const d2 = new Dog('Max', 'Poodle');

d1.colors.push('brown');
console.log(d2.colors); // ['black', 'white'] ✅
d1.sayName();           // Rex ✅
d1.bark();              // Woof! ✅
console.log(d1 instanceof Dog);    // true ✅
console.log(d1 instanceof Animal); // true ✅
```

**封装为通用函数**：

```js
function inheritPrototype(Child, Parent) {
  const prototype = Object.create(Parent.prototype); // 创建父类原型的副本
  prototype.constructor = Child;                     // 修正 constructor
  Child.prototype = prototype;                       // 赋值给子类原型
}

function Dog(name, breed) {
  Animal.call(this, name);
  this.breed = breed;
}

inheritPrototype(Dog, Animal);
```

**优点**：
1. 只调用一次父类构造函数 ✅
2. 原型链完整，`instanceof` 正常工作 ✅
3. 引用类型不共享 ✅
4. 可以向父类传参 ✅
5. 父类原型方法可以继承 ✅

**这是 ES6 `class extends` 的底层实现原理。**

---

## 实战案例

### ES6 class 与寄生组合继承的对比

```js
// ES6 class 写法
class Animal {
  constructor(name) {
    this.name = name;
    this.colors = ['black', 'white'];
  }
  sayName() {
    console.log(this.name);
  }
}

class Dog extends Animal {
  constructor(name, breed) {
    super(name); // 等价于 Animal.call(this, name)
    this.breed = breed;
  }
  bark() {
    console.log('Woof!');
  }
}

// Babel 编译后的核心逻辑（简化）：
// Dog.prototype = Object.create(Animal.prototype)
// Dog.prototype.constructor = Dog
// 在 Dog 构造函数中调用 Animal.call(this, name)
```

### 多层继承

```js
function Animal(name) {
  this.name = name;
}
Animal.prototype.eat = function () {
  console.log(`${this.name} is eating`);
};

function Dog(name, breed) {
  Animal.call(this, name);
  this.breed = breed;
}
Dog.prototype = Object.create(Animal.prototype);
Dog.prototype.constructor = Dog;
Dog.prototype.bark = function () {
  console.log('Woof!');
};

function GoldenRetriever(name) {
  Dog.call(this, name, 'Golden Retriever');
}
GoldenRetriever.prototype = Object.create(Dog.prototype);
GoldenRetriever.prototype.constructor = GoldenRetriever;
GoldenRetriever.prototype.fetch = function () {
  console.log(`${this.name} fetches the ball!`);
};

const buddy = new GoldenRetriever('Buddy');
buddy.eat();   // Buddy is eating
buddy.bark();  // Woof!
buddy.fetch(); // Buddy fetches the ball!

console.log(buddy instanceof GoldenRetriever); // true
console.log(buddy instanceof Dog);             // true
console.log(buddy instanceof Animal);          // true
```

---

## 底层原理

### Object.create 的实现

```js
// Object.create(proto) 的 polyfill
Object.create = function (proto) {
  function F() {}
  F.prototype = proto;
  return new F();
};
```

`Object.create(Animal.prototype)` 创建了一个新对象，该对象的 `__proto__` 指向 `Animal.prototype`，但**不会执行 `Animal` 的构造函数**，这正是寄生组合继承优于组合继承的关键。

### 原型链结构对比

```
组合继承的原型链：
Dog 实例
  └── __proto__ → Dog.prototype（含多余的 name、colors 属性）
        └── __proto__ → Animal.prototype
              └── __proto__ → Object.prototype
                    └── __proto__ → null

寄生组合继承的原型链：
Dog 实例（含 name、colors 属性）
  └── __proto__ → Dog.prototype（干净，只有 Dog 的方法）
        └── __proto__ → Animal.prototype（含 sayName）
              └── __proto__ → Object.prototype
                    └── __proto__ → null
```

### constructor 为什么要修正

```js
Dog.prototype = Object.create(Animal.prototype);
// 此时 Dog.prototype.constructor === Animal（错误！）

Dog.prototype.constructor = Dog; // 修正
// 现在 Dog.prototype.constructor === Dog（正确）

// 如果不修正：
const d = new Dog('Rex', 'Husky');
console.log(d.constructor === Dog);    // false（错误）
console.log(d.constructor === Animal); // true（错误）
```

---

## 高频面试题解析

### Q1：六种继承方式的优缺点总结

| 方式 | 引用类型共享 | 可传参 | 继承原型方法 | 调用父类构造函数次数 |
|------|------------|--------|------------|------------------|
| 原型链继承 | ❌ 共享 | ❌ | ✅ | 1次 |
| 构造函数继承 | ✅ 独立 | ✅ | ❌ | 1次 |
| 组合继承 | ✅ 独立 | ✅ | ✅ | **2次** |
| 原型式继承 | ❌ 共享 | ❌ | ✅ | 0次 |
| 寄生式继承 | ❌ 共享 | ❌ | ✅ | 0次 |
| **寄生组合继承** | ✅ 独立 | ✅ | ✅ | **1次** ⭐ |

---

### Q2：ES6 class extends 的底层是什么？

ES6 的 `class extends` 本质上是**寄生组合继承**的语法糖：

1. `super(args)` → `Parent.call(this, args)`（借用构造函数）
2. `Child.prototype = Object.create(Parent.prototype)` → 设置原型链
3. `Child.prototype.constructor = Child` → 修正 constructor
4. `Object.setPrototypeOf(Child, Parent)` → 静态方法继承（ES6 额外做的）

---

### Q3：为什么组合继承会调用两次父类构造函数？

```js
function Dog(name) {
  Animal.call(this, name); // 第一次：实例化时调用
}
Dog.prototype = new Animal(); // 第二次：设置原型时调用
```

第二次调用 `new Animal()` 时，`Animal` 构造函数执行，`name` 和 `colors` 被写入 `Dog.prototype`，造成原型上有多余的属性。

寄生组合继承用 `Object.create(Animal.prototype)` 替代 `new Animal()`，只复制原型，不执行构造函数，从而避免了第二次调用。

---

### Q4：如何判断属性是实例属性还是原型属性？

```js
const d = new Dog('Rex', 'Husky');

// hasOwnProperty：只检查实例自身属性
console.log(d.hasOwnProperty('name'));    // true（实例属性）
console.log(d.hasOwnProperty('sayName')); // false（原型属性）

// in 操作符：检查整条原型链
console.log('name' in d);    // true
console.log('sayName' in d); // true

// 只检查原型属性
function isPrototypeProperty(obj, name) {
  return !obj.hasOwnProperty(name) && (name in obj);
}
```

---

### Q5：手写寄生组合继承

```js
function Animal(name) {
  this.name = name;
  this.colors = ['black', 'white'];
}
Animal.prototype.sayName = function () {
  console.log(this.name);
};

function Dog(name, breed) {
  Animal.call(this, name);
  this.breed = breed;
}

// 寄生组合继承核心
function inheritPrototype(Child, Parent) {
  Child.prototype = Object.create(Parent.prototype);
  Child.prototype.constructor = Child;
}

inheritPrototype(Dog, Animal);

Dog.prototype.bark = function () {
  console.log('Woof!');
};

const d = new Dog('Rex', 'Husky');
d.sayName(); // Rex
d.bark();    // Woof!
console.log(d instanceof Dog);    // true
console.log(d instanceof Animal); // true
```

---

## 总结与扩展

### 六种继承方式演进路径

```
原型链继承（引用类型共享）
    ↓ 解决引用类型共享
构造函数继承（无法继承原型方法）
    ↓ 两者结合
组合继承（父类构造函数调用两次）
    ↓ 引入 Object.create
寄生组合继承（最优解 ⭐）
    ↓ 语法糖
ES6 class extends
```

### 扩展阅读

- **ES6 class 的静态方法继承**：`Object.setPrototypeOf(Dog, Animal)` 使子类可以继承父类的静态方法，这是 ES5 继承方案做不到的
- **Mixin 模式**：当需要多继承时，可以用 `Object.assign(Target.prototype, MixinA, MixinB)` 实现
- **TypeScript 中的继承**：TS 的 `extends` 编译后就是寄生组合继承，可以通过 `tsc --target ES5` 查看编译结果

### 记忆口诀

> **原型链共享有问题，构造函数借用解引用；**  
> **组合继承两次调，寄生组合一次搞；**  
> **Object.create 是关键，constructor 别忘改。**
