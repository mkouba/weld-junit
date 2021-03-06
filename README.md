# Weld JUnit Extensions

[![Travis CI Build Status](https://img.shields.io/travis/weld/weld-junit/master.svg)](https://travis-ci.org/weld/weld-junit)
[![Maven Central](http://img.shields.io/maven-central/v/org.jboss.weld/weld-junit4.svg)](http://search.maven.org/#search%7Cga%7C1%7Ca%3A%22weld-junit4%22)
[![License](https://img.shields.io/badge/license-Apache%20License%202.0-yellow.svg)](http://www.apache.org/licenses/LICENSE-2.0.html)


## JUnit 4

```xml
<dependency>
  <groupId>org.jboss.weld</groupId>
  <artifactId>weld-junit4</artifactId>
  <version>${version.weld-junit}</version>
</dependency>
```

### WeldInitiator

`org.jboss.weld.junit.WeldInitiator` is a `TestRule` (JUnit 4.9+) which allows to start a Weld container per test method execution.
The container is configured through a provided `org.jboss.weld.environment.se.Weld` instance - see also `WeldInitiator.of(Weld)` static method.

A convenient static method `WeldInitiator.of(Class<?>...)` is also provided - in this case, the container is optimized for testing purposes (with automatic discovery and concurrent deployment disabled) and only the given bean classes are considered.
`WeldInitiator` also implements `javax.enterprise.inject.Instance` and therefore might be used to perform programmatic lookup of bean instances.

```java
import static org.junit.Assert.assertEquals;

import org.junit.Rule;
import org.junit.Test;

public class SimpleTest {

    @Rule
    public WeldInitiator weld = WeldInitiator.of(Foo.class);

    @Test
    public void testFoo() {
        // Note that Weld container is started automatically

        // WeldInitiator can be used to perform programmatic lookup of beans
        assertEquals("baz", weld.select(Foo.class).get().getBaz());

        // WeldInitiator can be used to fire a CDI event
        weld.event().select(Baz.class).fire(new Baz());
    }

}
```

It's also possible to use the convenient static method `WeldInitiator.ofTestPackage()` - the container is optimized for testing purposes and all the classes from the test class package are added.

```java

public class AnotherSimpleTest {

    @Rule
    public WeldInitiator weld = WeldInitiator.ofTestPackage();

    @Test
    public void testFoo() {
        // Alpha comes from the same package as AnotherSimpleTest
        assertEquals(1, weld.select(Alpha.class).ping());
    }

}
```

`WeldInitiator.Builder` can be used to customize the final `WeldInitiator` instance, e.g. to activate a context for a given normal scope or to inject the test class.

#### Test class injection

Sometimes, the programmatic lookup can imply unnecessary overhead, e.g. an annotation literal must be used for parameterized types and qualifiers with members.
`WeldInitiator.Builder.inject(Object)` instructs the rule to inject the given non-contextual instance once the container is started, i.e. during each test method execution:

```java
public class InjectTest {

    @Rule
    public WeldInitiator weld = WeldInitiator.from(Foo.class).inject(this).build();

  // Gets injected by WeldInitiator when testFoo() is about to be run
    @Inject
    @MyQualifier
    Foo foo;

    @Test
    public void testFoo() {
        assertEquals(42, foo.getValue());
    }

}
```

#### Activate a context for a normal scope

`WeldInitiator.Builder.activate(Object)` makes it possible to activate and deactivate contexts for the given normal scopes for each test method execution:

```java
public class ContextsActivatedTest {

    @Rule
    public WeldInitiator weld = WeldInitiator.from(Foo.class, Oof.class)
            .activate(RequestScoped.class, SessionScoped.class).build();

    @Test
    public void testFoo() {
        // Contexts for @RequestScoped and @SessionScoped are active!
        weld.select(Foo.class).get().doSomethingImportant();
        weld.select(Oof.class).get().doSomethingVeryImportant();
    }
}
```