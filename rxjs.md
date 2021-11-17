# mergeMap
```js
import { fromEvent, of } from 'rxjs';
import { mergeMap, delay } from 'rxjs/operators';

// faking network request for save
const saveLocation = location => {
  return of(location).pipe(delay(500));
};
// streams
const click$ = fromEvent(document, 'click');

click$
  .pipe(
    mergeMap((e: MouseEvent) => {
      return saveLocation({
        x: e.clientX,
        y: e.clientY,
        timestamp: Date.now()
      });
    })
  )
  .subscribe(r => console.log('Saved!', r));
  ```

mergeMap，首先第一个功能是打平，所以还是flatMap名字形象一点！

比如上面这个例子，我们有一个鼠标点击的observable，当每次发送一个值时，我们调用远程的api请求，这样就又生产了一个observable，所以我们最终面对的是一个Observable[]。flatMap帮我们把这个array打平成一个observable。

反正当你的操作需要生产新的observable时使用就差不多了。

其实你还是对每次产生的observable进行订阅。只不过书写上看上去没产生新的observable好像一样。

mergeMap适用于写的操作，因为你不希望新的写操作去停止旧的写操作。而switchMap则适用于读的操作。

如果对于顺序有要求，则使用concatMap

就类似于
```
const arr1 = [1, 2, 3];

const res = arr1.map((el) => [el, el + 1, el + 2]);
//此时res = [[1,2,3],[2,3,4], [3,4,5]]
//使用flat把res变为[1,2,3,2,3,4,3,4,5]
```

# debounceTime
就是说我每次在源发消息的时候都等上固定的时间，如果中间没有新的消息产生，那么就在等待结束后把消息发出，如果中间有了新的消息，就从新开始计时，过去的消息就不会被发出了。

# withLatestFrom
就是类似于我source发出的时候，顺便把附带的observable的最新值也发出去。但是发送顺序完全是由source决定的。

# takeUtil
监听一个observable，如果这个observable发送了值或者complete，那么source就停止发送了

```ts
const notifier$ = fromEvent(document, 'click');
const source$ = interval(1000);

source$.pipe(takeUntil(notifier$)).subscribe(console.log);
```

# buffer
buffer(notifier)
将source发送的值保存在buffer中，直到notifier发送新的值，才把buffer的数组发送并清空buffer
```ts
import { fromEvent, interval } from 'rxjs';
import { buffer } from 'rxjs/operators';

const clicks = fromEvent(document, 'click');
const intervalEvents = interval(1000);
const buffered = intervalEvents.pipe(buffer(clicks));
buffered.subscribe(x => console.log(x));
```
比如上面的例子，会收集计时器的读数，每次新的点击释放收集的读数

# exhaustMap

```ts
const clicks = fromEvent(document, 'click');
const result = clicks.pipe(
  exhaustMap(ev => interval(1000).pipe(take(5)))
);
result.subscribe(x => console.log(x));
```
如上所示，只要在这个spin出来的observable没有完成的情况下（5秒内），点击事件是不会起作用的。只有当前的observable完成了，才能继续spin新的observable。

```ts
search$ = createEffect(() => 
  this.actions$.pipe(
    ofType(BookActions.search),
    exhaustMap(action =>
      this.googleBooksService.search(action.query)
    )
  )
);
```
比较常见的应用，当我发起多次请求，只有当前请求完成之后才能继续发送。


# switchMap
switchMap同exhaustMap的作用有点相反。switchMap会在产生新的内部obervable时，抛弃掉旧的observable。

比如当发送新的api请求后，就抛弃对旧的请求的监听。

```
import { fromEvent, interval } from 'rxjs';
import { switchMap } from 'rxjs/operators';

const clicks = fromEvent(document, 'click');
const result = clicks.pipe(switchMap((ev) => interval(1000)));
result.subscribe(x => console.log(x));
```
以上会在有新的点击的时候，停止旧的计时器。

# catchError
这个operator很容易引起疑惑。我们来看一下官网给出的类型定义
 ```ts
catchError<T, O extends ObservableInput<any>>(selector: (err: any, caught: Observable<T>) => O): OperatorFunction<T, T | ObservedValueOf<O>>
```
我们可以看到这边的operatorFunction是把源observable T转化成一个包含源observable T和自己catch到错误产生的值的一个新的observable。

```ts
import { of } from 'rxjs';
import { map, catchError } from 'rxjs/operators';

of(1, 2, 3, 4, 5).pipe(
    map(n => {
  	   if (n === 4) {
	       throw 'four!';
      }
     return n;
    }),
    catchError(err => of('I', 'II', 'III', 'IV', 'V')),
  )
  .subscribe(x => console.log(x));
  // 1, 2, 3, I, II, III, IV, V
```
所以上面这个例子，我们不需要用高阶的map操作符，尽管看上去catchError的回调函数是返回一个新的observable的，其实经过这层转换，只是把error转换成一些列新的值罢了。另外我们也可以看到新的observable是个混合值类型的，既有number也有string。

甚至也可以在catch中返回源observable,功能类似于retry操作符
```ts
of(1, 2, 3, 4, 5)
  .pipe(
    map((n) => {
      if (n === 4) {
        throw 'four!';
      }
      return n;
    }),
    catchError((err, caught) => caught),
    take(30)
  )
  .subscribe((x) => console.log(x));
// 1, 2, 3, 1, 2, 3, ...
```

