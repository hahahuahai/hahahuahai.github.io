---
layout:     post
title:      fairygui源码阅读（4）——Timer
subtitle:   fairygui
date:       2020-02-28
author:     huahai
catalog: true
tags:
    - fairygui
---



## Timer

FairyGUI里面有个很实用的计时器引擎（Timer Engine）。例如：让某个窗口打开之后，过几秒自动关闭的功能就可以用计时器，把关闭窗口的函数放到计时器里，设好关闭的时间即可。

这个计时器的基本原理是，创建一个不在hierarchy里显示的GameObject，名叫[FairyGUI.Timers]。这个GameObject上挂载着计时器引擎脚本TimerEngine，这个TimerEngine是继承于MonoBehaviour的，在Update函数里会调用计时器引擎里面存储的计时器函数。

```c#
        public Timers()
        {
            _inst = this;
            gameObject = new GameObject("[FairyGUI.Timers]");
            gameObject.hideFlags = HideFlags.HideInHierarchy;
            gameObject.SetActive(true);
            Object.DontDestroyOnLoad(gameObject);

            _engine = gameObject.AddComponent<TimersEngine>();

            _items = new Dictionary<TimerCallback, Anymous_T>();
            _toAdd = new Dictionary<TimerCallback, Anymous_T>();
            _toRemove = new List<Anymous_T>();
            _pool = new List<Anymous_T>(100);
        }
```

这是Timers类的构造函数。

```c#
    class TimersEngine : MonoBehaviour
    {
        void Update()
        {
            Timers.inst.Update();
        }
    }
```

这是挂载在GameObject中的TimersEngine脚本。