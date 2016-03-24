## ECMAScript Observable ##

这个提议引入了**可观测（Observable）**类型到 ECMAScript 标准库。**可观测**类型可用于基于模型推送的数据源，比如 DOM 事件，定时器和 socket。此外，可观测类型还是：

- **可复合的**：可观测类型可以由高阶的组合器组合。
- **惰性的**：可观测类型并不发送数据直到有观察者订阅。

### 例子：观察键盘事件

我们可以使用**Observab**构造器创建一个函数，返回支持任意 DOM 元素和事件类型的可观察事件流。

```js
function listen(element, eventName) {
    return new Observable(observer => {
        // 创建一个发送数据给下流的事件处理器
        let handler = event => observer.next(event);

        // 给元素附上事件处理器
        element.addEventListener(eventName, handler, true);

        // 返回一个函数用来取消事件流
        return _ => {
            // 从元素上去除事件处理器
            element.removeEventListener(eventName, handler, true);
        };
    });
}
```

我们可以使用标准的组合器来过滤和 map 事件流，就像操作数组一样。

```js
// 返回一个特殊键按下的观察对象
function commandKeys(element) {
    let keyCommands = { "38": "up", "40": "down" };

    return listen(element, "keydown")
        .filter(event => event.keyCode in keyCommands)
        .map(event => keyCommands[event.keyCode])
}
```

*注意：这个提议不包含 "fileter" 和 "map" 方法。它们也许会在未来的说明中添加进来。*

当我们想要消费事件流时，我们用**观察者（observer）**来订阅。

```js
let subscription = commandKeys(inputElement).subscribe({
    next(val) { console.log("Received key command: " + val) },
    error(err) { console.log("Received an error: " + err) },
    complete() { console.log("Stream complete") },
});
```

**subscribe** 方法返回的对象允许我们随时取消订阅。取消订阅时，Observable 的清除函数将会执行。

```js
// 调用了这个方法时候，将不会再发送任何事件。
subscription.unsubscribe();
```

或者，我们可以使用 **forEach** 方法来订阅一个 Observable，这时候接受一个回调函数作为参数并返回一个 Promise。

```js
commandKeys(inputElement).forEach(val => {
    console.log("Received key command: " + val);
});
```

```js
Observable.of(1, 2, 3, 4, 5)
    .map(n => n * n)
    .filter(n => n > 10)
    .forEach(n => console.log(n))
    .then(_ => console.log("All done!"));
```

### 动机 ###


可观测类型代表了一种处理异步事件流的基础协议。它在来自环境推向应用的数据流建模过程中尤其高效，比如用户界面事件。通过提供 Observable 作为 ECMAScript 标准库的一个组件，我们使平台和应用可以共享一个通用的基于推送的流协议。

### 实现 ###

- [RxJS 5](https://github.com/ReactiveX/RxJS)
- [zen-observable](https://github.com/zenparsing/zen-observable)

### Running Tests ###

To run the unit tests, install the **es-observable-tests** package into your project.

```
npm install es-observable-tests
```

Then call the exported `runTests` function with the constructor you want to test.

```js
require("es-observable-tests").runTests(MyObservable);
```

### API ###

#### Observable ####

一个 Observable 代表一系列的可以被观察的值。

```js
interface Observable {

    constructor(subscriber : SubscriberFunction);

    // Subscribes to the sequence
    subscribe(observer : Observer) : Subscription;

    // Subscribes to the sequence with a callback, returning a promise
    forEach(onNext : any => any) : Promise;

    // Returns itself
    [Symbol.observable]() : Observable;

    // Converts items to an Observable
    static of(...items) : Observable;

    // Converts an observable or iterable to an Observable
    static from(observable) : Observable;

    // Subclassing support
    static get [Symbol.species]() : Constructor;

}

interface Subscription {

    // Cancels the subscription
    unsubscribe() : void;
}

function SubscriberFunction(observer: SubscriptionObserver) : (void => void)|Subscription;
```

#### Observable.of ####

`Observable.of` 创建一个以参数作为值得 Observable。这些值在后面的事件循环中都是被异步的分发。

```js
Observable.of("red", "green", "blue").subscribe({
    next(color) {
        console.log(color);
    }
});

/*
 > "red"
 > "green"
 > "blue"
*/
```

#### Observable.from ####

`Observable.from` 将它的参数转化为 Observable。

- 如果参数有 `Symbol.observable` 方法，那么它将返回调用这个函数的结果。如果结果对象不是 Observable 的实例，那么它将被包裹进一个 Observable 来代理订阅。
- 否则，就假设参数是可遍历的，并且遍历的值在后面的时间循环中被异步的分发。

转化支持 `Symbol.observable` 函数的对象为一个 Observable：

```js
Observable.from({
    [Symbol.observable]() {
        return new Observable(observer => {
            setTimeout(_=> {
                observer.next("hello");
                observer.next("world");
                observer.complete();
            }, 2000);
        });
    }
}).subscribe({
    next(value) {
        console.log(value);
    }
});

/*
 > "hello"
 > "world"
*/

let observable = new Observable(observer => {});
Observable.from(observable) === observable; // true

```

转化一个可遍历的值为 Observable：

```js
Observable.from(["mercury", "venus", "earth"]).subscribe({
    next(value) {
        console.log(value);
    }
});

/*
 > "mercury"
 > "venus"
 > "earth"
*/
```

#### 观察者（Observer） ####

观察者是用来从 Observable 接收数据的，作为 **subscribe** 函数的参数。

所有的方法都是可选的。

```js
interface Observer {

    // Receives the next value in the sequence
    next(value);

    // Receives the sequence error
    error(errorValue);

    // Receives the sequence completion value
    complete(completeValue);
}
```

#### 订阅观察者（SubscriptionObserver） ####

订阅观察者是规范化的观察者，它把观察者包裹起来提供给 **subscribe** 方法。

```js
interface SubscriptionObserver {

    // Sends the next value in the sequence
    next(value);

    // Sends the sequence error
    error(errorValue);

    // Sends the sequence completion value
    complete(completeValue);
}
```
