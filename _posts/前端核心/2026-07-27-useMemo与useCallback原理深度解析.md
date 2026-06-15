---
title: useMemo与useCallback原理深度解析
date: 2026-07-27
categories: [前端核心, React]
tags: [前端, React, Hooks, useMemo, useCallback]
description: 深度解析React中两个缓存类Hooks——useMemo与useCallback——的核心原理，涵盖依赖项比较机制、缓存策略、Fiber节点上的存储结构、以及在真实项目中的适用场景和误用分析。
---

## 一句话概括

useMemo 和 useCallback 是 React 提供的两个"值缓存" Hook，通过在 Fiber 节点的 memoizedState 链表中存储上一次的计算结果和依赖数组，在每次渲染时通过浅比较依赖数组来判断缓存是否过期——useMemo 缓存任意计算值，useCallback 缓存函数引用，两者都用于减少不必要的子组件重渲染。

## 背景与意义

### 为什么需要缓存？

在 React 中，每次组件渲染时，组件体内的所有代码都会重新执行一遍：

```jsx
function ExpensiveComponent({ data }) {
  // 每次渲染都会重新计算！
  const sortedData = data.sort((a, b) => b.value - a.value);
  
  // 每次渲染都会创建新的函数！
  const handleClick = () => {
    console.log('Clicked:', sortedData);
  };
  
  return <Child onClick={handleClick} data={sortedData} />;
}
```

这带来了两个问题：

**问题一：无意义的重复计算。** 如果 `data` 没有变化，`sortedData` 的结果和上一次是一样的，重新计算浪费 CPU。

**问题二：无意义的子组件重渲染。** `handleClick` 每次都是"新"的函数引用，导致 `React.memo` 包裹的 `<Child>` 组件在浅比较 `onClick` 时认为 props 变了，触发不必要的重渲染。

### React 的"宁可多做也不要做错"哲学

React 的默认行为是：**每次渲染都重新计算所有值、重新创建所有函数。** 这是"安全的默认值"——因为只有在重新计算后才能保证值是最新的。

但 React 也提供了"逃生舱"：如果你能确定输入没变，就可以用 `useMemo` 和 `useCallback` 告诉 React "跳过这次计算，用上次的结果"。

### 与 Vue/Svelte 的区别

在 Vue 的响应式系统中，计算属性（computed）天然是懒求值的，当依赖的响应式数据没有变化时，计算属性不会重新计算。在 Svelte 编译时系统中，反应性声明（`$:`）在编译器层面做依赖追踪。

React 采用了不同的方式——**手动声明依赖数组**。这意味着开发者必须显式指出"这个值依赖于什么"。这是一个设计决策：`显式依赖声明 < 隐式依赖追踪`，但这也带来了依赖数组写错的常见陷阱。

## 概念与定义

### useMemo

返回一个**记忆化值**，仅在依赖项变化时重新计算：

```typescript
type useMemo<T> = (
  factory: () => T,      // 创建值的工厂函数
  deps: any[] | null | void,  // 依赖数组
) => T;
```

```jsx
// useMemo 的使用方式
const sortedList = useMemo(() => {
  return largeList.sort((a, b) => b.value - a.value);
}, [largeList]);  // 仅当 largeList 变化时重新排序
```

### useCallback

返回一个**记忆化函数引用**，仅在依赖项变化时更新：

```typescript
type useCallback<T extends (...args: any[]) => any> = (
  callback: T,           // 回调函数
  deps: any[] | null | void,  // 依赖数组
) => T;
```

```jsx
// useCallback 的使用方式
const handleSubmit = useCallback((data) => {
  // 使用 name 和 age
  saveToAPI({ name, age, ...data });
}, [name, age]);  // 仅当 name 或 age 变化时创建新函数
```

### Fiber 上的存储结构

useMemo 和 useCallback 的值存储在 FunctionComponent Fiber 节点的 **memoizedState 链表**中，这是一个按照 Hook 调用顺序串联的单向链表：

```typescript
// Fiber 节点上的 Hook 链表结构
type Hook = {
  memoizedState: any,      // 缓存的值（useMemo 的结果 / useCallback 的函数）
  baseState: any,          // 基础状态
  baseQueue: Update<any, any> | null,
  queue: UpdateQueue<any, any> | null,
  next: Hook | null,        // 指向下一个 Hook
};
```

每个 Hook 在链表中都有一个固定位置（由调用顺序决定），这就是 **Hook 调用规则**（不能在条件/循环中调用 Hook）的来源。

### 依赖数组（Dependencies Array）

```typescript
type Deps = ReadonlyArray<any> | null;

// 依赖数组的浅比较
function areHookInputsEqual(
  nextDeps: Deps,
  prevDeps: Deps,
): boolean {
  if (prevDeps === null) return false;
  
  // Object.is 比较每个依赖项
  for (let i = 0; i < prevDeps.length && i < nextDeps.length; i++) {
    if (Object.is(nextDeps[i], prevDeps[i])) {
      continue;
    }
    return false;
  }
  return true;
}
```

## 核心知识点拆解

### 1. 内部实现：mount 与 update 阶段

像所有 Hook 一样，useMemo 和 useCallback 在 **mount（首次渲染）** 和 **update（后续渲染）** 阶段有不同的行为：

**mount 阶段（首次渲染）：**

```typescript
function mountMemo<T>(factory: () => T, deps: Deps): T {
  // 新建 Hook 节点
  const hook = mountWorkInProgressHook();
  
  // 计算初始值
  const nextValue = factory();
  
  // 存储到 Hook.memoizedState
  hook.memoizedState = [nextValue, deps];
  
  return nextValue;
}

function mountCallback<T>(callback: T, deps: Deps): T {
  // 新建 Hook 节点
  const hook = mountWorkInProgressHook();
  
  // 直接存储函数和依赖
  hook.memoizedState = [callback, deps];
  
  return callback;
}
```

**update 阶段（后续渲染）：**

```typescript
function updateMemo<T>(factory: () => T, deps: Deps): T {
  const hook = updateWorkInProgressHook();
  const nextDeps = deps === undefined ? null : deps;
  
  // 获取上一次存储的值和依赖
  const prevState = hook.memoizedState;
  
  if (prevState !== null) {
    const prevDeps = prevState[1]; // 上一次的依赖数组
    
    // 比较依赖是否变化
    if (nextDeps !== null && areHookInputsEqual(nextDeps, prevDeps)) {
      // 依赖没变 → 返回缓存的值
      return prevState[0];
    }
  }
  
  // 依赖变了 → 重新计算
  const nextValue = factory();
  hook.memoizedState = [nextValue, nextDeps];
  return nextValue;
}

function updateCallback<T>(callback: T, deps: Deps): T {
  const hook = updateWorkInProgressHook();
  const nextDeps = deps === undefined ? null : deps;
  
  // 获取上一次存储的值和依赖
  const prevState = hook.memoizedState;
  
  if (prevState !== null) {
    const prevDeps = prevState[1];
    
    // 比较依赖
    if (nextDeps !== null && areHookInputsEqual(nextDeps, prevDeps)) {
      // 依赖没变 → 返回上一次的函数引用
      return prevState[0];
    }
  }
  
  // 依赖变了 → 存储新的函数引用
  hook.memoizedState = [callback, nextDeps];
  return callback;
}
```

关键观察：**useCallback(fn, deps) 本质上等价于 useMemo(() => fn, deps)**。两者的底层实现几乎完全相同，只是 useMemo 的 factory 函数会被执行，而 useCallback 直接存储传入的函数。

### 2. useMemo 与 useCallback 的存储结构对比

```
useMemo(() => computedValue, [a, b]):

Hook.memoizedState = [
  computedValue,     // 工厂函数计算的结果
  [a, b]             // 依赖数组
]

useCallback(handleFn, [a, b]):

Hook.memoizedState = [
  handleFn,          // 函数本身
  [a, b]             // 依赖数组
]
```

两者在 Fiber 上的存储格式完全一致（都是一个 `[value, deps]` 数组），区别只是 value 的来源不同。

### 3. 依赖数组的浅比较(Object.is)

```typescript
// React 使用的比较函数
function areHookInputsEqual(nextDeps: Array<mixed>, prevDeps: Array<mixed>): boolean {
  if (__DEV__) {
    // 开发环境下检查依赖数量
    checkDepsLength(prevDeps, nextDeps);
  }
  
  for (let i = 0; i < prevDeps.length && i < nextDeps.length; i++) {
    // 使用 Object.is 进行"浅比较"
    if (Object.is(nextDeps[i], prevDeps[i])) {
      continue;
    }
    return false;
  }
  
  return true;
}
```

浅比较意味着：
```javascript
Object.is(1, 1)         // true → 基本类型值相同认为没变
Object.is('abc', 'abc') // true
Object.is({}, {})       // false → 新对象引用不相同！
Object.is([], [])       // false
```

所以如果依赖项是对象/数组/函数，每次渲染创建新引用时，浅比较会始终返回 false，useMemo/useCallback 每次都会重新计算。

## 最小示例

{% raw %}
```jsx
import React, { useState, useMemo, useCallback } from 'react';

function MemoAndCallbackDemo() {
  const [count, setCount] = useState(0);
  const [items, setItems] = useState(['A', 'B', 'C', 'D', 'E']);
  const [filter, setFilter] = useState('');

  // --- useMemo 示例：缓存计算值 ---
  // 不缓存的情况：每次渲染都重新过滤
  // const filteredItems = items.filter(item => item.includes(filter));

  // 缓存的情况：只有 items 或 filter 变了才重新计算
  const filteredItems = useMemo(
    () => {
      console.log('🔄 useMemo 重新计算过滤结果');
      if (!filter) return items;
      return items.filter(item => 
        item.toLowerCase().includes(filter.toLowerCase())
      );
    },
    [items, filter]
  );

  // --- useCallback 示例：缓存函数引用 ---
  // 不缓存的情况：每次渲染都创建新的 handleAdd 函数
  // const handleAdd = () => setItems(prev => [...prev, `Item ${prev.length + 1}`]);

  // 缓存的情况：依赖为空 → 生命周期内引用不变
  const handleAdd = useCallback(() => {
    console.log('➕ 添加新项');
    setItems(prev => [...prev, `Item ${prev.length + 1}`]);
  }, []);  // setItems 是稳定的，不需要放在依赖中

  // 缓存依赖于 count 的情况
  const handleIncrement = useCallback(() => {
    console.log('🔢 递增（当前 count 将变为：', count + 1, '）');
    setCount(c => c + 1);  // 使用函数式更新，不需要依赖 count
  }, []);  // ✅ 使用函数式更新，count 不用写在 deps 里

  const handleNotifyCount = useCallback(() => {
    alert(`当前 count: ${count}`);
  }, [count]);  // 因为需要读取 count 的最新值，必须依赖

  return (
    <div style={{ fontFamily: 'system-ui, sans-serif', maxWidth: 500 }}>
      <h3>📦 useMemo & useCallback 演示</h3>

      <div style={{ marginBottom: 16 }}>
        <strong>计数：{count}</strong>
        <div style={{ marginTop: 8, display: 'flex', gap: 8 }}>
          <button onClick={handleIncrement} style={btnStyle}>+1</button>
          <button onClick={handleNotifyCount} style={btnStyle}>
            弹出 count
          </button>
          <button onClick={() => setCount(0)} style={btnStyle}>
            重置
          </button>
        </div>
      </div>

      <div style={{ marginBottom: 12 }}>
        <input
          value={filter}
          onChange={e => setFilter(e.target.value)}
          placeholder="过滤列表..."
          style={{
            padding: '6px 10px', borderRadius: 6, border: '1px solid #ccc',
            width: 200, fontSize: 14,
          }}
        />
        <button onClick={handleAdd} style={{ ...btnStyle, marginLeft: 8 }}>
          新增
        </button>
      </div>

      <div style={{ border: '1px solid #e0e0e0', borderRadius: 8, padding: 12 }}>
        <p style={{ margin: '0 0 8px', fontSize: 13, color: '#666' }}>
          过滤后的列表（共 {filteredItems.length} 项）：
        </p>
        <ul style={{ paddingLeft: 20, margin: 0 }}>
          {filteredItems.map((item, idx) => (
            <li key={idx} style={{ margin: '4px 0' }}>{item}</li>
          ))}
        </ul>
      </div>

      <UpdateLog />
    </div>
  );
}

function UpdateLog() {
  const [logs, setLogs] = useState([]);
  const [trigger, setTrigger] = useState(0);

  // 使用 React DevTools 来观察
  // 或者在控制台查看 useMemo 重新计算的日志
  return (
    <div style={{ marginTop: 12, fontSize: 12, color: '#999' }}>
      <p>💡 打开控制台查看 "🔄 useMemo 重新计算过滤结果" 日志</p>
      <p>当只点击 +1 时（items 和 filter 不变），useMemo 不会重新计算</p>
      <p>当新增或输入过滤时，useMemo 会重新计算</p>
    </div>
  );
}

const btnStyle = {
  padding: '6px 14px',
  border: '1px solid #ccc',
  borderRadius: 6,
  background: '#fff',
  cursor: 'pointer',
  fontSize: 13,
};

export default MemoAndCallbackDemo;
```
{% endraw %}

## 实战案例（含完整代码）

### 案例：性能敏感型数据表格

一个包含数据排序、过滤、页计算的表格组件，展示 useMemo 和 useCallback 在性能敏感场景中的正确用法：

{% raw %}
```jsx
import React, { useState, useMemo, useCallback, memo } from 'react';

// 生成测试数据
function generateData(count = 1000) {
  const departments = ['工程', '产品', '设计', '市场', '人事', '财务', '运营'];
  const statuses = ['活跃', '离线', '忙碌'];
  
  return Array.from({ length: count }, (_, i) => ({
    id: i + 1,
    name: `员工_${String(i + 1).padStart(4, '0')}`,
    age: 20 + Math.floor(Math.random() * 40),
    department: departments[Math.floor(Math.random() * departments.length)],
    salary: 8000 + Math.floor(Math.random() * 42000),
    status: statuses[Math.floor(Math.random() * statuses.length)],
    joinDate: new Date(2020, Math.floor(Math.random() * 40), 1 + Math.floor(Math.random() * 28))
      .toISOString().slice(0, 10),
  }));
}

// memo 包裹的子组件：只关心收到的 props 是否变化
const TableRow = memo(function TableRow({ row, isSelected, onSelect, onToggleStatus }) {
  return (
    <tr
      style={{
        background: isSelected ? '#e8f0fe' : (row.id % 2 === 0 ? '#fafafa' : '#fff'),
        cursor: 'pointer',
        transition: 'background 0.15s',
      }}
      onClick={() => onSelect(row.id)}
    >
      <td>{row.id}</td>
      <td><strong>{row.name}</strong></td>
      <td>{row.age}</td>
      <td>{row.department}</td>
      <td>{row.salary.toLocaleString()}</td>
      <td>
        <span style={{
          display: 'inline-block',
          padding: '2px 8px',
          borderRadius: 12,
          fontSize: 12,
          background: row.status === '活跃' ? '#e8f5e9' :
                      row.status === '忙碌' ? '#fff3e0' : '#f5f5f5',
          color: row.status === '活跃' ? '#2e7d32' :
                 row.status === '忙碌' ? '#e65100' : '#757575',
        }}>
          {row.status}
        </span>
      </td>
      <td>{row.joinDate}</td>
      <td>
        <button
          onClick={(e) => {
            e.stopPropagation();
            onToggleStatus(row.id);
          }}
          style={{
            padding: '2px 8px',
            border: '1px solid #ccc',
            borderRadius: 4,
            background: '#fff',
            cursor: 'pointer',
            fontSize: 11,
          }}
        >
          切换
        </button>
      </td>
    </tr>
  );
});

function DataTable() {
  const [data] = useState(() => generateData(1000));
  const [department, setDepartment] = useState('');
  const [status, setStatus] = useState('');
  const [sortColumn, setSortColumn] = useState('id');
  const [sortDirection, setSortDirection] = useState('asc');
  const [selectedIds, setSelectedIds] = useState(new Set());
  const [page, setPage] = useState(0);
  const PAGE_SIZE = 20;

  // --- 筛选：data 不变时不会重新筛选 ---
  const filteredData = useMemo(() => {
    console.log('📊 useMemo: 筛选数据');
    let result = data;
    
    if (department) {
      result = result.filter(row => row.department === department);
    }
    if (status) {
      result = result.filter(row => row.status === status);
    }
    
    return result;
  }, [data, department, status]);

  // --- 排序：筛选后数据不变时不会重新排序 ---
  const sortedData = useMemo(() => {
    console.log('📊 useMemo: 排序数据');
    const sorted = [...filteredData].sort((a, b) => {
      const aVal = a[sortColumn];
      const bVal = b[sortColumn];
      const cmp = typeof aVal === 'number' ? aVal - bVal : String(aVal).localeCompare(String(bVal));
      return sortDirection === 'asc' ? cmp : -cmp;
    });
    return sorted;
  }, [filteredData, sortColumn, sortDirection]);

  // --- 分页：不需要缓存，直接切片 ---
  const pageData = useMemo(() => {
    console.log('📊 useMemo: 分页切片');
    const start = page * PAGE_SIZE;
    return sortedData.slice(start, start + PAGE_SIZE);
  }, [sortedData, page]);

  // --- 统计信息：缓存汇总数据 ---
  const stats = useMemo(() => {
    console.log('📊 useMemo: 计算统计信息');
    const total = filteredData.length;
    const totalSalary = filteredData.reduce((sum, r) => sum + r.salary, 0);
    const avgSalary = total > 0 ? Math.round(totalSalary / total) : 0;
    const maxSalary = total > 0 ? Math.max(...filteredData.map(r => r.salary)) : 0;
    return { total, avgSalary, maxSalary };
  }, [filteredData]);

  // --- useCallback 缓存回调 ---
  const handleSort = useCallback((column) => {
    setSortColumn(prev => {
      if (prev === column) {
        setSortDirection(dir => dir === 'asc' ? 'desc' : 'asc');
        return prev;
      }
      setSortDirection('asc');
      return column;
    });
  }, []);

  const handleSelect = useCallback((id) => {
    setSelectedIds(prev => {
      const next = new Set(prev);
      if (next.has(id)) {
        next.delete(id);
      } else {
        next.add(id);
      }
      return next;
    });
  }, []);

  const handleToggleStatus = useCallback((id) => {
    // 实际应用中会 dispatch action 修改数据
    console.log('切换状态：', id);
  }, []);

  const totalPages = Math.ceil(sortedData.length / PAGE_SIZE);

  // 提取所有部门（只需计算一次）
  const departments = useMemo(
    () => [...new Set(data.map(row => row.department))],
    [data]
  );

  return (
    <div style={{ fontFamily: 'system-ui, sans-serif', maxWidth: 900, margin: '0 auto' }}>
      <h3>📋 员工数据表（性能优化示例）</h3>

      {/* 筛选栏 */}
      <div style={{ display: 'flex', gap: 12, marginBottom: 12, alignItems: 'center' }}>
        <select
          value={department}
          onChange={e => { setDepartment(e.target.value); setPage(0); }}
          style={selectStyle}
        >
          <option value="">全部部门</option>
          {departments.map(d => (
            <option key={d} value={d}>{d}</option>
          ))}
        </select>
        <select
          value={status}
          onChange={e => { setStatus(e.target.value); setPage(0); }}
          style={selectStyle}
        >
          <option value="">全部状态</option>
          <option value="活跃">活跃</option>
          <option value="忙碌">忙碌</option>
          <option value="离线">离线</option>
        </select>

        <span style={{ marginLeft: 'auto', fontSize: 13, color: '#666' }}>
          共 {stats.total} 条 · 平均薪资 ¥{stats.avgSalary.toLocaleString()} · 
          最高 ¥{stats.maxSalary.toLocaleString()}
        </span>
      </div>

      {/* 表格 */}
      <div style={{ overflowX: 'auto', border: '1px solid #e0e0e0', borderRadius: 8 }}>
        <table style={{ width: '100%', borderCollapse: 'collapse', fontSize: 13 }}>
          <thead>
            <tr style={{ background: '#f5f5f5' }}>
              {['id', 'name', 'age', 'department', 'salary', 'status', 'joinDate'].map(col => (
                <th
                  key={col}
                  onClick={() => handleSort(col)}
                  style={{
                    padding: '8px 12px',
                    textAlign: 'left',
                    cursor: 'pointer',
                    userSelect: 'none',
                    borderBottom: sortColumn === col ? '2px solid #1a73e8' : '2px solid transparent',
                  }}
                >
                  {col === 'id' ? 'ID' :
                   col === 'name' ? '姓名' :
                   col === 'age' ? '年龄' :
                   col === 'department' ? '部门' :
                   col === 'salary' ? '薪资' :
                   col === 'status' ? '状态' : '入职日期'}
                  {sortColumn === col && (sortDirection === 'asc' ? ' ▲' : ' ▼')}
                </th>
              ))}
              <th style={{ padding: '8px 12px' }}>操作</th>
            </tr>
          </thead>
          <tbody>
            {pageData.length === 0 ? (
              <tr><td colSpan={8} style={{ padding: 24, textAlign: 'center', color: '#999' }}>暂无数据</td></tr>
            ) : (
              pageData.map(row => (
                <TableRow
                  key={row.id}
                  row={row}
                  isSelected={selectedIds.has(row.id)}
                  onSelect={handleSelect}
                  onToggleStatus={handleToggleStatus}
                />
              ))
            )}
          </tbody>
        </table>
      </div>

      {/* 分页 */}
      <div style={{ display: 'flex', justifyContent: 'center', gap: 4, marginTop: 12 }}>
        <button onClick={() => setPage(0)} disabled={page === 0} style={pageBtnStyle}>首页</button>
        <button onClick={() => setPage(p => Math.max(0, p - 1))} disabled={page === 0} style={pageBtnStyle}>上一页</button>
        <span style={{ padding: '4px 12px', fontSize: 13, color: '#666' }}>
          第 {page + 1} / {totalPages} 页
        </span>
        <button onClick={() => setPage(p => Math.min(totalPages - 1, p + 1))} disabled={page >= totalPages - 1} style={pageBtnStyle}>下一页</button>
        <button onClick={() => setPage(totalPages - 1)} disabled={page >= totalPages - 1} style={pageBtnStyle}>末页</button>
      </div>

      {/* 性能提示 */}
      <div style={{
        marginTop: 12,
        background: '#f0f7ff',
        border: '1px solid #bbdefb',
        borderRadius: 8,
        padding: 12,
        fontSize: 12,
        color: '#555',
      }}>
        <strong>💡 性能优化说明：</strong>
        <ul style={{ margin: '4px 0 0 0', paddingLeft: 20 }}>
          <li><strong>筛选、排序、分页、统计</strong>：都用了 useMemo，只有依赖变化时才重新计算</li>
          <li><strong>TableRow</strong>：被 React.memo 包裹，props 不变时跳过重渲染</li>
          <li><strong>handleSelect/handleToggleStatus</strong>：用 useCallback 缓存，不会导致 TableRow 误重渲染</li>
          <li>点击"切换"按钮或"选择行"时，其余行不会重渲染</li>
        </ul>
      </div>
    </div>
  );
}

const selectStyle = {
  padding: '6px 10px',
  borderRadius: 6,
  border: '1px solid #ccc',
  fontSize: 13,
};

const pageBtnStyle = {
  padding: '4px 12px',
  border: '1px solid #ccc',
  borderRadius: 4,
  background: '#fff',
  cursor: 'pointer',
  fontSize: 12,
};

export default DataTable;
```
{% endraw %}

## 底层原理（含源码分析）

### 1. areHookInputsEqual 的详细实现

```typescript
// 源码位置：packages/react-reconciler/src/ReactFiberHooks.js

function areHookInputsEqual(
  nextDeps: Array<mixed>,
  prevDeps: Array<mixed> | null,
): boolean {
  if (prevDeps === null) {
    // 开发模式下有警告：依赖为 null 表示每次都重新计算
    return false;
  }
  
  // React 在开发环境下会检查依赖数组的长度一致性
  // 如果前后长度不同，打印警告
  if (__DEV__) {
    if (nextDeps.length !== prevDeps.length) {
      console.error(
        'The final argument passed to useMemo changed size between renders. ' +
        'The order and size of this array must remain constant.\n\n' +
        'Previous: %s\n' +
        'Incoming: %s',
        `[${prevDeps.join(', ')}]`,
        `[${nextDeps.join(', ')}]`,
      );
    }
  }
  
  // 逐个使用 Object.is 比较
  for (let i = 0; i < prevDeps.length && i < nextDeps.length; i++) {
    if (Object.is(nextDeps[i], prevDeps[i])) {
      continue;
    }
    return false;
  }
  
  return true;
}
```

### 2. mountWorkInProgressHook 与 updateWorkInProgressHook

这两个函数是所有 Hook 的基础，负责在 Fiber 的 memoizedState 链表中创建和定位 Hook 节点：

```typescript
// mount 阶段：创建新的 Hook 节点
function mountWorkInProgressHook(): Hook {
  const hook: Hook = {
    memoizedState: null,
    baseState: null,
    baseQueue: null,
    queue: null,
    next: null,
  };
  
  // 将新 Hook 连接到最后
  if (workInProgressHook === null) {
    // 第一个 Hook
    currentlyRenderingFiber.memoizedState = workInProgressHook = hook;
  } else {
    // 后续 Hook → 追加到链表末尾
    workInProgressHook = workInProgressHook.next = hook;
  }
  
  return workInProgressHook;
}

// update 阶段：定位到当前 Hook 位置
function updateWorkInProgressHook(): Hook {
  let nextCurrentHook: Hook | null;
  
  if (currentHook === null) {
    // 当前 Fiber 上的第一个 Hook
    const current = currentlyRenderingFiber.alternate;
    nextCurrentHook = current !== null ? current.memoizedState : null;
  } else {
    nextCurrentHook = currentHook.next;
  }
  
  // 更新 workInProgress 链
  let nextWorkInProgressHook: Hook | null;
  if (workInProgressHook === null) {
    nextWorkInProgressHook = currentlyRenderingFiber.memoizedState;
  } else {
    nextWorkInProgressHook = workInProgressHook.next;
  }
  
  // ... 确保顺序一致 ...
  
  // 定位到当前 Hook
  currentHook = nextCurrentHook;
  const newHook: Hook = {
    memoizedState: currentHook.memoizedState,
    baseState: currentHook.baseState,
    baseQueue: currentHook.baseQueue,
    queue: currentHook.queue,
    next: null,
  };
  
  // 连接到 workInProgress 链
  if (workInProgressHook === null) {
    currentlyRenderingFiber.memoizedState = workInProgressHook = newHook;
  } else {
    workInProgressHook = workInProgressHook.next = newHook;
  }
  
  return workInProgressHook;
}
```

### 3. React.memo 与 useCallback 的配合

`React.memo` 内部使用 `checkPropTypes` 进行 props 的浅比较。当 useCallback 缓存了函数引用时，React.memo 在比较 props 时就能判断 `onClick` 没有变化：

```typescript
// React.memo 的简化实现
function memo(type, compare) {
  return {
    $$typeof: REACT_MEMO_TYPE,
    type,
    compare: compare === undefined ? shallowEqual : compare,
  };
}

// shallowEqual 的比较
function shallowEqual(objA: mixed, objB: mixed): boolean {
  if (Object.is(objA, objB)) {
    return true;
  }
  
  if (typeof objA !== 'object' || objA === null ||
      typeof objB !== 'object' || objB === null) {
    return false;
  }
  
  const keysA = Object.keys(objA);
  const keysB = Object.keys(objB);
  
  if (keysA.length !== keysB.length) return false;
  
  for (let i = 0; i < keysA.length; i++) {
    if (!hasOwnProperty.call(objB, keysA[i]) ||
        !Object.is(objA[keysA[i]], objB[keysA[i]])) {
      return false;
    }
  }
  
  return true;
}
```

所以当 `useCallback` 返回相同的函数引用时，React.memo 的 shallowEqual 检查 `onClick={handleClick}` 时会认为前后一致，跳过子组件的重渲染。

## 高频面试题解析

### Q1：useMemo 和 useCallback 有什么区别？什么时候该用哪个？

**参考答案：**

**本质区别：**
- `useMemo` 缓存**计算的值**（可以是任意类型的值）
- `useCallback` 缓存**函数引用**（本质上是 `useMemo(() => fn, deps)` 的语法糖）

**选择原则：**

| 使用场景 | 使用什么 | 示例 |
|---------|---------|------|
| 需要缓存计算结果（大数据排序、过滤） | `useMemo` | `useMemo(() => expensiveCalculation(data), [data])` |
| 需要将函数传给子组件（配合 React.memo） | `useCallback` | `useCallback(() => doSomething(id), [id])` |
| 在 useEffect 中作为依赖 | `useCallback` | `useCallback(fn, [deps])` |
| 既不用传子组件也不用作为依赖 | **什么都不用** | 直接定义在组件体内即可 |

**常见误解：**
- ❌ "所有函数都应该用 useCallback 包裹"——不必要，函数每次创建的成本远低于 useCallback 本身的比较成本
- ❌ "useMemo 能提高所有计算的速度"——只有计算成本确实高（如 O(n²) 以上）时才值得
- ✅ "useCallback 和 useMemo 的主要价值是减少子组件重渲染，不是减少自身运算"

### Q2：依赖数组写错了会有什么后果？如何避免？

**参考答案：**

**依赖数组写错的两种情况：**

1. **缺少依赖（漏写）：**
```jsx
// ❌ 错误：依赖中缺少 count
const handleClick = useCallback(() => {
  setCount(count + 1);  // 始终使用初始 count 值（闭包陷阱）
}, []);

// 点击后：count 始终显示为 1（因为 handleClick 闭包中的 count 永远是 0）
```

2. **多余的依赖：**
```jsx
// ❌ 不必要：setState 函数是稳定的，不需要作为依赖
const handleClick = useCallback(() => {
  setCount(c => c + 1);  // 使用函数式更新
}, [setCount]);  // setCount 实际上不会变

// 虽然不会出 Bug，但让代码更难阅读
```

**推荐的写法：**
```jsx
// ✅ 使用函数式更新 + useRef 引用
const countRef = useRef(count);
countRef.current = count;

const handleClick = useCallback(() => {
  // countRef.current 始终是最新值
  console.log(countRef.current);
  setCount(c => c + 1);
}, []); // 不用依赖 count
```

**ESLint 插件**：使用 `eslint-plugin-react-hooks` 的 `exhaustive-deps` 规则自动检查依赖项完整性。在开发中养成"看到警告就修正"的习惯。

### Q3：useMemo 和 useCallback 本身的性能开销有多大？它们真的能优化性能吗？

**参考答案：**

**开销分析：** 每次渲染时，useMemo/useCallback 的开销包括：
1. 从 Hook 链表中定位到当前 Hook（O(1) 指针操作）
2. 读取上一次存储的 memoizedState
3. 浅比较依赖数组（O(n)，n 为依赖数组长度）
4. 决定是否重新计算或返回缓存值

对于一个有 3 个依赖的 useMemo，每次渲染的总开销约为 0.001-0.01ms。

**是否值得使用：**

```jsx
// 场景一：值得使用（跳过昂贵的计算）
const sortedData = useMemo(() => {
  return largeArray.sort(/* 复杂比较 */);
}, [largeArray]);
// 10000 条数据的排序 ≈ 5-10ms
// useMemo 的开销 < 0.01ms
// 收益 > 1000 倍 ✅

// 场景二：不值得使用（廉价计算）
const total = useMemo(() => items.length, [items]);
// items.length 的读取时间是 < 0.0001ms
// useMemo 本身的比较开销 > 直接读取的开销
// 反而更慢 ❌

// 场景三：使用 useCallback 避免子组件重渲染（整体收益）
const Parent = () => {
  const handleClick = useCallback(() => {
    doSomething(id);
  }, [id]);
  
  return <ExpensiveChild onClick={handleClick} />;
  // 如果不 useCallback，ExpensiveChild 每次渲染都重新执行
  // 即使它的 DOM 可能完全没变
};
```

**总结：**
- **只对真正昂贵的计算使用 useMemo**（遍历大型数组、复杂数学计算等）
- **在传递给 React.memo 子组件的回调上使用 useCallback**
- **不要对每个函数和值都包裹缓存**——"缓存击败了缓存"（代码可读性下降 + 性能反效果）

## 总结与扩展

### 总结

useMemo 和 useCallback 是 React Hooks 体系中的缓存工具，它们：

- **存储机制**：在 Fiber 的 memoizedState 链表中存储 `[value, deps]` 元组
- **比较机制**：使用 Object.is 对依赖数组做浅比较
- **生命周期**：mount 创建缓存，update 决定是否命中缓存
- **核心价值**：减少子组件重渲染（配合 React.memo）和减少昂贵计算

### 扩展：即将到来的 React Compiler（React Forget）

React 团队正在开发的**新编译器**（React Forget 或 React Compiler）将对 useMemo 和 useCallback 产生重大影响。

React Compiler 的目标是：**自动进行记忆化**。它会在编译阶段分析组件的函数体，自动判断哪些值需要缓存、哪些函数引用需要保持稳定，并在生成的代码中自动插入类似 useMemo/useCallback 的缓存代码。

这意味着：
- 未来开发者可能不需要手动使用 useMemo/useCallback
- React Compiler 会自动推断依赖关系，消除"忘记写依赖"的 Bug
- 代码会更简洁，同时保持高性能

但这个编译器不会让学习 useMemo/useCallback 变得无意义——**理解"值缓存"的概念和"依赖追踪"的语义**，对于理解 React Compiler 的工作原理和调试编译后的代码至关重要。
