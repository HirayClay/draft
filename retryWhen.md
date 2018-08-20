
### retryWhen
R:
retryWhen: 当发生错误的时候通过notificationHandler 函数返回的observable（记作t）来决定是否重试

```java
retryWhen(new Func1<Observable<? extends Throwable>, Observable<Notification>>() {
            public Observable<Notification> call(Observable<? extends Throwable> throwableObservable) {

             //Observable.just(null)

            }
        })
```
由于内部实现的原因，不能像官方源码解释的那样，简单粗暴返回一个 “Observable.just(null)”,虽然也是会调用onNext,但是
和传入的throwableObservable没有任何的关系；这里就和内部实现有关;内部有一段代码：
```java
final Observable<?> restarts = controlHandlerFunction.call(
                terminals.lift(new Operator<Notification<?>, Notification<?>>() {
                    @Override
                    public Subscriber<? super Notification<?>> call(final Subscriber<? super Notification<?>> filteredTerminals) {
                        return new Subscriber<Notification<?>>(filteredTerminals) {
                            @Override
                            public void onCompleted() {
                                filteredTerminals.onCompleted();
                            }

                            @Override
                            public void onError(Throwable e) {
                                filteredTerminals.onError(e);
                            }

                            @Override
                            public void onNext(Notification<?> t) {
                                if (t.isOnCompleted() && stopOnComplete) {
                                    filteredTerminals.onCompleted();
                                } else if (t.isOnError() && stopOnError) {
                                    filteredTerminals.onError(t.getThrowable());
                                } else {
                                    filteredTerminals.onNext(t);
                                }
                            }

                            @Override
                            public void setProducer(Producer producer) {
                                producer.request(Long.MAX_VALUE);
                            }
                        };
                    }
                }));
```
controlHandlerFunction 就是我们自己定义的notificationHandler,这里terminals是一个BehaviroSubject.在这段代码里面先当一个Observable来看。