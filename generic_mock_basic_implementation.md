
Generic mocking for tests</br>
Part one - create basic implementation
=========================

This is kind of independent follow up of my previous post - [Creating testable interfaces](https://github.com/hliberacki/hliberacki.github.io/blob/master/testable_interfaces.md#creating-testable-interfaces). Previously we made it able to inject mocks, directly into our class which needs to be tested, and in this post I'll focus on creating the mock itself.

Rational - is there a problem?
-------------------------------
Lately I myself had an issue of creating unit tests, both for implementation of my module and the one that was created by third party, so I wasn't able to make it fully or even partially in a TDD (test driven development) way. Moreover, the requirements for test coverage was set as 100/100 (100% function and 100% branch) - which is really high, and in most cases unrealistic. 

Luckily, I had an advantage by the fact that the module which I had to write tests for wasn't too big, and not that much complicated - as I thought by the time of starting. But I've quickly realized that even though there are no many functions to test, unfortunately there is a lot of _if_ statements which made tons of branches. 
