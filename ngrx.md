# Action
action的创建方式
```ts
import { createAction, props } from '@ngrx/store';

export const login = createAction(
  '[Login Page] Login',
  props<{ username: string; password: string }>()
);
```

[]方括号里面是action的种类，这个种类是用来对于同一个area的action做一个汇总。

area可以是一个元素组件，一个后台接口或者是一个浏览器接口。

后面跟的是具体的事件。

action如果有payload可以用props来定义

# reducer
每一个reducer可以处理多个action，通过on方法来associate。

一个action也可以被多个reducer处理。

当一个action被dispatch后，每一个reducer都会接受，但是通过on的方式来决定处理不处理。

state的注册有两个层级，一个是root，通过StoreModule.forRoot({key: reducer})来创建。另一个是feature，通过StoreModule.forFeature(featureToken, reducer)来创建。

对于懒加载的module，其注册的feature state也会懒加载。

# Selector
selector用来选取state的slice

createSelector创建下来的函数是pure的，正是因为这一性质，它还有memoization的功能，只有input不变，output直接返回上一个值。

可以通过release函数来释放记忆值。

select类似于pipe，可以传入很多函数来做链式变化

```ts
import { createSelector } from '@ngrx/store';

export interface State {
  counter1: number;
  counter2: number;
}

export const selectCounter1 = (state: State) => state.counter1;
export const selectCounter2 = (state: State) => state.counter2;
export const selectTotal = createSelector(
  selectCounter1,
  selectCounter2,
  (counter1, counter2) => counter1 + counter2
); // selectTotal has a memoized value of null, because it has not yet been invoked.

let state = { counter1: 3, counter2: 4 };

selectTotal(state); // computes the sum of 3 & 4, returning 7. selectTotal now has a memoized value of 7
selectTotal(state); // does not compute the sum of 3 & 4. selectTotal instead returns the memoized value of 7

state = { ...state, counter2: 5 };

selectTotal(state); // computes the sum of 3 & 5, returning 8. selectTotal now has a memoized value of 8
```

selector的key要对应我们注册reducer时候的key，不然选不中


createFeatureSelector是createSelector的简化版本，如果要通过createSelector来实现，那么只要传入两个函数，第一个是identity函数，第二个选取feature

```ts
const selectBooks = createFeatureSelector<ReadonlyArray<Book>>('books');
const selectBooks2 = createSelector(
  (state: AppState) => state,
  (state: AppState) => state.books
);
const selector3 = (state: AppState) => state.books;
```

以上三个select是等效的。
但是create*函数都会有一个记忆功能。谁稀罕你这个功能！

# effect
关键点
- Effects使得我们的component同service脱钩，让它更纯，只负责选择一个state和发起一个action
- Effects是长久的服务，它监听**每一个**从store派发的action
- 通过ofType操作符来选取自己感兴趣的action
- Effect的操作可以是同步也可以是异步，并返回新的action（注：不一定返回新的action，见最下面的例子）

```
import { Injectable } from '@angular/core';
import { Actions, createEffect, ofType } from '@ngrx/effects';
import { EMPTY } from 'rxjs';
import { map, mergeMap, catchError } from 'rxjs/operators';
import { MoviesService } from './movies.service';

@Injectable()
export class MovieEffects {

  loadMovies$ = createEffect(() => this.actions$.pipe(
    ofType('[Movies Page] Load Movies'),
    mergeMap(() => this.moviesService.getAll()
      .pipe(
        map(movies => ({ type: '[Movies API] Movies Loaded Success', payload: movies })),
        catchError(() => EMPTY)
      ))
    )
  );

  constructor(
    private actions$: Actions,
    private moviesService: MoviesService
  ) {}
}
```
effect的创建，使用class，注入Actions和所需要的数据获取的service。
创建一个针对特定action的监听的源。

注：这边的catchError会在请求失败时，不做任何相应。这样这个action源就不会中断了。如果你在catch里throw了error，那么这个持续监听就失败了。所以在action源的高阶处理器里的回调里，我们catch到错误就返回EMPTY。这是我总结的recipe。

还有你这个catchError是一定要写的，不catch就在service里直接error，那么合并后的源也在此时被终止了。

事件源不限于我们通过dispatch派发的事件，也可以是路由状态变化等。

```ts
import { Injectable } from '@angular/core';
import { Actions, ofType, createEffect } from '@ngrx/effects';
import { of } from 'rxjs';
import { catchError, exhaustMap, map } from 'rxjs/operators';
import {
  LoginPageActions,
  AuthApiActions,
} from '../actions';
import { Credentials } from '../models/user';
import { AuthService } from '../services/auth.service';

@Injectable()
export class AuthEffects {
  login$ = createEffect(() =>
    this.actions$.pipe(
      ofType(LoginPageActions.login),
      exhaustMap(action =>
        this.authService.login(action.credentials).pipe(
          map(user => AuthApiActions.loginSuccess({ user })),
          catchError(error => of(AuthApiActions.loginFailure({ error })))
        )
      )
    )
  );

  constructor(
    private actions$: Actions,
    private authService: AuthService
  ) {}
}
```
在这里我们用exhaustMap，这样可以我们发起请求后会等待其返回结果，无论是失败或成功。当成功或失败，我们发送不同的action来告诉store改变state

```ts
import { Injectable } from '@angular/core';
import { Store } from '@ngrx/store';
import { Actions, ofType, createEffect, concatLatestFrom } from '@ngrx/effects';
import { tap } from 'rxjs/operators';
import { CollectionApiActions } from '../actions';
import * as fromBooks from '../reducers';

@Injectable()
export class CollectionEffects {
  addBookToCollectionSuccess$ = createEffect(
    () =>
      this.actions$.pipe(
        ofType(CollectionApiActions.addBookSuccess),
        concatLatestFrom(action => this.store.select(fromBooks.getCollectionBookIds)),
        tap(([action, bookCollection]) => {
          if (bookCollection.length === 1) {
            window.alert('Congrats on adding your first book!');
          } else {
            window.alert('You have added book number ' + bookCollection.length);
          }
        })
      ),
    { dispatch: false }
  );

  constructor(
    private actions$: Actions,
    private store: Store<fromBooks.State>
  ) {}
}
```
这个例子，同时用到了store来获取部分的state，达到的效果就是可以针对不同的收藏数来给出不一样的通知。我们看到effect不一定是要产生新的action，只要是side effect的东西都可以封装成effect！

用concatLatestFrom来获取最近的collection数量，注意我们只要在添加成功后才会去获取数量。


# tips
使用createAction得到的actionCreator函数，要获得它们返回的action类，可以有两种方式
```ts
import { createAction, union } from '@ngrx/store';
const all = union({ logout, logoutConfirmation, logoutConfirmationDismiss });
export type AuthActionsUnion = typeof all;

export type AuthApiActionsUnion = ReturnType<
  typeof loginSuccess | typeof loginFailure | typeof loginRedirect
>;
```
一种是用的ngrx的union函数，传入actions的键值对构成的对象。不推荐。

另一种是用ts的工具类ReturnType，推荐

目录结构：
- actions
- components
- containers
- effects
- models
- reducers
  - a.reducer.ts
  - b.reducer.ts
  - index.ts
- services
- routing.module.ts
- module.ts

reducer里面定义自己负责的state interface，定义初始的state，定义selector。

reducer就是一个switch case，switch的值是action的type，就是个string
在index.ts里定义module下的state，定义reducers，是一个ActionReducerMap，它接受两个泛型参数，一个是module下的state，一个是actions的union type
```ts
import {
  createSelector,
  createFeatureSelector,
  ActionReducerMap,
} from '@ngrx/store';
import * as fromRoot from '../../reducers';
import * as fromAuth from './auth.reducer';
import * as fromLoginPage from './login-page.reducer';
import { AuthApiActions } from '../actions';

export interface AuthState {
  status: fromAuth.State;
  loginPage: fromLoginPage.State;
}

export interface State extends fromRoot.State {
  auth: AuthState;
}

//键值对，对应到自己下面的子reducer，key名字跟后面的selector要统一
export const reducers: ActionReducerMap<
  AuthState,
  AuthApiActions.AuthApiActionsUnion
> = {
  status: fromAuth.reducer,
  loginPage: fromLoginPage.reducer,
};

//选取当前module下的state
export const selectAuthState = createFeatureSelector<State, AuthState>('auth');
//选取user status {user}
export const selectAuthStatusState = createSelector(
  selectAuthState,
  (state: AuthState) => state.status
);
export const getUser = createSelector(selectAuthStatusState, fromAuth.getUser);
export const getLoggedIn = createSelector(getUser, user => !!user);

export const selectLoginPageState = createSelector(
  selectAuthState,
  (state: AuthState) => state.loginPage
);
export const getLoginPageError = createSelector(
  selectLoginPageState,
  fromLoginPage.getError
);
export const getLoginPagePending = createSelector(
  selectLoginPageState,
  fromLoginPage.getPending
);

```

因为createFeatureSelector需要parent state的type，所以我们需要导入parent state的type，我们还把这个state extend了，是因为我们要从parent state里选取我们当前的（这个模块定义的）state。你不extend的话，ts会报错，因为它找不到parent state下有你传入的key（'auth')。挺搞的。

也就是说，每一个module的reducer index.ts文件下都要定义自己的state要被如何选中，也就是自己这块state叫什么名字。

```ts 
# auth.module.ts
StoreModule.forFeature('auth', reducers),
```
在module下面注册你的store，注意key跟你在index.ts下定义的key保持一致。

reducer会听所有的action的！

effect也可以监听已经被reducer监听的流。

比如用户点了login的按钮，那么state里pending就会变成是true，同时effect也监听到了login这个动作，发送api请求。

effect可以有很多乱七八糟的依赖注入，可以有各种奇葩逻辑。

没有dispatch新的action的effect，一般都要有一个tap操作符来做点事情，比如路由跳转。

有些action可能什么都不做，那就可以不处理。比方说用户取消了一个弹窗，state不会改变（弹窗的隐藏是由local state掌管的，通常是ui库），这些action，都写到default分支里，原样返回state，原样的state可以不用拷贝，直接返回就行。














