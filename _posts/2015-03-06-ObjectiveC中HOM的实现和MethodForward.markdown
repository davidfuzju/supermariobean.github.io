---
layout: post
category: iOS
---

## HOM

偶然遇到HOM这个概念，一番查阅并手写了一部分代码后，记下此文，同时也顺带复习了Objective-C的Method Forward机制。

HOM全称[Higher Order Message](http://en.wikipedia.org/wiki/Higher_order_message)，wiki百科的解释如下:

>A higher order message (HOM) in a computer programming language is a form of higher-order programming that allows messages that have other messages as arguments. The concept was introduced at MacHack 2003[1][2] by Marcel Weiher and presented in a more complete form in 2005 by Marcel Weiher and Stéphane Ducasse.[3] Loops can be written without naming the collections looped over, higher order messages can be viewed as a form of point-free or tacit programming.

    ...
    
    if (self.delegate && [self.delegate respondsToSelector:@selector(doSomething)]) {
        [self.delegate doSomething];
    }

    ...

对于每一次调用，我们都要判断delegate是否存在并且其是否可以响应某一个方法，而如果使用HOM的话，上面的方法可以重写为:

    [[self.delegate try] doSomething];

大致意思为，先向self.delegate发送`try`方法，如果self.delegate响应`doSomething`方法，则调用，如果self.delegate不响应`doSomething`方法，则直接返回，不做任何动作。

结合上面HOM的解释，考虑其具体实现应该是消息`try`将消息`doSomething`作为自己的参数，在`try`的内部逻辑中判断self.delegate是否可以响应以决定是否调用，以免在代理对象上调用不存方法导致`doesNotRecognizeSelector:`的调用而抛出异常。

为了让消息`doSomething`能够成为消息`try`的参数，我们需要让`[self.delegate try]`返回一个NSProxy的子类对象，我们将返回的这个对象叫做`Trampoline`(更多详细大家可以参考[这篇文章](http://www.cocoadev.com/index.pl?HigherOrderMessaging))，我更喜欢叫它`蹦床`，由于是NSProxy的子类对象，所以当执行`[Trampoline doSomething]`的时候，会触发`Method Forward`流程，详细的流程后面的章节会详细介绍，这里暂且不表，基于`消息转发`流程，就可以将上层逻辑注入到消息try中去，实现HOM。

## Method Forward

消息转发我觉得大家并不陌生，[Apple Documents](https://developer.apple.com/library/mac/documentation/Cocoa/Conceptual/ObjCRuntimeGuide/Introduction/Introduction.html#//apple_ref/doc/uid/TP40008048-CH1-SW1)有详细的描述，`Effetive Objective-2.0`中`Chapter 2: Objects, Messaging, and the Runtime`的`Item 12: Understand Method Forward`也有相对简练的解释（不过书中对于`full message forward`的流程描述缺少了`- (IMP)methodForSelector:(SEL)aSelector`的阶段，个人不太理解为何作者会意外遗漏）。

当一个消息发送给一个对象，如果该对象无法`响应`(`responds`)就会触发消息转发流程，流程分为三个阶段，如下：

+ 动态方法解析 `Dynamic Method Resolution`

  该阶段的核心方法为：

      + (BOOL)resolveClassMethod:(SEL)sel       // for class method
      + (BOOL)resolveInstanceMethod:(SEL)sel   // for instance method

  当对象无法响应消息时，针对该消息是`类方法`还是`实例方法`，会调用上述中的方法之一。调用后，用户根据自己的逻辑在该方法内，也就是运行时，选择是否为当前对象增加一个和参数sel一样的方法，并通过返回`YES` or `NO`来告知程序你是否为对象增加了可以响应的方法。如果返回`YES`，消息会被重新再发送一遍；如果返回`NO`，直接进入下一步流程

  >Tip: Objective-C中的关键字@dynamic就是利用了这个特性，也就是告诉编译器在编译阶段不要为属性生成`setter`和`getter`，而是通过运行时在`动态方法解析`过程中加入到对象中，当第一次将方法加入对象后，调用流程就和普通的方法相同了。
  >Tip: 另外，方法`- (BOOL)respondsToSelector:(SEL)aSelector`或`+ (BOOL)instancesRespondToSelector:(SEL)aSelector;`的调用也会触发1`动态方法解析`流程

+ 快速消息转发 或 接收者替换 `fast forwarding` or `replacement recerver`

  该阶段的核心方法为：

      - (id)forwardingTargetForSelector:(SEL)aSelector

  当进入该阶段时，该方法会被调用，传入的是未得到响应并且未在`动态方法解析`流程加入对象的方法签名。调用后，用户根据自己的逻辑在该方法内，也就是运行时，选择返回一个有效对象（主要是指有意义的能够响应该方法签名的对象）或者返回nil或者self。前者，系统会自动将该方法转发给返回的有效对象；后者，系统会结束目前的流程，进入下一个流程。

+ 标准消息转发 或 完整消息转发 `normal forwarding` or `full forwarding`

  该阶段的核心方法为：

      - (NSMethodSignature *)methodSignatureForSelector:(SEL)aSelector
      - (void)forwardInvocation:(NSInvocation *)anInvocation

  当进入该阶段时，首先`- (NSMethodSignature *)methodSignatureForSelector:(SEL)aSelector`会被调用，要求用户根据自己的逻辑在该方法内，也就是运行时，返回一个`NSMethodSignature`对象或者nil，如果是nil则该流程结束，抛出异常；如果是一个对象，那么系统会根据这个对象生成一个`NSInvocation`对象并调用`- (void)forwardInvocation:(NSInvocation *)anInvocation`，在此方法中，用户根据自己逻辑处理，没有什么限制。

`Method Forward`大致会经历以上三个阶段，`动态方法解析`非常轻量，但是只能增加传入的方法签名；`快速转发`相对轻量，但是只能做转发消息到其他对象的行为；`标准消息转发`最重量，但是拥有最高的自由度。

## HOM实现

基于以上的回顾，我们来考虑如何实现HOM下的`try`方法。为了获得更高的自由度，我们选择在`标准消息转发`阶段来实现，让`try`方法返回一个注入了用户逻辑的NSProxy子类对象，当后续的`doSomething`方法发送到这个NSProxy子类对象时，会经过`标准消息转发`在方法回调中执行注入的用户逻辑。

NSObject (SMBDelegateTry)

    @interface NSObject (SMBDelegateTry)
    - (id)try;
    @end
    
    @implementation NSObject (SMBDelegateTry)
    
    - (id)try {
        return [SMBTrampoline trampolineWithSelectorHandler:^(SEL _selector_) {
            return [self methodSignatureForSelector:_selector_] ?: [NSMethodSignature signatureWithObjCTypes:"@@:"];
        } invocationHandler:^(NSInvocation *_invocation_) {
            if ([self respondsToSelector:_invocation_.selector]) {
                [_invocation_ invokeWithTarget:self];
            }
        }];
    }
    
    @end

SMBTrampoline


    @interface SMBTrampoline : NSProxy
    
    + (instancetype)trampolineWithSelectorHandler:(NSMethodSignature *(^)(SEL))selectorHandler
      invocationHandler:(void (^)(NSInvocation *))invocationHandler;
    
    @end

    @interface SMBTrampoline ()
    
    @property (readwrite, nonatomic, copy) NSMethodSignature *(^selectorHandler)(SEL);
    @property (readwrite, nonatomic, copy) void (^invocationHandler)(NSInvocation *);
    
    @end
    
    @implementation SMBTrampoline
    
    
    + (instancetype)trampolineWithSelectorHandler:(NSMethodSignature *(^)(SEL))selectorHandler
      invocationHandler:(void (^)(NSInvocation *))invocationHandler {
    	SMBTrampoline *result = [self alloc];
    	result.selectorHandler = [selectorHandler copy];
    	result.invocationHandler = [invocationHandler copy];
    	return result;
    }
    
    - (NSMethodSignature *)methodSignatureForSelector:(SEL)selector {
    	return self.selectorHandler(selector);
    }
    
    - (void)forwardInvocation:(NSInvocation *)invocation {
    	self.invocationHandler(invocation);
    }
    
    @end

最终的使用代码

    [[self.delegate try] doSomething];

当self.delegate响应doSomething会正常执行，当self.delegate不响应时什么也不做，不会崩溃。

从代码来看，`try`的逻辑通过NSObject(SMBDelegateTry)封装在SMBTrampoline对象中，`try`返回SMBTrampoline对象，其后的消息`doSomething`通过`标准消息转发`流程被捕获，交于从NSObject(SMBDelegateTry)注入的逻辑处理，处理完毕后结束。

## 总结

本文主要是大致描述了`HOM`的概念和简单应用的基本实现，顺带复习了一下`Method forward`机制。
