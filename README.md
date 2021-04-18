# Vue技巧-接口管理

整理自文章 [Vue tricks: smart api module for VueJS](https://itnext.io/vue-tricks-smart-api-module-for-vuejs-b0cae563e67b)

> 接口请求是项目开发中不可或缺的一部分，api模块的封装关系到后面项目的维护和扩展。

## 开始

[仓库地址](https://github.com/YooYooY/vue-api-module)

在项目开发过程中，不同的模块可能对应不同的api文档，好的接口模块能够起到事倍功半的效果。

下面👇 是项目中经过封装后抛出的请求模块：

```js
export const $api = {
  users: new UsersApiService(),
  posts: new PostsApiService(),
  albums: new AlbumsApiService()
};
```

`users`、`posts`、`albums` 对应不同模块的接口服务。

封装后的模块在组件中的使用：

```js
// 查找
this.$api.users.get(userId);
// 删除
this.$api.users.delete(userId);
// 新增
this.$api.users.add({ name:"new user" });
// 修改
this.$api.users.update(userId,{ name:"update" });
```

通常在项目中，接口的CRUD(增查改删)操作在模块中可能是基础功能的存在，还有像统一的请求头信息、路径前缀、相同的数据格式等等。

## 基础类

在编写接口模块之前，先创建基础类统一管理：

```js
class BaseApiService {
  baseUrl = "https://jsonplaceholder.typicode.com";
  resource;

  constructor(resource) {
    if (!resource) throw new Error("Resource is not provided");
    this.resource = resource;
  }

  getUrl(id = "") {
    return `${this.baseUrl}/${this.resource}/${id}`;
  }

  handleErrors(err) {
    // Note: 这里抛出错误参数
    console.log({ message: "Errors is handled here", err });
  }
}
```

### 创建子类

#### ReadOnlyApiService

只读接口（查找）服务：

```js
class ReadOnlyApiService extends BaseApiService {
  constructor(resource) {
    super(resource);
  }
  async fetch(config = {}) {
    try {
      const response = await fetch(this.getUrl(), config);
      return await response.json();
    } catch (err) {
      this.handleErrors(err);
    }
  }
  async get(id) {
    try {
      if (!id) throw Error("Id is not provided");
      const response = await fetch(this.getUrl(id));
      return await response.json();
    } catch (err) {
      this.handleErrors(err);
    }
  }
}
```

#### ModelApiService

基础模块服务：
```js
class ModelApiService extends ReadOnlyApiService {
  constructor(resource) {
    super(resource);
  }
  async post(data = {}) {
    try {
      const response = await fetch(this.getUrl(), {
        method: "POST",
        body: JSON.stringify(data)
      });
      const { id } = response.json();
      return id;
    } catch (err) {
      this.handleErrors(err);
    }
  }
  async put(id, data = {}) {
    if (!id) throw Error("Id is not provided");
    try {
      const response = await fetch(this.getUrl(id), {
        method: "PUT",
        body: JSON.stringify(data)
      });
      const { id: responseId } = response.json();
      return responseId;
    } catch (err) {
      this.handleErrors(err);
    }
  }
  async delete(id) {
    if (!id) throw Error("Id is not provided");
    try {
      await fetch(this.getUrl(id), {
        method: "DELETE"
      });
      return true;
    } catch (err) {
      this.handleErrors(err);
    }
  }
}
```

两个子类的区别在于，`ReadOnlyApiService` 只做读取数据的操作：

- `fetch` 查找资源
- `get` 根据参数id查找资源

`ModelApiService` 扩展了 `ReadOnlyApiService` 的能力，新增了三个方法写入数据：

- `post` 新增资源
- `put` 查找资源
- `delete` 删除资源

## 扩展接口服务

接下来可以针对不同模块的请求逻辑做扩展方法，例如 `UsersApiService` 资源服务：

```js
class UsersApiService extends ReadOnlyApiService {
  constructor() {
    super("users");
  }
  get(id){}
  add(id){}
  delet(id){}
  update(id){}
}
```

## 在组件和stroe中使用

### 注入全局组件

将接口服务注入到vue中，这里通过mixin的方式注入，编写`plugins/mixins.js`如下：

```js
import Vue from "vue";
import { $api } from "@/services/api";

// 全局注入
Vue.mixin({
  computed: {
    $api: () => $api
  }
});
```

在 `main.js` 中引入：
```js
import Vue from "vue";
import App from "./App.vue";
import "@/plugins/mixins";
```

### store插件

store也是经常使用接口服务的地方，编写`plugins/storePlugins.js`如下：

```js
import { $api } from "@/services/api";

export default function(store) {
  try {
    store.$api = $api;
  } catch (e) {
    console.error(e);
  }
}
```

在store中引入：
```js
...
import storePlugins from "@/plugins/storePlugins";

...

export default new Vuex.Store({
  plugins: [storePlugins],
  state: {
    ...
```

注入完成后，在组件和store中都可以通过`this.$api`方便的调用接口服务了。