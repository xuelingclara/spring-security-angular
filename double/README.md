# Multiple UI Applications and a Gateway: Single Page Application with Spring and Angular JS

In this article we continue [our discussion][fifth] of how to use [Spring Security](http://projects.spring.io/spring-security) with [Angular JS](http://angularjs.org) in a "single page application". Here we show how to use [Spring Session](http://projects.spring.io/spring-security-oauth/) together with [Spring Cloud](http://projects.spring.io/spring-cloud/) to combine the features of the systems we built in parts II and IV, and actually end up building 3 single page applications with quite different responsibilities. The aim is to build a Gateway (like in [part IV][fourth]) that is used not only for API resources but also to load the UI from a backend server. We can simplify the token-wrangling bits of [part II][second] by using the Gateway to pass through the authentication to the backends. Then we extend the system to show how we can make local, granular access decisions in the backends, while still controlling identity and authentication at the Gateway. This is a very powerful model for building distributed systems in general, and has a number of benefits that we can explore as we introduce the features in the code we build.

> Reminder: if you are working through this article with the sample application, be sure to clear your browser cache of cookies and HTTP Basic credentials. In Chrome the best way to do that for a single server is to open a new incognito window.

[first]: http://spring.io/blog/2015/01/12/spring-and-angular-js-a-secure-single-page-application (First Article in the Series)
[second]: http://spring.io/blog/2015/01/12/the-login-page-angular-js-and-spring-security-part-ii (Second Article in the Series)
[third]: http://spring.io/blog/2015/01/20/the-resource-server-angular-js-and-spring-security-part-iii (Third Article in the Series)
[fourth]: http://spring.io/blog/2015/01/28/the-api-gateway-pattern-angular-js-and-spring-security-part-iv (Fourth Article in the Series)
[fifth]: https://spring.io/blog/2015/02/03/sso-with-oauth2-angular-js-and-spring-security-part-v (Fifth Article in the Series)

## Target Architecture

Here's a picture of the basic system we are going to build to start with:

![Components of the System](https://raw.githubusercontent.com/dsyer/spring-security-angular/master/double/double-simple.png)

Like the other sample applications in this series it has a UI (HTML and JavaScript) and a Resource server. Like the sample in [Part IV][fourth] it has a Gateway, but here it is separate, not part of the UI. The UI effectively becomes part of the backend, giving us even more choice to re-configure and re-implement features, and also bringing other benefits as we will see.

The browser goes to the Gateway for everything and it doesn't have to know about the architecture of the backend (fundamentally, it has no idea that there is a back end). One of the things the browser does in this Gateway is authentication, e.g. it sends a username and password like in [Part II][second], and it gets a cookie in return. On subsequent requests it presents the cookie automatically and the Gateway passes it through to the backends. No code needs to be written on the client to enable the cookie passing. The backends use the cookie to authenticate and because all components share a session they share the same information about the user. Contrast this with [Part V][fifth] where the cookie had to be converted to an access token in the Gateway, and the access token then had to be independently decoded by all the backend components.

As in [Part IV][fourth] the Gateway simplifies the interaction between clients and servers, and it presents a small, well-defined surface on which to deal with security. For example, we don't need to worry about [Common Origin Resource Sharing](http://en.wikipedia.org/wiki/Cross-origin_resource_sharing), which is a welcome relief since it is easy to get wrong.

The source code for the complete project we are going to build is in [Github here](https://github.com/dsyer/spring-security-angular/tree/master/double), so you can just clone the project and work directly from there if you want. There is an extra component in the end state of this system ("double-admin") so ignore that for now.

## Building the Backend

In this architecture the backend is very similar to the ["spring-session"](https://github.com/dsyer/spring-security-angular/tree/master/spring-session) sample we built in [Part III][third], with the exception that it doesn't actually need a login page. The easiest way to get to what we want here is probably to copy the "resource" server from Part III and take the UI from the ["basic"](https://github.com/dsyer/spring-security-angular/tree/master/basic) sample in [Part I][first]. To get from the "basic" UI to the one we want here, we need only to add a couple of dependencies (like when we first used [Spring Session](https://github.com/spring-projects/spring-session/) in Part III:

```xml
<dependency>
  <groupId>org.springframework.session</groupId>
  <artifactId>spring-session</artifactId>
  <version>1.0.0.RELEASE</version>
</dependency>
<dependency>
  <groupId>org.springframework.boot</groupId>
  <artifactId>spring-boot-starter-redis</artifactId>
</dependency>
```

and add the `@EnableRedisHttpSession` annotation to the main application class:

```java
@SpringBootApplication
@EnableRedisHttpSession
public class UiApplication {

public static void main(String[] args) {
    SpringApplication.run(UiApplication.class, args);
  }

}
```

Since this is now a UI there is no need for the "/resource" endpoint. When you have done that you will have a very simple Angular application (the same as in the "basic" sample), which simplifies testing and reasoning about its behaviour greatly.

Lastly, we want this server to run as a backend, so we'll give it a non-default port to listen on (in `application.properties`):

```properties
server.port: 8081
security.sessions: NEVER
```

If that's the *only* entry in `application.properties` then the application will be secure and accessible to a user called "user" with a password that is random, but printed on the console (at log level INFO) on startup.

## The Resource Server

The Resource server is easy to generate from one of our existing samples. It is the same as the "vanilla" Resource server in [Part III][third] with the Spring Session and Redis features added. It's really very simple: just a "/resource" endpoint and `@EnableRedisHttpSession` to get the distributed session data. We want this server to have a non-default port to listen on, and we want to be able to look up authentication in the session so we need this (in `application.properties`):

```properties
server.port: 9000
security.sessions: NEVER
```

The completed sample is [here in github](https://github.com/dsyer/spring-security-angular/double/resource) if you want to take a peek.

## The Gateway

For an initial implementation of a Gateway (the simplest thing that could possibly work) we can just take an empty Spring Boot web application and add the `@EnableZuulProxy` annotation. As we saw in [Part I][first] there are several ways to do that, and one is to use the [Spring Initializr](http://start.spring.io) to generate a skeleton project. Even easier, is to use the [Spring Cloud Initializr](http://cloud-start.spring.io) which is the same thing, but for [Spring Cloud](http://cloud.spring.io) applications. Using the same sequence of command line operations as in Part I:

```
$ mkdir gateway && cd gateway
$ curl https://cloud-start.spring.io/starter.tgz -d style=web \
-d style=security -d style=cloud-zuul -d name=gateway \
-d style=redis | tar -xzvf - 
```

You can then import that project (it's a normal Maven Java project by default) into your favourite IDE, or just work with the files and "mvn" on the command line. There is a version [in github](https://github.com/dsyer/spring-security-angular/double/gateway) if you want to go from there, but it has a few extra features that we don't need yet.

Starting from the blank Initializr application, we need to add the Spring Session dependency (like in the UI above), and the `@EnableRedisHttpSession` annotation:

```java
@SpringBootApplication
@EnableRedisHttpSession
public class GatewayApplication {

public static void main(String[] args) {
    SpringApplication.run(GatewayApplication.class, args);
  }

}
```

The Gateway is ready to run, but it doesn't yet know about our backend services, so let's just set that up in its `application.yml` (renaming from `application.properties` if you did the curl thing above):

```
zuul:
  routes:
    ui:
      url: http://localhost:8081
    resource:
      url: http://locahost:9000
security:
  user:
    password:
      password
  sessions: ALWAYS
```

There are 2 routes in the proxy, one each for the UI and resource server, and we have set up a default password and a session persistence strategy (telling Spring Security to always create a session on authentication). This last bit is important because we want authentication and therefore sessions to be managed in the Gateway.

## Up and Running

We now have three components, running on 3 ports. If you point the browser at http://localhost:8080/ui/ you should get an HTTP Basic challenge, and you can authenticate as "user/password" (your credentials in the Gateway), and once you do that you should see your message in the UI, via a backend call through the proxy to the Resource server.

The interactions between the browser and the backend can be seen in your browser if you use some developer tools (usually F12 opens this up, works in Chrome by default, requires a plugin in Firefox). Here's a summary:


Verb | Path    | Status | Response
-----|---------|--------|---------
GET  | /ui/    | 401    | Browser prompts for authentication
GET  | /ui/    | 200    | index.html
GET  | /ui/css/angular-bootstrap.css | 200 | Twitter bootstrap CSS
GET  | /ui/js/angular-bootstrap.js  | 200 | Bootstrap and Angular JS
GET  | /ui/js/hello.js              | 200 | Application logic
GET  | /ui/user    | 200    | authentication
GET  | /resource/  | 200 | JSON greeting

You might not see the 401 because the browser treats the home page load as a single interaction. All requests are proxied (there is no content in the Gateway yet).

Hurrah, it works! You have 

## Adding a Login Form

Just as in the "basic" sample in [Part I][first] we can now add a login form to the Gateway, e.g. by copying the code from [Part II][second]. When we do that we can also add a basic navigation UI in the Gateway, so the user doesn't have to know the path to the UI backend in the proxy. So let's first copy the static assets from the UI backend into the Gateway, delete the message rendering and insert a login form into our home page (in the `<body/>` somewhere):

```html
<div class="container" ng-show="!authenticated">
  <form role="form" ng-submit="login()">
    <div class="form-group">
      <label for="username">Username:</label> <input type="text"
        class="form-control" id="username" name="username"
        ng-model="credentials.username" />
    </div>
    <div class="form-group">
      <label for="password">Password:</label> <input type="password"
        class="form-control" id="password" name="password"
        ng-model="credentials.password" />
    </div>
    <button type="submit" class="btn btn-primary">Submit</button>
  </form>
</div>
```

Instead of the message rendering we will have a nice big navigation button:

```html
<div class="container" ng-show="authenticated">
  <a class="btn btn-primary" href="/ui/">Go To User Interface</a>
</div>
```

If you are looking at the sample in github, it also has a minimal navigation bar with a "Logout" button. Here's the login form in a screenshot:

![Login Page](https://raw.githubusercontent.com/dsyer/spring-security-angular/master/double/login.png)

To support the login form we need some JavaScript implementing the `login()` function we declared in the `<form/>`, and we need to set the `authenticated` flag so that the home page will render different content depending on whether or not we are authenticated. For example:

```javascript
angular.module('hello', []).controller('navigation',
function($scope, $http) {

  ...
  
  authenticate();
  
  $scope.credentials = {};

$scope.login = function() {
    authenticate($scope.credentials, function() {
      if ($scope.authenticated) {
        console.log("Login succeeded")
        $scope.error = false;
        $scope.authenticated = true;
      } else {
        console.log("Login failed")
        $scope.error = true;
        $scope.authenticated = false;
      }
    })
  };

}
```

where the implementation of the `authenticate()` function is similar to that in [Part II][second]:

```javascript
var authenticate = function(credentials, callback) {

  var headers = credentials ? {
    authorization : "Basic "
        + btoa(credentials.username + ":"
            + credentials.password)
  } : {};

  $http.get('user', {
    headers : headers
  }).success(function(data) {
    if (data.name) {
      $scope.authenticated = true;
    } else {
      $scope.authenticated = false;
    }
    callback && callback();
  }).error(function() {
    $scope.authenticated = false;
    callback && callback();
  });

}
```

We can simply use the `$scope` to store the `authenticated` flag because there is only one controller in such a simple application.

If we run this enhanced Gateway, instead of having to remember the URL for the UI we can just load the home page and follow links. Here's the home page for an authenticated user:

![Home Page](https://raw.githubusercontent.com/dsyer/spring-security-angular/master/double/home.png)

## Granular Access Decisions in the Backend

Up to now our application is functionally very similar to the one in [Part III][third] or [Part IV][fourth], but with an additional dedicated Gateway. The advantage of the extra layer is not yet apparent, but we can emphasise it by expanding the system a bit. Suppose we want to use that Gateway to expose another backend UI, for users to "administrate" the content in the main UI, and that we want to restrict access to this feature to users with special roles. So we will add an "Admin" application behind the proxy, and the system will look like this:

![Components of the System](https://raw.githubusercontent.com/dsyer/spring-security-angular/master/double/double-components.png)

There is a new component (Admin) and a new route in the Gateway in `application.yml`:

```yaml
zuul:
  routes:
    ui:
      url: http://localhost:8081
    admin:
      url: http://localhost:8082
    resource:
      url: http://localhost:9000
```

The fact that the existing UI is available to users in the "USER" role is indicated on the block diagram above in the Gateway box (green lettering), as is the fact that the "ADMIN" role is needed to go to the Admin application. The access decision for the "ADMIN" role could be applied in the Gateway, in which case it would appear in a `WebSecurityConfigurerAdapter`, or it could be applied in the Admin application itself (which is probably preferable and we will see how to do that below).

In addition, suppose that inside the Admin application we want to distinguish between "READER" and "WRITER" roles, so that we can permit (let's say) users who are auditors to view the changes made by the main admin users. This is a granular access decision, where the rule is only known, and should only be known, in the backend application. In the Gateway we only need to ensure that our user accounts have the roles needed, and this information is available, but the Gateway doesn't need to know how to interpret it. In the Gateway we create user accounts to keep the sample application self-contained:

```javascript
@Configuration
public class SecurityConfiguration extends WebSecurityConfigurerAdapter {

  @Autowired
  public void globalUserDetails(AuthenticationManagerBuilder auth) throws Exception {
    auth.inMemoryAuthentication()
      .withUser("user").password("password").roles("USER")
    .and()
      .withUser("admin").password("admin").roles("USER", "ADMIN", "READER", "WRITER")
    .and()
      .withUser("audit").password("audit").roles("USER", "ADMIN", "READER");
  }
  
}
```

where the "admin" user has been enhanced with 3 new roles ("ADMIN", "READER" and "WRITER") and we have also added an "audit" user with "ADMIN" access, but not "WRITER".

> Aside: In a production system the user account data would be managed in a backend database (most likely a directory service), not hard coded in the Spring Configuration. Sample applications connecting to such a database are easy to find on the internet, for example in the [Spring Security Samples](https://github.com/spring-projects/spring-security/tree/master/samples).

The access decisions go in the Admin application. For the "ADMIN" role (which is required globally for this backend) we do it in Spring Security:

```java
@Configuration
public class SecurityConfiguration extends WebSecurityConfigurerAdapter {

@Override
  protected void configure(HttpSecurity http) throws Exception {
    http
    ...
      .authorizeRequests()
        .antMatchers("/index.html", "/login", "/").permitAll()
        .antMatchers("/admin/**").hasRole("ADMIN")
        .anyRequest().authenticated()
    ...
  }
  
}
```

For the "READER" and "WRITER" roles the application itself is split, and since the application is implemented in JavaScript, that is where we need to make the access decision. One way to do this is to have a home page with a computed view embedded in it:

```html
<div class="container">
  <h1>Admin</h1>
  <div ng-show="authenticated" ng-include="template"></div>
  <div ng-show="!authenticated" ng-include="'unauthenticated.html'"></div>
</div>
```

Angular JS evaluates the "ng-include" attribute value as an expression, and then uses the result to load a template. 

> Tip: A more complex application might use other mechanisms to modularize itself, e.g. the `$routeProvider` service that we used in nearly all the other applications in this series. 

The `template` variable is initialized in our controller, first by defining a utility function:

```javascript
var computeDefaultTemplate = function(user) {
  $scope.template = user && user.roles
      && user.roles.indexOf("ROLE_WRITER")>0 ? "write.html" : "read.html";		
}
```

then by using the utility function when the controller loads:

```javascript
angular.module('admin', []).controller('home',

function($scope, $http) {
	
  $http.get('user').success(function(data) {
    if (data.name) {
      $scope.authenticated = true;
      $scope.user = data;
      computeDefaultTemplate(data);
    } else {
      $scope.authenticated = false;
    }
    $scope.error = null
  })
  ...
      
})
```

the first thing the application does is look at the usual (for this series) "/user" endpoint, then it extracts some data, sets the authenticated flag, and if the user is authenticated, computes the template by looking at the user data. 

To support this function on the backend we need an endpoint, e.g. in our main application class:

```java
@SpringBootApplication
@RestController
@EnableRedisHttpSession
public class AdminApplication {

  @RequestMapping("/user")
  public Map<String, Object> user(Principal user) {
    Map<String, Object> map = new LinkedHashMap<String, Object>();
    map.put("name", user.getName());
    map.put("roles", AuthorityUtils.authorityListToSet(((Authentication) user)
        .getAuthorities()));
    return map;
  }

  public static void main(String[] args) {
    SpringApplication.run(AdminApplication.class, args);
  }

}
```

> Note: the role names come back from the "/user" endpoint with the "ROLE_" prefix so we can distinguish them from other kinds of authorities (it's a Spring Security thing). Thus the "ROLE_" prefix is needed in the JavaScript, but not in the Spring Security configuration, where it is clear from the method names that "roles" are the focus of the operations.

## Why are we Here?

Now we have a nice little system with 2 independent user interfaces and a backend Resource server, all protected by the same authentication in a Gateway. The fact that the Gateway acts as a micro-proxy makes the implementation of the backend security concerns extremely simple, and they are free to concentrate on their own business concerns. The use of Spring Session has (again) avoided a huge amount of hassle and potential errors.

A powerful feature is that the backends can independently have any kind of authentication they like, and this can, for example, be useful for testing purposes (e.g. you can go directly to the UI if you know its physical address and a set of local credentials). The Gateway imposes a completely unrelated set of constraints, as long as it can authenticate users and assign metadata to them that satisfy the access rules in the backends. This is an excellent design for being able to independently develop and test the backend components. If we wanted to, we could go back to an external OAuth2 server (like in [Part V][fifth], or even something completely different) for the authentication at the Gateway, and the backends would not need to be touched.

A bonus feature of this architecture (single Gateway controlling authentication, and shared session token across all components) is that "Single Logout", a feature we identified as difficult to implement in [Part V][fifth], comes for free. To be more precise, one particular approach to the user experience of single logout is automatically available in our finished system: if a user logs out of any of the UIs (Gateway, UI backend or Admin backend), he is logged out of all the others, assuming that each individual UI implemented a "logout" feature the same way (invalidating the session).

> Thanks: I would like to thank again everyone who helped me develop this series, and in particular [Rob Winch](http://spring.io/team/rwinch) and [Thorsten Späth](https://twitter.com/thspaeth) for their careful reviews of the articles and sources code. Since [Part I][first] was published it hasn't changed much but all the other parts have evolved in response to comments and insights from readers, so thank you also to anyone who read the articles and took the trouble to join in the discussion.
