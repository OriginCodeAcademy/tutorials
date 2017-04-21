# Authentication in CSharp Part 2

In this demo students will be shown how to use the authentication setup in the previous guide.

## Sources
[Token Based Authentication - Taiseer Joudeh](http://bitoftech.net/2014/06/09/angularjs-token-authentication-using-asp-net-web-api-2-owin-asp-net-identity/)<br />
[ASP.NET Identity with Entity Framework](http://odetocode.com/blogs/scott/archive/2014/01/03/asp-net-identity-with-the-entity-framework.aspx)

## Table of Contents

- [ ] Add an authentifaction factory
- [ ] Add a register feature
- [ ] Add a login feature

## Add an authentication factory

- [ ] Add this factory to your project - call it `authentication.factory.js`

```js
(function() {
    'use strict';

    angular
        .module('app')
        .factory('authFactory', authFactory);

    authFactory.$inject = ['apiUrl', '$http', '$q', 'localStorageService'];

    /* @ngInject */
    function authFactory(apiUrl, $http, $q, localStorageService) {
        var service = {
            initialize: initialize,
            register: register,
            login: login,
            logout: logout
        };

        service.isAuth = false;
        service.username = '';

        return service;

        ////////////////

        function initialize() {
            var authData = localStorageService.get('authorizationData');
            if(authData) {
                service.isAuth = false;
                service.username = '';
            }
        }

        function register(registration) {
            logout();

            var defer = $q.defer();
            
            $http.post(apiUrl + 'register', registration).then(
                function(response) {
                    defer.resolve(response.data);
                },
                function(error) {
                    console.log(error);
                    defer.reject(error);
                }
            );
            
            return defer.promise;
        }

        function login(username, password) {
            var data = "grant_type=password&username=" + username +
                       "&password=" + password;

            var defer = $q.defer();
            
            $http.post(apiUrl + 'token',  data, { headers: { 'Content-Type': 'application/x-www-form-urlencoded' } }).then(
                function(response) {
                    localStorageService.set('authorizationData', {
                        token: response.data.access_token,
                        username: username
                    });

                    service.isAuth = true;
                    service.username = username;

                    defer.resolve(response.data);
                },
                function(error) {
                    logout();
                    console.log(error);
                    defer.reject(error);
                }
            );
            
            return defer.promise;
        }

        function logout() {
            localStorageService.remove('authorizationData');

            service.isAuth = false;
            service.username = '';
        }
    }
})();
```

## Add a Register Feature

- [ ] Create a `register` folder in the `src` folder.
	- [ ] Create an HTML file
	- [ ] Create an Angular controller

**register.controller.js**

```js
(function() {
    'use strict';

    angular
        .module('app')
        .controller('RegisterController', RegisterController);

    RegisterController.$inject = ['authFactory', '$state'];

    /* @ngInject */
    function RegisterController(authFactory, $state) {
        var vm = this;
        
        vm.registration = {
        	username: '',
        	password: '',
        	confirmPassword: ''
        };

        vm.register = register;

        ////////////////

        function register() {
        	authFactory.register(vm.registration).then(
        		function(response) {
        			alert('Registration successful! Please login.');
        			$state.go('login');
        		},
        		function(response) {
        			alert('Registration form invalid');
        		}
      		);
        }
    }
})();
```

## Add a Login Feature

- [ ] Create a `login` folder in the `src` folder.
	- [ ] Create an HTML file
	- [ ] Create an Angular controller

**login.controller.js**

```js
(function() {
    'use strict';

    angular
        .module('app')
        .controller('LoginController', LoginController);

    LoginController.$inject = ['$state', 'authFactory'];

    /* @ngInject */
    function LoginController($state, authFactory) {
        var vm = this;

        vm.login = login;        

        ////////////////

        function login() {
        	authFactory.login(vm.username, vm.password).then(
        		function(response) {
        			$state.go('app.dashboard');
        		},
        		function(error) {
        			alert(error.error_description);
        		}
      		);
        }
    }
})();
```

## Refactor State Structure

### Before
```js
$stateProvider
	.state('dashboard', { 
		url: '/dashboard', 
		controller: 'DashboardController as dashboard', 
		templateUrl: 'app/dashboard/dashboard.html'
	})
	.state('student', {
		url: '/student',
		abstract: true,
		template: '<div ui-view></div>'
	})
		.state('student.grid', {
			url: '/grid',
			controller: 'StudentGridController as studentGrid',
			templateUrl: 'app/student/student.grid.html'
		})
		.state('student.detail', {
			url: '/detail?studentId',
			controller: 'StudentDetailController as studentDetail',
			templateUrl: 'app/student/student.detail.html'
		})
	.state('project', {
		url: '/project',
		abstract: true,
		template: '<div ui-view></div>'
	})
		.state('project.grid', {
			url: '/grid',
			controller: 'ProjectGridController as projectGrid',
			templateUrl: 'app/project/project.grid.html'
		})
		.state('project.detail', {
			url: '/detail?projectId',
			controller: 'ProjectDetailController as projectDetail',
			templateUrl: 'app/project/project.detail.html'
		});
```

### After

```js
$stateProvider
	.state('register', {
		url: '/register',
		controller: 'RegisterController as register',
		templateUrl: 'app/register/register.html'
	})
	.state('login', {
		url: '/login',
		controller: 'LoginController as login',
		templateUrl: '/app/login/login.html'
	})
	.state('app', {
		url: '/app',
		abstract: true,
		template: '<div ui-view></div>'
	})
		.state('app.dashboard', { 
			url: '/dashboard', 
			controller: 'DashboardController as dashboard', 
			templateUrl: 'app/dashboard/dashboard.html'
		})
		.state('app.student', {
			url: '/student',
			abstract: true,
			template: '<div ui-view></div>'
		})
			.state('app.student.grid', {
				url: '/grid',
				controller: 'StudentGridController as studentGrid',
				templateUrl: 'app/student/student.grid.html'
			})
			.state('app.student.detail', {
				url: '/detail?studentId',
				controller: 'StudentDetailController as studentDetail',
				templateUrl: 'app/student/student.detail.html'
			})
		.state('app.project', {
			url: '/project',
			abstract: true,
			template: '<div ui-view></div>'
		})
			.state('app.project.grid', {
				url: '/grid',
				controller: 'ProjectGridController as projectGrid',
				templateUrl: 'app/project/project.grid.html'
			})
			.state('app.project.detail', {
				url: '/detail?projectId',
				controller: 'ProjectDetailController as projectDetail',
				templateUrl: 'app/project/project.detail.html'
			});
```

## Demo Registration and Login

Show a successful registration and a successful login - but note that the dashboard does not work. Show the `401 Unauthorized` error. Explain that we're not sending the JWT along with every request, so our requests are unauthorized.

## Implement `authInterceptorService`

Rather than adding the token to every single one of our HTTP requests - let's instead write an Interceptor, which will intercept every http request made by our application and add the token before letting it go off to the server.

**authInterceptor.service.js**

```js
(function() {
    'use strict';

    angular
        .module('app')
        .factory('authInterceptorService', authInterceptorService);

    authInterceptorService.$inject = ['$q', '$location', 'localStorageService'];

    /* @ngInject */
    function authInterceptorService($q, $location, localStorageService) {
        var service = {
            request: request,
            responseError: responseError
        };
        return service;

        ////////////////

        function request(httpRequest) {
        	httpRequest.headers = httpRequest.headers || {};

        	var authData = localStorageService.get('authorizationData');

        	if(authData) {
        		httpRequest.headers.Authorization = 'Bearer ' + authData.token;
        	}

        	return config;
        }

        function responseError(httpResponse) {
        	if(httpResponse.status === 401) {
        		$location.path('/login');
        	}
        	return $q.reject(httpResponse);
        }
    }
})();
```

Finally - in our app.config function, we need to add the following

```js

.config(function($stateProvider, $urlRouterProvider, $httpProvider) {
	$httpProvider.interceptors.push('authInterceptorService');
});

```
