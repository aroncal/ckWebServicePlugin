# ckWebServicePlugin (for symfony 1.2)

The ckWebServicePlugin allows you to build a webservice api for your symfony applications.
It comes with a powerful and easy to use wsdl generator, which creates WS-I compliant wsdl files for a maximum of interopability with PHP, .NET and Java clients.

# Requirements

*   PHP >= 5.2.4
*   php_soap extension

# Installation

To install the latest release, execute:

    > symfony plugin:install ckWebServicePlugin

or to install the current revision, checkout the HEAD revision into a `plugins/ckWebServicePlugin` folder:

    > svn co http://svn.symfony-project.com/plugins/ckWebServicePlugin/branches/1.2/

>**CAUTION**
>The HEAD revision of the branch is not guaranteed to be stable all the time!

Now configure the plugin how it is described in the next section and clear your cache afterwards.

# Configuration

The configuration can be devided into two parts. A `basic` one, which is mandatory and has to be done in order to get the plugin working.
The second, `advanced`, part is only required under certain circumstances and if you want to leverage the full power of the plugin.
So if you are using this plugin the first time you can skip the `Advanced` section.

## Basic

### app.yml

Configure general plugin settings in your application's `app.yml` file.

```yaml
all:
  # because by default every filter condition is true, we have to set this var
  # to off in all other environments
  enable_soap_parameter: off

# your environment for webservice mode
soap:
  # enable the `ckSoapParameterFilter`
  enable_soap_parameter: on
  ck_web_service_plugin:
    # the location of your wsdl file
    wsdl: %SF_WEB_DIR%/myWebService.wsdl
    # the class that will be registered as handler for webservice requests
    handler: ckSoapHandler
```

You will propably have to change the `wsdl` and `handler` option after you have run the `webservice:generate-wsdl` task.

### factories.yml

Enable the ``ckWebServiceController`` in your application's `factories.yml` file.

```yaml
# your environment for webservice mode
soap:
  controller:
    class: ckWebServiceController
```

### filters.yml

Enable the `ckSoapParameterFilter` in your application's `filters.yml` file.

```yaml
soap_parameter:
  class: ckSoapParameterFilter
  param:
    # `app_enable_soap_parameter` has to be set to `on` so the filter is only enabled in soap mode
    condition: %APP_ENABLE_SOAP_PARAMETER%
```

## Advanced

### app.yml

In your application's `app.yml` file you have some more options to configure the internally used SoapServer.

These are:

*   setting the persistence mode:

    ```php
    # your environment for webservice mode
    soap:
      # ...
      ck_web_service_plugin:
        # ...
        persist: <?php echoln(SOAP_PERSISTENCE_SESSION) ?>
    ```
    
    For further information see documentation on `SoapServer::setPersistence()`.

*   setting the `$options` array used by `SoapServer::__construct()`:

    ```php
    # your environment for webservice mode
    soap:
      # ...
      ck_web_service_plugin:
        # ...
        soap_options:
          encoding: utf-8
          soap_version: <?php echoln(SOAP_1_2) ?>
    ```
    
    For further information see documentation on `SoapServer::__construct()`.

*   configuring SOAP Headers:

    ```yaml
    # your environment for webservice mode
    soap:
      # ...
      ck_web_service_plugin:
        # ...
        soap_headers:
          # the name of the soap header
          MySoapHeader:
            # the corresponding data class
            class: MySoapHeaderDataClass
    ```
    
    For more details about the usage of SOAP Headers read the section `Using SOAP Headers`.

### module.yml

Every action, which should be callable in webservice mode, needs some configuration so the parameters are accessable through `sfRequest::getParameter()` and the proper value is returned as result.
This configuration is automaticly done by the `webservice:generate-wsdl` task, if you don't use the task or want to customize something you have to change the `module.yml` file corresponding to the action.

An example `module.yml` file:

```yaml
# your environment for webservice mode
soap:
  # the action name
  action_name:
    # ordered list of the parameters
    parameter: [first_param, second_param]
    # the result adapter
    result:
      # the result adapter class, extending `ckAbstractResultAdapter`
      class: ckPropertyResultAdapter
      # result adapter specific parameters array
      param:
        property: result
```

The result adapters will be explained in more detail in the section `Understanding result adapters`.

# Using the `webservice:generate-wsdl` task

Now it is time to start making our actions available as a webservice.

This is best explained with an example, we will use the following action, which will multiply two numbers and is in an application named `frontend`:

```php
    <?php

    // apps/frontend/modules/math/actions/actions.class.php
    class mathActions extends sfActions
    {
      /**
       * An action multiplying two numbers.
       *
       * @param sfRequest $request A sfRequest instance
       */
      public function executeMultiply($request)
      {
        $factorA = $request->getParameter('a');
        $factorB = $request->getParameter('b');

        if(is_numeric($factorA) && is_numeric($factorB))
        {
          $this->result = $factorA * $factorB;

          return sfView::SUCCESS;
        }
        else
        {
          return sfView::ERROR;
        }
      }
    }
```

The only thing we will have to do is updating the doc comment:

```php
    <?php

    // apps/frontend/modules/math/actions/actions.class.php
    class mathActions extends sfActions
    {
      /**
       * An action multiplying two numbers.
       *
       * @WSMethod(webservice='MathApi')
       *
       * @param double $a Factor A
       * @param double $b Factor B
       *
       * @return double The result
       */
      public function executeMultiply($request)
      {
        // nothing changed here...
      }
    }
```

Changes:

*   a `@WSMethod` annotation was added to mark the action as available in the `MathApi` webservice,
*   for every parameter accessible through `sfRequest::getParameter()` a `@param` doc tag with type, name and description was added,
*   a `@return` doc tag with type and description was added,
*   the `@param` doc tag for `$request` was removed, because it is not a real parameter required by the action `multiply`.

Now we are ready to execute the `webservice:generate-wsdl` task. It is explained in detail in the section `The `webservice:generate-wsdl` task in detail`.

Execute the task with:

    > symfony webservice:generate-wsdl frontend MathApi http://localhost/

>**TIP**
>Change `http://localhost/` to the url you are using for development!

The task will generate a `MathApi.wsdl` and `MathApi.php` in your project's `web/` folder.
Further the task will generate a `MathApiHandler.class.php` and a `BaseMathApiHandler.class.php` in the application's `lib/` folder.

We have to change the `wsdl` option in the application's `app.yml` file to `MathApi.wsdl` and the `handler` option to `MathApiHandler`:

```yaml
    // apps/frontend/config/app.yml
    # your environment for webservice mode
    soap:
      # ...
      ck_web_service_plugin:
        # the location of your wsdl file, relative to your project's `web/` folder
        wsdl: %SF_WEB_DIR%/MathApi.wsdl
        # the class which will be registered as handler for webservice requests
        handler: MathApiHandler
```

and we have to clear the cache.

Now it is time to create a test script to ensure everything is working properly. Please refer to
 the section `Functional Testing` to see how to setup the test environment.

The script will be named `mathApiTest.php` and placed under the project's `test/functional/` folder. It should look the following way:

```php
    <?php

    // test/functional/mathApiTest.php
    $app   = 'frontend';
    $debug = true;

    include_once(dirname(__FILE__).'/../bootstrap/soaptest.php');

    $c = new ckTestSoapClient();

    // test executeMultiply
    $c->math_multiply(5, 2)    // call the action
      ->isFaultEmpty()         // check there are no errors
      ->isType('', 'double')   // check the result type is double
      ->is('', 10);            // check the result value is 10
```

You see the name of the webservice method follows the scheme `<moduleName>_<actionName>`, because this might be not descriptive enough or an alternative scheme is desired,
 we will see how to change the method name. To do this we have to change again the action's doc comment:

```
    <?php

    // apps/frontend/modules/math/actions/actions.class.php
    class mathActions extends sfActions
    {
      /**
       * An action multiplying two numbers.
       *
       * @WSMethod(name='SimpleMultiply', webservice='MathApi')
       *
       * @param double $a Factor A
       * @param double $b Factor B
       *
       * @return double The result
       */
      public function executeMultiply($request)
      {
        // nothing changed here...
      }
    }
```

Changes:

*   the parameter `name` was added to the `@WSMethod` annotation to specify the method name.

Now we have to regenerate the wsdl, execute:

    > symfony webservice:generate-wsdl frontend MathApi http://localhost/

Finally our test script has to be updated:

```php
    <?php

    // test/functional/mathApiTest.php
    $app   = 'frontend';
    $debug = true;

    include_once(dirname(__FILE__).'/../bootstrap/soaptest.php');

    $c = new ckTestSoapClient();

    // test executeMultiply
    $c->SimpleMultiply(5, 2)
      ->isFaultEmpty()
      ->isType('', 'double')
      ->is('', 10);
```

You now have a basic overview how to use the plugin, the following sections will explain more advanced features.

# The `webservice:generate-wsdl` task in detail

Its general syntax is:

    > symfony webservice:generate-wsdl [--environment=soap] [--enabledebug] app_name webservice_name webservice_base_url

It will do the following things:

*   look through all modules of `'app_name'` for actions with the `@WSMethod` annotation,
*   add the marked actions to the wsdl definition, if:
    *   the `@WSMethod` annotation's `webservice` parameter equals the `'webservice_name'` argument or
    *   the `'environment'` option has its default value `soap` and the `@WSMethod` annotation's `webservice` parameter is missing,
*   save the wsdl definition to your project's `web/` folder as `'webservice_name'.wsdl`,
*   create a new controller in your project's `web/` folder with name `'webservice_name'.php`,
*   add a `'webservice_name'Handler.class.php` and a `Base'webservice_name'Handler.class.php` to the `'app_name'`'s `lib/` folder.

The arguments explained in detail:

*   `app_name`:
    *   specifies the application, which is searched for actions marked with the `@WSMethod` annotation
*   `webservice_name`:
    *   specifies the name of the webservice
*   `webservice_base_url`:
    *   specifies the url under which the webservice will be accessible

The options explained in detail:

*   `environment` (short `e`):
    *   sets the environment for webservice mode
    *   defaults to `soap`
    *   if you change it, don't forget to change the configuration files accordingly
*   `enabledebug` (short `d`):
    *   enables the debug mode in the generated controller
    *   defaults to `false`

# Understanding result adapters

Until now it wasn't explained how the result of an action is got, we have just seen, that the result was assigned to the `$this->result` property and a `sfView` constant was returned, like `sfView::SUCCESS`.

Because an action should be reusable in web and webservice mode, we can't rely on the return value, because in web mode it always has to be a template name.
For this reason the result adpater pattern was introduced. This means to get the action result an adapter object of a subclass of `ckAbstractResultAdapter` is used.
Which one is used is determined by the configuration in the action's `module.yml` file how it is shown in the `Configuration`->`Advanced`->`module.yml` section.
The `param` array in the `module.yml` file is passed to the result adapter's constructor and contains adapter specific settings.

There are three built-in adapters:

*   `ckPropertyResultAdapter`:
    *   gets the result from a property of the action object
    *   parameters:
        *   `property`:
            *   specifies the property name
            *   defaults to `result`
    *   if there is only one property, this one is returned, also its name doesn't match the specified `property`
*   `ckMethodResultAdapter`:
    *   gets the result from a method call on the action object
    *   parameters:
        *   `method`:
            *   specifies the method name
*   `ckRenderResultAdapter`:
    *   executes the standard render pipeline and returns the resulting text
    *   the `sf_format` is set to `soap` so template file names have to end with `.soap.php`, e.g.: `indexSuccess.soap.php`
    *   if this adapter is used the return value has to be `string`
    *   parameters:
        *   none

You can easily implement your own adapters by extending the `ckAbstractResultAdapter` class and overriding the abstract `ckAbstractResultAdapter::getResult()` method.

# Using arrays and objects as parameters or result values

In the previous examples only simple types have been used for parameters and result values, but you propably want to use objects, arrays of simple types or arrays of complex types.
To illustrate these features we will stick to the example used earlier.

Let's say we want to multiply any number of factors, not only two:

```php
    <?php

    // apps/frontend/modules/math/actions/actions.class.php
    class mathActions extends sfActions
    {
      /**
       * An action multiplying any number of factors.
       *
       * @WSMethod(name='SimpleMultiply', webservice='MathApi')
       *
       * @param double[] $factors An array of factors
       *
       * @return double The result
       */
      public function executeMultiply($request)
      {
        $this->result = 1;

        foreach($request->getParameter('factors') as $factor)
        {
          $this->result *= $factor;
        }
      }
    }
```

Changes:

*   the `@param` doc tags for factor `$a` and `$b` have been replaced with one `@param` doc tag for the factors array,
*   the action body changed to iterate over the array of factors.

As you can see the array type is indicated by the `[]`, you can add the square brackets to any type to identify an array of the type should be used.

Because array types are complex data types, we have to add a mapping to the application's `app.yml` file:

```yaml
    // apps/frontend/config/app.yml
    soap:
      # ...
      ck_web_service_plugin:
        # ...
        soap_options:
          classmap:
            # mapping of wsdl types to PHP types
            DoubleArray: ckGenericArray
```

>**TIP**
>The generated array type names follow the scheme `<TypeName>Array`.
>
>Use `ckGenericArray` as PHP mapping type for any array type you use, so you can use the transferred array object like a normal PHP array (iterate, index, ...).

The last thing to do is: regenerate the wsdl file with the `webservice:generate-wsdl` task and clear the cache.

Our test script might look like this now:

```php
    <?php

    // test/functional/mathApiTest.php
    $app   = 'frontend';
    $debug = true;

    include_once(dirname(__FILE__).'/../bootstrap/soaptest.php');

    $c = new ckTestSoapClient();

    // test executeMultiply
    $c->SimpleMultiply(array(1, 2, 3, 4))
      ->isFaultEmpty()
      ->isType('', 'double')
      ->is('', 24);
```

As example for the use of classes, we will implement the multiplication example for complex numbers.

Because complex numbers aren't nativly supported in PHP, we have to create our own `ComplexNumber.class.php` in the applications `lib/` folder with the following content:

```php
    <?php

    // apps/frontend/lib/ComplexNumber.class.php
    class ComplexNumber
    {
      /**
       * @var double
       */
      public $realPart;

      /**
       * @var double
       */
      public $imaginaryPart;

      public function __construct($realPart, $imaginaryPart)
      {
        $this->realPart      = $realPart;
        $this->imaginaryPart = $imaginaryPart;
      }

      public function __toString()
      {
        return sprintf('%.2f + %.2fi', $this->realPart, $this->imaginaryPart);
      }

      public function multiply($c)
      {
        $real      = $this->realPart * $c->realPart - $this->imaginaryPart * $c->imaginaryPart;
        $imaginary = $this->realPart * $c->imaginaryPart - $this->imaginaryPart * $c->realPart;
        return new ComplexNumber($real, $imaginary);
      }
    }
```

It is important to add the `@var` doc tag with the type and an optional desciption to the properties of the class, so they will appear in the wsdl file.

Now let's modify the `mathActions` class by adding a new action, called `ComplexMultiply`:

```php
    <?php

    // apps/frontend/modules/math/actions/actions.class.php
    class mathActions extends sfActions
    {
      // nothing changed here...

      /**
       * An action multiplying any number of complex factors.
       *
       * @WSMethod(name='ComplexMultiply', webservice='MathApi')
       *
       * @param ComplexNumber[] $input
       *
       * @return ComplexNumber
       */
      public function executeComplexMultiply($request)
      {
        $this->result = new ComplexNumber(1, 0);

        foreach($request->getParameter('input') as $c)
        {
          $this->result = $this->result->multiply($c);
        }
      }
    }
```

Again we have to update the `classmap` in the application's `app.yml` file:

```yaml
    // apps/frontend/config/app.yml
    soap:
      # ...
      ck_web_service_plugin:
        # ...
        soap_options:
          classmap:
            # mapping of wsdl types to PHP types
            DoubleArray:        ckGenericArray
            ComplexNumber:      ComplexNumber
            ComplexNumberArray: ckGenericArray
```

Finally regenerate the wsdl file once more and clear the cache.

Our updated test script will look something like this:

```php
    <?php

    // test/functional/mathApiTest.php
    $app   = 'frontend';
    $debug = true;

    include_once(dirname(__FILE__).'/../bootstrap/soaptest.php');

    class ClientComplexNumber
    {
      public $realPart;

      public $imaginaryPart;

      public function __construct($realPart, $imaginaryPart)
      {
        $this->realPart      = $realPart;
        $this->imaginaryPart = $imaginaryPart;
      }
    }

    $options = array(
      'classmap' => array(
        'ComplexNumber' => 'ClientComplexNumber',
      ),
    );

    $c = new ckTestSoapClient($options);

    // test executeMultiply
    // ...

    // test executeComplexMultiply
    $cn = new ClientComplexNumber(1, 0);

    $c->ComplexMultiply(array(clone $cn, clone $cn))
      ->isFaultEmpty()
      ->isType('', 'ClientComplexNumber')
      ->isType('realPart', 'double')
      ->is('realPart', 1)
      ->isType('imaginaryPart', 'double')
      ->is('imaginaryPart', 0);
```

As you see, we have added a lightweight definition of the `ComplexNumber` class called `ClientComplexNumber`, because it is likely that you don't have the same class definition at client and server, only the names and types of the properties will match.

Often the objects you want to return or pass in as parameter are not as simple as the shown `ComplexNumber`, e.g. Doctrine or Propel objects or objects which have a JavaBean-style class, so the properties are only accessible through getter and setter methods.

The plugin also offers a solution for this problem. Therefor it introduces so called property strategies, they have two purposes, the first is to determine which properties a class has when the wsdl is generated, the second purpose is to access those properties
at runtime.

There are already four property strategy implementations:

*   ckDefaultPropertyStrategy:
    *   provides access to all public properties
    *   each property needs to have a proper doc comment so its type can be determined
    *   it is used if there is no other property strategy specified (so it was already used in the background in the `ComplexNumber` example)
*   ckBeanPropertyStrategy:
    *   provides access to JavaBean-like properties with getter and setter methods
    *   each getter method needs to have proper doc comment so the property type can be determined
*   ckDoctrinePropertyStrategy:
    *   provides access to all properties you defined in your `schema.yml` for Doctrine objects
*   ckPropelPropertyStrategy:
    *   provides access to all properties you defined in your `schema.yml` for Propel objects

You can implement your own property strategy by extending `ckAbstractPropertyStrategy`.

To apply a property strategy to a class you have to add a `@PropertyStrategy` annotation to the class with the class name of the property strategy as parameter.

Here are two examples:

*   JavaBean-like class:

```php
        <?php

        // apps/frontend/lib/UserBean.class.php
        /**
         * @PropertyStrategy('ckBeanPropertyStrategy')
         */
        class UserBean
        {
          private $_name;

          /**
           * Gets the user name.
           *
           * @return string The user name
           */
          public function getName()
          {
            return $this->_name;
          }

          /**
           * Sets the user name to a given value.
           *
           * @param string $name A name
           */
          public function setName($name)
          {
            $this->_name = $name;
          }
        }
```

*   Doctrine class:

```php
        <?php

        // lib/model/doctrine/Article.class.php
        /**
         * @PropertyStrategy('ckDoctrinePropertyStrategy')
         */
        class Article extends BaseArticle
        {
        }
```

There is one important thing you have to note:

If you you want to use those classes as parameter for your methods, you have to map them in your `app.yml` to `ckGenericObjectAdapter_<classname>`.
So the `app.yml` for the `Article` and `UserBean` class shown above would look like:

```yaml
    // apps/frontend/config/app.yml
    soap:
      # ...
      ck_web_service_plugin:
        # ...
        soap_options:
          classmap:
            # mapping of wsdl types to PHP types
            UserBean: ckGenericObjectAdapter_UserBean
            Article:  ckGenericObjectAdapter_Article
```

Collections in Doctrine and Propel objects are represented as arrays in the wsdl, so suppose the `Article` class has many `Comment` objects and `Comment` is also a Doctrine class annotated with `@PropertyStrategy('ckDoctrinePropertyStrategy')`.
The `app.yml` would be:

```yaml
    // apps/frontend/config/app.yml
    soap:
      # ...
      ck_web_service_plugin:
        # ...
        soap_options:
          classmap:
            # mapping of wsdl types to PHP types
            UserBean:     ckGenericObjectAdapter_UserBean
            Article:      ckGenericObjectAdapter_Article
            Comment:      ckGenericObjectAdapter_Comment
            CommentArray: ckGenericArray
```

This is everything you have to do to use such complex classes!

>**TIP**
>Do not forget to validate objects you get as a parameter!

In this section you have learned how to work with arrays and classes, the next section covers the usage of SOAP Headers.

# Using SOAP Headers

SOAP Headers provide a way to send additional information, which are not directly or semantically related to the original method call.
An good example for this are authentication information, so the use of a certain method can be restricted to a group of users.

To demonstrate the support for SOAP Headers, we will stick to the simple multiplication example used previously.

>**CAUTION**
>The authentication mechanism used here is not secure unless you use https, it is just used for demonstration purpose and to keep the example simple!

First we will modify the `mathActions` class the following way:

```php
    <?php

    // apps/frontend/modules/math/actions/actions.class.php
    class mathActions extends sfActions
    {
      /**
       * An action multiplying two numbers.
       *
       * @WSMethod(name='SimpleMultiply', webservice='MathApi')
       * @WSHeader(name='AuthHeader', type='AuthData')
       *
       * @param double $a Factor A
       * @param double $b Factor B
       *
       * @return double The result
       */
      public function executeMultiply($request)
      {
        $factorA = $request->getParameter('a');
        $factorB = $request->getParameter('b');

        if($this->getUser()->isAuthenticated() && is_numeric($factorA) && is_numeric($factorB))
        {
          $this->result = $factorA * $factorB;

          return sfView::SUCCESS;
        }
        else
        {
          return sfView::ERROR;
        }
      }
    }
```

Changes:

*   a `@WSHeader` annotation was added, specifying the name (`AuthHeader`) of the SOAP Header and the data class (`AuthData`), which holds the data of the SOAP header,
*   an authentication check was added, so the multiplication is only done, if the user was authenticated successfully.

To get this example working we have to define the `AuthData` class, so let's create a `AuthData.class.php` file in the application's `lib` folder with the following content:

```php
    <?php

    // apps/frontend/lib/AuthData.class.php
    class AuthData
    {
      /**
       * @var string
       */
       public $username;

       /**
        * @var string
        */
       public $password;
    }
```

Afterwards we have to edit the application's `app.yml` file:

```yaml
    // apps/frontend/config/app.yml
    soap:
      # ...
      ck_web_service_plugin:
        # ...
        soap_headers:
          AuthHeader:
            class: AuthData
```

When the application receives a SOAP Header a `webservice.handle_header` event is dispatched (it is a `notifyUntil` event), it has two attributes, the first is `header` holding the name of the header and the second is `data` containing an instance of the header's data class.
To do the authentication stuff in our example we will define an `AuthHeaderListener` class by creating an `AuthHeaderListener.class.php` in the application's `lib/` folder with the following content:

```php
    <?php

    // apps/frontend/lib/AuthHeaderListener.class.php
    class AuthHeaderListener
    {
      const HEADER = 'AuthHeader';

      public static function handleAuthHeader($event)
      {
        if($event['header'] == self::HEADER)
        {
          if($event['data']->username == 'test' && $event['data']->password == 'secret')
          {
            sfContext::getInstance()->getUser()->setAuthenticated(true);
          }

          return true;
        }
        else
        {
          return false;
        }
      }
    }
```

We have to register this event listener in the application's configuration class (assuming the application's name is `frontend`, this would be `frontendConfiguration.class.php`).

The modified configuration class would look something like this:

```php
    <?php

    // apps/frontend/config/frontendConfiguration.class.php
    class frontendConfiguration extends sfApplicationConfiguration
    {
      public function configure()
      {
        $this->dispatcher->connect('webservice.handle_header', array('AuthHeaderListener', 'handleAuthHeader'));
      }
    }
```

The example is now ready to work, regenerate the wsdl file and clear the cache.

The last missing thing is the updated test script:

```php
    <?php

    // test/functional/mathApiTest.php
    $app   = 'frontend';
    $debug = true;

    include_once(dirname(__FILE__).'/../bootstrap/soaptest.php');

    class ClientComplexNumber
    {
      // ...
    }

    class ClientAuthData
    {
      public $username;
      public $password;

      public function __construct($username, $password)
      {
        $this->username = $username;
        $this->password = $password;
      }
    }

    $options = array(
      'classmap' => array(
        'ComplexNumber' => 'ClientComplexNumber',
        'AuthHeader'    => 'ClientAuthData',
      ),
    );

    $c = new ckTestSoapClient($options);

    // test executeMultiply
    $authData = new ClientAuthData('test', 'secret');

    $c->addRequestHeader('AuthHeaderElement', $authData)
      ->SimpleMultiply(5, 2)
      ->isFaultEmpty()
      ->isHeaderType('AuthHeaderElement', 'ClientAuthData')
      ->isHeader('AuthHeaderElement.username', 'test')
      ->isHeader('AuthHeaderElement.password', 'secret')
      ->isType('', 'double')
      ->is('', 10);

    // test executeComplexMultiply
    // ...
```

>**TIP**
>When adding or accessing a SOAP Header its name has to end with `Element`.

This section demonstrated the use of SOAP Headers, so now you have seen nearly all features this plugin has to offer.

# Throwing SOAP Faults

The equivalent to exceptions in SOAP are so called SOAP Faults. The plugin supports a simple translation of exceptions to SOAP Faults, but also allows you to throw
 your own SOAP Faults.

To demonstrate the feature we will extend the authentication example from the last section.

First to demonstrate what happens if an exception is thrown, we will modify the `multiply` action to throw an exception if the user is not authenticated.
The modified `mathActions` class will look like this:

```php
    <?php

    // apps/frontend/modules/math/actions/actions.class.php
    class mathActions extends sfActions
    {
      /**
       * An action multiplying two numbers.
       *
       * @WSMethod(name='SimpleMultiply', webservice='MathApi')
       * @WSHeader(name='AuthHeader', type='AuthData')
       *
       * @param double $a Factor A
       * @param double $b Factor B
       *
       * @return double The result
       */
      public function executeMultiply($request)
      {
        if(!$this->getUser()->isAuthenticated())
        {
          throw new sfSecurityException('Unauthenticated user!');
        }

        $factorA = $request->getParameter('a');
        $factorB = $request->getParameter('b');

        if(is_numeric($factorA) && is_numeric($factorB))
        {
          $this->result = $factorA * $factorB;

          return sfView::SUCCESS;
        }
        else
        {
          return sfView::ERROR;
        }
      }
    }
```

How the exception is translated to a SOAP Fault depends on the value of `sf_debug`. If debugging is disabled every exception will be
translated to a standard SOAP Fault with the message `'Internal Server Error'`, otherwise if debugging is enabled the message, the type
and the stack trace of the exception will be send to the client.

An example test script for the method with debugging enabled:

```php
    <?php

    // test/functional/mathApiTest.php
    $app   = 'frontend';
    $debug = true;

    include_once(dirname(__FILE__).'/../bootstrap/soaptest.php');

    $options = array(
      'classmap' => array(
      ),
    );

    $c = new ckTestSoapClient($options);
    $c->SimpleMultiply(2, 5)
      ->hasFault('Unauthenticated user!')
      ;
```

A test script for the same method but with debugging disabled:

```php
    <?php

    // test/functional/mathApiTest.php
    $app   = 'frontend';
    $debug = false;

    include_once(dirname(__FILE__).'/../bootstrap/soaptest.php');

    $options = array(
      'classmap' => array(
      ),
    );

    $c = new ckTestSoapClient($options);
    $c->SimpleMultiply(2, 5)
      ->hasFault('Internal Server Error')
      ;
```

The next example shows how to throw our own SOAP Fault if we are in webservice mode.

```php
    <?php

    // apps/frontend/modules/math/actions/actions.class.php
    class mathActions extends sfActions
    {
      /**
       * An action multiplying two numbers.
       *
       * @WSMethod(name='SimpleMultiply', webservice='MathApi')
       * @WSHeader(name='AuthHeader', type='AuthData')
       *
       * @param double $a Factor A
       * @param double $b Factor B
       *
       * @return double The result
       */
      public function executeMultiply($request)
      {
        if(!$this->getUser()->isAuthenticated())
        {
          $e = $this->isSoapRequest() ? new SoapFault('Server', 'Unauthenticated user!') : new sfSecurityException('Unauthenticated user!');
          throw $e;
        }

        $factorA = $request->getParameter('a');
        $factorB = $request->getParameter('b');

        if(is_numeric($factorA) && is_numeric($factorB))
        {
          $this->result = $factorA * $factorB;

          return sfView::SUCCESS;
        }
        else
        {
          return sfView::ERROR;
        }
      }
    }
```

# Functional Testing

The symfony framework promotes the paradigm of test driven development, so it is just natural that this plugin offers you
possibilities to test your webservices. The following two sections show you how to setup a test environment and how to use
`ckTestSoapClient` for testing.

## Test Environment Setup

The setup of a test environment is similar to the configuration described in the section `Configuration`, only the environment name
changes from `soap` to `soaptest`, though you can use any other name you like.

The changes to the configuration files are:

*   `app.yml`:

    Copy the configuration of the `soap` to the `soaptest` environment, e.g.:
    
    ```yaml
        # ...
        soaptest:
          enable_soap_parameter: on
          ck_web_service_plugin:
            wsdl: %SF_WEB_DIR%/myWebService.wsdl
            handler: ckSoapHandler
    ```

*   `factories.yml`:

    Add the following configuration:
    
    ```yaml
        # ...
        soaptest:
          storage:
            class: sfSessionTestStorage
            param:
              session_path: %SF_TEST_CACHE_DIR%/sessions
          controller:
            class: ckWebServiceController
    ```

*   `filters.yml`:

    Remains unchanged, because it is environment independent.

To finish the setup you have to create a bootstrap script for the `soaptest` environment in the project's `test/bootstrap/` folder.

It will be named `soaptest.php` and will have the following content:

```php
    <?php

    require_once dirname(__FILE__).'/../../config/ProjectConfiguration.class.php';
    $configuration = ProjectConfiguration::getApplicationConfiguration($app, isset($env) ? $env : 'soaptest', isset($debug) ? $debug : true);
    require_once($configuration->getSymfonyLibDir().'/vendor/lime/lime.php');

    sfContext::createInstance($configuration);

    // remove all cache
    sfToolkit::clearDirectory(sfConfig::get('sf_app_cache_dir'));
```

This is the same as the default `functional.php` script except the environment parameter of `ProjectConfiguration::getApplicationConfiguration()` can be changed with the `$env` variable and defaults to `soaptest`.

>**TIP**
>You have to run the `webservice:generate-wsdl` task always twice, once for the `soap` environment and once for the `soaptest` environment, do not forget to set the `--environment` switch to the proper value.

## Using `ckTestSoapClient`

The `ckTestSoapClient` class lets you dispatch webservice requests to your symfony application without the need of a webserver.
Additionally it offers several evaluation methods for the result of each request.

A good starting point for every test script is the following template:

```php
    <?php

    $app   = 'frontend';
    $debug = true;

    include_once(dirname(__FILE__).'/../bootstrap/soaptest.php');

    $options = array(
      'classmap' => array(
      ),
    );

    $c = new ckTestSoapClient($options);
```

Change the `$app` variable to the name of the application you want to test.

>**TIP**
>The `$options` array supplied to the constructor is the same as the one of PHP's SoapClient constructor.

Calling a SOAP Action is quite easy, just use it as it would be a method of the `ckTestSoapClient` object:

```php
<?php

$c->myMethod($param1, $param2);
```

The call does not directly return the result, instead it returns the `ckTestSoapClient` object, this offers you a so called fluent interface
 how it is often found in the symfony framework.

To get the actual result you have to call the `getResult()` method:

```php
<?php

$result = $c->myMethod($param1, $param2)
            ->getResult();
```

For evaluating the result, the `ckTestSoapClient` class offers three methods: `is()` checks the value, `isType()` checks the type and `isCount()` checks the element count,
 useful when the result is an array.

The first argument is always a child element selector, so you can easily access and check properties or array elements, the second argument is a value to check against.

A child element selector can either be empty so the result itself is accessed or arbitrary count of property names or array indexes separated by a `.` (dot).

Some examples for selectors:

*   `''` accesses the result,
*   `'name'` accesses the `name` property of the result object,
*   `'1.name'` accesses the `name` property of the second object in the result array,
*   `'cities.0.name'` accesses the `name` property of the first object in the `cities` array, which is a property of the result object.

Various examples for the use of the three methods `is()`, `isType()` and `isCount()` can be found in the test scripts given in this README.

You can also add SOAP Headers for the next request with the `addRequestHeader()` method, whichs first parameter is the header name and the second is the data object, e.g.:

```php
<?php

$c->addRequestHeader('MyHeaderElement', new MyHeaderData('content'))
  ->myMethodWithHeader();
```

The headers are cleared after each request, so do not forget to add them again if you need them more then once.

Similar to the evaluation methods for the result, there are three methods to evaluate the response headers. These are `isHeader()`, `isHeaderType()` and `isHeaderCount()`.
The parameter list is the same, but the child element selector has to contain at least the header name, e.g.:

```php
<?php

$c->addRequestHeader('MyHeaderElement', new MyHeaderData('content'))
  ->myMethodWithHeader()
  ->isHeader('MyHeaderElement.myContent', 'content');
```

The `ckTestSoapClient` has also methods to check the result for SoapFaults. One method is `isFaultEmpty()` it is usefull to check that the response contains no SoapFaults, e.g.:

```php
<?php

$c->myMethod()
  ->isFaultEmpty()
  ->is('', 1);
```

Another method is `hasFault()` it checks if a SoapFault with the given message exists.

```php
<?php

$c->myMethod()
  ->hasFault('Internal Server Error');
```

The method is at all quite similiar to the `throwsException()` method of `sfTestBrowser`.

So finally this section has shown you how to write functional tests for your webservices by using the `ckTestSoapClient` class.

# Reference

## Supported simple types

All primitive PHP types are supported:

*   `string` maps to `xsd:string`
*   `integer` or `int` maps to `xsd:int`
*   `float` or `double` maps to `xsd:double`
*   `boolean` or `bool` maps to `xsd:boolean`

# Tips'n Tricks

## Disable wsdl caching during development

If you often regenerate your wsdl file during development, you propably want to disable caching of this file, so changes become usable immediatly.

You can do this by modifying your `php.ini`:

```ini
soap.wsdl_cache_enabled=0
```

## Checking for webservice mode

If you want to check in an action if it is executed in webservice mode, you can use the `isSoapRequest()` method, e.g.:

```php
    <?php

    class FooActions extends sfActions
    {
      /**
       * Some description...
       *
       * @WSMethod(webservice='MyApi')
       */
      public function executeBar($request)
      {
        if($this->isSoapRequest())
        {
          // do this only in webservice mode...
        }

        // do this always...
      }
    }
```

## Create multiple webservices for one application

In earlier releases there could only exist one webservice per symfony application.
This has changed with the introduction of the `@WSMethod` annotation, you can now specify
to which webservice an action belongs with the `webservice` parameter of the annotation.
But it is important to notice that there has to be one environment per webservice!

## Adding an action to multiple webservices

It is possible to add an action to more then one webservice, because the `webservice` parameter of the
`@WSMethod` annotation accepts also an array of values. The action shown in the following example will be
available in the webservices `MyWebserviceA` and `MyWebserviceB`:

```php
    <?php

    class FooActions extends sfActions
    {
      /**
       * Some description...
       *
       * @WSMethod(webservice={'MyWebserviceA', 'MyWebserviceB'})
       */
      public function executeBar($request)
      {
        // ...
      }
    }
```

# Support

If you have any questions concerning the use of the plugin, send me an email to: christian-kerl [at] web [dot] de

# Contribution

If you have feature suggestions, bug reports, patches, usage examples for the documentation
 or want to become an active contributor, send me an email to: christian-kerl [at] web [dot] de

In case you use the Symfony Trac ticket system for bug reports assign the ticket to `chrisk` or send me an email with a link to the ticket!
This ensures I notice and work on the ticket in time.

Any help is welcome!
