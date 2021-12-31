---
title: antd table 滚动加载
toc: true
date: 2021-09-22 16:08:10
tags:
categories: 业务
---

## 背景

最近在工作中有一个列表的滚动加载功能需求，了解了一下发现`antd`的`table`是不支持滚动加载的，只有`list`支持。因为原来的业务代码是用`table`做的，为了不改变原来的代码，我打算自己写个`table`的滚动加载功能。

<!-- more -->
## 实现

想要实现一个滚动加载，目标就是实现对`table`的滚动行为进行监听。第一个想法就是获取`Table`的`dom`，然后对可滚动的`table-body`添加一个监听事件，这里我们利用`useRef`来获取`dom`。出现问题了，会报错，`ts`提示我们不能将`MutableRefObject`这个类型传递到`table`的`props`中。

```jsx
const target = useRef(null);
<Table
  ref={target}
  dataSource={dataSource}
  columns={columns}
  loading={tableLoading}
  scroll={{ y: 200 }}
/>
```
![](https://lost-and-find.oss-cn-hangzhou.aliyuncs.com/blog/scrollTable.png?versionId=CAEQKBiBgMDE87nN4BciIDIwYTQzOTNhMjkwODRmMjJiNDNiNzc4MTU1NTY3ODdh)

既然`Table`不能传入`ref`，那在外面包一层`div`总行了吧，于是代码变成了这样

```jsx
const target = useRef(null);
<div ref={target}>
  <Table
    dataSource={dataSource}
    columns={columns}
    loading={tableLoading}
    scroll={{ y: 200 }}
  />
</div>
```

控制台打印一下`target.current`，可以获取`dom`对象，到这我们的第一步目的就达成了。

![](https://lost-and-find.oss-cn-hangzhou.aliyuncs.com/blog/image-20210924003831132.png?versionId=CAEQKBiBgMCgrI3O4BciIGJkNmFjZWEwZDNkODRlMzQ5ZGI2YTYxNWM4YzIxMGQ4)

接下来我们在`target.current`中获取`table-body`的`dom`对象，并对`table-body`的滚动进行监听，然后通过条件`crollHeight - scrollTop === clientHeight`来判断滚动条是否滚到底了，滚到底了就去异步加载数据。完整代码如下：

```jsx
import React, { useEffect, useState, useRef } from 'react';
import { Table } from 'antd';

const data = [
  {
    key: '1',
    name: '胡彦斌',
    age: 1,
    address: '西湖区湖底公园1号',
  },
  {
    key: '2',
    name: '胡彦祖',
    age: 2,
    address: '西湖区湖底公园1号',
  },
  {
    key: '3',
    name: '胡彦斌',
    age: 3,
    address: '西湖区湖底公园1号',
  },
  {
    key: '4',
    name: '胡彦祖',
    age: 4,
    address: '西湖区湖底公园1号',
  },
  {
    key: '5',
    name: '胡彦斌',
    age: 5,
    address: '西湖区湖底公园1号',
  },
  {
    key: '6',
    name: '胡彦祖',
    age: 6,
    address: '西湖区湖底公园1号',
  },
];

const columns = [
  {
    title: '姓名',
    dataIndex: 'name',
    key: 'name',
  },
  {
    title: '年龄',
    dataIndex: 'age',
    key: 'age',
  },
  {
    title: '住址',
    dataIndex: 'address',
    key: 'address',
  },
];

const ScrollTable = () => {
  const [dataSource, setDataSource] = useState(data);
  const [tableLoading, setTableLoading] = useState(false);
  const target = useRef(null);
  useEffect(() => {
    const dom = (target.current as HTMLElement | null)?.querySelector(
      '.ant-table-body',
    ) as HTMLElement;
    dom.addEventListener('scroll', () => scrollEvent());
    return () => {
      dom.removeEventListener('scroll', scrollEvent);
    };
  }, []);

  const scrollEvent = () => {
    const dom = (target.current as HTMLElement | null)?.querySelector(
      '.ant-table-body',
    ) as HTMLElement;
    if (dom.scrollHeight - dom.scrollTop === dom.clientHeight) {
      // 在这里去获取并更新数据
      setTableLoading(true);
      setTimeout(() => {
        console.log('已滚动到底部');
        let arr: any = [];
        setDataSource(arr);
        setTableLoading(false);
      }, 1000);
    }
  };

  return (
    <div
      ref={target}
      style={{ width: '70%', margin: 'auto', paddingTop: '100px' }}
    >
      <Table
        dataSource={dataSource}
        columns={columns}
        loading={tableLoading}
        scroll={{ y: 200 }}
      />
    </div>
  );
};

export default ScrollTable;
```



