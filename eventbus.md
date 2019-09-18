eventbus是一个很常见的android库，平时开发用的也比较多
但是会用和了解它的原理是不一样的，今天主要通过正常的一条eventbus流程，来分析一下它的内部实现
首先还是看一下基本用法
```
在oncrate或者onresume注册 根据需求在不同的地方注册
EventBus.getDefault().register(this)

对应的在相应的ondestory onstop中取消注册
EventBus.getDefault().unregister(this)

注册完毕后发送事件，所有的注册了TestEvent的页面，@Subscribe会收到事件触发
EventBus.getDefault().post(TestEvent())

//threadmode 还有POSTING BACKGROUND ASYNC
@Subscribe(threadMode = ThreadMode.MAIN)
    fun testEvent(event: TestEvent) {
     
    }
```

接下来开始分析第一句
```
public void register(Object subscriber) {
        Class<?> subscriberClass = subscriber.getClass();
        List<SubscriberMethod> subscriberMethods = subscriberMethodFinder.findSubscriberMethods(subscriberClass);
        synchronized (this) {
            for (SubscriberMethod subscriberMethod : subscriberMethods) {
                subscribe(subscriber, subscriberMethod);
            }
        }
    }
```
首先获取了当前类的class，并且传入了subscriberMethodFinder. findSubscriberMethods

```
    List<SubscriberMethod> findSubscriberMethods(Class<?> subscriberClass) {

        ......省略
        List<SubscriberMethod> subscriberMethods = METHOD_CACHE.get(subscriberClass);
    
      subscriberMethods = findUsingReflection(subscriberClass);
      return subscriberMethods;
}
```
省略了很多代码，根据初始化的变量判断，最终主要看findUsingReflection方法，将得到的subscriberMethods返回出去

```
private List<SubscriberMethod> findUsingReflection(Class<?> subscriberClass) {
      //新建一个FindState
        FindState findState = prepareFindState();
    //初始化FindState
        findState.initForSubscriber(subscriberClass);
        while (findState.clazz != null) {
            findUsingReflectionInSingleClass(findState);
            findState.moveToSuperclass();
        }
        return getMethodsAndRelease(findState);
    }
```

主要看关键的代码findUsingReflectionInSingleClass
```
private void findUsingReflectionInSingleClass(FindState findState) {
        Method[] methods;
        try {
            // This is faster than getMethods, especially when subscribers are fat classes like Activities
            methods = findState.clazz.getDeclaredMethods();
        } catch (Throwable th) {
            // Workaround for java.lang.NoClassDefFoundError, see https://github.com/greenrobot/EventBus/issues/149
            methods = findState.clazz.getMethods();
            findState.skipSuperClasses = true;
        }
        for (Method method : methods) {
            int modifiers = method.getModifiers();
            if ((modifiers & Modifier.PUBLIC) != 0 && (modifiers & MODIFIERS_IGNORE) == 0) {
                Class<?>[] parameterTypes = method.getParameterTypes();
                if (parameterTypes.length == 1) {
                    Subscribe subscribeAnnotation = method.getAnnotation(Subscribe.class);
                    if (subscribeAnnotation != null) {
                        Class<?> eventType = parameterTypes[0];
                        if (findState.checkAdd(method, eventType)) {
                            ThreadMode threadMode = subscribeAnnotation.threadMode();
                            findState.subscriberMethods.add(new SubscriberMethod(method, eventType, threadMode,
                                    subscribeAnnotation.priority(), subscribeAnnotation.sticky()));
                        }
                    }
                } else if (strictMethodVerification && method.isAnnotationPresent(Subscribe.class)) {
                    String methodName = method.getDeclaringClass().getName() + "." + method.getName();
                    throw new EventBusException("@Subscribe method " + methodName +
                            "must have exactly 1 parameter but has " + parameterTypes.length);
                }
            } else if (strictMethodVerification && method.isAnnotationPresent(Subscribe.class)) {
                String methodName = method.getDeclaringClass().getName() + "." + method.getName();
                throw new EventBusException(methodName +
                        " is a illegal @Subscribe method: must be public, non-static, and non-abstract");
            }
        }
    }
```

这个方法也是register过程中最重要的方法
可以看到根据反射，分别去寻找类中标记了@Subscribe的方法 ，拿到它的方法名，threadMode，优先级,方法是否是public,是否是粘性事件，并且把这些方法都封装在了findState.subscriberMethods中

也就是说最开始List<SubscriberMethod> subscriberMethods = subscriberMethodFinder.findSubscriberMethods(subscriberClass);
这个方法，我们拿到了这个类中所有的标记了@ Subscribe的方法和他们的属性
接下来在看
```
for (SubscriberMethod subscriberMethod : subscriberMethods) {
                subscribe(subscriber, subscriberMethod);
            }
```

将所有的subscriberMethods进行循环
```
private void subscribe(Object subscriber, SubscriberMethod subscriberMethod) {
        ....上面代码省略主要是把subscriberMethod中的属性拿出来，分别用一个集合存储主要
        if (subscriberMethod.sticky) {
            if (eventInheritance) {
                // Existing sticky events of all subclasses of eventType have to be considered.
                // Note: Iterating over all events may be inefficient with lots of sticky events,
                // thus data structure should be changed to allow a more efficient lookup
                // (e.g. an additional map storing sub classes of super classes: Class -> List<Class>).
                Set<Map.Entry<Class<?>, Object>> entries = stickyEvents.entrySet();
                for (Map.Entry<Class<?>, Object> entry : entries) {
                    Class<?> candidateEventType = entry.getKey();
                    if (eventType.isAssignableFrom(candidateEventType)) {
                        Object stickyEvent = entry.getValue();
                        checkPostStickyEventToSubscription(newSubscription, stickyEvent);
                    }
                }
            } else {
                Object stickyEvent = stickyEvents.get(eventType);
                checkPostStickyEventToSubscription(newSubscription, stickyEvent);
            }
        }
    }
```
主要看下面的代码如果事件是粘性事件
那么就会调用
checkPostStickyEventToSubscription
```
private void postToSubscription(Subscription subscription, Object event, boolean isMainThread) {
        switch (subscription.subscriberMethod.threadMode) {
            case POSTING:
                invokeSubscriber(subscription, event);
                break;
            case MAIN:
                if (isMainThread) {
                    invokeSubscriber(subscription, event);
                } else {
                    mainThreadPoster.enqueue(subscription, event);
                }
                break;
            case BACKGROUND:
                if (isMainThread) {
                    backgroundPoster.enqueue(subscription, event);
                } else {
                    invokeSubscriber(subscription, event);
                }
                break;
            case ASYNC:
                asyncPoster.enqueue(subscription, event);
                break;
            default:
                throw new IllegalStateException("Unknown thread mode: " + subscription.subscriberMethod.threadMode);
        }
    }
```
也就是说如果有这个粘性事件已经发送过，那么在这个界面注册后就会马上执行，
Sticky事件只指事件消费者在事件发布之后才注册的也能接收到该事件的特殊类型
通过invokeSubscriber
```
 void invokeSubscriber(Subscription subscription, Object event) {
        try {
            subscription.subscriberMethod.method.invoke(subscription.subscriber, event);
        } catch (InvocationTargetException e) {
            handleSubscriberException(subscription, event, e.getCause());
        } catch (IllegalAccessException e) {
            throw new IllegalStateException("Unexpected exception", e);
        }
    }
```
调用方法的invoke方法触发事件
正好在这个地方分析一下threadmode
根据源码的posting 可以看到没有进行任何的线程切换
也就是说如果你指定了POSTING，那么你是在主线程发送的就是在主线程接收，你是在异步线程就是在异步线程接收
```
case POSTING:
                invokeSubscriber(subscription, event);
                break;
```
接下来看MAIN
如果发送线程不是主线程，就是调用mainThreadPoster切换到主线程，如果是主线程就直接调用invoke方法触发事件
```
case MAIN:
                if (isMainThread) {
                    invokeSubscriber(subscription, event);
                } else {
                    mainThreadPoster.enqueue(subscription, event);
                }
                break;
```
剩下的就不分析都是同样的原理，到这里register就分析完了

unregister流程就比较简单了
```
public synchronized void unregister(Object subscriber) {
        List<Class<?>> subscribedTypes = typesBySubscriber.get(subscriber);
        if (subscribedTypes != null) {
            for (Class<?> eventType : subscribedTypes) {
                unsubscribeByEventType(subscriber, eventType);
            }
            typesBySubscriber.remove(subscriber);
        } else {
            Log.w(TAG, "Subscriber to unregister was not registered before: " + subscriber.getClass());
        }
    }

 private void unsubscribeByEventType(Object subscriber, Class<?> eventType) {
        List<Subscription> subscriptions = subscriptionsByEventType.get(eventType);
        if (subscriptions != null) {
            int size = subscriptions.size();
            for (int i = 0; i < size; i++) {
                Subscription subscription = subscriptions.get(i);
                if (subscription.subscriber == subscriber) {
                    subscription.active = false;
                    subscriptions.remove(i);
                    i--;
                    size--;
                }
            }
        }
    }
```
就是把我们在register中添加的
subscriptionsByEventType
typesBySubscriber
根据传入的object移除集合中的数据
这样在后续post事件的时候，根据事件类型名字，unregister的页面就不会接收到事件

接下来看最后一个
```
EventBus.getDefault().post(TestEvent())
```

```
public void post(Object event) {
        PostingThreadState postingState = currentPostingThreadState.get();
        List<Object> eventQueue = postingState.eventQueue;
        eventQueue.add(event);

        if (!postingState.isPosting) {
            postingState.isMainThread = Looper.getMainLooper() == Looper.myLooper();
            postingState.isPosting = true;
            if (postingState.canceled) {
                throw new EventBusException("Internal error. Abort state was not reset");
            }
            try {
                while (!eventQueue.isEmpty()) {
                    //主要看这句代码
                    postSingleEvent(eventQueue.remove(0), postingState);
                }
            } finally {
                postingState.isPosting = false;
                postingState.isMainThread = false;
            }
        }
    }
```
前面主要是获取当前发送线程的状态，如果没有在发送，就发送eventQueue队列中的成员。
主要看postSingleEvent这一句代码

```
private void postSingleEvent(Object event, PostingThreadState postingState) throws Error {
        Class<?> eventClass = event.getClass();
        boolean subscriptionFound = false;
        if (eventInheritance) {
            List<Class<?>> eventTypes = lookupAllEventTypes(eventClass);
            int countTypes = eventTypes.size();
            for (int h = 0; h < countTypes; h++) {
                Class<?> clazz = eventTypes.get(h);
                subscriptionFound |= postSingleEventForEventType(event, postingState, clazz);
            }
        } else {
            subscriptionFound = postSingleEventForEventType(event, postingState, eventClass);
        }
        if (!subscriptionFound) {
            if (logNoSubscriberMessages) {
                Log.d(TAG, "No subscribers registered for event " + eventClass);
            }
            if (sendNoSubscriberEvent && eventClass != NoSubscriberEvent.class &&
                    eventClass != SubscriberExceptionEvent.class) {
                post(new NoSubscriberEvent(this, event));
            }
        }
    }
```
这里通过postSingleEventForEventType查询post过来的event是否有注册过，如果没有注册过就会发送一个NoSubscriberEvent事件，这个事件再次到达这个方法的时候被消费

如果查询有注册就会进入postSingleEventForEventType方法里面的postToSubscription

```
private boolean postSingleEventForEventType(Object event, PostingThreadState postingState, Class<?> eventClass) {
        CopyOnWriteArrayList<Subscription> subscriptions;
        synchronized (this) {
            subscriptions = subscriptionsByEventType.get(eventClass);
        }
        if (subscriptions != null && !subscriptions.isEmpty()) {
            for (Subscription subscription : subscriptions) {
                postingState.event = event;
                postingState.subscription = subscription;
                boolean aborted = false;
                try {
                    postToSubscription(subscription, event, postingState.isMainThread);
                    aborted = postingState.canceled;
                } finally {
                    postingState.event = null;
                    postingState.subscription = null;
                    postingState.canceled = false;
                }
                if (aborted) {
                    break;
                }
            }
            return true;
        }
        return false;
    }
```

postToSubscription这个方法前面已经分析过来 就是通过不同线程调用方法的invoke触发事件到这里一个整的流程就结束了，这篇文章也只是简单的跟踪了一篇这个流程，关于eventbus还有很多知识，包括里面的缓存，里面的队列的工作留着以后有时间在继续分析