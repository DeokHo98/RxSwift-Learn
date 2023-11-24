# Hot&Cold Observable
Reactive 공식문서엔 설명이 이렇게 되어있다.    
```
“Hot” and “Cold” Observables
When does an Observable begin emitting its sequence of items?
It depends on the Observable.
A “hot” Observable may begin emitting items as soon as it is created, 
and so any observer who later subscribes to that Observable may start observing the sequence somewhere in the middle. 
A “cold” Observable,
on the other hand,
waits until an observer subscribes to it before it begins to emit items,
and so such an observer is guaranteed to see the whole sequence from the beginning.


Observable이 언제 어떤 순서의 아이템을 방출하는지는 Observable에 달려있는데,
여기서 Hot Observable은 생성되는 즉시 방출하기 시작하는데 이 때,
중간의 아이템을 방출하고 있더라고 그 아이템을 방출하기 시작합니다.
반면에 Cold Observable은 Observer가 구독할 때까지 기다리고 구독하는 순간부터 방출하기 시작합니다.
그리고 Observer에게 방출하는 아이템의 전부를 볼 수 있게 보장합니다.
```

## Hot Observable
Hot Observable은 구독 여부에 상관 없이 이벤트를 발생시킨다.    
일단 동작하기 시작하면 리소스를 사용한다.   
구독했을때 이벤트 시퀀스를 처음부터 관찰하지 못한다.     
구독하는 시점에 따라서 전달받는 이벤트 또한 다르다.     
ex) Subject     

## Cold Observable
Cold Observable는 observer가 구독하기 전까지는 이벤트가 방출되지 않음
Observable이 observer마다 새로운 시퀀스를 시작함
ex) just, single, creat, timer, UIEvent

## Share
share는 Cold Observable의 이벤트 방출의 중복을 일으키지 않게 딱 한번만 방출시키게 하는 연산자이다.    
이로인해 중복실행을 막는것이다.         
api 호출같은 역할을 하는 Observable 같은경우 중복되면 그만큼의 자원소비가 되는것이기 때문에.      
그런것을 방지하기 위해서 사용한다.    
```
        let observable = Observable<Int>.create { observer in
            print("api 호출 시작이라고 생각")
            observer.onNext(1)
            observer.onNext(2)
            observer.onNext(3)
            observer.onNext(4)
            observer.onNext(5)
            observer.onNext(6)
            observer.onNext(7)
            observer.onNext(8)
            observer.onNext(9)
            observer.onNext(10)
            return Disposables.create()
        }

        _ = observable.subscribe(onNext: { element in
            print("first: \(element)")
        })

        _ = observable.subscribe(onNext: { element in
            print("second: \(element)")
        })
        
api 호출 시작이라고 생각
first: 1
first: 2
first: 3
first: 4
first: 5
first: 6
first: 7
first: 8
first: 9
first: 10
api 호출 시작이라고 생각
second: 1
second: 2
second: 3
second: 4
second: 5
second: 6
second: 7
second: 8
second: 9
second: 10
```
실행 시켜보면.     
이렇게 api 호출이 두번 실행되는데.        
여기서 share를 해주면.       
```
        let observable = Observable<Int>.create { observer in
            print("api 호출 시작이라고 생각")
            observer.onNext(1)
            observer.onNext(2)
            observer.onNext(3)
            observer.onNext(4)
            observer.onNext(5)
            observer.onNext(6)
            observer.onNext(7)
            observer.onNext(8)
            observer.onNext(9)
            observer.onNext(10)
            return Disposables.create()
        }.share()

        _ = observable.subscribe(onNext: { element in
            print("first: \(element)")
        })

        _ = observable.subscribe(onNext: { element in
            print("second: \(element)")
        })
        
api 호출 시작이라고 생각
first: 1
first: 2
first: 3
first: 4
first: 5
first: 6
first: 7
first: 8
first: 9
first: 10
```
이렇게 api 호출이 한번만 일어나게 된다.      
중복실행되지 않기때문에 두번재 구독자에게 이벤트가 전달되지 않는데.      
두번째 구독자에게도 이벤트를 전달하고 싶다면 replay 파라미터를 이용하면 된다.     
마지막 이벤트를 기준으로 share의 replay 파라미터에 넣어준 숫자만큼의 이벤트를 방출한다.       
replay 파람에 5를 넣는경우에는 이렇게 결과가 나온다.     
```
second: 6
second: 7
second: 8
second: 9
second: 10
```
