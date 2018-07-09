<div ng-app="app" ng-controller="controller">
	<div id="menu">
		<ul>
          <li><a href="#/spring/mvc/01.Spring%E6%95%B4%E5%90%88Shiro%E5%B9%B6%E6%89%A9%E5%B1%95%E4%BD%BF%E7%94%A8EL%E8%A1%A8%E8%BE%BE%E5%BC%8F.html">01.Spring整合Shiro并扩展使用EL表达式</a></li>
          <li><a href="#/spring/mvc/01.%E8%B7%AF%E5%BE%84%E5%8F%98%E9%87%8F%E4%B8%AD%E4%BD%BF%E7%94%A8%E6%AD%A3%E5%88%99%E8%A1%A8%E8%BE%BE%E5%BC%8F%E5%8C%B9%E9%85%8D.html">01.路径变量中使用正则表达式匹配</a></li>
          <li><a href="#/spring/mvc/02.%E9%80%9A%E9%85%8D%E7%AC%A6%E5%8C%B9%E9%85%8D.html">02.通配符匹配</a></li>
          <li><a href="#/spring/mvc/03.DispatcherServlet.properties.html">03.DispatcherServlet.properties</a></li>
          <li><a href="#/spring/mvc/04.RequestContextUtils%E5%B7%A5%E5%85%B7%E7%B1%BB.html">04.RequestContextUtils工具类</a></li>
          <li><a href="#/spring/mvc/05.RedirectAttributes.html">05.RedirectAttributes</a></li>
          <li><a href="#/spring/mvc/06.theme%E9%80%89%E6%8B%A9.html">06.theme选择</a></li>
          <li><a href="#/spring/mvc/07.SpringMVC%E6%96%87%E4%BB%B6%E4%B8%8A%E4%BC%A0%E5%AF%B9Servlet3%E7%9A%84%E6%94%AF%E6%8C%81.html">07.SpringMVC文件上传对Servlet3的支持</a></li>
          <li><a href="#/spring/mvc/08.SpringMVC%E4%B9%8BControllerAdvice.html">08.SpringMVC之ControllerAdvice</a></li>
          <li><a href="#/spring/mvc/09.SpringMVC%E4%B9%8BResponseEntityExceptionHandler.html">09.SpringMVC之ResponseEntityExceptionHandler</a></li>
          <li><a href="#/spring/mvc/10.ResponseStatus.html">10.ResponseStatus</a></li>
          <li><a href="#/spring/mvc/11.%E9%80%9A%E8%BF%87%E7%A8%8B%E5%BA%8F%E5%AE%9A%E4%B9%89DispatcherServlet.html">11.通过程序定义DispatcherServlet</a></li>
          <li><a href="#/spring/mvc/12.%E7%9B%B4%E6%8E%A5%E6%8C%87%E5%AE%9A%E8%B7%AF%E5%BE%84%E5%AF%B9%E5%BA%94%E7%9A%84%E8%A7%86%E5%9B%BE%E5%90%8D%E7%A7%B0.html">12.直接指定路径对应的视图名称</a></li>
          <li><a href="#/spring/mvc/13.%E6%8C%87%E5%AE%9A%E9%9D%99%E6%80%81%E8%B5%84%E6%BA%90%E8%B7%AF%E5%BE%84.html">13.指定静态资源路径</a></li>
          <li><a href="#/spring/mvc/14.SpringMVC%E5%AF%B9Servlet3%E5%BC%82%E6%AD%A5%E8%AF%B7%E6%B1%82%E7%9A%84%E6%94%AF%E6%8C%81.html">14.SpringMVC对Servlet3异步请求的支持</a></li>
        </ul>
	</div>

	<div id="mainContent" ng-view>
	
	</div>

</div>
<script src="src/js/angular.1.2.32.js"></script>
<script src="src/js/angular-route.js"></script>
<style>
	#menu {
		width: 300px;
		margin-left: 50px;
		margin-top: 450px;
		border: 1px solid #cdf;
		float: left;
		
	}
	#mainContent {
		border: 1px solid red;
		position: absolute;
		top: 0px;
		left: ; 0px;
	}
</style>

<script type="text/javascript">
	angular.module('app', ['ngRoute']).controller('controller', function($scope) {
			
	})
	.config(['$routeProvider', function($routeProvider){
		$routeProvider
		.when('/spring/mvc/01.Spring%E6%95%B4%E5%90%88Shiro%E5%B9%B6%E6%89%A9%E5%B1%95%E4%BD%BF%E7%94%A8EL%E8%A1%A8%E8%BE%BE%E5%BC%8F.html',{templateUrl:'spring/mvc/01.Spring%E6%95%B4%E5%90%88Shiro%E5%B9%B6%E6%89%A9%E5%B1%95%E4%BD%BF%E7%94%A8EL%E8%A1%A8%E8%BE%BE%E5%BC%8F.html'})
		.when('/spring/mvc/01.%E8%B7%AF%E5%BE%84%E5%8F%98%E9%87%8F%E4%B8%AD%E4%BD%BF%E7%94%A8%E6%AD%A3%E5%88%99%E8%A1%A8%E8%BE%BE%E5%BC%8F%E5%8C%B9%E9%85%8D.html',{templateUrl:'spring/mvc/01.%E8%B7%AF%E5%BE%84%E5%8F%98%E9%87%8F%E4%B8%AD%E4%BD%BF%E7%94%A8%E6%AD%A3%E5%88%99%E8%A1%A8%E8%BE%BE%E5%BC%8F%E5%8C%B9%E9%85%8D.html'})
		.when('/spring/mvc/02.%E9%80%9A%E9%85%8D%E7%AC%A6%E5%8C%B9%E9%85%8D.html',{templateUrl:'spring/mvc/02.%E9%80%9A%E9%85%8D%E7%AC%A6%E5%8C%B9%E9%85%8D.html'})
		.when('/spring/mvc/03.DispatcherServlet.properties.html',{templateUrl:'spring/mvc/03.DispatcherServlet.properties.html'})
		.when('/spring/mvc/04.RequestContextUtils%E5%B7%A5%E5%85%B7%E7%B1%BB.html',{templateUrl:'spring/mvc/04.RequestContextUtils%E5%B7%A5%E5%85%B7%E7%B1%BB.html'})
		.when('/spring/mvc/05.RedirectAttributes.html',{templateUrl:'spring/mvc/05.RedirectAttributes.html'})
		.when('/spring/mvc/06.theme%E9%80%89%E6%8B%A9.html',{templateUrl:'spring/mvc/06.theme%E9%80%89%E6%8B%A9.html'})
		.when('/spring/mvc/07.SpringMVC%E6%96%87%E4%BB%B6%E4%B8%8A%E4%BC%A0%E5%AF%B9Servlet3%E7%9A%84%E6%94%AF%E6%8C%81.html',{templateUrl:'spring/mvc/07.SpringMVC%E6%96%87%E4%BB%B6%E4%B8%8A%E4%BC%A0%E5%AF%B9Servlet3%E7%9A%84%E6%94%AF%E6%8C%81.html'})
		.when('/spring/mvc/08.SpringMVC%E4%B9%8BControllerAdvice.html',{templateUrl:'spring/mvc/08.SpringMVC%E4%B9%8BControllerAdvice.html'})
		.when('/spring/mvc/09.SpringMVC%E4%B9%8BResponseEntityExceptionHandler.html',{templateUrl:'spring/mvc/09.SpringMVC%E4%B9%8BResponseEntityExceptionHandler.html'})
		.when('/spring/mvc/10.ResponseStatus.html',{templateUrl:'spring/mvc/10.ResponseStatus.html'})
		.when('/spring/mvc/11.%E9%80%9A%E8%BF%87%E7%A8%8B%E5%BA%8F%E5%AE%9A%E4%B9%89DispatcherServlet.html',{templateUrl:'spring/mvc/11.%E9%80%9A%E8%BF%87%E7%A8%8B%E5%BA%8F%E5%AE%9A%E4%B9%89DispatcherServlet.html'})
		.when('/spring/mvc/12.%E7%9B%B4%E6%8E%A5%E6%8C%87%E5%AE%9A%E8%B7%AF%E5%BE%84%E5%AF%B9%E5%BA%94%E7%9A%84%E8%A7%86%E5%9B%BE%E5%90%8D%E7%A7%B0.html',{templateUrl:'spring/mvc/12.%E7%9B%B4%E6%8E%A5%E6%8C%87%E5%AE%9A%E8%B7%AF%E5%BE%84%E5%AF%B9%E5%BA%94%E7%9A%84%E8%A7%86%E5%9B%BE%E5%90%8D%E7%A7%B0.html'})
		.when('/spring/mvc/13.%E6%8C%87%E5%AE%9A%E9%9D%99%E6%80%81%E8%B5%84%E6%BA%90%E8%B7%AF%E5%BE%84.html',{templateUrl:'spring/mvc/13.%E6%8C%87%E5%AE%9A%E9%9D%99%E6%80%81%E8%B5%84%E6%BA%90%E8%B7%AF%E5%BE%84.html'})
		.when('/spring/mvc/14.SpringMVC%E5%AF%B9Servlet3%E5%BC%82%E6%AD%A5%E8%AF%B7%E6%B1%82%E7%9A%84%E6%94%AF%E6%8C%81.html',{templateUrl:'spring/mvc/14.SpringMVC%E5%AF%B9Servlet3%E5%BC%82%E6%AD%A5%E8%AF%B7%E6%B1%82%E7%9A%84%E6%94%AF%E6%8C%81.html'})
		.otherwise({redirectTo:'/'});
	}]);
</script>