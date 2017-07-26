id: angular1
title:  angular手札:所谓语法
categories: font
tags: [angular,js]
---

## 基本概念
### 应用
app只是起名字的一种习惯，其本质是`模块(module)`。每个模块有自己独立的控制器、视图、服务、过滤器。。。
基本语法：
```js
var xxxApp = angular.module('xxxApp', ['module1', 'module2']);
```
- 第一个参数是应用的名字
- 第二个参数数组是依赖的其他模块。

可将通用的服务、过滤器、指令等封装到一个模块，在此引入。just like jar

### 控制器
控制器是一个函数，用于增强视图的作用域的功能。
基本语法：
```js
xxxApp.controller('XxxController', ['$scope', function($scope) {...}]);
```
- 第一个参数是控制器的名字。一般的， 首字母大写， 并且不用ctrl的简称。
- 第二个参数是一个函数，即控制器的本体。函数的入参是注入的各种服务，可用官方提供的也可用自定义的。

### 作用域
作用域是构成Angular应用的核心基础，在整个框架中都被广泛使用。作用域是`视图`和`控制器`之间的**胶水**。在应用将视图渲染并呈现给用户之前，视图中的模板会和作用域进行连接，然后应用会对`DOM`进行设置以便将属性变化通知给Angular。
用域是应用状态的基础。基于`动态绑定`，可以依赖视图在修改数据时立刻更新`$scope`，也可依赖`$scope`在其发生变化时立刻重新渲染视图。
Angular在启动并生成视图时，会将根`ng-app`元素同`$rootScope`进行绑定。`$rootScope`是所有`$scope`对象的最上层。
> $rootScope是Angular中最接近全局作用域的对象。在$rootScope上附加太多业务逻辑并不是好主意，这与
污染JavaScript的全局作用域是一样的。

### 服务
服务，可以类比MVC中的业务逻辑层，将业务逻辑从控制器中剥离出来可以使控制器更轻更干净，更符合软件开发中职责单一的原则，程序的维护成本也相应降低。

angular官方提供了许多服务，特点是名称以`$`开头，因此自定义服务**不建议**以`$`开头。
常用的服务有
- `$rootScope` 根作用域
- `$scope` 作用域
- `$http` 支持http请求
- `$location` 抽象了`window.location`，提供对url的操作
- `$routeParams` 支持Angular路由传参
- `$filter` 支持js调用过滤器

## 来点高级的下酒菜
### 自定义服务
*如何编写自定义服务呢？Angular提供了多种方式。*

**Factory**
用`Factory`就是创建一个对象，为它添加属性，然后把它返回出来。当注入到`Controller`中时，就可以直接用了。
```js
// 声明
feedbackApp.factory('contentService', function() {
    function filterTextContent(content) {
        // 替换图片
        var textContent = content.replace(/<img[^>]+>/g, "[图]");
        // 替换超链接
        textContent = textContent.replace(/<a[^>]+>/g, "[链接]");
        // 无html格式的内容
        textContent = textContent.replace(/<[^>]+>/g,"");
        return textContent;
    }

    return {
        filterText: filterTextContent
    }
})

// 使用
feedbackApp.controller('FeedbackDetailController', ['$scope','contentService',
    function($scope, contentService) {
      // 替换图片和链接
      $scope.feedback.textContent = contentService.filterText($scope.feedback.content);
    }   
])
```

**Service**
`Service`是用`new`关键字实例化的。因此，应该给`this`添加属性，然后返回`this`。
```js
// 声明
feedbackApp.Service('contentService', function() {
    this.filterText = function() {
        // 替换图片
        var textContent = content.replace(/<img[^>]+>/g, "[图]");
        // 替换超链接
        textContent = textContent.replace(/<a[^>]+>/g, "[链接]");
        // 无html格式的内容
        textContent = textContent.replace(/<[^>]+>/g,"");
        return textContent;
    }
})

// 使用
feedbackApp.controller('FeedbackDetailController', ['$scope','contentService',
    function($scope, contentService) {
      // 替换图片和链接
      $scope.feedback.textContent = contentService.filterText($scope.feedback.content);
    }   
])
```

**Provider**
`Provider`是唯一一种可以传进`.config()`函数的`service`。当想在`service`对象启动前，先进行模块范围的配置，那就应该用`provider`。
```js
// 声明
feedbackApp.provider('contentProvider', function() {
    // 只有这行可以在app.config()中使用
    this.thingFromConfig = '';

    // 只有$get返回的对象上的属性才能在Controller中使用
    this.$get = function() {
        return {
            filterTextContent: function() {
                // 替换图片
                var textContent = content.replace(/<img[^>]+>/g, "[图]");
                // 替换超链接
                textContent = textContent.replace(/<a[^>]+>/g, "[链接]");
                // 无html格式的内容
                textContent = textContent.replace(/<[^>]+>/g,"");
                return textContent;
            },
            thingOnConfig: this.thingFromConfig
        }
    }
})

// 配置
feedbackApp.config(['contentProvider', function(contentProvider) {
    contentProvider.thingFromConfig = "啦啦啦，卖个萌"；
}])

// 使用
feedbackApp.controller('FeedbackDetailController', ['$scope','contentService',
    function($scope, contentService) {
      // 替换图片和链接
      $scope.feedback.textContent = contentService.filterText($scope.feedback.content);
      $scope.data.thingFromConfig = contentProvider.thingOnConfig;
    }   
])
```

### 过滤器
“增强型服务”，个人感觉呀。常用作渲染时对数值进行变化，比如时间的格式化、金币、数字状态变为文本状态等
举几个官方的栗子：
- date
```js
// html 中的用法
{ {date_expression | date:format:timezone} }
// js 中的用法
$filter('date')(date_expression, format, timezone)
```
  `format`是日期时间的格式，如'yyyy-MM-dd HH:mm:ss sss'。
  `timezone`是时区，如'+0430',相对于格林威治标准时间的东半球4小时30分
- currency
```js
// html
{ { currency_expression | currency : symbol : fractionSize} }
// js
$filter('currency')(amount, symbol, fractionSize)
```
  `symbol`金钱符号，显示在数值的左边，如￥$。`fractionSize`保留的小数位数，会四舍五入的哟。
- number
```js
{ { number_expression | number : fractionSize} }
```

*自定义自己的过滤器*

将后端传入的数值状态变为文字状态
```js
// 声明
feedbackApp.filter('statusText', function () {
    return function (input) {
        var text;
        switch (input) {
            case 0:
                text = '未分配';
                break;
            case 1:
                text = '已处理';
                break;
            default:
                text = '未知状态';
                break;
        }
        return text;
    }
});
// 使用
<span>{ {status | statusText} }</span>
```

### 指令
指令本质上是Angular扩展具有自定义功能的HTML元素的途径。
常见的已`ng-`开头的均是Angular提供的指令。

*布尔属性，简单易用。*
- ng-disabled
- ng-checked
- ng-readonly
- ng-selected
- ng-show
- ng-hide
- ng-if

*属性加强*
- ng-href
```js
<a ng-href="{ {customHref} }">我是一个连接</a>
```
- ng-src
```js
<img ng-src="{ {customSrc} }" alt="pic"></img>
```
- ng-class
```js
// scope变量绑定（不推荐）
<div class="{ {test} }"></div>

// 布尔， isActive为true则active，否则inactive
<div ng-class="{true: 'active', false: 'inactive'}[isActive]"></div>

// 多重class
<div ng-class="{'sel': isSel == $index, 'notsel': isSel != $index, 'red': isRed}"></div>
```

*so big*
- ng-app
- ng-controller
- ng-view

*so small*
- ng-init
```html
// 一般写示例代码会用
<div ng-init="greeting='hello'">
  { {greeting} }
</div>
```
- ng-bind / { {} }  单向绑定
- ng-model 双向绑定

*复杂一点的*
- ng-repeat
```html
<ul>
  <li ng-repeat="apple in apples">{ {apple.name} }</li>
</ul>
```
- ng-switch
```html
<div class="animate-switch-container"
  ng-switch on="selection">
    <div class="animate-switch" ng-switch-when="settings">Settings Div</div>
    <div class="animate-switch" ng-switch-when="home">Home Span</div>
    <div class="animate-switch" ng-switch-default>default</div>
</div>
```
- ng-select / ng-options
```html
// 只用ng-repeat效果不太好，下拉列表中会有一个空白选项
<select name="repeatSelect" id="repeatSelect" ng-model="data.repeatSelect">
  <option ng-repeat="option in data.availableOptions" value="{ {option.id} }">{ {option.name} }</option>
</select>
// 常用版
<select ng-model="data.selectedOption"
      ng-options="option.name as option.id for option in data.availableOptions">
</select>
// 高级版
<select ng-model="data.selectedOption"
      ng-options="option.name for option in data.availableOptions track by option.id">
</select>
// 看不懂版
<select ng-model="myColor"
      ng-options="color.name group by color.shade disable when color.notAnOption for color in colors">
</select>
```

### 路由
在单页应用中从一个视图跳转到另一个视图是非常普遍而重要的需求。随着视图越来越复杂，我们需要一个行之有效的
方式管理应用。路由应运而生，实乃天命之子。
> Angular 1.x版本后， 路由需要额外引入angular-route.js。并且在app中引入依赖ngRoute。
var feedbackApp = angular.module('feedbackApp',['angular.config', 'ngRoute']);

#### 布局模板
创建布局模板的目的是告诉angular将视图渲染到何处。通过指令`ng-view`定位。
```html
<!DOCTYPE html>
<html lang="zh-CN" ng-app="feedbackApp">
<head>
    <title>反馈小助手</title>
    <meta charset="utf-8"/>
    <meta name="viewport" content="width=device-width, initial-scale=1"/>
    <link rel="stylesheet" href="/css/bootstrap.min.css"/>
</head>

<body>

<ng-view/>

<!-- js引用 -->
<script src="/js/angular.min.js"></script>
<script src="/js/angular-route.min.js"></script>
<script src="/js/httpConfig.js"></script>
<script src="/front/feedback/feedbackApp.js"></script>
<script src="/front/feedback/controller/feedbackListController.js"></script>
</body>
</html>
```
ngView遵循一下规则：
- 每次出发`$routeChangeSuccess`事件，视图都会更新
- 如果某个模板通当前的路由相关联：
 - 创建一个新的作用域
 - 移出上一个视图和作用域
 - 将新的作用域同模板关联在一起
 - 如果路由有相关定义，将控制器和当前作用域关联起来
 - 触发`$viewContentLoaded`事件
 - 如果提供了onload属性，则调用

#### 配置路由
```js
watchdogApp.config(function($routeProvider){
    // 路由配置
    $routeProvider
        .when('/project', {
            templateUrl: '/front/bone/view/list.html',
            controller: 'ListController'
        })
        .when('/project/:pid', {
            templateUrl: '/front/bone/view/detail.html',
            controller: 'DetailController'
        })
        .otherwise({redirectTo:'/project'})
});
```
