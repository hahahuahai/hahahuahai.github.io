---
layout:     post
title:      fairygui源码阅读（3）——GObjectPool
subtitle:   fairygui
date:       2020-02-28
author:     huahai
catalog: true
tags:
    - fairygui
---



## GObjectPool

这是FairyGUI里面的对象池，主要用于GList。



```c#
/// <summary>
/// Callback function when a new object is creating.
/// </summary>
/// <param name="obj"></param>
public delegate void InitCallbackDelegate(GObject obj);

/// <summary>
/// Callback function when a new object is creating.
/// </summary>
public InitCallbackDelegate initCallback;
```

initCallback是一个对外公开的回调函数，会在新对象创建之后调用。



```c#
Dictionary<string, Queue<GObject>> _pool;
```

_pool是对象池，键是以内部id表达的url格式，值是GObject的队列，url相同的GObject都存于相同的队列中。

```c#
Transform _manager;
```

_manager是对象池中所有对象transform的父亲。



```c#
        /// <summary>
        /// 需要设置一个manager，加入池里的对象都成为这个manager的孩子
        /// </summary>
        /// <param name="manager"></param>
        public GObjectPool(Transform manager)
        {
            _manager = manager;
            _pool = new Dictionary<string, Queue<GObject>>();
        }
```

构造函数，参数是一个transform对象用于作为对象池中所有对象transform的父亲。

```c#
        /// <summary>
        /// Dispose all objects in the pool.
        /// </summary>
        public void Clear()
        {
            foreach (KeyValuePair<string, Queue<GObject>> kv in _pool)
            {
                Queue<GObject> list = kv.Value;
                foreach (GObject obj in list)
                    obj.Dispose();
            }
            _pool.Clear();
        }
```

清理对象池中的所有对象并释放。

```c#
        /// <summary>
        /// 
        /// </summary>
        public int count
        {
            get { return _pool.Count; }
        }
```

得到对象池中的对象数量。

```c#
        /// <summary>
        /// 
        /// </summary>
        /// <param name="url"></param>
        /// <returns></returns>
        public GObject GetObject(string url)
        {
            url = UIPackage.NormalizeURL(url);
            if (url == null)
                return null;

            Queue<GObject> arr;
            if (_pool.TryGetValue(url, out arr)
                && arr.Count > 0)
                return arr.Dequeue();

            GObject obj = UIPackage.CreateObjectFromURL(url);
            if (obj != null)
            {
                if (initCallback != null)
                    initCallback(obj);
            }

            return obj;
        }
```

从对象池中获取对象。

```c#
        /// <summary>
        /// 
        /// </summary>
        /// <param name="obj"></param>
        public void ReturnObject(GObject obj)
        {
            string url = obj.resourceURL;
            Queue<GObject> arr;
            if (!_pool.TryGetValue(url, out arr))
            {
                arr = new Queue<GObject>();
                _pool.Add(url, arr);
            }

            ToolSet.SetParent(obj.displayObject.cachedTransform, _manager);
            arr.Enqueue(obj);
        }
```

把对象存入对象池中。

完。