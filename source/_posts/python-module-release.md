---
title: python模块释放
date: 2018-10-14 23:53:06
tags:
---


# module释放


## module释放基本原理
Python模块释放问题，是我在工作中遇到的一个问题：使用python模块存放数据，并进行动态加载，使用完后，还需要将模块释放掉。  
当时怀疑python模块可能并未释放，从而导致内存泄漏。事实上，python模块的释放与其他python内对象的释放并没有什么不同：**当对象的引用计数为0，就会释放该对象**。  
Python内存垃圾回收主要使用引用计数，引用计数为0，就会立即进行垃圾回收。同时为了解决循环引用的问题，Python垃圾回收中引入了**标记-清除**、**分代回收**的垃圾回收机制。  
因此要释放Python模块，只要保证模块的引用计数为0：**删除所有对模块引用的地方（包括`sys.modules`）**。可以使用`sys.getrefcount`查看对象的引用计数，注意`sys.getrefcount`也会对当前对象的引用加1。  
模块释放的源码如下：  
```c
#define Py_DECREF(op)                                   \
    do {                                                \
        if(--((PyObject*)(op))->ob_refcnt != 0)         \
            _Py_CHECK_REFCNT(op)                        \
        else                                            \
        _Py_Dealloc((PyObject *)(op));                  \
    } while (0)
#define _Py_Dealloc(op) ((*Py_TYPE(op)->tp_dealloc)((PyObject *)(op)))
#define Py_TYPE(ob)             (((PyObject*)(ob))->ob_type)
```

module对象的`ob_type`是`PyModule_Type`，释放时调用了模块的释放函数`module_deallc`：
```c
PyTypeObject PyModule_Type = {
    ...
    (destructor)module_dealloc,                 /* tp_dealloc */
    ...
    PyObject_GC_Del,                            /* tp_free */
}

static void
module_dealloc(PyModuleObject *m)
{
    PyObject_GC_UnTrack(m);
    if (m->md_dict != NULL) {
        _PyModule_Clear((PyObject *)m); // 清理该模块引用的对象
        Py_DECREF(m->md_dict);
    }
    Py_TYPE(m)->tp_free((PyObject *)m);
}

void
PyObject_GC_Del(void *op)
{
    PyGC_Head *g = AS_GC(op);
    if (IS_TRACKED(op))
        gc_list_remove(g);
    if (generations[0].count > 0) {
        generations[0].count--;
    }
    PyObject_FREE(g);
}
```

## 例外
Python的**内建模块**（`_PyImport_Inittab`中的模块）和**扩展模块**（`_PyImport_DynLoadFiletab`表中指定的扩展类型）的释放略微不同，这两类模块在加载时，会调用`_PyImport_FixupExtension`放入**备份列表**（`extensions`全局变量）中，而加载完成返回的只是对**备份列表**中的module对象的拷贝。  
这就会导致一个问题，释放**内建模块**和**扩展模块**只是释放了对应模块的拷贝，而该模块并未被完全释放。


## 总结
module并不适合用来存放数据，一是module的加载机制效率低，二是module在Python内部也有自己的类型定义，加载进内存时，还会存放其他一些模块相关的信息，需要占用更多内存。


## 参考
《Python源码剖析》
