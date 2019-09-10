
Creating testable interfaces
======

Most projects, at some point requires to have unit tests with line coverage - and I encourage to have it from the beginning! But in most cases, this kind of requirements comes when project is in the last developing phases, therefore either whole code is already written or most of it is.

That situation is highly problematic especially in cases where we need to hit high coverage percentage. You would most likely find yourself creating tons of mocks, which would be massively hard to inject. Yet creating interfaces and classes which are easy to be tested by injecting mocked types might be easy.

For the need of this tutorial let's consider following, easy implementation as an example:

```cpp
struct SQL
{
    //Dummy SQL
    void foo(){ std::cout << "foo\n"; };
    void bar(){ std::cout << "bar\n"; };
};

class Implementation
{
public:
    Implementation(); //not default just to show the use case
    ~Implementation(); //not default just do show the use case
    
    void foo();
private:
    void bar();
    SQL mSQL;
};

Implementation::Implementation()
{
  std::cout << "constructor\n";
}

Implementation::~Implementation()
{
  std::cout << "deconstructor\n";
}

void Implementation::foo()
{
    mSQL.foo();
    bar();
}

void Implementation::bar()
{
    mSQL.bar();
}

int main()
{
    Implementation impl;
    impl.foo();
}
```
([live code - coliru](https://coliru.stacked-crooked.com/a/02736a02eb7e5848))

Above example shows a simple example of class accessing SQL database via SQL interface. Now let's consider that we need to provide unit tests for given `Implementation` class.

We need to take care of given SQL object, since unit tests should only test internal interface (itself) not anything else.
Excluding SQL from `Implementation` can be started from creating `virtual interface` which will represent the one from SQL:

```cpp
//Interface for SQL
struct ISQL
{
    virtual void foo(){};
    virtual void bar(){};
};

struct SQL : ISQL
{
    //Dummy SQL
    void foo() override { std::cout << "foo\n"; };
    void bar() override { std::cout << "bar\n"; };
};
```
([live code - coliru](https://coliru.stacked-crooked.com/a/efff3eb108878eda))

Afterwards we can just create a mock type, derived from `ISQL` interface, and then inject it into `Implementation` class:
```cpp
...
struct MockedSQL : ISQL
{
    void foo() override { std::cout << "MockedFoo\n"; };
    void bar() override { std::cout << "MockedBar\n"; };
};

...

class Implementation
{
public:
    Implementation(std::shared_ptr<ISQL> dateBase); //not default just to show the use case
    ~Implementation(); //not default just do show the use case
    
    void foo();
private:
    void bar();
    std::shared_ptr<ISQL> mSQL;
};

Implementation::Implementation(std::shared_ptr<ISQL> dateBase) : mSQL(dateBase)
{
  std::cout << "constructor\n";
}
```

Then usage of mocked and regular version:
```cpp
int main()
{
    Implementation impl(std::make_shared<SQL>());
    impl.foo();
    
    Implementation implInjected(std::make_shared<MockedSQL>());
    implInjected.foo();
}
```
([live code - coliru](https://coliru.stacked-crooked.com/a/db43778f88c73c6c))

Right now we can have sort of testable code, but most likely it would not work. Why? Let's highlight issues:
- In most cases SQL interface is 3rd party library and we are not able to change it
- Even if we would take a different approach and derive from SQL so we would not have to change it's inside. It still most likely would not work, because of private methods.
- This approach brings many changes to `Implementation` API, and it might not be possible to apply
- Even if all above would not be important, then using runtime polymorphism might not be the best scenario because of runtime overhead.

There is easier way of injecting mocked data into our classes, all we need is simple templates.

First we need to make our `Implementation` class, a template class with default parameter.

```cpp
template<class DateBase = SQL>
class Implementation
```
Creating `Implementation` as template gives us a chance to provide any type as DateBase managing one. But we would not like to break any existing code therefore we are making DateBase as SQL object if no type is provided - default type.

Next step is to use DateBase type instead of SQL within object definition. Remember, as we said before in regular use case DateBase would be just an alias to SQL.
```cpp
template<class DateBase = SQL>
class Implementation
{
public:
    Implementation(); //not default just to show the use case
    ~Implementation(); //not default just do show the use case
    
    void foo();
private:
    void bar();
    DateBase mSQL;
};
```
Those are all needed changes in header section (definition). In case where we have additional cpp file, as we are mimicing it here, we need to do the following.

```cpp
template<class DateBase>
Implementation<DateBase>::Implementation()
{
  std::cout << "constructor\n";
}

template<class DateBase>
Implementation<DateBase>::~Implementation()
{
  std::cout << "deconstructor\n";
}

template<class DateBase>
void Implementation<DateBase>::foo()
{
    mSQL.foo();
    bar();
}

template<class DateBase>
void Implementation<DateBase>::bar()
{
    mSQL.bar();
}
```
Each time we are referring to implementation section in cpp files, we would need to refer to right type. Before making our `Implementation` as template it would be sufficient to refer as `Implementation::` and method name. After we have made it a template type then we need to provide template argument. Since we are not aware at a time what the parameter will be, therefore we are using the same syntax as for defining `Implementation` which is `template<class DateBase>` and adding template argument as well `Implementation<DateBase>`.

After we have our class defined, it's time to use it. Here is how we can do it:
```cpp
int main()
{
    Implementation<> impl;
    impl.foo();
}
```
As you can see only addition thing is to add empty angle brackets to `Implementation` type. This is needed because we need to show the compiler that `Implementation` is a template, but we do not need to provide any type since we have made it as default.

This was true for code which was dedicated for pre-c++17 standard, if you are able to use c++17 and beyond then it's even easier, empty brackets can be removed.

```cpp
int main()
{
    Implementation impl; //C++17 and beyond
    impl.foo();
}
```

Ok so now we have the same situation as we had before the need of unit testing, lets inject our MockedSQL, and use it instead:

```cpp
struct MockedSQL
{
    void foo() { std::cout << "MockedFoo\n"; };
    void bar() { std::cout << "MockedBar\n"; };
};
...

int main()
{
    Implementation<> impl;
    impl.foo();
    
    Implementation<MockedSQL> implInjected;
    implInjected.foo();
}
```

MockedSQL is no needed to be derived from ISQL anymore, since template can be any type which fulfils SQL interface, so we would not have compilation issues. Just a note, it's wise to provide Concepts like checks, to ensure that DateBase is meeting the requirements of DateBase interface.

All we need to do is just provide Injected type as a template parameter `Implementation<MockedSQL>` and that's it.
See [full code in Coliru](https://coliru.stacked-crooked.com/a/b5be9e5193fa9c01)


Hubert Liberacki
