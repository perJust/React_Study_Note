1. fiber是一种数据结构

   > vdom
   
2. useState其实就是基于useReducer而来的

   > `useReducer(reducer, initial)`， useState就是内置一个reducer

   ```javascript
   // 
   function useState(initialState) {
   	return updateReducer(basicStateReducer, initialState);
   }
   function basicStateReducer(state, action) {
   	return typeof action === 'function'?action(state):action;
   }
   ```

3. hooks其实是通过`代数效应`的思想实现

4. 每次进行setState时都会创建一个全新的fiber树

5. 整个组件内hooks执行的核心思想

   ```note
   1、组件内所有的hook都挂在WorkInProgressHooks链表上（单向链表）；
   2、mount与update时，是使用的不同的hooks，其实就是将代码里if...else...抽离成不同的执行函数；如：useState中，mount时对应mountState，update时对应updateState；源码看详细1。
   3、对于workInProgressHooks来说，mount与update时：其实就是第2点的补充。每次进行hooks的调用时，会根据是否mount进行不同的拿取(也就相当于详细2中isMount省去，维持核心代码统一处理方式，将差别赋予中间执行变量)。如：mountState函数中执行的是 hook(当前正在执行的hook) = mountWorkInProgressHooks()；updateState函数中执行的是 hook(当前正在执行的hook) = updateWorkInProgressHooks()；促进理解看详细2.
   ```



```javascript
// 详细1
// 组件使用hooks不同
const HooksDispatcherOnMount: Dispatcher = {
  readContext,

  useCallback: mountCallback,
  useContext: readContext,
  useEffect: mountEffect,
  useImperativeHandle: mountImperativeHandle,
  useLayoutEffect: mountLayoutEffect,
  useMemo: mountMemo,
  useReducer: mountReducer,
  useRef: mountRef,
  useState: mountState,
  // ...
  unstable_isNewReconciler: enableNewReconciler,
};
const HooksDispatcherOnUpdate: Dispatcher = {
  readContext,

  useCallback: updateCallback,
  useContext: readContext,
  useEffect: updateEffect,
  useImperativeHandle: updateImperativeHandle,
  useLayoutEffect: updateLayoutEffect,
  useMemo: updateMemo,
  useReducer: updateReducer,
  useRef: updateRef,
  useState: updateState,
  // ...
  unstable_isNewReconciler: enableNewReconciler,
};

// 核心判断：初始赋值为空时，用HooksDispatcherOnMount
ReactCurrentDispatcher.current =
      current === null || current.memoizedState === null
        ? HooksDispatcherOnMount
        : HooksDispatcherOnUpdate;

// useMutableSource函数中进行
const dispatcher = ReactCurrentDispatcher.current;
// 然后开始使用
```



```javascript
// 详细2
// 实现简易useState

// 是否首次加载，真正源码中是将 if...else... 抽离成不同的执行  HooksDispatcherOnMount与HooksDispatcherOnUpdate
let isMount = true;
// 存当前组件的正在运行的hook，即：存的是正在运行的hook指向
let workInProgressHook = null; // 始终指向当前运行的hook

// 组件节点
const fiber = {
    stateNode: App, // 函数组件 stateNode指向本身
    // 存函数组件里定义的状态, 通过链表的方式存组件内定义的多个hook
    memoizedState: null, 
}

// 环状链表方便处理优先级
// setState 其实内部是基于dispatchAction
function dispatchAction(queue, action) {
    const update = { // 这个update就是链表尾部
        action,
        next: null
    }

    // 循环链表
    if(queue.pending === null) {
        update.next = update;
    } else { // 就是两步：新来的update需要放到上一个的后面，且新来的next需要连接链表的首部形成环
        // （类推得知）上一个的next（即queue.pending.next）其实就是链表的首部
        update.next = queue.pending.next;
        queue.pending.next = update;
    }
    queue.pending = update;

    schedule();
}
// 进行组件的调度
function schedule() {
    workInProgressHook = fiber.memoizedState; // 在初始化之前，先将workInProgressHook指向hook链表的第一个
    const app = fiber.stateNode(); // 初始化
    isMount = false; // 首次调用后，之后就是update

    // 返回结果，只是方便控制台测试
    return app;
}

// 整个组件内所有用到同样hook的其实是个单向链表  一个hook指向下一个hook
function useState(initialState) {
    let hook; // 用于存当前用的hook

    if(isMount) {  // !---  这里就相当于源码中  mountState
        hook = {
            memoizedState: initialState,
            next: null, // 链表指针下一个
            queue: { // 修改state的链表  将setState的操作全放入pending  这里是循环链表
                pending: null
            }
        }
        if(!fiber.memoizedState) { // 如果当前组件状态的链表是还未初始化的
            fiber.memoizedState = hook;
            workInProgressHook = hook;
        } else {
            workInProgressHook.next = hook;
        }
        workInProgressHook = hook; // 指向当前hook
    } else { // 	!---  这里就相当于源码中  updateState
        hook = workInProgressHook; // hook指向之前存的hook
        workInProgressHook = workInProgressHook.next; 
    }

    // 计算新的state
    let baseState = hook.memoizedState;
    if(hook.queue.pending) {
        let firstUpdate = hook.queue.pending.next;
        do {
            const action = firstUpdate.action;
            baseState = action(baseState); 
            firstUpdate = firstUpdate.next;
        } while (firstUpdate !== hook.queue.pending.next)
        // 计算完后，需要将action的链表清空
        hook.queue.pending = null;
    }
    hook.memoizedState = baseState;

    return [baseState, dispatchAction.bind(null, hook.queue)]; // setState锁定第一个参数为queue，所以外部调用setState时只传入action即可； 
    // 额外概念：fn1 = fn.bind(context, arg1, arg2 ...argn)方法会锁定函数this指向并且锁定argn个传参，fn1(arg)实际执行时arg是作为第n+1个传参执行
}
```

