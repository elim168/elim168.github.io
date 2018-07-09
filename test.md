<div ng-app="app" ng-controller="controller">
	<div id="menu">
		<ul>
			<li><a href="#/ehcache">开启JMX支持</a></li>
			<li><a href="#/test2">Test2</a></li>
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
		margin-left: -300px;
		margin-top: -100px;
		border: 1px solid #cdf;
		
	}
	#mainContent {
		border: 1px solid red;
		min-height: 200px;
	}
</style>

<script type="text/javascript">
	angular.module('app', ['ngRoute']).controller('controller', function($scope) {
			
	})
	.config(['$routeProvider', function($routeProvider){
		$routeProvider
		.when('/ehcache',{templateUrl:'ehcache/01.%E5%BC%80%E5%90%AFJMX%E6%94%AF%E6%8C%81.html'})
		.otherwise({redirectTo:'/'});
	}]);
</script>