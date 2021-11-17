# 引入外部样式脚本
- 通过cdn方式在html页面引入
- 通过angular.json的build里的style方式引入，需要重启进程
- 通过在全局scss文件里使用@import语句来引入样式，推荐。因为不需要重启
- 对于js文件，可以在main.ts里引用，也是全局的。

不能在多个module里declare相同的组件

自定义属性需要通过attr来绑定，如[attr.data-title]="",主要是指带有-的属性

要双向绑定的话，output需要设置成inputName后面再加上Change

ngModel在表单中需要设置name属性

ngModel可以绑定一个对象里的属性，不一定是要一个单独的属性。

ngElse后面跟的值可以是template reference，也可以是组件的一个变量。组件的变量需要reference一个template。

template reference通过@ViewChild('referenceName', {static: true}) ref: TemplateRef<any>在组件中访问

再通过在ngOnInit赋值的方式把这个reference赋值给组件变量

？安全导航运算符，或叫可选链。如hero?.name，即使hero未被初始化，也不会报错，只不过这一块不会被渲染

如a?.b?.c等同于`a && a.b && a.b.c`

！非空断言

只有当tsconfig开启了strictNullChecks后，这个操作符才会生效。
```ts
const name:string | null = "fefe";
ngOnInit(){
    const _name:string = this.name!;
}
```
如以上情况，即使我们给name初始化，但是因为类型包含null，ts会报错。同时注意也要关闭tslint关于！操作符的警告。

如果不用！操作符，就这样书写
```
if (this.name) {
    const _name:string = this.name;
}
```

所以!和?都是一种简写，都有对应的不使用的写法。

关于组件也是一种特殊的指令。

在我们写一个组件/指令时，我们会给它们一个selector，注意这边的selector跟css的selector是一样的。所以如果是element的，就类似h1,p，而如果是属性选择器，则是类似于input[type="text"] ，里面的方括号就是属性选择器。所以在创建属性指令的时候，我们的selector应该是[selectorName]的格式。你甚至可以把之前的元素便化成属性指令，套用到div上。

指令可以监听宿主。注意这里指的是在内部监听宿主的事件。通常情况下，只要在宿主上增加事件监听即可。

使用指令来访问宿主并不被推荐。原因跟webworker有关。

要监听就给函数添加@HostListener装饰器，参数是事件名和一个参数数组，第一个为事件。

如果指令接受input参数，可以给input参数设置同指令相同的别名，这样在外部就可以把指令和传参合并为一条传参语句

ng-template，ng-container
我们要通过逻辑的方式来将ng-template插入ng-container,只要获得这两个reference，然后调用后者的createEmbeddedView方法，把前者传入就可以了。就这么简单。

如果模板已经创立好，那每次insert不会创建新的进入视图中。

containerRef的move方法可以将已创建的模板插入到视图中。

创建embeddedview的时候可以传一个context对象，然后在模板里使用
let-var="propInContext"的方式来动态绑定一些值。 

除了通过ngContainer的方式来注入template，我们还有ngTemplateOutlet指令可以绑定一个templateRef。这样我们可以在组件外面传入这个templateRef，并在组件内部使用，同时还可以设置默认的templateRef。

ng-template可以设置变量，通过let-var="bindingVar"的格式，也可以通过let-var的格式，如果是后者，则会使用context对象的$implicit属性来赋值。

```ts
<ng-container *ngTemplateOutlet="templateRefExp; context: contextExp"></ng-container>
```

```ts
@Component({
  selector: 'ng-template-outlet-example',
  template: `
    <ng-container *ngTemplateOutlet="greet"></ng-container>
    <hr>
    <ng-container *ngTemplateOutlet="eng; context: myContext"></ng-container>
    <hr>
    <ng-container *ngTemplateOutlet="svk; context: myContext"></ng-container>
    <hr>
    <ng-template #greet><span>Hello</span></ng-template>
    <ng-template #eng let-name><span>Hello {{name}}!</span></ng-template>
    <ng-template #svk let-person="localSk"><span>Ahoj {{person}}!</span></ng-template>
`
})
class NgTemplateOutletExample {
  myContext = {$implicit: 'World', localSk: 'Svet'};
}
```

# viewChild
viewChild默认是在变更检测之后才能获得，所以一般可以在afterViewInit之后获得。要更早获得的话，可以传入{static:true}参数，适用于目标一开始就出现在模板上且没有使用ngIf这样的指令的情况下。










