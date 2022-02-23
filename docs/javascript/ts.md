# TS语法及使用

> TypeScript 由微软开发的自由和开源的编程语言。TypeScript 是 JavaScript 的一个超集，支持 ECMAScript 6 标准（[ES6 教程](https://www.runoob.com/w3cnote/es6-tutorial.html)）。TS是一种强类型语言

## 一.语言特性

TypeScript 是一种给 JavaScript 添加特性的语言扩展。增加的功能包括：

- 类型批注和编译时类型检查
- 类型推断
- 类型擦除
- 接口
- 枚举
- Mixin
- 泛型编程
- 名字空间
- 元组
- Await

以下功能是从 ECMA 2015 反向移植而来：

- 类
- 模块
- lambda 函数的箭头语法
- 可选参数以及默认参数

## 二.JavaScript 与 TypeScript 的区别

TypeScript 是 JavaScript 的超集，扩展了 JavaScript 的语法，因此现有的 JavaScript 代码可与 TypeScript 一起工作无需任何修改。

TypeScript 可处理已有的 JavaScript 代码，并只对其中的 TypeScript 代码进行编译。

越来越多的前端开源框架采用TypeScript编译 ,当下学习ts也十分必要。

## 三.基本使用

### 3.1安装及编译

全局安装：

```
npm install -g typescript
```

运行编译器

```
tsc xxx.ts
```

监听状态编译

```
tsc xxx.ts -w
```

编译选项有十分多的参数可选，可以参考以下文档

[编译选项文档](https://www.tslang.cn/docs/handbook/compiler-options.html)



输出结果为一个xxx.js文件，它包含了和输入文件中相同的JavsScript代码。 一切准备就绪，我们可以运行这个使用TypeScript写的JavaScript应用了！

### 3.2语法

我们创建如下结构

![avatar](https://picture.zhanghong110.top/docsify/16451634392150.png)

采用`tsc xxx.ts -w`语法对02(01没内容就单纯的hello word)进行编译，然后创建`index.html`

```html
<!DOCTYPE html>
<html>
    <meta charset="UTF-8">
    <title>Tiele</title>
    <body>

    <script src="02_basis.js"></script>
    </body>
</html>
```

我们依托该结构，采用TS与编译后的js对比来介绍下ts语法

##### 3.2.1 类型声明

基本类型

TS：

```typescript
//声明类型
let a: number 
a=10;
//a="hello" //就会报错，编译会报错，但是可以编译成JS

let b: string

//声明完直接赋值
let c: boolean = true

//自动类型判断，此时e就是布尔值不能再赋其它类型
let e = false

e=false
//e="asd" //报错
```

JS:

```javascript
//声明类型
var a;
a = 10;
//a="hello" //就会报错，编译会报错，但是可以编译成JS
var b;
//声明完直接赋值
var c = true;
//自动类型判断，此时e就是布尔值不能再赋其它类型
var e = false;
e = false;
```

> 可以看出ts语法是强类型语言，在声明的时候需要指定类型

!>值得注意的是，由于浏览器只会执行JS，也就是说TS最终还是转化为JS执行的，所以当类型不符时的报错也就是编译中报错，还是会编译下去且产生的JS由于不需要指定变量，所以还是能运行。

其他类型

TS:

```typescript
//object没啥意义
let a :object;
a={};
a=function (){};

//声明对象
//问号表示可选属性
let b: {name: string,age?:number};
b={name:"asd"}

//声明对象
//当需要除了name还有其它非必选属性时可以这么做，propName随便写啥string 指属性名类型，any指属性值是啥都可以
let c: {name:string,[propName: string]: any}
c={name:"213",age:18}

//声明函数（作用类似于接口）,两个参数为number，返回number
let d: (a: number,b: number)=>number;
d = function(n1:number,n2:number): number{
    return n1+n2;
}

//数组的两种声明方式

// 1.string [] 表示字符串数组
let e: string[];
e=['a','b'];

//2.等价于 let g: number[];
let g : Array<number>;
g=[1,2]


//元组：固定长度的数组
let h:[string,string]
h=["ads","Ads"];

//枚举
enum Gender{
    Male = 0,
    Femal = 1
}

let i: {name: string, gender:Gender};
i = {
    name:'孙悟空',
    gender: Gender.Male
}

//js知识点：==会自动类型转化 1=="1" //true  1==="1"//false
//console.log(i.gender === 1)
console.log(i.gender === Gender.Male) //true

//| &的用法，&不能直接连起来，没有意义
//let j: string | number;
let j: {name: string} & {age: number};
j={name:"sun",age: 18}

//类的别名
//let k :1|2|3|4|5;
//let l :1|2|3|4|5;
//很长很麻烦,就可以起别名,比如
type mytype = string;

type mytype2 =1|2|3|4|5;
let k: mytype2;
k=1;
//k=6;报错
```

JS:

```javascript
//object没啥意义
var a;
a = {};
a = function () { };

var b;
b = { name: "asd" };

var c;
c = { name: "213", age: 18 };

var d;
d = function (n1, n2) {
    return n1 + n2;
};

var e;
e = ['a', 'b'];

var g;
g = [1, 2];

var h;
h = ["ads", "Ads"];

//枚举
var Gender;
(function (Gender) {
    Gender[Gender["Male"] = 0] = "Male";
    Gender[Gender["Femal"] = 1] = "Femal";
})(Gender || (Gender = {}));

var i;
i = {
    name: '孙悟空',
    gender: Gender.Male
};

console.log(i.gender === Gender.Male);

var j;
j = { name: "sun", age: 18 };
var k;
k = 1;
```

##### 3.2.2 tsconfig.json配置文件

我们使用`tsc`命令编译的时候，编译器会从当前目录开始去查找`tsconfig.json`文件，逐级向上搜索父目录。这个文件中指定了用来编译这个项目的根文件和编译选项。

我们新建个项目，建立`src`文件夹在里面添加`tsconfig.json`文件

![avatar](https://picture.zhanghong110.top/docsify/16451687184194.png)

初步的配置文件如下

```typescript
{
/**
*          include指定哪些需要被编译 **表示任意目录 *表示任意文件
*          exclude被排除的文件，一般不需要设置
*/
    "include": [
         "./src/moduledemo/*"
    ],
    "exclude":[

    ],
    "compilerOptions": {
        //编译器选项
        "target": "es2015",//用来指定被编译成ES的版本，默认会转换成ES3版本 支持字段'es3', 'es5', 'es6', 'es2015', 'es2016', 'es2017',     //'es2018', 'es2019', 'es2020', 'esnext'
        "module": "es2015", //模块化的版本 'none', 'commonjs', 'amd', 'system', 'umd', 'es6', 'es2015', 'es2020', 'esnext'
        //"lib":[]  //一般不需要，指定使用的库，浏览器都有，如果要运行再nodejs里才要
        "outDir": "./dist",//指定编译后目录
        //"outFile": "" //把全局作用域的代码合并代码合并成一个文件
        "strict": false, //所有的严格检查都打开
        "allowJs": false, //是否将JS代码也编译过去
        "checkJs": false, //检查JS语法,使得JS严格化
        "removeComments": false, //是否移除注释
        "noEmit": false,//不生成编译后的文件,单纯用来检查检查语法
        "noEmitOnError": false,//有错误不会编译到目标
        "alwaysStrict": false,//是否严格模式
        "noImplicitAny": false,//不允许隐式的any类型
        "noImplicitThis": false//不允许不明确类型的this  
    }

}
```

我们可以看到以上配置文件，包括 包含的路径，输出的路径 编译后的ES版本以及一些其他配置

切换到根目录下，`tsc -w`，将会去自动寻找该配置文件并编译，编译后根据当前配置的路径去编译输出

这个配置文件还有很多项目，想了解的可以参考以下链接

[配置文档](https://www.tslang.cn/docs/handbook/tsconfig-json.html)

我们看下`app.ts`及`m.ts`种的代码

app.ts:
```typescript
import {hi} from './m.js'
console.log(hi)
```
m.ts:

```typescript
export const hi ='nihao'
```

在根目录增加index.html(图中没有，自己增加下)，代码如下

```html
<!DOCTYPE html>
<html>
    <meta charset="UTF-8">
    <title>Tiele</title>
    <body>
    <script type="module" src="./dist/app.js"></script>
    </body>
</html>
```

此时我们在`vscode`使用`live server`（没有的`vscode`装个插件，这边主要是跨域请求不支持`file`头）打开，打开控制台可见模块导入成功

![avatar](https://picture.zhanghong110.top/docsify/16451694042075.png)

##### 3.2.3 webpack构建

>接下来我们来看下如何将`ts`和`webpack`结合使用

1.初始化工程

```
npm init
npm install -g webpack
```

`package.json`依赖引入`ts`

```json
{
  "name": "part3",
  "version": "1.0.0",
  "description": "",
  "main": "index.js",
  "scripts": {
    "test": "echo \"Error: no test specified\" && exit 1",
    "build": "webpack",
    "start": "webpack serve --open chrome.exe"
  },
  "keywords": [],
  "author": "",
  "license": "ISC",
  "devDependencies": {
    "@babel/core": "^7.14.3",
    "@babel/preset-env": "^7.14.2",
    "babel-loader": "^8.2.2",
    "clean-webpack-plugin": "^4.0.0-alpha.0",
    "corejs": "^1.0.0",
    "html-webpack-plugin": "^5.3.1",
    "ts-loader": "^9.2.2",
    "typescript": "^4.2.4",
    "webpack": "^5.37.1",
    "webpack-cli": "^4.7.0",
    "webpack-dev-server": "^3.11.2"
  }
}
```

2.我们需要创建一个`tsconfig.json`文件，它包含了输入文件列表以及编译选项。 在工程根目录下新建文件 `tsconfig.json`文件

剩下的文件自己创建下如下图所示

![avatar](https://picture.zhanghong110.top/docsify/16451696657459.png)

这次的`tsconfig.json`我们采用最低限度的配置

```typescript
{
    "compilerOptions": {
        "module": "ES2015",
        "target": "ES2015",
        "strict": true
    }
}
```

我们重点看下`webpack`怎么配置来耦合TS,`webpack.config.js`如下

```
const path = require('path')
const HTML = require('html-webpack-plugin')
const { CleanWebpackPlugin } = require('clean-webpack-plugin')

// webpack中所有的配置信息都因该写在module.exports中
module.exports = {
  mode: 'development',

  //指定入口文件
  entry: './src/index.ts',
  //指定打包文件所在目录
  output: {
    //指定打包文件所在目录
    path: path.resolve(__dirname, 'dist'),
    //打包后文件的文件
    filename: 'bundle.js',

    //告诉webpack不适用箭头函数以便兼容老浏览器
    environment: {
      arrowFunction: false
    }
  },

  //指定webpack打包时要使用的模块
  module: {
    //指定加载的规则
    rules: [
      {
        //test指定的是规则生效的文件,使用哪个Loader,排除什么开头的文件
        test: /\.ts$/,
        //加载器顺序从后向前执行，先转成JS在转成对应版本
        use: [
          {
            //baben复杂点所以用对象配置（浏览器兼容）
            loader: 'babel-loader',
            options: {
              //设置运行环境
              presets: [
                //指定环境插件
                [
                  '@babel/preset-env',
                  //配置信息
                  {
                    targets: {
                      //适配浏览器
                      chrome: '58',
                      ie: '11'
                    },
                    //版本,作用就是来浏览器，类似promis这种函数babel实现不了转换，必须用corejs才可以
                    corejs: 3,
                    //按需加载
                    useBuiltIns: 'usage'
                  }
                ]
              ]
            }
          },

          //   {
          //     loader: 'babel-loader',
          //     options: {
          //       presets: [
          //         [
          //           '@babel/env',
          //           {
          //             targets: {
          //               browsers: ['> 1%', 'last 2 versions']
          //             }
          //           }
          //         ]
          //       ]
          //     }
          //   },

          'ts-loader'
        ],
        exclude: /node_modules/
      }
    ]
  },

  //配置webpack插件
  plugins: [
    new CleanWebpackPlugin(),
    new HTML({
      title: '这是自定义模板'
      // template:"" //可以自定义模板
    })
  ],

  // 用来设置引用模块,告诉webpack哪些结尾的文件可以作为模块使用
  resolve: {
    extensions: ['.ts', '.js']
  }
}
```

我们`index.js`里只有

```typescript
console.log('123')
```

`index.html`如下

```html
<!DOCTYPE html>
<html>
    <meta charset="UTF-8">
    <title>Tiele</title>
    <body>

    </body>
</html>

```

运行`npm run start`可以看到打印成功

![avatar](https://picture.zhanghong110.top/docsify/16451710427999.png)

> 到这边我们`webpack`就和`ts`集成成功了

##### 3.2.4 基本语法

> 学完了集成方式，我们来学习一下`ts`的基本语法

###### 3.2.4.1 类的简介

TS:

```typescript
//使用class关键词定义一个类,与JAVA十分相似
class Person {
  static sex = '男'

  static sayHello() {
    console.log('静态方法')
  }

  readonly name: string = '悟空' //readonly类似final
  age: number = 18

  sayHello() {
    console.log('普通方法')
  }
}

const per = new Person()

//对象属性直接定义，用对象访问
per.age = 19
//per.name="123"; //只读不能重新赋值

console.log('tag', per)
console.log('name', per.name)
console.log('age', per.age)

//可以重新赋值
Person.sex = 's'

//static 定义静态变量（类似JAVA成员变量），用类访问
console.log('sex', Person.sex)

//方法,静态方法和普通方法可以重名
Person.sayHello()

per.sayHello()
```

JS:

```javascript
"use strict";
//使用class关键词定义一个类,与JAVA十分相似
class Person {
    constructor() {
        this.name = '悟空'; //readonly类似final
        this.age = 18;
    }
    static sayHello() {
        console.log('静态方法');
    }
    sayHello() {
        console.log('普通方法');
    }
}
Person.sex = '男';
const per = new Person();
//对象属性直接定义，用对象访问
per.age = 19;
//per.name="123"; //只读不能重新赋值
console.log('tag', per);
console.log('name', per.name);
console.log('age', per.age);
//可以重新赋值
Person.sex = 's';
//static 定义静态变量（类似JAVA成员变量），用类访问
console.log('sex', Person.sex);
//方法,静态方法和普通方法可以重名
Person.sayHello();
per.sayHello();
```

我们看下以上代码的输出结果，不难看出，TS的类与java类十分的类似。有JAVA基础的同学应该不难理解

![avatar](https://picture.zhanghong110.top/docsify/16451727043900.png)

###### 3.2.4.2 构造函数和this

TS:

```typescript
class Dog{
    // name: string;
    // age: number;

    //构造
    //会在对象创建时调用
    constructor(){
        //this和JAVA一样，谁调用就是指谁，就是说谁实例化了DOG，产生的对象就是this的指向。JS相对比较复杂TS和JAVA比较像
        console.log('构造函数执行了',this)
    }

    break(){
        alert('汪汪汪')
    }
}

new Dog();

class Cat{
    name: string;
    age: number;

    constructor(name: string, age: number){
       this.name=name;
       this.age=age;
    }

    break(){
        alert('喵喵喵')
    }
}

const cat = new Cat("小白",2);
console.log(cat.name)
console.log(cat.age)
cat.break()
```

JS:

```javascript
"use strict";
class Dog {
    // name: string;
    // age: number;
    //构造
    //会在对象创建时调用
    constructor() {
        //this和JAVA一样，谁调用就是指谁，就是说谁实例化了DOG，产生的对象就是this的指向。JS相对比较复杂TS和JAVA比较像
        console.log('构造函数执行了', this);
    }
    break() {
        alert('汪汪汪');
    }
}
new Dog();
class Cat {
    constructor(name, age) {
        this.name = name;
        this.age = age;
    }
    break() {
        alert('喵喵喵');
    }
}
const cat = new Cat("小白", 2);
console.log(cat.name);
console.log(cat.age);
cat.break();
```

看下输出，没啥好说的，JS相对还更复杂一些，TS基本就是JAVA的复刻。

![avatar](https://picture.zhanghong110.top/docsify/16451730597139.png)

###### 3.2.4.3 继承及抽象类

TS:

```typescript
//为了避免作用域冲突
(function () {
  //抽象类，不能被实列化
  abstract class Animal {
    name: string
    age: number
    constructor(name: string, age: number) {
      this.name = name
      this.age = age
    }

    //抽象方法，只能定义在抽象类中，子类必须重写抽象方法
    abstract sayHello(): void
  }

  class Dog extends Animal {
    constructor(name: string, age: number) {
      //子类的构造需要调用父类的，ts不会默认赠送
      super(name, age)
    }

    //重写了父类的方法
    sayHello() {
      console.log('汪汪汪')
    }
  }

  const dog = new Dog('旺财', 5)
  console.log('name', dog.name)
  console.log('age', dog.age)
  dog.sayHello()
})()
```

JS:

```javascript
"use strict";
//为了避免作用域冲突
(function () {
    //抽象类，不能被实列化
    class Animal {
        constructor(name, age) {
            this.name = name;
            this.age = age;
        }
    }
    class Dog extends Animal {
        constructor(name, age) {
            //子类的构造需要调用父类的，ts不会默认赠送
            super(name, age);
        }
        //重写了父类的方法
        sayHello() {
            console.log('汪汪汪');
        }
    }
    const dog = new Dog('旺财', 5);
    console.log('name', dog.name);
    console.log('age', dog.age);
    dog.sayHello();
})();
```

我们看下以上代码，输出的结果，ts子类调用父类不会赠送构造函数，抽象类不能被实例化只能被继承继承后需要重写抽象方法这点和java也十分类似。

![avatar](https://picture.zhanghong110.top/docsify/16451733324180.png)

###### 3.2.4.4 接口

TS:

```typescript
(function () {
  //原来的写法,定义一个类型
  type myType = {
    name: string
    age: number
  }

  //interface
  interface myInterface {
    name: string
    age: number
  }

  //以下可见，接口和类型十分类似，用来定义一个类中应该包含哪些属性。
  //区别在于接口可以重名，type不行，接口重名定义，改接口类型必须要包含重名的所有属性

  const obj: myType = {
    name: 'sss',
    age: 111
  }

  const obj2: myInterface = {
    name: 'sss',
    age: 111
  }

  //interface
  interface myInterface2 {
    name: string
    //接口中不准有实现方法
    sayHello(): void
  }

  //实现类，必须实现接口方法
  class MyClass implements myInterface2 {
    name: string

    constructor(name: string) {
      this.name = name
    }

    sayHello(): void {
      throw new Error('Method not implemented.')
    }
  }
})()
```

JS:

```javascript
"use strict";
(function () {
    //以下可见，接口和类型十分类似，用来定义一个类中应该包含哪些属性。
    //区别在于接口可以重名，type不行，接口重名定义，改接口类型必须要包含重名的所有属性
    const obj = {
        name: 'sss',
        age: 111
    };
    const obj2 = {
        name: 'sss',
        age: 111
    };
    //实现类，必须实现接口方法
    class MyClass {
        constructor(name) {
            this.name = name;
        }
        sayHello() {
            throw new Error('Method not implemented.');
        }
    }
})();
```

这个没啥输出我们就不看了，可以看到接口还是和java十分类似23333333

###### 3.2.4.5 属性的封装

TS:

```typescript
(function () {
  //存在的问题，属性没有被封装，外部可以任意访问
  class Person {
    name: string
    age: number

    constructor(name: string, age: number) {
      this.name = name
      this.age = age
    }
  }

  const per = new Person('悟空', 18)

  //修改
  class Person2 {
    private name: string
    private age: number

    constructor(name: string, age: number) {
      this.name = name
      this.age = age
    }

    getName() {
      return this.name
    }
    setName(value: string) {
      this.name = value
    }
  }

  const per2 = new Person2('悟空', 18)
  //per2.name 报错，注意虽然报错但是可以成功，JS没有这玩意。但是编码阶段会报错

  per2.setName('悟空空')
  console.log('per2', per2)

  //TS中还有隐式的get set，可以让对象直接调用属性，但是其实是通过get set方法的。
  class Person3 {
    private _name: string

    constructor(name: string) {
      this._name = name
    }

    get name() {
      return this._name
    }
    set name(value: string) {
      this.name = value
    }
  }

  const per3 = new Person3('悟空')
  //实际上还是需要和属性不重名，没啥意思说白了就是调方法。
  per3.name = '空空'
  console.log(per3)
})()
```

JS:

```javascript
"use strict";
(function () {
    //存在的问题，属性没有被封装，外部可以任意访问
    class Person {
        constructor(name, age) {
            this.name = name;
            this.age = age;
        }
    }
    const per = new Person('悟空', 18);
    //修改
    class Person2 {
        constructor(name, age) {
            this.name = name;
            this.age = age;
        }
        getName() {
            return this.name;
        }
        setName(value) {
            this.name = value;
        }
    }
    const per2 = new Person2('悟空', 18);
    //per2.name 报错，注意虽然报错但是可以成功，JS没有这玩意。但是编码阶段会报错
    per2.setName('悟空空');
    console.log('per2', per2);
    //TS中还有隐式的get set，可以让对象直接调用属性，但是其实是通过get set方法的。
    class Person3 {
        constructor(name) {
            this._name = name;
        }
        get name() {
            return this._name;
        }
        set name(value) {
            this.name = value;
        }
    }
    const per3 = new Person3('悟空');
    //实际上还是需要和属性不重名，没啥意思说白了就是调方法。
    per3.name = '空空';
    console.log(per3);
})();
```

我们可以看到ts的类封装也和java十分类似，其它真没啥讲了。

![avatar](https://picture.zhanghong110.top/docsify/16451753385221.png)

###### 3.2.4.6 泛型

TS:

```typescript
(function(){

    //存在问题，无法知道传入类型和返回值类型是否相同
    function fn(a: any): any{
        return a;
    }

    /**
     * 在定义函数或者类型，如果遇到不明确的可以使用泛型
     * 好处在于知道返回值和传入值相同
     */
    function fn2<T>(a: T): T{
        return a;
    }

    //使用
    let res = fn2(10);
    //若是不能推断出来，可以手动指定
    let res2 =fn2<string>("hello");

    //多个泛型
    function fn3<T,K>(a: T,b: K) :T{
        console.log(b)
        return a;
    }
    fn3<number,string>(231,"abc")


    interface Inter{
        length: number;
    }
    //希望这个泛型实现inter接口,这边无论是接口还是类都是extends
    function fn4<T extends Inter>(a: T): number{
        return Text.length;
    }
    //调用要求，必须此参数里有length属性，所以其实不是必须实现，而是有这个Inter的属性，这个和JAVA还是有区别的
    let re = fn("asd")

    console.log(re)

})()
```

JS:

```javascript
"use strict";
(function () {
    //存在问题，无法知道传入类型和返回值类型是否相同
    function fn(a) {
        return a;
    }
    /**
     * 在定义函数或者类型，如果遇到不明确的可以使用泛型
     * 好处在于知道返回值和传入值相同
     */
    function fn2(a) {
        return a;
    }
    //使用
    let res = fn2(10);
    //若是不能推断出来，可以手动指定
    let res2 = fn2("hello");
    //多个泛型
    function fn3(a, b) {
        console.log(b);
        return a;
    }
    fn3(231, "abc");
    //希望这个泛型实现inter接口,这边无论是接口还是类都是extends
    function fn4(a) {
        return Text.length;
    }
    //调用要求，必须此参数里有length属性，所以其实不是必须实现，而是有这个Inter的属性，这个和JAVA还是有区别的
    let re = fn("asd");
    console.log(re);
})();
```

这个分别输出了

```
abc
asd
```

我们可以发现，泛型还是和java有一点点小区别的，但是也不是很大



> 做个总结的话就是TS语法和java还是有很大相似性的，说明JS作为弱类型语言在开发上还是存在不少问题的。

我们对于TS的学习就到这里，如果有想进一步了解的可以参考以下地址

[配置文档](https://www.tslang.cn/docs/home.html)

