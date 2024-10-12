In this repo, I list some techs I have used in my two recents projects and share some technical details in regards to how I have used them.


# CI/CD Pipeline in My Project

![CI/CD Pipeline](./assets/cicd.png)


In my project, I use Gogs, Drone, and Harbor to build a CI/CD pipeline. Gogs is used for code repo. With Harbor, I build a 
image registry. Drone automates the image build, push, and deployment process. It has enabled me to realize automation using just 
a YAML file.

Technical docs: [Gogs](gogs.md), [Harbor](harbor.md), [Drone](drone.md)




# IoC and Dependency Injection

[Inverse of Control](https://medium.com/@amitkma/understanding-inversion-of-control-ioc-principle-163b1dc97454) (IoC) is a design principle that suggests "inversing the control flow of a program". Implementing this
principle helps increase the modularity of the software, decouple the codependent components, and make your software more extensible.


In my two most recent Go projects, I have reused the same IoC container design. In short, I have built an IoC container to register the API handlers and another one to register different controllers. 

Define a basic container structure:
```go
type IocContainer struct {
	store map[string]IocObject
}
```

Basically, the container is a map, in which the keys are simply the components names, and the values are just the IoC object. For example, 
a module is dependent on another module. Instead of creating an instance inside it, it can just get an instance of it from the container by using the module's name. An example 
will be provided below.

Degine an `IocObject` interface so that any object who implements this interface would be also an `IocObeject`.

```go
type IocObject interface {
	Init() error

	Name() string
}
```

I also defined a `GinApiHandler` interface to register the APIs

```go
type GinApiHandler interface {
	Registry(gin.IRouter)
}
```


`IocContainer` has three important methods:
```go
{
    Init() error
	Get(name string) IocObject
    Register(obj IocObject)
    RouterRegistry(router gin.IRouter)
}
```
Implementations:
```go

func (c *IocContainer) Init() error {
    //initialize the IocObjects in the hash map.
    //the objects who have implemented the IocObject interface will have the method Init(). Within this method is the initialization logic.
	for _, obj := range c.store {
		if err := obj.Init(); err != nil {
			return err
		}
	}
	return nil
}

func (c *IocContainer) Register(obj IocObject) {
    //This method should be called with Go initializes the project. 
    c.store[obj.Name()] = obj

}

func (c *IocContainer) Get(name string) IocObject {
    //this method is used to get the IocObject within the container.
    return c.store[name]
}

func (c *IocContainer) RouterRegistry(router gin.IRouter) {
    //all the API handlers need to implement the `GinApiHandler` interface
    //the registry() method will register all the APIs to the Gin Router
    //Upon startup, this method will be called to register the APIs
    for _, obj := range c.store {
        if apiHandler, ok := obj.(GinApiHandler); ok {
            apiHandler.Registry(router)
        }
    }
}
```


Below is an example for a controller.

In my `MyBlog` project, I have two components: `token` service and `user` service. `token` service has two important responsibilities: login and logout.
Upon login, `token` service needs to call the `user` service to query the user and compare the information fetched by `user` service with the information from the request.
Thus, `token` service actually depends on the `user` service. However, with my Ioc container, the dependency is removed since `token` service can just get a global 
`user` service in the container.


Define `token` service

```go
type TokenServiceImpl struct {
	...
	user user.Service
}
```


Implement the `IocObject` interface

```go
func (t *TokenServiceImpl) Init() error {
	...
    /* notice here instead of creating a new instance of user service, we just get the user service instance from container.
	and this instance is only created once
    */
	t.user = ioc.DefaultControllerContainer().Get(user.AppName).(user.Service)
	return nil
}

func (t *TokenServiceImpl) Name() string {
	return token.AppName
}
```

Implement an `init()` function to register the IocObject to the container upon project startup. After startup, the `Init()` method above will be called in the main routine to initialize the object.

```go
func init() {
	ioc.DefaultControllerContainer().Register(&TokenServiceImpl{})
}
```


Below is an example for API Handler. API handler depends on controller to deal with the core data logic

Define the structure. 
```go
type TokenApiHandler struct {
	svc token.Service
}
```

Implement the `IocObject` interface

```go
func (h *TokenApiHandler) Init() error {
    /*
    here we also get the instance from the container
    */
	h.svc = ioc.DefaultControllerContainer().Get(token.AppName).(token.Service)
	return nil
}

func (h *TokenApiHandler) Name() string {
	return token.AppName
}
```

Since we need to make it a GinApiHandler, we also need to implement the `GinApiHandler` interface
```go
func (h *TokenApiHandler) Registry(router gin.IRouter) {
	v1 := router.Group("v1")
	v1.POST("/tokens/", h.Login)
	v1.DELETE("/tokens/", h.Logout)
}
```

With the dependency injection pattern, the project is more extensible.



# RabbitMQ




