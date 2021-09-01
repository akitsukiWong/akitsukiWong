`IndexedDB` 是一种底层 API，用于在客户端存储大量的结构化数据

首先回顾一下前端储存数据常用的 `Cookie` 和 `WebStorage`

- Cookie 是服务器发送到用户浏览器并保存在本地的一小块数据，它会在浏览器下次向同一服务器再发起请求时被携带并发送到服务器上,通常用于保持用户的登录状态（储存大小不能超过4KB）

Web Storage 包含如下两种机制：
- SessionStorage 为每一个给定的源维持一个独立的存储区域，仅在当前会话下有效，关闭页面或浏览器后被清除（大小为5M左右）
- LocalStorage 同样的功能，但是在浏览器关闭，然后重新打开后数据仍然存在。除非被清除，否则永久保存（大小为5M左右）

都是在客户端永久存储数据，那`IndexedDB`与`LocalStorage`有什么区别呢

- LocalStorage是同步的，IndexedDB操作是异步的，存储一个较大的数据时，不会因为写入数据慢而导致页面阻塞
- 储存空间不同，LocalStorage大小为5M左右，IndexedDB一般来说不少于 250M，理论上没有上限
- 都通过键值对存储数据，但LocalStorage的数据并不是按对象形式存储，它存储的数据都是字符串形式，如果你想让LocalStorage存储对象，就需要借助JSON.stringify()能将对象变成字符串形式，再用JSON.parse()将字符串还原成对象。IndexedDB可以直接存储对象数组，甚至是二进制数据（ArrayBuffer 对象和 Blob 对象）
- IndexedDB使用索引存储数据，各种数据库操作放在事务中执行，提供了更为复杂的查询数据的方式

## IndexedDB的用法

首先判断浏览器是否支持`IndexedDB`

```
window.indexedDB = window.indexedDB || window.mozIndexedDB || window.webkitIndexedDB || window.msIndexedDB;

if(!window.indexedDB)
{
    console.log("你的浏览器不支持IndexedDB");
}
```

当浏览器支持`IndexedDB`就可以创建一个请求去打开数据库。

```
let request = window.indexedDB.open(dbInfo.name, dbInfo.version);
```

这个方法接受两个参数，第一个参数是字符串，表示数据库的名字。如果指定的数据库不存在，就会新建数据库。第二个参数是整数，表示数据库的版本。如果省略，打开已有数据库时，默认为当前版本；新建数据库时，默认为1。

`indexedDB.open()`方法返回一个 `IDBRequest` 对象。这个对象通过三种事件`error`、`success`、`upgradeneeded`，处理打开数据库的操作结果。


### error事件表示打开数据库失败
```
request.onerror = function (e) {
  console.log('数据库打开错误');
  console.dir(e);
};
```

### success 事件
```
let db;

request.onsuccess = function (e) {
  db = request.result;	//通过request对象的result属性拿到数据库对象
  console.log('数据库打开成功');
};
```


### upgradeneeded 事件

触发条件：
- 第一次打开数据库，数据库不存在，就会初始化数据库，版本从无到有
- 版本号更新，指定的版本号大于数据库的实际版本号
    
```
let db;

request.onupgradeneeded = function (e) {
  db = e.target.result;
  console.log('数据库初始化或更新成功');
}
```

一般在onupgradeneeded事件里新建对象仓库（新建表）

### 新建对象仓库（新建表）


```
let db;

request.onupgradeneeded = function (e) {
  db = e.target.result;
  console.log('数据库初始化或更新成功');
  let objectStore = db.createObjectStore('person', { keyPath: 'id', autoIncrement: true });
}
```
以上新建person表，主键为id，id自增

### 使用索引

索引的意义在于，可以让你搜索任意字段，也就是说从任意字段拿到数据记录。如果不建立索引，默认只能搜索主键（即从主键取值）。

```
request.onupgradeneeded = function(event) {
  db = event.target.result;
  var objectStore = db.createObjectStore('person', { keyPath: 'id' });
  objectStore.createIndex('name', 'name', { unique: false });
  objectStore.createIndex('email', 'email', { unique: true });
}
```
上面代码中，`IDBObject.createIndex()`的三个参数分别为索引名称、索引所在的属性、配置对象（说明该属性是否包含重复的值）

这样，就可以使用 `name` 或者 `email` 来找到对应的数据记录

### 增
```
function add() {
  var request = db.transaction(['person'], 'readwrite')
    .objectStore('person')
    .add({ id: 1, name: '张三', age: 24, email: 'zhangsan@example.com' });

  request.onsuccess = function (event) {
    console.log('数据写入成功');
  };

  request.onerror = function (event) {
    console.log('数据写入失败');
  }
}

add();
```

为了往数据库里新增数据，我们首先需要创建一个事务，并要求具有读写权限。在indexedDB里任何的存取对象的操作都需要放在事务里执行。

事务模式：
 - readonly 只读
 - readwrite 读写


### 删
```
function remove() {
  var request = db.transaction(['person'], 'readwrite')
    .objectStore('person')
    .delete(1);

  request.onsuccess = function (event) {
    console.log('数据删除成功');
  };
}

remove();
```
### 改
```
function update() {
  var request = db.transaction(['person'], 'readwrite')
    .objectStore('person')
    .put({ id: 1, name: '李四', age: 35, email: 'lisi@example.com' });

  request.onsuccess = function (event) {
    console.log('数据更新成功');
  };

  request.onerror = function (event) {
    console.log('数据更新失败');
  }
}

update();
```

### 查
#### 获取所有数据

##### getAll()
```
function readAll() {
  var objectStore = db.transaction('person').objectStore('person');

  objectStore.getAll().onsuccess = function (event) {
     data = event.target.result; // 所有数据
  };
}

readAll();
```

##### 通过指针
```
function readAll() {
  var objectStore = db.transaction('person').objectStore('person');

   objectStore.openCursor().onsuccess = function (event) {
     var cursor = event.target.result;

     if (cursor) {
       console.log('Id: ' + cursor.key);
       console.log('Name: ' + cursor.value.name);
       console.log('Age: ' + cursor.value.age);
       console.log('Email: ' + cursor.value.email);
       cursor.continue();
    } else {
      console.log('没有更多数据了！');
    }
  };
}

readAll();
```

#### 查询某数据

##### 通过get()
```
function read() {
   var transaction = db.transaction(['person']);
   var objectStore = transaction.objectStore('person');
   var request = objectStore.get(1);

   request.onerror = function(event) {
     console.log('事务失败');
   };

   request.onsuccess = function( event) {
      if (request.result) {
        console.log('Name: ' + request.result.name);
        console.log('Age: ' + request.result.age);
        console.log('Email: ' + request.result.email);
      } else {
        console.log('未获得数据记录');
      }
   };
}

read();
```

##### 通过索引
```
var transaction = db.transaction(['person'], 'readonly');
var store = transaction.objectStore('person');
var index = store.index('name');
var request = index.get('李四');

request.onsuccess = function (e) {
  var result = e.target.result;
  if (result) {
    // ...
  } else {
    // ...
  }
}
```