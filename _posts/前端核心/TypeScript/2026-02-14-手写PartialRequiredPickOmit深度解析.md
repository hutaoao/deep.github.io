---
title: "手写 Partial / Required / Pick / Omit"
date: 2026-02-14
categories: ["前端核心", TypeScript]
tags: ["TypeScript", "工具类型", "类型体操", "手写实现"]
description: "深入剖析TypeScript内置工具类型Partial/Required/Pick/Omit的底层实现原理，从零手写理解映射类型、条件类型、keyof等核心机制"
---

# 手写：Partial / Required / Pick / Omit

> TypeScript 内置工具类型是我们日常开发中最常用的利器；本文将深入剖析 `Partial<T>`、`Required<T>`、`Pick<T, K>`、`Omit<T, K>` 四个工具类型的底层实现原理，并带你从零手写这些工具类型，深入理解映射类型、keyof、条件类型等核心机制的协同工作方式。

---

## 一、概念定义

### 1.1 四个工具类型的作用

| 工具类型 | 作用 | 示例 |
|---------|------|------|
| `Partial<T>` | 将 T 的所有属性变为可选 | `Partial<User>` |
| `Required<T>` | 将 T 的所有属性变为必填 | `Required<Partial<User>>` |
| `Pick<T, K>` | 从 T 中挑选出 K 指定的属性 | `Pick<User, 'name' \| 'age'>` |
| `Omit<T, K>` | 从 T 中排除 K 指定的属性 | `Omit<User, 'email'>` |

### 1.2 基础类型准备

```typescript
interface User {
  id: number;
  name: string;
  age: number;
  email: string;
  isActive: boolean;
}

// 用于后续示例
const user: User = {
  id: 1,
  name: 'Alice',
  age: 25,
  email: 'alice@example.com',
  isActive: true
};
```

---

## 二、最小示例

### 2.1 Partial 最小示例

```typescript
// 内置 Partial
type PartialUser = Partial<User>;
// 等价于：
// {
//   id?: number;
//   name?: string;
//   age?: number;
//   email?: string;
//   isActive?: boolean;
// }

const updateUser = (id: number, updates: Partial<User>) => {
  // updates 中的所有字段都是可选的
  // 可以只传需要更新的字段
};

updateUser(1, { name: 'Bob' });       // ✅ 合法
updateUser(1, { age: 30, email: 'new@example.com' }); // ✅ 合法
```

### 2.2 Required 最小示例

```typescript
type MaybeUser = Partial<User>;
type DefinitelyUser = Required<MaybeUser>;
// 所有 ? 可选属性变回必填

const createUser = (user: Required<Partial<User>>) => {
  // user 现在所有字段都是必填的
};
```

### 2.3 Pick 最小示例

```typescript
// 只需要用户的名字和邮箱
type ContactInfo = Pick<User, 'name' | 'email'>;
// 等价于：
// {
//   name: string;
//   email: string;
// }

const sendEmail = (contact: Pick<User, 'name' | 'email'>) => {
  console.log(`Sending to ${contact.email}`);
};
```

### 2.4 Omit 最小示例

```typescript
// 排除敏感信息
type PublicUser = Omit<User, 'email'>;
// 等价于：
// {
//   id: number;
//   name: string;
//   age: number;
//   isActive: boolean;
// }

const displayUser = (user: Omit<User, 'email'>) => {
  // 无法访问 user.email
};
```

---

## 三、核心知识点拆解

### 3.1 映射类型（Mapped Types）基础

四个工具类型的核心都是**映射类型**：

```typescript
// 映射类型基本语法
type Mapped<T> = {
  [P in keyof T]?: T[P];
};
//     ↑          ↑     ↑
//   属性名    可选修饰符  属性值类型
```

- `keyof T`：获取 T 的所有属性名组成的联合类型
- `P in keyof T`：遍历每个属性名
- `T[P]`：获取属性 P 的类型
- `?`：可选修饰符（可加可不加）

### 3.2 readonly 与 ? 修饰符操作

TypeScript 支持通过前缀 `+` / `-` 来增删修饰符：

```typescript
// 移除可选属性
type RemoveOptional<T> = {
  [P in keyof T]-?: T[P];
};

// 移除 readonly
type Mutable<T> = {
  -readonly [P in keyof T]: T[P];
};

// 同时添加 readonly 和 ?
type ReadonlyPartial<T> = {
  +readonly [P in keyof T]+?: T[P];
};
```

> `+` 可以省略，`-` 不能省略。
> `?` 表示可选，`-?` 表示移除可选（变为必填）。
> `readonly` 表示只读，`-readonly` 表示移除只读。

### 3.3 keyof 与索引访问类型

```typescript
interface User {
  id: number;
  name: string;
}

type UserKeys = keyof User;    // "id" | "name"
type IdType = User['id'];      // number
type NameType = User['name'];  // string

// 联合类型索引访问
type SomeType = User['id' | 'name'];  // number | string
```

### 3.4 never 类型在 Omit 中的作用

`Omit<T, K>` 的实现利用了 `Exclude<K, keyof T>` 和 `never`：

```typescript
// 当 P 在 K 中时，返回 never，该属性被排除
type OmitHelper<T, K extends keyof T> = {
  [P in keyof T as P extends K ? never : P]: T[P];
};
```

---

## 四、手写实现

### 4.1 手写 Partial

```typescript
// 内置实现（简化版）
type Partial<T> = {
  [P in keyof T]?: T[P];
};

// 验证
interface Todo {
  title: string;
  description: string;
}

type PartialTodo = Partial<Todo>;
// 结果：
// {
//   title?: string;
//   description?: string;
// }
```

**手写要点：**
1. 使用 `keyof T` 获取所有属性名
2. 使用 `in` 遍历属性名
3. 添加 `?` 修饰符使属性变为可选
4. 值类型保持 `T[P]` 不变

### 4.2 手写 Required

```typescript
// 内置实现（简化版）
type Required<T> = {
  [P in keyof T]-?: T[P];
};

// 验证
type PartialUser = Partial<User>;
type RestoredUser = Required<PartialUser>;
// 所有属性恢复为必填
```

**手写要点：**
1. 使用 `-?` 移除可选修饰符
2. 其余与 Partial 对称

### 4.3 手写 Pick

```typescript
// 内置实现
type Pick<T, K extends keyof T> = {
  [P in K]: T[P];
};

// 验证
type NameAndAge = Pick<User, 'name' | 'age'>;
// 结果：
// {
//   name: string;
//   age: number;
// }
```

**手写要点：**
1. 泛型约束 `K extends keyof T` 确保 K 是 T 的属性子集
2. 映射 `P in K`（注意：不是 `P in keyof T`）
3. 值类型 `T[P]` 保持不变

### 4.4 手写 Omit

`Omit` 有两种实现方式：

**方式一：结合 Exclude（官方实现）**

```typescript
// 官方实现
type Omit<T, K extends keyof any> = Pick<T, Exclude<keyof T, K>>;

// 分步理解：
// 1. Exclude<keyof T, K> → 排除 K 后的属性名联合类型
// 2. Pick<T, ...> → 挑选出剩余属性
```

**方式二：使用 as 子句（TS 4.1+）**

```typescript
// 使用 as 子句的实现
type Omit<T, K extends keyof any> = {
  [P in keyof T as P extends K ? never : P]: T[P];
};

// 验证
type UserWithoutEmail = Omit<User, 'email'>;
// 结果：{ id, name, age, isActive }（不含 email）
```

**手写要点：**
1. `K extends keyof any` 约束 K 为 `string | number | symbol`
2. 使用 `as` 子句或 `Exclude` + `Pick` 组合来排除属性
3. `never` 类型的属性会被自动排除

### 4.5 完整手写实现（含注释）

```typescript
// ========== 手写 Partial ==========
type MyPartial<T> = {
  [P in keyof T]?: T[P];
};

// ========== 手写 Required ==========
type MyRequired<T> = {
  [P in keyof T]-?: T[P];
};

// ========== 手写 Pick ==========
type MyPick<T, K extends keyof T> = {
  [P in K]: T[P];
};

// ========== 手写 Omit（方式一：as 子句）==========
type MyOmit<T, K extends keyof any> = {
  [P in keyof T as P extends K ? never : P]: T[P];
};

// ========== 手写 Omit（方式二：Exclude + Pick）==========
type MyExclude<T, U> = T extends U ? never : T;

type MyOmit2<T, K extends keyof any> = MyPick<T, MyExclude<keyof T, K>>;

// ========== 验证 ==========
type Test1 = MyPartial<User>;
// ✅ 所有属性变为可选

type Test2 = MyRequired<Partial<User>>;
// ✅ 所有属性变为必填

type Test3 = MyPick<User, 'name' | 'age'>;
// ✅ 只有 name 和 age

type Test4 = MyOmit<User, 'email' | 'isActive'>;
// ✅ 排除了 email 和 isActive
```

---

## 五、实战案例

### 5.1 表单更新场景（Partial 实战）

```typescript
interface UserForm {
  username: string;
  password: string;
  nickname: string;
  avatar: string;
  bio: string;
}

// 更新表单：所有字段可选
type UpdateUserForm = Partial<UserForm>;

const updateProfile = (id: number, data: UpdateUserForm) => {
  // 只更新传入的字段
  return prisma.user.update({
    where: { id },
    data
  });
};

// 使用
updateProfile(1, { nickname: '新昵称' });  // ✅ 只更新昵称
```

### 5.2 API 响应封装（Pick 实战）

```typescript
interface ApiResponse<T> {
  code: number;
  message: string;
  data: T;
  timestamp: number;
  requestId: string;
}

// 前端只需要 code、message、data
type FrontendResponse<T> = Pick<ApiResponse<T>, 'code' | 'message' | 'data'>;

const handleResponse = (res: FrontendResponse<User>) => {
  if (res.code === 0) {
    console.log(res.data.name);
  }
};
```

### 5.3 排除敏感字段（Omit 实战）

```typescript
interface UserModel {
  id: number;
  name: string;
  email: string;
  passwordHash: string;    // 敏感信息
  salt: string;             // 敏感信息
  lastLoginIp: string;      // 敏感信息
  createdAt: Date;
}

// 返回给前端时排除敏感字段
type SafeUser = Omit<UserModel, 'passwordHash' | 'salt' | 'lastLoginIp'>;

const getUserApi = (id: number): SafeUser => {
  const user = db.findUser(id);
  return user;  // ✅ 类型安全，无法返回敏感字段
};
```

### 5.4 组合使用：更新 DTO

```typescript
// 创建用户 DTO（所有字段必填）
type CreateUserDto = Pick<User, 'name' | 'email' | 'age'>;

// 更新用户 DTO（所有字段可选，且排除 id）
type UpdateUserDto = Partial<Omit<Pick<User, 'name' | 'email' | 'age'>, 'age'>>;
// 等价于：{ name?: string; email?: string }
```

---

## 六、底层原理

### 6.1 TypeScript 源码中的实现

在 TypeScript 源码（`lib.es5.d.ts`）中：

```typescript
// Partial 源码
type Partial<T> = {
  [P in keyof T]?: T[P];
};

// Required 源码
type Required<T> = {
  [P in keyof T]-?: T[P];
};

// Readonly 源码
type Readonly<T> = {
  readonly [P in keyof T]: T[P];
};

// Pick 源码
type Pick<T, K extends keyof T> = {
  [P in K]: T[P];
};

// Omit 源码（通过 Pick + Exclude 实现）
type Exclude<T, U> = T extends U ? never : T;
type Omit<T, K extends keyof any> = Pick<T, Exclude<keyof T, K>>;
```

### 6.2 映射类型的编译过程

以 `Partial<User>` 为例：

```
1. keyof User → "id" | "name" | "age" | "email" | "isActive"
2. 遍历每个属性名 P：
   - P = "id"    → id?: number
   - P = "name"  → name?: string
   - P = "age"   → age?: number
   - P = "email" → email?: string
   - P = "isActive" → isActive?: boolean
3. 合并为一个新类型
```

### 6.3 条件类型在 Omit 中的角色

```typescript
// Exclude 的实现原理
type Exclude<T, U> = T extends U ? never : T;

// 示例：Exclude<'a' | 'b' | 'c', 'a' | 'c'>
// 分布式条件类型：
//   'a' extends 'a' | 'c' ? never : 'a'  → never
//   'b' extends 'a' | 'c' ? never : 'b'  → 'b'
//   'c' extends 'a' | 'c' ? never : 'c'  → never
// 结果：'b'
```

### 6.4 为什么 Omit 不用直接映射？

```typescript
// 尝试直接实现 Omit（错误方式）
type WrongOmit<T, K extends keyof T> = {
  [P in keyof T]: T[P];  // ❌ 无法排除属性
};

// 正确方式：使用 as 子句（TS 4.1+）
type RightOmit<T, K extends keyof any> = {
  [P in keyof T as P extends K ? never : P]: T[P];
};
```

---

## 七、面试题解析

### 7.1 手写 Partial（高频）

**题目**：手写实现 TypeScript 的 `Partial<T>` 工具类型。

```typescript
// 答案
type MyPartial<T> = {
  [P in keyof T]?: T[P];
};
```

**追问**：如何实现 `PartialByKeys<T, K>`，只将指定属性变为可选？

```typescript
type PartialByKeys<T, K extends keyof T> = {
  [P in keyof T as P extends K ? P : never]?: T[P];
} & {
  [P in keyof T as P extends K ? never : P]: T[P];
};
// 更优解：
type PartialByKeys<T, K extends keyof T> = Omit<T, K> & Partial<Pick<T, K>>;
```

### 7.2 Pick 和 Omit 的区别？

| 特性 | Pick | Omit |
|------|------|------|
| 作用 | 挑选指定属性 | 排除指定属性 |
| 泛型参数 | `Pick<T, K>` | `Omit<T, K>` |
| K 的约束 | `K extends keyof T` | `K extends keyof any` |
| 实现方式 | 直接映射 `P in K` | `Pick<T, Exclude<keyof T, K>>` |

### 7.3 实现一个 RequiredByKeys

**题目**：只将指定属性变为必填。

```typescript
type RequiredByKeys<T, K extends keyof T> = Omit<T, K> & Required<Pick<T, K>>;

// 验证
type Test = RequiredByKeys<Partial<User>, 'id' | 'name'>;
// { id: number; name: string; age?: number; email?: string; isActive?: boolean }
```

### 7.4 实现一个 Mutable（移除 readonly）

```typescript
type Mutable<T> = {
  -readonly [P in keyof T]: T[P];
};

// 验证
interface ReadonlyUser {
  readonly id: number;
  readonly name: string;
}

type MutableUser = Mutable<ReadonlyUser>;
// { id: number; name: string }  （不再 readonly）
```

### 7.5 深度 Partial（递归 Partial）

**题目**：实现 `DeepPartial<T>`，将所有嵌套属性也变为可选。

```typescript
type DeepPartial<T> = {
  [P in keyof T]?: T[P] extends object
    ? DeepPartial<T[P]>
    : T[P];
};

// 验证
interface Nested {
  a: {
    b: {
      c: number;
    };
  };
}

type Test = DeepPartial<Nested>;
// { a?: { b?: { c?: number } } }
```

---

## 八、总结

### 8.1 核心要点回顾

| 工具类型 | 核心机制 | 关键语法 |
|---------|---------|---------|
| `Partial<T>` | 映射类型 + 可选修饰符 | `[P in keyof T]?: T[P]` |
| `Required<T>` | 映射类型 + 移除可选 | `[P in keyof T]-?: T[P]` |
| `Pick<T, K>` | 映射类型 + 泛型约束 | `[P in K]: T[P]` |
| `Omit<T, K>` | Exclude + Pick 或 as 子句 | `as P extends K ? never : P` |

### 8.2 记忆口诀

> **Partial 加 `?`，Required 减 `?`**
> **Pick 挑 K，Omit 排 K**
> **映射类型是核心，keyof 遍历属性名**

### 8.3 实战选择指南

- 更新场景 → `Partial<T>`（部分更新）
- 创建场景 → `Required<T>`（确保必填）
- 字段精选 → `Pick<T, K>`（只要某些字段）
- 字段排除 → `Omit<T, K>`（排除敏感/无关字段）
- 组合使用 → `Partial<Pick<T, K>>`（部分字段可选）

---

## 参考资料

- [TypeScript 官方文档 - 映射类型](https://www.typescriptlang.org/docs/handbook/2/mapped-types.html)
- [TypeScript 官方文档 - 工具类型](https://www.typescriptlang.org/docs/handbook/utility-types.html)
- [TypeScript GitHub - lib.es5.d.ts](https://github.com/microsoft/TypeScript/blob/main/lib/lib.es5.d.ts)
