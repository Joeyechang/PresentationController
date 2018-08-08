UIKit 允许通过 delegate 模式（让主视图控制器采用UIViewControllerTransitioningDelegate）自定义 view controller 的模态视图弹出。

每次呈现新的视图控制器时，UIKit 都会询问其代理是否应该使用自定义 transition 。

![custom transition](https://upload-images.jianshu.io/upload_images/130752-243a6476733e7faa.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

UIKit 调用 animationController（forPresented：presents：source :) 来查看是否返回了 UIViewControllerAnimatedTransitioning。 如果该方法返回 nil，则 UIKit 使用内置转换。 如果 UIKit 接收 UIViewControllerAnimatedTransitioning 对象，则 UIKit 将该对象用作转换的动画控制器。

![animator](https://upload-images.jianshu.io/upload_images/130752-ff43e738e3efbbc4.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

UIKit 首先要求自定义的 animation controller 以秒为单位给出 transition 持续时间，然后调用  animateTransition（using:)，在该方法中我们可以可以访问屏幕上的 current view controller 以及将要显示的新 view controller，根据需要我们可以淡化（fade），缩放（scale），旋转（rotate）和操作（manipulate）现有 view 和新 view 。

##### Implementing transition delegates

创建集成于 NSObject 的类 PopAnimator 。该类将作为动画管理类。书写代码如下：

```
class PopAnimator: NSObject,UIViewControllerAnimatedTransitioning {
   
    let duration = 1.0
    var presenting = true
    var originFrame = CGRect.zero
    var dismissCompletion: (()->Void)?
    
    func transitionDuration(using transitionContext: UIViewControllerContextTransitioning?) -> TimeInterval {
       return duration
    }
    
    func animateTransition(using transitionContext: UIViewControllerContextTransitioning) {
        
        let containerView = transitionContext.containerView
        let toView = transitionContext.view(forKey: .to)!
        containerView.addSubview(toView)
        let herbView = presenting ? toView :
            transitionContext.view(forKey: .from)!
        let initialFrame = presenting ? originFrame : herbView.frame
        let finalFrame = presenting ? herbView.frame : originFrame
        let xScaleFactor = presenting ?
            initialFrame.width / finalFrame.width :
            finalFrame.width / initialFrame.width
        
        let yScaleFactor = presenting ?
            initialFrame.height / finalFrame.height :
            finalFrame.height / initialFrame.height
         let scaleTransform = CGAffineTransform(scaleX: xScaleFactor, y: yScaleFactor)// 宽缩放为原来的 xScaleFactor 倍，高缩放为原来 的yScaleFactor 倍，中心点位置不变
        
        if presenting {
            herbView.transform = scaleTransform
            herbView.center = CGPoint( x: initialFrame.midX, y: initialFrame.midY)
            herbView.clipsToBounds = true
        }
        
        containerView.addSubview(toView)
        containerView.bringSubview(toFront: herbView)
        
        UIView.animate(withDuration: duration, delay:0.0,
                       usingSpringWithDamping: 0.4, initialSpringVelocity: 0.0, animations: {
            herbView.transform = self.presenting ?
               CGAffineTransform.identity : scaleTransform
            herbView.center = CGPoint(x: finalFrame.midX, y: finalFrame.midY)
        }, completion: { _ in
            if !self.presenting {
                self.dismissCompletion?()
            }
            transitionContext.completeTransition(true)
        })
    }
    
}
```

1. 该类实现 UIViewControllerAnimatedTransitioning 协议。
2. 方法 transitionDuration(using transitionContext: UIViewControllerContextTransitioning?) 设定动画时间。
3. 方法 animateTransition(using transitionContext: UIViewControllerContextTransitioning)  设置具体的动画效果。
xScaleFactor ，yScaleFactor 两行代码，检测初始和最终动画帧，然后计算在每个视图之间设置动画时需要在每个轴上应用的比例因子。

在 ViewController 中定义全局变量 let transition = PopAnimator() , 实现 UIViewControllerTransitioningDelegate 代理方法： 

```
extension ViewController: UIViewControllerTransitioningDelegate {
    
    func animationController(forPresented presented: UIViewController, presenting: UIViewController, source: UIViewController) -> UIViewControllerAnimatedTransitioning? {
        transition.originFrame = selectedImage!.superview!.convert(selectedImage!.frame, to: nil)
        
        transition.presenting = true
        selectedImage!.isHidden = true
        
        return transition
    }
    
    func animationController(forDismissed dismissed: UIViewController) -> UIViewControllerAnimatedTransitioning? {
        transition.presenting = false
        return transition
    }
}
```
这样弹出模态视图的时候，就会执行 PopAnimator 管理类的动画了。

最终效果图:

![Presentation Animations](https://upload-images.jianshu.io/upload_images/130752-062c68114f370e4c.gif?imageMogr2/auto-orient/strip)
