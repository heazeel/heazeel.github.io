---
title: antd table的滚动加载
toc: true
tags: js
categories: js
---

## 背景
最近在工作中有一个列表的滚动加载功能需求，了解了一下发现antd的table是不支持滚动加载的，只有list支持。因为原来的业务代码是用table做的，为了不改变原来的代码，我打算自己写个table的滚动加载功能。

## 实现
想要实现一个滚动加载，无非就是实现一个scroll滚动位置的监听，所以我的目标就转换为实现对table的滚动行为监听。

第一个想法就是获取Table的ref，但是用useRef这个hook来获取table的dom会报错
```jsx
const tableRef = useRef(null);
<Table
  ref={tableRef}
  dataSource={dataSource}
  columns={columns}
  loading={tableLoading}
  scroll={{ y: 200 }}
/>
```
! [scroll-table](../../public/../../public/img/scroll-table.png)