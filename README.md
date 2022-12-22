# flutter的foundation其他小工具

在foundation模块下有一个文件是object.dart，这个文件唯一的一个函数就是工具函数objectRuntimeType。

这个objectRuntimeType是很有意思，也很实用的。它存在最主要的原因是，在一个runtime调用toString会对性能产生负面的影响。所以把它封装了一下放在assert里面执行，巧妙地利用了**assert在release模式下不会执行**的特点。其中传入的optimizedValue是一个常量，可以是任何字符串，在release模式下，这个函数最后会返回optimizedValue，来保证性能和逻辑的连贯性不受影响。实际代码如下：

```dart
String objectRuntimeType(Object object, String optimizedValue) {
  assert(() {
    optimizedValue = object.runtimeType.toString();
    return true;
  }());
  return optimizedValue;
}
```



foundation模块下有一个文件是collections.dart,这个文件提供了四个集合的工具函数，分别是：setEquals，listEquals，mapEquals，binarySearch。

setEquals，listEquals和mapEquals这些函数是判断传入的两个是不是一样的，这个逻辑是比较容易理解的。我就选set类型的代码展示一下，并在代码中加了注释：

```kotlin
bool setEquals<T>(Set<T> a, Set<T> b) {
  if (a == null)
    return b == null;
  
  if (b == null || a.length != b.length)
    return false;
  
  if (identical(a, b))       //这个就是确定指针(引用)指向的是不是同一块内存地址
    return true;
  
  for (final T value in a) { //和listEquals及mapEquals相比仅这一块不一样，目的是判断内容是否一致
    if (!b.contains(value))
      return false;
  }
  
  return true;
}
```



binarySearch这个函数要求传入两个参数，第一个是**已经排好序的list**，第二个是需要查找的value。内部实现可以理解成一个简单的**二分法**实现的，代码如下：

```dart
int binarySearch<T extends Comparable<Object>>(List<T> sortedList, T value) {
  int min = 0;
  int max = sortedList.length;
  while (min < max) {
    final int mid = min + ((max - min) >> 1); //这个方式还是很有新意的，值得借鉴
    final T element = sortedList[mid];
    final int comp = element.compareTo(value);
    if (comp == 0) {
      return mid;
    }
    if (comp < 0) {
      min = mid + 1;
    } else {
      max = mid;
    }
  }
  return -1;
}
```



foundation模块下有一个文件是constants.dart，这个文件里提供了5个常量。其中最常用到的是kReleaseMode

- **kReleaseMode是否在release 模式下**

- kProfileMode是否在profile 模式下

- kDebugMode是否在debug 模式下

- precisionErrorTolerance这个值是double类型，是double类型数值的误差范围

- kIsWeb是否是web环境

  

foundation模块下有一个文件是profile.dart, 这个文件只有一个profile函数。这个函数已经废弃，功能完全可以用上文提到的constants.dart文件里的kReleaseMode替代。



foundation模块下有platform.dart, 里面有三个很有用的东西：

- 开发项目时，常用到的TargetPlatform枚举:**android，fuchsia, iOS, linux, macOS, windows**.
- 获取默认的defaultTargetPlatform，这个一般就是当前设备的系统类型，在_platform_io.dart文件里，封装了一个获取defaultTargetPlatform类型的方法。获取设备类型优先使用Theme.of获取，在写更底层的实现无法用Theme.of是，才用这个获取。
- debugDefaultTargetPlatformOverride这个就是重写defaultTargetPlatform的值，这个属性通常在测试的时候才会用到。



foundation模块下有一个文件是synchronous_future.dart。
SynchronousFuture是一个class，这个的功能和new Future.value相似，只是他是同步的。这个内部实现代码不多，基本都能看的明白，代码如下：

```dart
class SynchronousFuture<T> implements Future<T> {
  SynchronousFuture(this._value);

  final T _value;

  @override
  Stream<T> asStream() {
    final StreamController<T> controller = StreamController<T>();
    controller.add(_value);
    controller.close();
    return controller.stream;
  }

  @override
  Future<T> catchError(Function onError, { bool test(Object error) }) => Completer<T>().future;

  @override
  Future<E> then<E>(FutureOr<E> f(T value), { Function onError }) {
    final dynamic result = f(_value);
    if (result is Future<E>)
      return result;
    return SynchronousFuture<E>(result as E);
  }

  @override
  Future<T> timeout(Duration timeLimit, { FutureOr<T> onTimeout() }) {
    return Future<T>.value(_value).timeout(timeLimit, onTimeout: onTimeout);
  }

  @override
  Future<T> whenComplete(FutureOr<dynamic> action()) {
    try {
      final FutureOr<dynamic> result = action();
      if (result is Future)
        return result.then<T>((dynamic value) => _value);
      return this;
    } catch (e, stack) {
      return Future<T>.error(e, stack);
    }
  }
}
```

SynchronousFuture是很少有场景用到的，如果必须要用到，要特别注意，因为很不好debug这种双峰行为。我写了一个简单的使用示例：

```csharp
debugPrint('ssss00000');
SynchronousFuture(4).then((value){
  debugPrint('ssss11111');
}).whenComplete((){
  debugPrint('ssss22222');
}).catchError((error) {
  debugPrint('ssss33333');
});
debugPrint('ssss=====');
```

输出的结果是：

```undefined
flutter: ssss00000
flutter: ssss11111
flutter: ssss22222
flutter: ssss=====
```

从输出的结果上大家应该能更直观的了解了。