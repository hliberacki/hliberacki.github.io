
Generic mocking for tests</br>
Part one - create basic implementation
=========================

This is kind of independent follow up of my previous post - [Creating testable interfaces](https://github.com/hliberacki/hliberacki.github.io/blob/master/testable_interfaces.md#creating-testable-interfaces). Previously we made it able to inject mocks, directly into our class which needs to be tested, and in this post I'll focus on creating the mock itself.

Rational - is there a problem?
-------------------------------
Lately I myself had an issue of creating unit tests, both for implementation of my module and the one that was created by third party, so I wasn't able to make it fully or even partially in a TDD (test driven development) way. Moreover, the requirements for test coverage was set as 100/100 (100% function and 100% branch) - which is really high, and in most cases unrealistic. 

Luckily, I had an advantage by the fact that the module which I had to write tests for wasn't too big, and not that much complicated - as I thought by the time of starting. But I've quickly realized that even though there are no many functions to test, unfortunately there is a lot of _if_ statements which made tons of branches.

To test the module, I had to also create a few mocks, which at first seemed easy. Quite quickly I've realized that even within the scope of testing one function, mocked methods have to return different values. Because of the number of branches, mocked functionalities started to grow exponentially, and eventually it became overcomplicated. In addition to that I realized that mocks which I will be injecting are private within my context. Moreover, they are being moved, so I was losing my handle (reference) to private injected mock. I could overcome that by having references to internal flags and variables within mock, but that's just ugly. In the end I would have some kind of Frankenstein code. 

Easier approach
---------------

My aim was to create generic mocking object, which decouples implementation and definition. Only needed implementation for each test case would be injected. From mock perspective I would like to provide the function name which has to be called. 

Easiest way to hold multiple different object types within one container, is using ```std::tuple```.

```cpp
template<class... Callable>
class GenericMock
{
public:
    GenericMock(Callable... callables): m_Callables{callables...} {}
private:
    std::tuple<Callable...> m_Callables;
};
```

```GenericMock``` will have a container of any callable objects, which will represent functions in our interface which need to be mocked.

Remaining problem is how to access needed function from our container. ```std::tuple``` gives us possibility to use ```std::get``` for taking needed object out, it gives as two way of doing so. First is accessing by providing Index number, yet this is definitely not sufficient, because in that case we would need to always provide all functions and always in the same order. Second way is more likely to be used, since it returns object by it type.

Yet to achieve that we need to provide objects type uniqueness, therefore we will use tags. So, let's create some class that we need to mock, before finishing implementation of ```GenericMock```.

```cpp
template<class Injected>
struct ToTest
{
    void foo() { obj.foo(); }   
    auto bar(std::string s)
    {
        obj.foo();
        return obj.bar(s);
    }
    
    Injected obj;
};
```

Now we need to create tags for each method, let's use _ prefix just to show that this is a tag.

```cpp
using _foo = std::function<void()>;
using _bar = std::function<int(std::string)>;
````
Now we can finish the implementation of ```GenericMock```

```cpp
template<class... Callable>
class GenericMock
{
public:
    GenericMock(Callable... callables): m_Callables{callables...} {}
    template<class Tag, class... Args> auto call(Args... args) 
    { 
        auto& callable = std::get<Tag>(m_Callables);
        return callable(args...);
    } 
private:
    std::tuple<Callable...> m_Callables;
};
```

As mentioned before, we are using ```std::get``` to get callable object by tag, and afterwards when the object is found we call it with given arguments. 

So, let's create the mock for ```ToTest``` class.

```cpp
template<class... Callables>
class CustomMock
{
public:
    CustomMock(Callables... callables) : m_ImplProvider{callables...} {};
    
    //interface
    void foo()
    {
        return call<::tag::_foo>();
    }
    
    int bar(std::string s)
    {
        return call<::tag::_bar>(s);
    }
    
private:
    template<class Tag, class... Args>
    auto call(Args... args)
    {
        return m_ImplProvider.template call<Tag, Args...>(args...);
    }
    
    GenericMock<Callables...> m_ImplProvider;
};
```

We need to provide all needed mocked functions into our mock class. Class is aggregating ```GenericMock```, which will be our provider of injected implementations. 

To simplify a calling of functions there is addition method ```call```, same as in ```GenericMock```. This is needed only to get rid of need of providing additional ```template``` for each function call, which might be misleading.

in each method we are just using our implementation provider, instead of defining logic.
```cpp
int bar(std::string s)
{
    return call<::tag::_bar>(s);
}
````

Here is how we can use it
```cpp
int main()
{
    { //test foo
        CustomMock<::tag::_foo> mock{ 
            ::tag::_foo{ []() { std::cout << "mocked" << std::endl; } } 
                };
                
        ToTest<decltype(mock)> t {mock};
        t.foo();
    }
    
    { //test foobar
        CustomMock<::tag::_foo,
                   ::tag::_bar> mock{ 
            ::tag::_foo{ []()              { std::cout << "mocked" << std::endl; } },
            ::tag::_bar{ [](std::string s) { std::cout << s << std::endl; return 1; } }
                };
                
        ToTest<decltype(mock)> t {mock};
        
        std::cout << t.bar("crap") <<std::endl;
    }
}
```
See full code in [Coliru](http://coliru.stacked-crooked.com/a/1935a3d4fa0c394c)

This is a very simple implementation in next post I'll create more robust one - and add type traits to ensure that callable are invokable and each injected type is unique. 

Hubert Liberacki
