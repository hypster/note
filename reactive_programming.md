Reactive programming v.s. imperative programming

imperative
- state
- condition
- statement
- describe how program operates

Reactive
- stateless
- function/expression
- describe what the program should accomplish

observable
- observable同event是不同的，要类比的话，还是function同observable最为类似。

event.addEventListener时候，event下面会注册listener，而observable是不会的，它根本不知道谁注册了，调用subscribe只不过是类似于调用一个function（observable execution）

observable的四个要素

1. create，可以通过
```js
const subscription = new Observable(function subscribe(subscriber){
  //calling subscribe methods such as ...
  subscriber.next()
  subscriber.complete()
  subscribe.error()
})
```
也可以通过其他available的creation function来创建

2. subscribe
通过observable.subscribe来subscribe

3. execution
就是执行new observable的时候constructor里传入的subscribe函数
observable允许这样的执行pattern:
```
next*(error|complete)?
```

4. dispose，通过unsubscribe来dispose
很多时候observable的stream是infinite的，所以observer在完成当前的任务时需要及时unsubscribe来释放内存

### higher order piped operator
- concatAll, 很多时候我们面对的数据是这样的，Observable<Observable<Type>>，比方说当我们有一个动态的url数组，当我们继续去做http请求时，每次请求又会返回一个obser
vable，这时候我们希望最终的数据结构是平的，我们就会用这个操作符。它底下的逻辑是去依次subscribe每一个observable，直到获取它的值，然后再把结果concat起来。

- mergeAll， ？？？


- switchAll，当下一个新的observable arrive的时候就去取消之前的subscription，再去订阅新的

- exhaust，只要有observable未完成，就等到它完成，期间进来的observable都丢弃，完成后，再去订阅新的observable

- concatMap， mergeMap， switchMap， exhaustMap，将map和前面的功能合并

### 自定义operator
- 可以使用pipe operator来对一些经常使用的系列操作符进行封装
- 要全新创建新的operator，稍微烦了一点,一般用不到，不然还要rxjs干嘛，但是可以更好理解piped operator的本质

记住complete/error/next和前一个observable没有一一对应的关系哦！完全由当前的subscriber来决定，所以teardown logic很重要，如果你subscribe了一个无限发射的observable，你只要前面几个值，那你完事之后就应该要去取消订阅，不然就很浪费了。注意，你一旦subscribe了source observable，如果没有subscribe的话，source是不知道你其实已经完成了，是会一直发射的！！

这种loose structure其实也不太方便对不对？
```ts
import { Observable, of } from 'rxjs';

//每一个之前observable产生的value都给它延后一定时间，可以想象marble diagram
function delay<T>(delayInMillis: number) {
  //所有piped operator都默认带参数，这里是delay，都用operator()的方式产生具体的operator，其结构就是（previousObservable) => newObservable
  return (observable: Observable<T>) =>
    new Observable<T>((subscriber) => {
      //这里跟直接构造observable是一样的，只是多了对delay参数和原始observable的closure（层级不一样但都是closure），这个subscribe逻辑每次有subscriber去subscribe都会被执行，可以看成function subscribe (subscriber){...},如果看不习惯箭头函数的话
      
      //piped operator本身是订阅者，同时还要负责通知新的订阅者！
      const allTimerIDs = new Set(); 
      let hasCompleted = false;
      const subscription = observable.subscribe({ //订阅原来的observable，这就是为什么每一个新的operator虽然产生新的observable，但都会订阅旧的observable
        next(value) {
          //每一个值都延迟，理解piped既要负责完成对上一个observable的值进行加工（delay）还要通知下面的subscriber（next，error， complete，时机up to you）！
          const timerID = setTimeout(() => {
            subscriber.next(value);
            //从set清除完成的timerId
            allTimerIDs.delete(timerID);
            //这里是一般情况，当前面的(source)observable已经喊停了，而我们发现我们也没有要处理的事情了，我们也喊停。注意source喊停意味着再也不可能会有next的通知！！也就是说这个set当下是空的，那么以后永远都是空的！
            if (hasCompleted && allTimerIDs.size === 0) {
              subscriber.complete();
            }
          }, delayInMillis);

          //将注册的timer加入set
          allTimerIDs.add(timerID);
        },
        error(err) {
          //错误要proprogate到下个subscriber
          subscriber.error(err);
        },
        complete() {
          //当前面的observable通知完成时，我们的新的observable也“基本”要完成了，hasCompleted就类似于这个基本的状态，只剩下那些还没有被触发的timer事件要完成了
          hasCompleted = true;
          if (allTimerIDs.size === 0) { //当完全清空了，我们可以通知下一个subscriber我们完成了，注意有两个地方调用了complete，一般当前面一个通知完成时，新的应该还没完成，但是严谨的代码不会忘了可能的情况！记住这个情况只会发生一次，也就是说错过了就没办法complete了！不会有next事件了！
            subscriber.complete();
          }
        },
      });

      //注意返回unsubscribe函数对象，调用的话，将取消对source observable的订阅，imagine一个source源源不断给你发数据
      //注意teardown的逻辑针对的是当前的订阅逻辑的execution，当前就订阅了之前的observable，和设定了一些timer，那么就要反过来清除这些操作，注意这逻辑不但可以通过手动unsubscribe来执行，也会在complete/error发生后执行，毕竟你的complete是可以arbitrary的，没有source的complete和当前complete的一对一关系！
      return () => {
        subscription.unsubscribe();
        //也要清除之前设定的timer
        for (const timerID of allTimerIDs) {
          clearTimeout(timerID);
        }
      };
    });
}

// Try it out!
const subscription = of(1, 2, 3).pipe(delay(1000)).subscribe(console.log);
// subscription.unsubscribe();
```
### subscription
subscription可以通过add方法叠加，这样在unsubscribe的时候，可以将add过的所有subscription的execution都停止，并释放其资源。

### subject
subject可以被近似为eventEmitter，它是multicast的，也就是允许多个observer去subscribe，它会maintain listener的列表。

naming很重要，下次不要问我observable和subject的区别了！

subject也会observer，注意observer的标志只是一个具有next，error，complete方法的对象！不要理解错了！observable可以调用这个observer的这些方法来达到通知的功能！

只不过subject的这些方法，只是值传递，但是是同时传给所有的它的subscriber，这是它的不一样。

类似于时间线在subject这边做了一个fork

创建subject，`new Subject<Type>()`

除了使用
```js
subject.subscribe(subscriberA)
subject.subscribe(subscriberB)
observable.subscribe(subject)
```
这样的方式来创建一个multicasted observable

也可以用multicast的操作符

```js
const multicasted = observable.pipe(multicast(subject))
multicasted.subscribe(subscriberA)
multicasted.subscribe(subscriberB)
const subscription = multicasted.connect()
```

multicast返回一个connectableObservable，它的subscribe方式同普通的observable已经不一样了。当调用其connect方法时等价于observable.subscribe(subject)

有时候要管理多个observable很麻烦，refCount操作符可以把connectableObservable封装成一个具有自动connect和自动unsubscribe from source的一个observable。也就是说当有人subscribe了，它就connect了，当全部的subscriber unsubscribe了它就会unsubscribe from source。它的原理就是计数。从0到1就connect，从1到0就unconnect

#### BehaviorSubject
同subject的区别是它会自动记录最近一次发射的值，当有新的observer subscribe时会自动将当前的值通知给它。比方说我们关心的是一个人的年龄的话，我们肯定是想知道它当前的年龄，而不是说他过了一段时间才知道他未来的年龄。

它创建时可以给一个初始值，observer订阅时会自动接受这个初始值。

#### ReplaySubject
比起BehaviorSubject可以设定一个buffer数和一个值的最长的保留时间，也就是这个值要被留下来需要两个条件都要满足。

#### AsyncSubject
只有当这个subject complete了才会把最后一个值通知给observer，类似于后面加了一个last操作符。

#### voidSubject
目前（v7）默认创建不带type的subject，等同于创建void类型的subject，这时不能访问subject的值，也标注这个subject产生的值是不重要的。而在以前的版本，则是默认是any类型，它会让ts停止类型检查。



































































