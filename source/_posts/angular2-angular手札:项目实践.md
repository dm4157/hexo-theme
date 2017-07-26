id: angular2
title: angular手札:项目实践
categories: font
tags: [angular,js]
---

> 项目中的一些用法

## 配置App
### 路由中的\#号
Angular框架定义了自己的前端路由控制器，通过不同URL实现单面(ng-app)对视图(ng-view)的部署刷新，并支持HTML5的历史记录功能。
默认情况下是不启动`HTML5模式`的，URL中会包括一个#号，用来区别是Angular管理的路径还是WebServer管理的路径。
```html
比如：
http://watchdog.com/#/
http://watchdog.com/#/project
http://watchdog.com/#/diagram
```
这种体验其实不太友好，像锚点。不过Angular提供了一种HTML5模式的路由，可以直接去掉#号。
通过设置`$locationProvider`就行了。HTML5模式要求html页面中包含`<base/>`标签。不过在做微信开发时，发现微信浏览器似乎无法解析`<base/>`(也可能是打开方式不对)，只好关掉base；
```js
watchdogApp.config(function($locationProvider){
    // 打开html5模式,否则路由跳转会加#; 去掉对<base/>的依赖
    $locationProvider.html5Mode({
        enabled: true,
        requireBase: false
    });
  }
)
```

### 拦截器
Angular提供了拦截器，概念与服务端的拦截器没什么区别， 我们可以在拦截器里做一些统一的处理，比如对错误信息的统一处理。
拦截器的定义如下：
```js
$provide.factory('myHttpInterceptor', function($q, dependency1, dependency2) {
  return {
    // 请求成功
    'request': function(config) {
      return config;
    },
    // 请求失败
   'requestError': function(rejection) {
      if (canRecover(rejection)) {
        return responseOrNewPromise
      }
      return $q.reject(rejection);
    },
    // 响应成功
    'response': function(response) {
      return response;
    },
    // 响应失败
   'responseError': function(rejection) {
      if (canRecover(rejection)) {
        return responseOrNewPromise
      }
      return $q.reject(rejection);
    }
  };
});
```
在项目中目前使用了简单的异常拦截，对所有失败的请求统一处理，并弹出异常信息：
```js
feedbackApp.config(function($provide, $httpProvider) {
    // 配置异常拦截器
    $provide.factory('ErrorInterceptor', function($q) {
        return {
            "responseError": function(response) {
                if (response.status == 0) {
                    // session过期是ajax请求会收到这个状态的回馈, 刷新页面以跳到登录页
                    window.location.reload();
                } else if (response.data && response.data.message) {
                    // 弹出异常信息
                    alert(response.data.message);
                } else {
                    // 这里可以讨论一下，如果不加这句，服务失败时页面无响应， 加了服务失败时会频繁跳出提示
                    alert("哎呀，服务器开小差了，再试一次！");
                }
                return $q.reject(response);
            }
        };
    });
    // 注入异常拦截器
    $httpProvider.interceptors.push('ErrorInterceptor');
  }
)
```

### $http的参数们
在使用Angular与后端（如Spring MVC）交互数据的时候，常遇见的问题是接收参数问题。事故常发生在`POST`请求上。
后端代码：
```java
/**
 * 新增项目
 */
@RequestMapping(method = RequestMethod.POST)
public Project newProject(Project project) {
    projectService.newProject(project);
    return project;
}
```
前端请求：
```js
// 请求根路径
$scope.baseUrl = "/project";
...   
// 新增项目
$http.post($scope.baseUrl, project)
    .success(function(data) {
        $scope.projects.push(data);
    });
```
通常，这次请求会以服务端500收场。此时打开浏览器控制台，可查看到如下信息：
![POST配置前](http://7xrbi6.com1.z0.glb.clouddn.com/img-htppPOST%E8%AE%BE%E7%BD%AE%E4%B9%8B%E5%89%8D.png)
Angular中`$http`服务的**POST**和**PUT**方法的传参方式默认是payload，即将对象解析为字符串整个传到后台，这种情况下应该如何接收参数呢？
你有两种方式：

**改变后端**
```java
@RequestMapping(method = RequestMethod.POST)
public Project newProject(@RequestBody Project project) {
    projectService.newProject(project);
    return project;
}
```
关键在于`@RequestBody`，这个注解将请求体中所有的内容都作为`project`对象的内容，并通过spring将`JSON`字符串转换为`project`对象。

**改变前端**
改变Angular默认的提交方式，使用表单提交的模式，核心代码如下：
```js
.config(['$httpProvider',function($httpProvider){
    $httpProvider.defaults.headers.post['Content-Type'] = 'application/x-www-form-urlencoded;charset=utf-8';
    $httpProvider.defaults.headers.put['Content-Type'] = 'application/x-www-form-urlencoded;charset=utf-8';
}])
```
变更之后请求的参数如下：
![POST配置后](http://7xrbi6.com1.z0.glb.clouddn.com/img-htppPOST%E8%AE%BE%E7%BD%AE%E4%B9%8B%E5%90%8E.png)
在某些我也不知道的情况下（当时没有深究），会出现此种方式也不可以的情况，那么就要上杀手锏:将请求的对象转化成键值对的形式提交。结合上面的配置可解决绝大部分问题（目前没有遇到问题），而这个配置基本上是每个Angular应用的默认配置，所以将之抽象为一个模块，在需要的时候引用进来。完整代码如下：
```js
// 声明
(function(){
    angular.module('angular.config', []).config(['$httpProvider',function($httpProvider){
        $httpProvider.defaults.headers.post['Content-Type'] = 'application/x-www-form-urlencoded;charset=utf-8';
        $httpProvider.defaults.headers.put['Content-Type'] = 'application/x-www-form-urlencoded;charset=utf-8';

        var param = function(obj) {
            var query = '', name, value, fullSubName, subName, subValue, innerObj, i;

            for(name in obj) {
                value = obj[name];

                if(value instanceof Array) {
                    for(i=0; i<value.length; ++i) {
                        subValue = value[i];
                        fullSubName = name + '[' + i + ']';
                        innerObj = {};
                        innerObj[fullSubName] = subValue;
                        query += param(innerObj) + '&';
                    }
                }
                else if(value instanceof Object) {
                    for(subName in value) {
                        subValue = value[subName];
                        fullSubName = name + '.' + subName;
                        innerObj = {};
                        innerObj[fullSubName] = subValue;
                        query += param(innerObj) + '&';
                    }
                }
                else if(value !== undefined && value !== null)
                    query += encodeURIComponent(name) + '=' + encodeURIComponent(value) + '&';
            }
            return query.length ? query.substr(0, query.length - 1) : query;
        };

        $httpProvider.defaults.transformRequest = [function(data) {
            return angular.isObject(data) && String(data) !== '[object File]' ? param(data) : data;
        }];
    }]);
}())

// 使用
var watchdogApp = angular.module('watchdogApp',['angular.config', ...]);
```


## Controller的基本结构
### 定义基本变量
用Angular开发最大的特点就是对象化，通过改变对象的属性来控制前端的展示。
>栗子：在做列表查询时，通常把所有查询条件归纳为`query`对象，每次分页查询都将对象传给后端。同时根据对象的属性来显示筛选项

此处不便多言，直接上代码。
```js
// tab页切换
$rootScope.curTab = 'project';

// 基本url
$scope.projectUrl = "/project/";
$scope.statisticUrl = "/statistic/";
$scope.notificationUrl = "/notification/";

// 当前页的项目id
$scope.pid = $routeParams.pid;
```

### 初始化页面
在进入页面时一般只会获得最基本的信息，比如列表页的页码、详情页的实体id，那么在`controller`中就要对页面进行初始化。
>非Angular中，列表页数据或实体信息一般是在后台控制器中查询好，放到`model`里然后跳转页面。而在Angular中往往是先跳转到单页，再请求数据。

```js
/**初始化*/
$scope.init = function() {
    $scope.initProject();
    $scope.initStatistic();
    $scope.initNotification();
}
// 定义好初始化方法，一定要记得调用，并且顺序不能错
$scope.init();
```

### 页面交互函数
页面上的增删改查都在此声明，比如`ng-click`。
```js
// 删除通知项
$scope.deleteNotification = function(index) {
  ...
}
// 显示全部内容
$scope.showAllContent = function() {
  ...
};
```

### 记得“擦屁股“
如果页面中存在周期型任务，记得在销毁`controller`时处理掉，比如`$interval`、JS的各种监听。
```js
$scope.$on('$destroy', function() {
       $interval.cancel($scope.freshDiagram);
       console.log("停止刷新前任");
   });
```

## Angular中的增删改查
### 增
- 在弹层中使用双向绑定，将输入的信息绑定到`$scope`的属性中
- 使用`$http`将数据提交给服务端，一般使用**POST**
- 接收后端返回的对象（或对象id），并将之添加到`$scope`的实体列表里（如果有的话）

```html
<div class="modal-header">
    <h3 class="modal-title">关注一个项目</h3>
</div>
<div class="modal-body">
    <form>
        <div class="form-group">
            <input type="text" class="form-control" ng-model="project.projectName" placeholder="项目名称写上来">
        </div>
        <div class="form-group">
            <textarea class="form-control" rows="3" ng-model="project.description" placeholder="简单介绍下吧"></textarea>
        </div>
    </form>
</div>
<div class="modal-footer">
    <button class="btn btn-primary" type="button" ng-click="ok()">新增</button>
</div>
```
```js
$scope.ok = function() {
  $http.post($scope.baseUrl, project)
      .success(function(data) {
          $scope.projects.push(data);
      });
}
```

### 删
使用`delete`方法，删除成功后，从数据列表中删除这个对象
```html
<div ng-repeat="project in projects">
  <button ng-click="deleteProject($index)">删除</button>
</div>
```
```js
// 删除项目
$scope.deleteProject = function(index) {
   if(!confirm("确实要删除该项目么?相关统计阈和通知项都会被删除.")) {
       return ;
   }

   $http.delete($scope.projectUrl + $scope.projects[index].id + "/info")
       .success(function() {
           $scope.projects.splice(index, 1);
       })
}
```

### 改
修改数据时，记得复制一份更改，更改完毕后再更新回去
```js
// 复制要修改的对象， 否则在弹层中改动，列表中实际数据也会跟着变动
var editModel = angular.extend({}, $scope.project);
// 修改完毕后记得更新回去
$http.put($scope.projectUrl, project)
    .success(function() {
        $scope.project = project;
    })
```
### 查
- 将页面查询用到的字段归纳到一个对象`query`中，包括筛选条件、当前页码（用于翻页）等
- 监听`query`的变化，根据需求调用查询方法
- 查询成功后，更新`$scope`里的列表数据

```html
<div class="boxWrapBlue pd_10 mt_15">
    <div class="clearfix searchFilter pd_6">
        <div class="left">
            <div>
                <span class="">关键字：</span>
                <input type="text" class="txtNew tt280" ng-model="query.keyword" placeholder="请输入SIM卡号、设备号、手机号等"/>

                <span class="ml_15 pl_20">入库时间：</span>
                <dui-date-picker date="query.dateFrom"></dui-date-picker> -
                <dui-date-picker date="query.dateTo"></dui-date-picker>
            </div>

        </div>
        <div class="right center bdL_ccc" style="width:120px;">
            <a href="javascript:;" class="btn-small btn-blue in_block" ng-click="queryData()">搜索</a>
        </div>
    </div>
    <div class="boxWrapFilter2" style="border-top:none;padding:10px 6px;">
    	<span class="pr_10">部门：</span>
        <duitree tree-type="popup"  datasource-type="all" selected-data="query.org" contain-employee="true" org-status="1" can-reset="true" searchable="true" reject-public="true"></duitree>
        <span class="ml_20 pl_20">在职状态：</span>
        <search-filter class="searchFilter" datasource="dictionary.empStatus" selected="query.empStatus" allvalue="" alltext="不限" valuealias="value" textalias="text"></search-filter>
        <span class="ml_20 pl_20">手机类型：</span>
        <search-filter class="searchFilter" datasource="dictionary.phoneType" selected="query.phoneType" allvalue="" alltext="不限" valuealias="value" textalias="text"></search-filter>
    </div>
    <div class="bdT_ccc mt_10 pt_10">
        <p style="padding:10px 6px;">
            <span>状态：</span>
            <status-search-filter class="searchFilter" datasource="dictionary.statusDictionary" count-map="subTotal" selected="query.curStatus" />
        </p>
    </div>
</div>
```
```js
// 归纳
//查询条件
$scope.query = {
    keyword: '',
    empStatus: '',
    phoneType: '',
    dateFrom:null,
    dateTo:null,
    org:
    {
        type:'',
        id:0
    }, //部门对象
    currentPage:1, //当前页号
    curStatus:0 // 当前选中的手机状态，0是全部
};

// 监控查询参数变化
$scope.$watch('query', function(oldValue, newValue){
    // 根据组织架构选择器的选项，判断是员工还是组织，并向后台发送相应的请求参数
    if ($scope.query.org) {
        if ($scope.query.org.type == '员工') {
            $scope.query.empNo = $scope.query.org.id;
        } else {
            $scope.query.orgNo = $scope.query.org.id;
        }
    }
    if (oldValue.keyword != newValue.keyword
        || oldValue.dateFrom != newValue.dateFrom
        || oldValue.dateTo != newValue.dateTo) {
        return;
    }
    // 如果变化的不是页码，将页码初始化
    if (oldValue.currentPage == newValue.currentPage) {
        $scope.query.currentPage = 1;
    }
    // 一旦改变就重新查询
    $scope.queryData();
}, true);

// 通用查询方法
$scope.queryData = function() {
   // 根据组织架构选择器的选项，判断是员工还是组织，并向后台发送相应的请求参数
   if ($scope.query.org) {
       if ($scope.query.org.type == '员工') {
           $scope.query.empNo = $scope.query.org.id;
       } else {
           $scope.query.orgNo = $scope.query.org.id;
       }
   }

   $http.get($scope.baseUrl + "/list/" + $scope.query.currentPage, {params:$scope.query})
       .success(function(data){
           $scope.paginate = data.paginate;
           $scope.subTotal = data.subTotal;
       });

    //请求完要把临时加上的参数删掉
   if($scope.query.empNo) {
       delete $scope.query.empNo;
   }
   if($scope.query.orgNo) {
       delete $scope.query.orgNo;
   }
}
```

## 附录
###简单Controller的完整代码
```js
feedbackApp.controller('FeedbackListController', ['$scope', '$http', '$location', 'anchorScroll', function($scope, $http, $location, anchorScroll) {
    // 是否到了最后
    $scope.endLoad = false;
    // 加载中
    $scope.loading = false;
    // 反馈的根路径
    $scope.feedbackUrl = "/feedback/";
    // 当前页码
    $scope.curPage = 1;
    // 初始化反馈列表
    $scope.feedbacks = [];

    // 加载中toast
    $scope.loadingToast = {
        "isLoading" : "",
        "loadingMsg" : "加载中"
    };

    // 查询反馈分页数据
    $scope.queryFeedback = function(pageNo) {
        // 打开加载状态
        $scope.loading = true;

        // 个人反馈总数
        $http.get($scope.feedbackUrl + "count")
            .success(function(data) {
                $scope.count = data;
            });

        // 分页数据
        $http.get($scope.feedbackUrl + "page/" + pageNo)
            .success(function(data) {
                if (!data || data.length == 0) {
                    $scope.endLoad = true;
                }
                $scope.feedbacks = $scope.feedbacks.concat(data);
                $scope.pageNo = pageNo;

                // 关闭加载
                $scope.loading = false;
                $scope.loadingToast = {
                    "isLoading" : "ng-hide",
                    "loadingMsg" : ""
                };
            })
            .error(function() {
                $scope.loading = false;
                $scope.loadingToast = {
                    "isLoading" : "ng-hide",
                    "loadingMsg" : ""
                };
            });
    }

    // 初始化第一页数据
    $scope.queryFeedback(1);

    // 返回到顶部
    $scope.turnTop = function() {
        anchorScroll.toView('#top', true, 0);
    }

    // 滚动到底部加载
    $(window).scroll(function () {
        // 获得滚动高度
        $scope.height = document.body.scrollTop;

        // 滚动到底部加载
        if (!$scope.endLoad && !$scope.loading && ($(document).scrollTop() + $(window).height() >= $(document).height())) {
            $scope.queryFeedback($scope.pageNo + 1);
        }

        $scope.$apply();
    });

    $scope.$on('$destroy', function() {
        $(window).unbind('scroll');
    });

    // 进入详情页
    $scope.detail = function(index) {
        $location.path($scope.feedbackUrl + "detail/" + $scope.feedbacks[index].id);
    }

}])
```
