# Partial<Type>

这个type可以把传入的type的所有属性变成optional的。也就是扩展成传入的type的所有subset。

使用场景：当你要更新某一个entity时，你可以使用这个Partial类来修饰你的更新参数。
```ts
interface Todo {
  title: string;
  description: string;
}
 
function updateTodo(todo: Todo, fieldsToUpdate: Partial<Todo>) {
  return { ...todo, ...fieldsToUpdate };
}
 
const todo1 = {
  title: "organize desk",
  description: "clear clutter",
};
 
const todo2 = updateTodo(todo1, {
  description: "throw out trash",
});
```
上面这个例子里的updateTodo就可以既确保参数的安全，也可以最大程度允许参数的自由。

# ReturnType<Type>
当你有一个function，如果你想获得它的return type，就是用这个类

应用场景：你有一个构造函数，返回一个类，你需要获得这个类，那么就可以`ReturnType<typeof funcName>`



