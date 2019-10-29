---
title: ' [Threejs]Object3d对象如何获取指定名字的子元素？'
date: 2019-10-29 23:19:16
tags: threejs Object3D
---

## 现有的获取对象的方法

- getObjectById()

根据指定的id获取对应的对象，总是返回第一个匹配到的对象

- getObjectByName()

根据指定的name获取对应的对象，总是返回第一个匹配到的对象

- getObjectByProperty()

根据指定的属性(键值对)获取对应的对象，总是返回第一个匹配到的对象

<!--more-->

其实**getObjectById**和**getObjectByName**的方法内部都是调用了**getObjectByProperty**方法，我们打开它的源码，逻辑具体如下:

```javascript
// ...
   getObjectById: function ( id ) {
      return this.getObjectByProperty( 'id', id );
   },
   getObjectByName: function ( name ) {
      return this.getObjectByProperty( 'name', name );
   },
   getObjectByProperty: function ( name, value ) {
      if ( this[ name ] === value ) return this;
      for ( var i = 0, l = this.children.length; i < l; i ++ ) {
         var child = this.children[ i ];
         var object = child.getObjectByProperty( name, value );
         if ( object !== undefined ) {
            return object;
         }
      }
      return undefined;
   },
// ...
```

## Object3D对象如何获取指定名字的子元素?

对比以上三个方法，看起来都不太适合解决这个问题．假设多个不同的Object3D的child中都有一个name/id相同的对象，那么通过这些方法找到的对象无法确定是哪个Object3D的．

我当前的一种解决方法是，我们可以在Object3D类(而不是Scene类)上扩展一个方法出来，比如*getChildByName*, 参考如下:

```javascript
  getChildByName(childName: string): Object3D {
    const getChild = (obj: Object3D) => {
      if (obj.name === childName) {
        return obj;
      }
      if (obj.children.length <= 0) {
        return null;
      }

      let c = null;
      obj.children.forEach(child => {
        const tmp = getChild(child);
        if (tmp != null) {
          c = tmp;
        }
      });
      return c;
    };

    return getChild(this);
  }
```

可以定义一个Object3D的子类，把这个getChildByName方法给添加进来子类中．调用的时候，类似这样即可:

```javascript
// const obj = xxx; //Object3D
obj.getChildByName('someName');
```
