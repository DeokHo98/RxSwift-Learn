# Extension Reactive
RxSwift를 Extension해서 사용하는 방법에는 3가지 가 있다.    

## ControlEvent
```
button.rx.tap
.bind()
```
이런 방식의 코드는 많이 사용 했을껀데    
이는 이미 버튼에 extension 되어있는 코드다.    
원래는 사실 이렇게 구현되어 있다.    
```
// UIButton + Rx
extension Reactive where Base: UIButton {
    
    /// Reactive wrapper for `TouchUpInside` control event.
    public var tap: ControlEvent<Void> {
        controlEvent(.touchUpInside)
    }
}
```
이런 식으로 button.rx.tap 처럼 커스텀에서 구현 할 수가 있다는 뜻이다.     
ControlEvent는 값을 구독 할 수는 있지만 값을 주입시키는 못한다.
    
## ControlProperty
ControlProperty는 값을 주입시키는것도 가능하고, 값을 구독 할 수도있다.     
예를들면 textFiled.text의 경우 ControlProperty이다.      
```
extension Reactive where Base: UITextField {
    /// Reactive wrapper for `text` property.
    public var text: ControlProperty<String?> {
        value
    }
}
```
그래서 textFiled.text는 
```
Observable.just("1234")
    .bind(to: textField.text)
```
처럼 값을 주입시킬 수도
```
textField.rx.text
   .subscribe { }
```
값을 구독하는것도 가능하다.    

ControlProperty와 ControlEvent의 값을 구독하는것의 차이는.   
ControlEvent는 구독시 초기값을 방출하지 않는데.     
ControlProperty는 구독시 초기값을 방출한다.     
    
## Binder
값을 주입시키는것은 가능하지만 값을 구독 하는 것은 불가능하다.
대표 적인 예가 label.text 이다.     
label.text는 값을 주입은 가능하지만 구독은 되지 않는다.
    
실무에선 Binder를 가장 많이 사용했던것 같다.     
Binder 예시.            
```
extension Reactive where Base: UIButton {
    var isSelectedToggle: Binder<Void> {
        return Binder(base) { button, _ in
            button.isSelected = !button.isSelected
        }
    }
}

extension Reactive where Base: UIViewController {
    var alert: Binder<Message> {
        return Binder(base) { viewController, text in
            viewController.alert(text.title, msg: text.message)
        }
    }
    
    var popToRootViewController: Binder<Void> {
        return Binder(base) { viewController, _ in
            viewController.navigationController?.popToRootViewController(animated: true)
        }
    }
    
    var popViewController: Binder<Void> {
        return Binder(base) { viewController, _ in
            viewController.navigationController?.popViewController(animated: true)
        }
    }
}
```

