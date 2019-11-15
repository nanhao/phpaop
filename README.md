# phpaop
Phpaop is a simple php7 extension for AOP (Aspect Oriented Programming), which allow you to attach a piece of code before/after a method or function in the easiest way.

## What is AOP?
Let's assume the following class:
```php
class MyClass
{
    public function method1()
    {
        log(); // write some log here
        
        // main logic of method1
        // ...
    }
    
    public function method2()
    {
        log(); // write some log here
        
        // main logic of method2
        // ...
    }
}
```
We see log() appears both at the start of method1() and method2(). They are necessary while they are not part of main logic of the methods.  
In fact, log() may appear repeatedly in many other methods across your system. These log() form an aspect of the system. With AOP, we have a better way to organize them:
```php
class MyClass
{
    public function method1()
    {       
        // main logic of method1
        // ...
    }
    
    public function method2()
    {   
        // main logic of method2
        // ...
    }
}

PHPAOP::add_advice([
    'before@MyClass::method1',
    'before@MyClass::method2',
], function() {
    log();
});
```
With codes above, we put the log aspect in a single place, writing log() only once. PHPAOP::add_advice() will magically attach it to the beginning of method1 and method2.  
By this way, we gain at least two advantages:
- We extract the aspect, making it easier to maintain.
- We keep the main logic of methods clean, which also make them easier to maintain.

Logging is only one typical aspect. Other common aspects include access control, statistics and so on.  

## Installation

```bash
git clone https://github.com/nanhao/phpaop.git
cd phpaop
phpize
./configure
make
make test
make install
```

Add the following lines to your php.ini
```bash
[phpaop]
extension=phpaop.so
```

## Usage
```php
PHPAOP::add_advice([
    'before@class_name::method_name',
    'after@class_name::method_name',
    'before@function_name',
], function($joinpoint, $args, $ret) {
    // todo
});
```

## Two types of advice
There are before-advice and after-advice:
```php
before@class_name::method_name
after@class_name::method_name
```  
Before-advice is attached to the beginning of the target code, while after-advice is attached to the end of the target code.

## Before-advice
Before-advice is executed **after** the caller calls the callee and **before** the callee receives the arguments, which means:
```php
function sum($a, $b = 10) {
    return $a + $b;
}

PHPAOP::add_advice(['before@sum'], function($joinpoint, $args, $ret) {
    var_dump($joinpoint);
    var_dump($args);
    var_dump($ret);
});

sum(1);
```
The above code will output:
```text
string(8) "before@sum"
array(2) {
  ["a"]=>
  int(1)
}
NULL
```
- Because the default value of $b is set when the callee receives the arguments, we can't find $b in the $args array. In other words, $args represents arguments actually passed by the caller instead of that recieved by the callee.
- $ret is NULL due to obvious reasons.

## After-advice
After-advice is executed **after** the return statement of the callee, So you can get the real return value by reading $ret. However, there is a particular scenario under which $ret is set to NULL even when the real return value seems not NULL:
```php
function sum($a, $b) {
    return $a + $b;
}

PHPAOP::add_advice(['after@sum'], function($joinpoint, $args, $ret) {
    var_dump($ret);
});

sum(1, 2);
```
The above code output NULL instead of 3. The reason is:
- The return value of sum(1, 2) is not assigned to other variables. It's unused, so the underlying engine of php discards the return value on optimization purpose.

## When to call PHPAOP::add_advice?
- PHPAOP::add_advice **can** be called before the target code's definition:
```php
// ok
PHPAOP::add_advice(['after@sum'], function($joinpoint, $args, $ret) {
    var_dump($ret);
});

function sum($a, $b) {
    return $a + $b;
}

sum(1, 2);
```
- PHPAOP::add_advice **should** be called before the target code's execution:
```php
// bad. advice will not run
function sum($a, $b) {
    return $a + $b;
}

sum(1, 2);

PHPAOP::add_advice(['after@sum'], function($joinpoint, $args, $ret) {
    var_dump($ret);
});
``` 

## The execution of an advice may trigger another advice. 
Consider the following code:
```php
PHPAOP::add_advice(['after@sum'], function($joinpoint, $args, $ret) {
    sum(3, 4);
});

function sum($a, $b) {
    return $a + $b;
}

sum(1, 2);
```
If you run the above script, it will cause a fetal error:
```text
Fatal error: advice recursion detected: after@sum
```
Phpaop is built with a mechanism that within an advice, another advice may be triggered. But, recursion is not allowed.