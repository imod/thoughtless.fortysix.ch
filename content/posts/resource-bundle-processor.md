---
title: "the problem with java resource bundles"
date: 2021-03-04T17:45:24+01:00
draft: false
---

If you ever worked with java resource bundles (`PropertyResourceBundle`),
you probably also came a cross the situation where you needed to replace some tokens within the message to get the final text.

<!--more-->

e.g.

```text
welcome.message = Hello {0} {1}
goodby.message = By {0}
```

```java
Locale locale = ...
ResourceBundle bundle = ResourceBundle.getBundle("mypackage.ExampleResource", locale);

String messagePattern = bundle.getString("welcome.message");
String message = MessageFormat.format(messagePattern, "John", "Doe");

assertEquals("Hello John Doe", messages);
```

The API is easy and does not need a lot of explanation.
But there are a couple of things that can go wrong or are anoing:

- use of the wrong key to get the message out of the bundle.
- message keys distributed all over the code base.
- pass the wrong number of parameters to the `MessageFormat`.
- loading the resource bundle over and over again.

## ease handling of java resource bundles

Lets see how we can improve some of the points above..

- use of the wrong key to get the message out of the bundle.

Sounds easy enough to be solved: lets introduce an class or enum holding all the awailable keys.

```java
public enum MessageKey {

 WELCOME_MESSAGE("welcome.message"),
 GOODBY_MESSAGE("goodby.message");

 private final String key;

 private MessageKey(String key) {
  this.key = key;
 }

 public String getKey() {
  return key;
 }
}
```

- message keys distributed all over the code base.

If we take the enum holding all the key definitions a step further,
we could add a method for each key and resolve the message right from within this method. e.g.

```java
 public static String getMessage(Locale locale, MessageKey key) {
  ResourceBundle bundle = ResourceBundle.getBundle("mypackage.ExampleResource", locale != null ? locale : Locale.ENGLISH);
  try {
   return bundle.getString(key.getKey());
  } catch (MissingResourceException e) {
   return "[" + key + "]";
  }
 }
```

- pass the wrong number of parameters to the `MessageFormat`.

This could be solved by adding a method for each of the keys and take the exact number of arguments we need to pass to the `MessageFormat`:

```java
 public static String getWelcomeMessage(Locale locale, String firstName, String lastName) {
  String messagePattern = getMessage(locale, WELCOME_MESSAGE);
  return MessageFormat.format(messagePattern, firstName, lastName);
 }
```

Sure we could go on with this and add a new method every time we add a new message to the bundle.
But what if the messages changes?
How do you ensure the method gets updated correctly? Every time?
We just add unit test you might say,
nothing about this,
but what if you could ensure the resource bundle is automatically reflected in your utility class?

This is exaclty what Resource Bundle Processor does:
It reads your resource bundle and generates a class that reflects all the messages from your bundle.

## resource bundle processor

The [Resource Bundle Processor](https://github.com/imod/resource-bundle-gen) is a small Java Annotation Processor (JSR-269) to generate a utility class to simplify the handling of resource bundles.

With the genearated utility class (e.g. `ApplicationBundle`), the example from above would result in the following code:

```java
ApplicationBundle bundle = new ApplicationBundle();

Locale locale = ...
String message = bundle.welcomeMessage(locale, "John", "Doe");

assertEquals("Hello John Doe", message);
```

The generated class will have a method for each resource bundle key taking a locale and if the message contains placeholders (e.g. `Hello {0} {1}`),
it also takes the exact same number of String arguments.

## how to

Annotate a class with `@ResourceBundle` and tell it which bundle it should act on.
e.g.:

```java
@ResourceBundle(bundle = "mypackage.ExampleResource")
public class Application {
}
```

This will generate the following method in the new utility class:

```java
   public String welcomeMessage(Locale locale, String arg1, String arg2) {
      String messagePattern = holder.getBundle(locale).getString("welcome.message");
      return MessageFormat.format(messagePattern, arg1, arg2);
   }
```

Unfortunatly this method still has a downside:
the order of arguments matters!
If the message changes from `Hello {0} {1}` to `Hello {1} {0}`,
we get an unexpected result.

To overcome this problem it also supports 'names arguments':

```text
welcome.message = Hello {firstName} {lastName}
```

```java
@ResourceBundle(bundle = "mypackage.ExampleResource", type = COMMONS_TEXT)
public class Application {
}
```

This will result in the following generated method und resolve the order issue:

```java
    public String welcomeMessage(Locale locale, String firstName, String lastName) {
        String messagePattern = holder.getBundle(locale).getString("welcome.message");
        Map<String, String> values = new HashMap<>();
        values.put("firstName", firstName);
        values.put("lastName", lastName);
        StringSubstitutor sub = new StringSubstitutor(values, "{", "}");
        return sub.replace(messagePattern);
    }
```

### configuration

More details about the configuration options can be found on the project documentation site: [resource-bundle-processor](https://github.com/imod/resource-bundle-gen)

## limitations

- The current version only supports resource bundles in the form of properties files (no XML!).
- A annotation processor is only triggered in case a java file changes, but the resource bundles actually are `properties`-files, therefore a compilation of the java class annotated with `@ResourceBundle` might need to be triggered in case the bundle file changes. e.g. `mvn clean verify`
- The current version requires the default bundle to include all the messages,
  all methods will be generated out of the entries of the default bundle.
