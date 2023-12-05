# RxSwift와 MVVM을 사용하면서 고민해본점

## ViewModel 안에서 왜 Driver를 사용 하는가?
깃헙에 보이는 많은 MVVM + RxSwift를 사용한 코드들에서 흔히 보여지는 Input Output을 사용한 MVVM패턴에서     
input 과 output 혹은 output을 Driver를 사용하는 흔한 경우를 볼수 있는데      
왜 Driver를 사용하는것일까?     
Driver는 항상 메인쓰레드에서 동작한다 또한 내부에 share(replay: 1, scope: .whileConnected) 가 구현되어 있다.     
그리고 에러상황이 발생할 경우 onErrorJustReturn을 통해 에러 대신 기본값을 반환한다.     
하지만 실제로 실무에서 ViewModel안에서 Driver를 사용하는 이점에 대해서 잘 모르겠다     
싧무에서 API 호출을 한뒤에 Observable로 데이터를 가져오는 예시를 대강 만들어 보았다     
```
//api 호출 상황을 만들기 위해 계산이 오래 걸리는 코드를 생성
func requestApi(isError: Bool) -> Observable<String> {
     return Observable<String>.create { anyObserver -> Disposable in
         performLongCalculation()
         if isError {
             let error = RxError.noElements
             anyObserver.onError(error)
         } else {
             anyObserver.onNext("ok")
             anyObserver.onCompleted()
         }
         
         return Disposables.create()
      
     }
 }
 
 func performLongCalculation() {
     let startTime = CFAbsoluteTimeGetCurrent()
     
     var result: Double = 0
     for _ in 0..<3000000 {
         result += sin(Double.random(in: 0...1))
     }
     
     let endTime = CFAbsoluteTimeGetCurrent()
     let elapsedTime = endTime - startTime
     print("debug \("계산이 완료되었습니다. 경과된 시간: \(elapsedTime)초")")
 }

// ViewModel
final class ViewModel {
    
    private var 강제에러 = true
    private let disposeBag = DisposeBag()
    
    struct Input {
        let trigger = Driver<Void>()
    }
    
    struct Output {
        let text: Driver<String>
    }
    
    func transform(input: Input) -> Output {
        let text = input.trigger
             .flatMapLatest { [weak self] _ -> Observable<String> in
                 self?.강제에러 = !(self?.강제에러 ?? true)
                 return requestApi(isError: self?.강제에러 ?? true)
             }
             .asDriver(onErrorJustReturn: "Error Default")

        return Output(text: text)
    }
    
}

//ViewController

let triggerButton = UIButton()

    override func viewDidLoad() {
        super.viewDidLoad()

   let viewModel = ViewModel()
   let input = ViewModel.Input(trigger: triggerButton.rx.tap.asDriver)
   let output = viewModel.transform(input: input)
        
        output.text
            .bind(to: label.rx.text)
            .disposed(by: disposeBag)
        
    }
```

나는 처음에 흔히 이런 코드에서 api에서 에러를 반환하더라도 text에 asDriver를 했으니 에러가 반환 될 때마다 "Error Default"가 나와주겠지? 라고 생각을 했었다          
하지만 실제 코드를 확인해 보면 "Error Default"는 단 한번만 반환되고 그 뒤엔 스트림이 끊어져 더이상 이벤트를 반환 하지 않는다.      
스트림이 끊어지지 않고 에러가 반환 될때마다 "Error Default"를 대신 반환 하고 싶다면 driver text에 asDriver를 하는것이 아니라     
requestApi에 asDriver를 해 줘야한다.    
```
        let text = input.trigger
             .flatMapLatest { [weak self] _ -> Driver<String> in
                 self?.강제에러 = !(self?.강제에러 ?? true)
                 return requestApi(isError: self?.강제에러 ?? true)
                     .asDriver(onErrorJustReturn: "Error Default")
             }
```
이렇게 하면 스트림이 끊어지지 않고 "Error Default"를 계속 반환 하게된다.    
그러다 다시 정상 이벤트가 발생하면 정상적으로 반환한다.    
하지만 또 이런생각이 들게 된다.   
"api 호출을 한뒤에 받아오는 데이터에 대해서 굳이 Driver를 써야할 이유가 있을까?"      
Driver는 메인쓰레드에서 동작한다 그런데 api 호출을 한뒤에 받아오는 데이터에 대해서 꼭 메인 쓰레드에서 처리할 필요가 있을까?     
방금 위에 코드를 그대로 사용하고 ViewController에 새로운 버튼을 한게 만들고 trigger 버튼을 누른 동시에 새로운 버튼을 눌러보자 그 버튼의 이벤트가 동작하는가??     
아니 동작하지 않을것이다 오래 걸리는 작업을 메인쓰레드 에서 하고 있으니 말이다.     
```
//ViewControlelr
let triggerButton = UIButton()
let newButton = UIButton()

    override func viewDidLoad() {
        super.viewDidLoad()

   let viewModel = ViewModel()
   let input = ViewModel.Input(trigger: triggerButton.rx.tap.asDriver)
   let output = viewModel.transform(input: input)
        
        output.text
            .bind(to: label.rx.text)
            .disposed(by: disposeBag)

        newButton.rx.tap
            .subscribe(onNext: { _ in
                print("newButtonTap")
            }).disposed(by: disposeBag)
    }
```
trigger버튼을 누르고 바로 newButton을 눌러도 프린트는 실행되지 않으며 apiRequest가 모두 끝나야 그때 실행된다.      
이런 상황을 만들게되는 Driver를 굳이 input에 쓸필요가 있을까?      
만약 이런 복잡한 데이터 조합이나, api 호출이 아닌 단순한 작업의 경우는 계속해서 메인 쓰레드를 사용해도 무방하다고 생각하지만     
복잡한 작업이나 api 호출의 경우는 그 작업이 끝날동안 아무런 UI 조작을 하지 못하기 때문에 observeOn을 제대로 사용하여 비동기 적으로 진행해줘야 한다고 생각한다.      
이런 흐름을 만드는 코드 예시      

```
//ViewModel
    struct Input {
        let trigger: Observable<Void>
    }
    
    struct Output {
        let text: Observable<String>
    }
    
    func transform(input: Input) -> Output {
        let text = input.trigger
            .observe(on: ConcurrentDispatchQueueScheduler(qos: .default))
            .flatMapLatest { [weak self] _ -> Observable<String> in
                self?.강제에러 = !(self?.강제에러 ?? true)
                return requestApi(isError: self?.강제에러 ?? false)
                    .catchAndReturn("Error Default")
            }
        
        return Output(text: text)
    }

//ViewController

    override func viewDidLoad() {
        super.viewDidLoad()
        
        newButton.rx.tap
            .subscribe(onNext: { _ in
                print("newButtonTap")
            }).disposed(by: disposeBag)
        
        output.text
            .asDriver(onErrorJustReturn: "")
            .drive(label.rx.text)
            .disposed(by: disposeBag)
        
    }
```
이렇게 되면 프린트문도 비동기적으로 출력 되고      
UI에 적용될때만 Driver를 사용해주면 쓰레드 이슈도 생기지 않을 것이다.    
- ex       
   

## 결론
Driver를 ViewModel 안에서 굳이 사용할 필요는 없는것 같다.
ViewModel 안에서 내가 비동기적인 작업을 할수도 있기 때문.      
Driver의 원래 사용목적을 생각하면 스트림이 공유되고 메인쓰레드에서만 동작하기 때문에    
viewModel의 output으로 나가고난뒤 View에서 asDriver를 하는것을 Best라고 생각한다. 
         
ViewModel에서 observable, subject, Relay를 사용하는것이 좋아보임.          
하지만 기본적으로 input output 패턴 자체는 UI관련 작업이기때문에.     
(사용자의 화면 이벤트를 input으로 받아 어떤 작업을한뒤 output으로 내보내는것이 ViewModel의 기본).     
error, complete가 불리지 않게하기위해 Relay 사용을 지향하는것이 좋아 보임.  
                   
       


   
