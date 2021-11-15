Angular的核心：
 - component
 - template
    - directive:
- 依赖注入

Angular CLI常用指令：
```
ng build	
ng serve	
ng generate	
ng test	
ng e2e
```


reference的用法：#name

`*ngIf="canEdit; else noEdit"`

# DI
## 服务

服务需要provider才能使用。

一个比方就是：
医生的处方->provider，
药 -> service，
只有有了处方，才能得到并使用这个药。
provider有三个重要内容：
Token, Where, and what

Token用来唯一locate服务，where指明需要的地方，what指明具体需要什么

用药方的比喻就是开的什么药，给谁吃，和具体怎么吃

where有四个层级
Platform， root， module，和any，前三者都是singleton，后者是不限个数的

### 对于自己写的服务：

要么需要Injectable装饰器， 提供providedIn参数

或者写入ngModule的providers数组里，这样才能正常在constructor里使用服务。

但是推荐使用Injectable，因为ngModule无法执行Treeshaking，所以如果你声明了provider但是最后没有在构造器里使用，angular还是会将服务包含进build里

### 对于第三方或者angular内置的服务，如httpclient：

则需在需要依赖的module的装饰器的import数组中加入该服务所属的module！

NG的Provider有三个成分，一个是Token，一个是provideredIn，一个是实际使用的值，默认就是被装饰的class，也可以使用别的值，如一个字符串


## property binding

要在template中使用[propertyName]="value"不要忘了使用Input装饰器来修饰属性

对于未知的custom/web component，可以在app module的schema数组里设定相关的schema来bypass known property error

## class binding

## single class binding
`[class.className]="expression"`
这种syntax是ng compiler层面定义的语法

## multi class binding,新加的
`[class]="obj"`
这种语法是每个人都可以定义的directive语法

## ngClass directive
`[ngClass] = "expression"`

ngClass和class没什么区别了

## pipe
自己创建的pipe需要加入module的declarations数组里

要使用第三方的pipe则需要将相应的module加入import数组里

## template variable

基础用法 `#name`

对于ngForm这种directive可以`#name="ngForm"`

## content projection
view vs content

对于multi-content，使用`<ng-content>标签，再加上`select="css/attribute selector"`来指定内容应该对应哪个ng-content

如你有一个input元素，你在模版里添加了#name这样的reference，那么你就可以用@ViewChild()装饰器来获取type为ElementRef的视图元素。
注意视图元素只有在ngAfterViewInit后才能访问。

同样适用于TemplateRef和指令，注意获取单个指令还需要标注exportAs


## lifecycle
绑定的数据只有在ngOnInit钩子后才能访问

##Form
在template里使用`#formName="ngForm"`
`@ViewChild("formName") var: ngForm`

或者在template直接写ngForm
`@ViewChild(ngForm)`即可。

对于ViewChild的改变，可以用ngOnChange hook来观察，注意此时如果要修改parent的property，必须在下一个change detection cycle来改变，因为此时parent的view已经改变过了。通常使用setTimeout或者promise.resolve来触发新的cycle。
而contentChild则不存在这个问题。因为content composition发生在view只前

## View Encapsulation
Angular有三种style的嵌入方式，default是emulated，每个元素会生成独特的token，并且对于的style也会添加这些token的属性修饰，来模拟原生的shawdow dom，style也写入head里

Shadow Dom自支持样式的隔离，style写在shadow dom里

None的话只是在head里添加相应的样式，不做其他处理

注意不要mix三种模式

## Component Interaction
parent 给child传值，child要用@Input装饰器
### 下接受上
#### setter
对于单一的property改变，可以用js的setter函数来做出响应 ｀@Input() set propname(){
    ...
}｀

#### onChange
对于关联的prop change，可以使用ngOnChanges hook来做出响应，如针对大小版本的变动，你要写一条log信息，那这个信息是要同时包含这两个变动的，你不太可能对两个属性分别作检测来完成这个要求。

### 上接受下
#### EventEmitter
child给parent传东西要用EventEmitter<>,
`e.emit(payload)`
parent模板`(event)="handler($event)"`

#### template reference
parent可以通过local variable来获取child的属性和方法！获取方式是#refName

但是上述方式只有在parent的template中才可以奏效，parent和child class之间并没有建立起联系。

#### Viewchild
要在parent的class中访问child的属性方法，需要使用viewChild来装饰。

#### 更复杂的情况可以用service
还可以通过service的方法来通信。通常这个service由rsjs Observable来实现。服务可以发布新的消息，也可以订阅新的消息。双方都可以发布和订阅，也比较灵活。

注意在child destroy的时候unsubscribe之前的stream，parent不需要，因为如果service加在parent上，那么service是会自动销毁的。

## Component styles
component有一些特别的css，主要是:host和:host-context

每一个元素都会有一个host element何其对应

functional form
`:host(.active) {}`

`:host-context(.className)`

host-context只能用functional form，当一个host上层的某个元素拥有这个class的时候，才会起效。如当body有某一个主题样式时，我们才对host元素做一些样式变动。

注意，该选择器仍然只针对host和其descendent，不是针对它的ancestor！

## Content projection
投射可以用来创建一些可扩展的组件，如对话框，样式边框是统一的，里面却存放不同的内容。

最普通的单槽投射就是用`<ng-content>`作为投射的地方

对于同时做多个投射的场景，使用select属性来区分被投射的元素，不加select属性的ng-content会自动回收未被其他有select属性吸收的内容。

### conditional projection
没看懂 https://angular.io/guide/content-projection#conditional-content-projection

ngProjectAs可以用来做更加复杂的projection，比方说你要投射的元素在另一个其他的元素里面

## 动态组件

### anchor directive
一下directive可以用来获取host元素
```
import { Directive, ViewContainerRef } from '@angular/core';

@Directive({
  selector: '[adHost]',
})
export class AdDirective {
  constructor(public viewContainerRef: ViewContainerRef) { }
}

```

### loading component
可以使用ng-template配合之前的anchor directive来告诉angular在哪里添加组件，ng-template不产生额外dom，所以是个不错的选择

### resolving component
ComponentFactoryResolver来获取ComponentFactory

directive的viewContainerRef可以createComponent，创建的组件再去绑定数据

这里绑定数据是统一的，因为所有的广告都实现了统一的接口，这个接口很简单，就一个data对象，但是data可以长任何样

### template

template的表达式变量resolve方式是这样的：
1. template variable
2. directive variable
3. class member

template里不要有side effect，不要改变别的值

### pipes
使用pipe的时候要注意，angular对于使用到pipe的场景，并不使用其default change detection机制，比如对于reference的变量，`array | filter`，这种写法不会触发angular对view的update。这么做是为了让程序能更快（更坑）。

解决方式，一种是创建新的数组，让pipe能察觉变化

另一种是用impure pipe，只要在Pipe装饰器里加入pure:false即可

注意不要放入heavy logic，不然会降低angular性能

async pipe可以处理Promise和Observable两种对象，包含了自动的订阅，自动的unsubscribe，避免造轮子

async是commonModule下面提供的

BroswerModule把commonModule reexport了

### 异步ajax请求

可以用angular自带的http client来发起请求，http返回Observable，用async来transform

或者可以custom一个pipe，包含cache的机制

注：我觉得不用pure:false,因为参数是url string，本身就是immutable的

可以包含多个pipe，每个pipe管理自己的data

pipe的优先级大于三目运算符


### Module
```ts
@NgModule({
  declarations: [HelloFrameworkComponent],
  exports: [HelloFrameworkComponent],
  imports: [
    SharedModule,
    MaterialModule,
  ],
})
```

一般module包含declarations， exports和imports：
- declarations：这个module需要什么component， directive，pipe
- exports：别的module要导入这个module，可以导入什么
- imports：同exports相对

一些module可以只做module的管理工作

比方建立一个sharedModule，里面只**导出export**通常要用的module，如` CommonModule,
    FormsModule,
    ReactiveFormsModule,
    RouterModule,`

对于经常用到的样式如material子模块，也可以这样管理

导入的module如果包含了必要的组件，directive，pipe，那么你就不需要再去declaration了，declaration是针对本模块内的其他组件directive互相知道彼此存在而用的，外部模块需要这些组件直接import本模块就可以了，前提本模块也把自己declare的东西放在了export里面！

所以说module一个功能是为了方便管理组件directive而建立的

exports makes the components, directives, and pipes available in modules that add this module to imports. exports can also be used to re-export modules such as CommonModule and FormsModule, which is often done in shared modules.

### property binding

[innerHTML]="str",innerHTML会自动去除危险的元素，如script tag，而interpolation则不会解析html，直接按照string输出

同interpolation区别
语法上，property binding如同：`[src]="imageURL"`

而interpolation则是`src="{{imageURL}}"`

在bind string类型时是基本相同的功能，但是如果需要绑定的是非字符串类型，那么就要用property binding了。


所以绑定数据时一般都是`[data]="propName"`
`[data]="{{expression}}"`，这种是违法的

property binding就是把一个component的属性绑到一个元素的属性上，所以没有表达式

{{}}这是interpolation语法

both interpolation and property binding can set only properties
colspan是attribute，colSpan是property

### attribute binding
[attr.attrName]="expression"

同property binding区别：attribute binding前面要加attr prefix

### class binding
加class prefix或一个class

### style binding
加style prefix或一个style

template binding的优先级大于host binding和directive binding

@HostBinding(attr)
可以把property bind到host的attr上
```
<comp-with-host-binding [class.special]="isSpecial" dirWithClassBinding></comp-with-host-binding>
```
如上，是否添加special的class会由template binding决定（也就是parent上的isSpecial属性），尽管在这个host component上也设置了host binding

注意null和undefined的区别，设置一个attr为null，那么该attr会被移除，而设置undefined，则这个attr不会被移除

`<comp-with-host-binding dirWithHostBinding></comp-with-host-binding>`
比如这个情况，如果directive binding设置了null，那么host binding将完全不起效，而如果directive binding设置了undefined，那么host-binding会起效

另外动态的binding优先级大于静态的binding

#### injecting attribute value
有些时候你想针对一些静态的属性做一些行为的改变。这个时候你可以在该component/directive的constructor里加入
`@Attribte('the type name you set on the host element') public propertyName`

注意同@Input的区别，@Input用来动态绑定，而Attribute则是用来静态绑定的

### event binding
```ts
@Directive({selector: '[myClick]'})
export class ClickDirective {
  @Output('myClick') clicks = new EventEmitter<string>(); //  @Output(alias) propertyName = ...

  toggle = false;

  constructor(el: ElementRef) {
    el.nativeElement
      .addEventListener('click', (event: Event) => {
        this.toggle = !this.toggle;
        this.clicks.emit(this.toggle ? 'Click!' : '');
      });
  }
}
```
这里的directive名字同其事件属性是一样的，就构成了一个custom的click event了，在构造器里设置了对于host element的click事件的包装

### two-way binding

实现two-way binding 假设input的name是bar，那么output的name要加后缀Change！

对于form元素，由于事件和值不follow这个pattern，所以要用ngModel directive

### template variable
在使用的時候要注意，对于ng-if，ng-for这样的structural directive，实际上angular会创建一个ng-template，如果要在外部访问这个reference，angular会报错。

 template variable默认就是绑定host element，要改变的话需要加上directive的别名，如`#itemForm="ngForm"`，这个别名在directive的exportAs属性中定义

 ngForm directive在引入formModule之后自动就被导入了，加入这个reference是为了更好观察form里面的值

angular可以使用svg作为template，非常强大，可以结合d3做动态图表

### directive
获取index：
`<div *ngFor="let item of items; let i=index">{{i + 1}} - {{item.name}}</div>`

设置trackBy,可以避免不必要的server请求
`<div *ngFor="let item of items; trackBy: trackByItems">
  ({{item.id}}) {{item.name}}
</div>`

#### ng-switch
```html
<div [ngSwitch]="currentItem.feature">
  <app-stout-item    *ngSwitchCase="'stout'"    [item]="currentItem"></app-stout-item>
  <app-device-item   *ngSwitchCase="'slim'"     [item]="currentItem"></app-device-item>
  <app-lost-item     *ngSwitchCase="'vintage'"  [item]="currentItem"></app-lost-item>
  <app-best-item     *ngSwitchCase="'bright'"   [item]="currentItem"></app-best-item>
  <div *ngSwitchCase="'bright'"> Are you as bright as {{currentItem.name}}?</div>
  <app-unknown-item  *ngSwitchDefault           [item]="currentItem"></app-unknown-item>
</div>
```
ng-switch由ngSwitch、*ngSwitchCase和*ngSwitchDefault组成
ng-switch绑定一个property，ngSwitchCase做匹配，ngSwitchDefault是在匹配不到的情况下的默认选项，只有一个case会被创建。

可以绑定任意类型值。

### attribute directive
构建custom directive，并同时接受parent的binding，只要将它的input property设置成directive selector name一样就行了

要处理host事件，引入HostListener装饰器，传入事件名就可以了。

ElementRef可以获取host的引用。在constructor注入就可以。

获取对应的dom元素用nativeElement属性

ngNonBindable可以阻止interpolation，binding，但是只对element的children生效，作用在element上的directive还是有效的！如：
```html
<div ngNonBindable [appHighlight]="'yellow'">This should not evaluate: {{ 1 +1 }}, but will highlight yellow.
</div>
```

### structural directive
创建方法同attribute directive大致一样，
只是在使用的时候用*符号，这样angular知道它是structural directive，在constructor会传入 TemplateRef<any>,ViewContainerRef这两个类型的参数，后者的createEmbeddedView可以创建一个新的view到container

clear方法可以清空创建的view

### DI
好处：
- 方便测试
- 方便部署不同环境

#### 手动书写DI

由于我们class自己不会手动创建依赖，那么必然系统的某处会去帮我们创建依赖，对于手动设置的依赖，我们就把这种映射关系写在模块或组件的providers数组里

provider能告诉angular如何取创建一个依赖，它需要
- token（provide）
- factory function（useFactory）
- dependency？（deps）

可以简单理解为一个哈希表，token为key，factory function为value

需要什么依赖，就通过对应的token查找，通过fatory函数来创建

token通过`new InjectionToken<T>`(nameStr)来创建，保证唯一性

此时只是确保了在需要的时候能够被正确的创建，但是通知angular去调用还需要我们自己去设置。

需通过对constructor里需要注入的parameter加入Inject装饰器做修饰，传入对应的token。这样angular就知道何时去调用了。

此时就完成了一个DI的配置




































































































