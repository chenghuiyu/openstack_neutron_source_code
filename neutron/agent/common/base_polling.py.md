## **base_polling.py**

主要实现了一些轮询的策略机制

### **BasePollingManager类**

主要定义了几个方法`force_polling()`、`polling_completed()`、`is_polling_required()`。
其中，`is_polling_required()`通过@property装饰器来实现方法到属性的转换。

### **AlwaysPoll类**

该类是`BasePollingManager`类的子类，覆盖了`is_polling_required()`方法。通过对内部属性_force_polling、_polling_completed的判断来返回poling_required的值。




> python学习小结：如何使用@property? Python内置的@property装饰器就是负责把一个方法变成属性调用，@property的实现比较复杂，我们先考察如何使用。把一个getter方法变成属性，只需要加上@property就可以了，此时，@property本身又创建了另一个装饰器@score.setter，负责把一个setter方法变成属性赋值，遇到@property，我们在对实例属性操作的时候，就知道该属性很可能不是直接暴露的，而是通过getter和setter方法来实现的。@property广泛应用在类的定义中，可以让调用者写出简短的代码，同时保证对参数进行必要的检查，这样，程序运行时就减少了出错的可能性。
