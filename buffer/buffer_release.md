ReferenceCounted的设计和ByteBuf的实现(AbstractReferenceCountedByteBuf)都不复杂。

但是对于如何使用ByteBuf, 在使用时候应该调用remain()/release()依然是个非常重要的话题。

＃　原则

究竟应该谁，在申明时候来销毁ByteBuf？

[netty的wiki](http://netty.io/wiki/reference-counted-objects.html)中给出建议：

** 通常的经验是谁最后访问引用计数对象，谁就负责销毁它**

具体来说是以下两点：

- 如果组件A把一个引用计数对象传给另一个组件B，那么组件A通常不需要销毁对象，而是把决定权交给组件B
- 如果一个组件不再访问一个引用计数对象，那么这个组件负责销毁它

看了一下[netty中的例子](http://netty.io/wiki/reference-counted-objects.html#wiki-h3-4),对照上面的原则和做法,总结如下.

## 传递buffer对象不返回

对于A调用B的情况,如果B的函数不会返回Buffer对象,代码类似如下:

```java
void a() {
	ByteBuf buffer = createBuffer();
	b(buffer);
}

void b(ByteBuf buffer) {}
```

会有以下几种情况:

1. 如果A传递buffer对象到B之后,A就放弃对buffer的控制

	之后只有B才控制(或者说访问)buffer,那么此时B就负有在使用完成之后销毁buffer的责任.

    ```java
    void a() {
        ByteBuf buffer = createBuffer();
        b(buffer);
    }

    void b(ByteBuf buffer) {
        try {
            // handle buffer
        } finally {
            buffer.release();
        }
    }
    ```
	因此b需要遵循的原则: 如果有buffer传入,在使用完成之后需要销毁.

2. 如果A传递buffer对象到B之后,A并没有放弃对buffer的控制

	但是b是无法区分,b在被调用时是不知道a是否要放弃控制权,因此b只能继续遵循上述的原则做一次release.

    此时a是知道自己的行为的,因此a可以在调用b之前,先行调用一次remain()方法. 这样a就可以在调用b之后继续保留对buffer的访问.

    ```java
    void a() {
        ByteBuf buffer = createBuffer();
        buffer.remain();
        try {
        	b(buffer);
        	// a continue to use buffer
        } finally {
        	buffer.release();
        }
    }

    void b(ByteBuf buffer) {
        try {
            // handle buffer
        } finally {
            buffer.release();
        }
    }
    ```

## 传递buffer对象又被返回

对于A调用B的情况,如果B的函数会返回Buffer对象,代码类似如下:

```java
void a() {
	ByteBuf buffer = createBuffer();
	buffer = b(buffer);
    // continue to access buffer
}

ByteBuf b(ByteBuf buffer) {}
```

会有以下几种情况:

1. B在处理完成之后又将buffer原样返回,此时B不再控制buffer,而A在获取返回的buffer之后需要继续访问buffer,因此A才是最后的访问者,因此这时是A需要担负销毁buffer的责任

    ```java
    void a() {
        ByteBuf buffer = createBuffer();
        try {
        	buffer = b(buffer);
            // continue to access buffer
        } finally {
        	buffer.release();
        }
    }

    ByteBuf b(ByteBuf buffer) {
    	// handle buffer
        return buffer;
    }
    ```
2. B在处理传入的buffer之后,又重新生成一个新的ByteBuf返回.此时对于A时无法判断返回的buffer对象和上面情况有什么不同,因此A只能继续遵循上述的原则对返回的ByteBuf做release(). 而B应该对输入的ByteBuf做release().

    ```java
    void a() {
        ByteBuf buffer = createBuffer();
        try {
        	buffer = b(buffer);
            // continue to access buffer
        } finally {
        	buffer.release();
        }
    }

    ByteBuf b(ByteBuf buffer) {
    	try {
    		// handle buffer
        } finally {
        	buffer.release();
        }
        ByteBuf newBuffer = createBuffer();
        return newBuffer;
    }
    ```
