
## 四种全局状态数据


在实际开发当中，会遇到四种全局状态数据：`异步数据（一般来自服务端）`、`同步数据`。同步数据又分为三种：`localstorage`、`cookie`、`内存`。在传统的 Vue3 当中，分别采用不同的机制来处理这些状态数据，而在 Zova 中只需要采用统一的`Model`机制




| 状态数据 | 传统的Vue3 | Zova |
| --- | --- | --- |
| 异步数据 | Pinia | Model |
| localstorage | Pinia \+ Localstorage | Model |
| cookie | Pinia \+ Cookie | Model |
| 内存 | Pinia | Model |


采用 Model 机制统一管理这些全局状态数据，就可以提供一些通用的系统能力，比如，`内存优化`、`持久化`和`SSR支持`等等，从而规范数据使用方式，简化代码结构，提升代码的可维护性


## 特性1\. 支持异步数据和同步数据


Zova Model 的基座是[TanStack Query](https://github.com)。TanStack Query 提供了强大的数据获取、缓存和更新能力。如果你没有使用过类似TanStack Query的数据管理机制，那么强烈建议了解一下，相信你一定会受到思想的洗礼
但是，TanStack Query 的核心是对异步数据（一般来自服务端）进行管理。Zova Model 在 TanStack Query 的基础上做了扩展，因此也支持同步数据的管理。换而言之，以下所述所有特性和能力同时适用于`异步数据`和`同步数据`


## 特性2\. 自动缓存


对获取的异步数据进行本地缓存，避免重复获取。对于同步数据，会自动针对 localstorage 或者 cookie 进行读写操作


## 特性3\. 自动更新


提供数据过期策略，在合适的时机自动更新


## 特性4\. 减少重复请求


在程序的多个地方同时访问数据，将只调用一次服务端 api。如果是同步数据，也只针对 localstorage 或者 cookie 调用一次操作


## 特性5\. 内存优化


通过 Zova Model 管理的数据，虽然是全局范围的状态，但是并不总是占用内存，而是提供了内存释放与回收的机制。具体而言，就是在创建 Vue 组件实例时根据业务的需要创建缓存数据，当 Vue 组件实例卸载时释放对缓存数据的引用，到达约定的过期时间如果仍然没有其他 Vue 组件引用，就会触发回收机制(GC)，完成对内存的释放，从而节约内存占用。这对于大型项目，用户需要长时间进行界面交互的场景，具有显著的好处


## 特性6\. 持久化


本地缓存可以持久化，当页面刷新时可以自动恢复，避免服务端调用。如果是异步数据，就会自动持久化到 IndexDB 中，从而满足大数据量的存储需要。如果是同步数据，就会自动持久化到 localstorage 或者 cookie


`内存优化`与`持久化`配合发挥作用，对于大型项目效果更佳明显。比如，第一次从服务端获取的数据，会生成本地缓存，并自动持久化。当页面不再使用并且过期时，会自动销毁本地缓存，从而释放内存。当再次访问该数据时，会自动从持久化中恢复本地缓存数据，而不是再次从服务端获取数据


## 特性7\. SSR支持


不同类型的状态数据，在 SSR 模式下也会有不同的实现机制。Zova Model 把这些状态数据的差异进行抹平，并且采用统一的机制进行水合，从而让 SSR 的实现更加自然、直观，显著降低了心智负担


## 特性8\. 自动命名空间隔离


Zova 通过 Model Bean 来管理数据。而 Bean 本身有唯一的标识，可以作为数据的命名空间，从而自动保证了 Bean 内部状态数据命名的唯一性，避免数据冲突


* 参见：[Bean标识](https://github.com)


## 如何创建一个Model Bean


Zova提供了VS Code插件，通过右键菜单可以非常便利的创建一个Model Bean



> 右键菜单 \- \[模块路径]: `Zova Create/Bean: Model`


依据提示输入 model bean 的名称，比如`todo`，VSCode 插件会自动添加 model bean 的代码骨架


比如，在 demo\-todo 模块中创建一个 Model Bean `todo`


`demo-todo/src/bean/model.todo.ts`



```
import { Model } from 'zova';
import { BeanModelBase } from 'zova-module-a-model';

@Model()
export class ModelTodo extends BeanModelBase {}

```

* 使用@Model 装饰器
* 继承自基类 BeanModelBase


## 异步数据


TanStack Query 的核心是对服务端数据进行管理。为简化起见，这里仅展示select方法的定义与使用:


* 完整代码示例，请参见：[demo\-todo](https://github.com)


### 如何定义



```
@Model()
export class ModelTodo {
  select() {
    return this.$useQueryExisting({
      queryKey: ['select'],
      queryFn: async () => {
        return this.scope.service.todo.select();
      },
    });
  }
}

```

* 调用$useQueryExisting 创建 Query 对象
	+ 为何不使用`$useQuery`方法？因为异步数据一般是在需要时才进行异步加载。因此我们需要确保在多次调用`select`方法时始终返回同一个 Query 对象，所以必须使用`$useQueryExisting`方法
* 传入 queryKey，确保本地缓存的唯一性
* 传入 queryFn，在合适的时机调用此函数获取服务端数据
	+ service.todo.select：参见[Api服务](https://github.com):[wgetCloud机场](https://tabijibiyori.org)


### 如何使用


`demo-todo/src/page/todo/controller.ts`



```
import { ModelTodo } from '../../bean/model.todo.js';

export class ControllerPageTodo {
  @Use()
  $$modelTodo: ModelTodo;
}

```

* 注入 Model Bean 实例：$$modelTodo


`demo-todo/src/page/todo/render.tsx`



```
export class RenderTodo {
  render() {
    const todos = this.$$modelTodo.select();
    return (
      <div>
        <div>isLoading: {todos.isLoading}div>
        <div>
          {todos.data?.map(item => {
            return <div>{item.title}div>;
          })}
        div>
      div>
    );
  }
}

```

* 调用 select 方法获取 Query 对象
	+ render 方法会多次执行，重复调用 select 方法返回的是同一个 Query 对象
* 直接使用 Query 对象中的状态和数据
	+ 参见：[TanStack Query: Queries](https://github.com)


### 如何支持SSR


在 SSR 模式下，我们需要这样使用异步数据：在服务端加载状态数据，然后通过 render 方法渲染成 html 字符串。状态数据和 html 字符串会同时发送到客户端，客户端在进行水合时仍然使用此相同的状态数据，从而保持状态的一致性


要实现以上逻辑，在 Zova Model 中只需要执行一个步骤：


`demo-todo/src/page/todo/controller.ts`



```
import { ModelTodo } from '../../bean/model.todo.js';

export class ControllerPageTodo {
  @Use()
  $$modelTodo: ModelTodo;

  protected async __init__() {
    const queryTodos = this.$$modelTodo.select();
    await queryTodos.suspense();
    if (queryTodos.error) throw queryTodos.error;
  }
}

```

* 只需要在`__init__`方法中调用`suspense`等待异步数据加载完成


## 同步数据: localstorage


由于服务端不支持`window.localStorage`，因此 localstorage 状态数据不参与 SSR 的水合过程


下面演示把用户信息存入 localstorage，当页面刷新时也会保持状态


### 如何定义



```
export class ModelUser extends BeanModelBase {
  user?: ServiceUserEntity;

  protected async __init__() {
    this.user = this.$useQueryLocal({
      queryKey: ['user'],
    });
  }
}

```

* 与`异步数据`定义不同，同步数据直接在初始化方法`__init__`中定义
* 调用$useQueryLocal 创建 Query 对象
* 传入 queryKey，确保本地缓存的唯一性


### 如何使用


直接像常规变量一样读取和设置数据



```
const user = this.user;
this.user = newUser;

```

## 同步数据: cookie


在服务端会自动使用`Request Header`中的 Cookies，在客户端会自动使用`document.cookie`，因此会自动保证 SSR 水合过程中 cookie 状态数据的一致性


下面演示把用户 Token 存入 cookie，当页面刷新时也会保持状态。这样，在 SSR 模式下，客户端和服务端都可以使用相同的`jwt token`访问后端 API 服务


### 如何定义



```
export class ModelUser extends BeanModelBase {
  token?: string;

  protected async __init__() {
    this.token = this.$useQueryCookie({
      queryKey: ['token'],
    });
  }
}

```

* 与`异步数据`定义不同，同步数据直接在初始化方法`__init__`中定义
* 调用$useQueryCookie 创建 Query 对象
* 传入 queryKey，确保本地缓存的唯一性


### 如何使用


直接像常规变量一样读取和设置数据



```
const token = this.token;
this.token = newToken;

```

## 同步数据: 内存


在 SSR 模式下，服务端定义的全局状态数据会同步到客户端，并自动完成水合


下面演示基于内存的全局状态数据


### 如何定义


`zova-ui-quasar/src/suite-vendor/a-quasar/modules/quasar-adapter/src/bean/model.theme.ts`



```
export class ModelTheme extends BeanModelBase {
  cBrand: string;

  protected async __init__() {
    this.cBrand = this.$useQueryMem({
      queryKey: ['cBrand'],
    });
  }
}

```

* 与`异步数据`定义不同，同步数据直接在初始化方法`__init__`中定义
* 调用$useQueryMem 创建 Query 对象
* 传入 queryKey，确保本地缓存的唯一性


### 如何使用


直接像常规变量一样读取和设置数据



```
const cBrand = this.cBrand;
this.cBrand = newValue;

```

## 结语


Zova 是一款支持 IOC 容器的 Vue3 框架，在代码风格上结合了Vue/React/Angular的优点，同时规避他们的缺点，让我们的开发体验更加优雅，减轻心智负担。Zova已经内置了大量实用、有趣的功能特性，Model机制仅仅是其中一个


Zova框架已经开源，欢迎关注，参与共建：[https://github.com/cabloy/zova](https://github.com)。可添加我的微信，入群交流：yangjian2025


