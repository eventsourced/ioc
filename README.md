# ioc

Javascript lacks one aspect of OOP systems that most have come to rely on, interfaces and dependency injection.
That's why we created an Inversion of Control (IOC) container for commonjs apps, modelled after the Laravel 5.1 IOC container.

It allows you to

- Create interfaces
- Create classes that implement those interfaces
- Bind classes to those classes
- Inject implementations of interfaces wherever you need them

## Usage

###Interfaces:
Creating an interface is pretty easy, here's an example of a PaymentService that is used in our domain.

```js
var InterfaceFactory = require('ioc').interfaceFactory;

var PaymentService = InterfaceFactory.make({
  chargeAccount: function(paymentDetails) {}
});

//This creates an interface that you can use when creating a class, like the following.

var ClassFactory = require('ioc').classFactory;

var StripeService = ClassFactory.make({
 
  _implements: [PaymentService],
  
  chargeAccount: function(paymentDetails) {
    
  }
});
```

If you've ever used BoackboneJS or StapesJs, the class constructor will look familiar. 
When it's created, the ClassFactory makes sure that the objects schema matches the interface defined. Simply pass in the interfaces as an array in the param "_implements", and the factory will take care of the rest.

You can pass multiple interfaces, the factory will make sure that they're all implemented.

###Injecting classes:

To bind a class to an interface, add the following in your bootstrap code.

```js
var PaymentService = require('./payment-service');
var StripeService = require('./stripe-service');
ioc.bind(PaymentService, StripeService);

//Or, the shorter version.

ioc.bind(require('./payment-service'), require('./stripe-service'));
```

This binds that interface to that concrete class. It also checks the interface, to make sure the concrete class implements that interface.
It handles single instances as well, so you can bind an instance like this.

```js
ioc.bind(PaymentService, new StripeService());
```
To create a concrete instance, call the following.

```js
var paymentService = ioc.make(PaymentService);
```
That's it, you now have a concrete instance of that interface.

###Dependency injection:

Now, in the above, you'll notice that the "chargeAccount" method is missing it's implementation details, it's infrastructure if you will.
So let's make a class that solves that problem and inject it in.

```js
var StripeApi = ClassFactory.make({
  charge: function(amount, currency, token){
    // make the api call to stripe
  }
});

//Here's now we tell our class about the dependencies it needs.

var StripeService = ClassFactory.make({
 
  _implements: [PaymentService],
  
  _dependencies: {
    stripeApi: StripeApi
  },
  
  chargeAccount: function(paymentDetails) {
    this.stripeApi.charge(paymentDetails.amount, paymentDetails.currency, paymentDetails.token);  
  }
});
```

"_dependencies" is a key value store for the dependencies. 
These dependencies are automatically created and associated when the class is made by the IOC, so we don't have to worry about it, it will just be there when we inject the class.

###Dependency trees

Dependency injection also works with interfaces, it will automatically create a concrete instance of that interface. This allows you to instantiate complex dependency trees, with minimal code and messiness.

###Extending classes

If you'd like to extend an existing class, you can do it like this.

```js
var LoggedStripeService = ClassFactory.make({

  _extends: StripeService,

  chargeAccount: function(paymentDetails) {
    console.log(paymentDetails);
    LoggedStripeService.parent.chargeAccount.call(this, paymentDetails);
  }
})
```

This makes it easy to add functionality to a class.

###Extending interfaces

You can also extend interfaces if you'd like

```js
var ReversablePaymentService = InterfaceFactory.make({
  
  _extends: [PaymentService],
  
  reverseChange: function(paymentDetails) {}
});
```

This makes combining interfaces a lot easier. It checks the method signatures as well, so if one interface overwrites another with a different method signature, it will throw an exception.

###Production and Development Environments

This library works by [duck-typing](https://en.wikipedia.org/wiki/Duck_typing), it checks the functions defined in the interface and matches them against the class. It does this whenever you define a class or bind a class to an interface. Now, this can cause a little slowdown, not much, but for some that's a deal breaker. That's why you can turn off the interface checking, like so.

```js
//Throw errors if interface methods are missing
ioc.enableInterfaceChecking();

//Ignore the checks altogether and just assume it's all ok
ioc.disableInterfaceChecking();
```

In development you can turn the method checking on, to make debugging easier (it's on by default). Then in production you turn it off to gain a little more speed.

###Constructors

The ClassFactory is using [stape.js](https://hay.github.io/stapes/) behind the scenes, so anything you can do with that, you can do with the ClassFactory. This includes creating constructors and all the other lovely stuff it adds. I've disabled the event aspects, as it's not core to this library, but if you really want it, just let me know.