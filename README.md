# SimplePromiseImplement
A very simple promise implement



<pre><code>

import UIKit

typealias PromiseHandler = (Resolve, Reject)->Void
typealias Resolve = (Any?)throws ->Any?
typealias Reject = (Error?)->Any?
typealias Callback = ()->Void

enum PromiseState:Int{
    case pending = 0
    case resolve = 1
    case reject  = 2
}
struct PromiseError: Error {
}
func delay(_ second:Double, closure:@escaping ()->()) {
    DispatchQueue.main.asyncAfter(
        deadline: DispatchTime.now() + Double(Int64(second * Double(NSEC_PER_SEC))) / Double(NSEC_PER_SEC), execute: closure)
}

class Promise{
    
    var state:PromiseState = .pending
    var value:Any?
    var error:Error?
    var resolveCallbackList = [Callback]()
    var rejectCallbackList = [Callback]()
    var doneCallbackList = [Callback]()
    
    init() {
        
    }
    init(value:Any) {
        fill(value)
    }
    init(callback:@escaping ((@escaping Resolve,@escaping Reject) throws ->Void)) {
        DispatchQueue.main.async {
            do {
                try callback(self.resolve, self.reject)
            } catch {
                self.reject(nil)
            }
        }
    }
    
    func resolve(_ data:Any?) -> Void {
        guard state == .pending else { return }
        value = data
        state = .resolve
        resolveCallbackList.forEach { $0() }
    }
    
    func reject(_ e:Error?) -> Void {
        guard state == .pending else { return }
        error = e
        state = .reject
        rejectCallbackList.forEach { $0() }
    }
    
    func fill(_ data:Any?){
        guard state == .pending else { return }
        value = data
        state = .resolve
    }
    
    func resolvePromise(promise2:Promise, x:Any?, resolve: Resolve?, reject: Reject?)throws {
  
        if let p = x as? Promise{
//            guard p != promise2 else {
//                reject(PromiseError())
//                return
//            }
            
            p.then({ (data) in
                if let p2  = data as? Promise{
                    do {
                        try self.resolvePromise(promise2: promise2, x: p2, resolve: resolve, reject: reject)
                    } catch {
                        reject?(error)
                    }
                    return nil
                }else{
                    do {
                        try resolve?(data)
                    } catch {
                        reject?(error)
                    }
                    return nil
                }
            }, { error in
                reject?(error)
            })
        }else{
            do {
                try resolve?(value)
            } catch {
                reject?(error)
            }
        }
    }
}

extension Promise{
    
    func then(_ onFulfilled:@escaping Resolve) -> Promise {
        return Promise { (resolve, reject) in
            self.resolveCallbackList.append({
                DispatchQueue.main.async {
                    do {
                        try onFulfilled(self.value)
                        self.then(resolve, reject)
                    } catch {
                        reject(error)
                    }
                }
            })
        }
    }
    
    func then(_  onFulfilled: Resolve?, _ onRejected: Reject?) ->Promise{
        var promise2:Promise!
        promise2 = Promise{ (resolve, reject) in
            
            switch self.state {
            case .resolve:
                DispatchQueue.main.async {
                    do {
                        let x = try onFulfilled?(self.value)
                        try self.resolvePromise(promise2: promise2, x: x, resolve: resolve, reject: reject)
                    } catch {
                        reject(error)
                    }
                }
                
            case .reject:
                DispatchQueue.main.async {
                    do {
                        print("reject \(self.value)")
                        let x = onRejected?(self.error)
                        try self.resolvePromise(promise2: promise2, x: x, resolve: resolve, reject: reject)
                    } catch {
                        reject(error)
                    }
                }
            case .pending:
                self.resolveCallbackList.append({
                    DispatchQueue.main.async {
                        do {
                            let x = try onFulfilled?(self.value)
                            try self.resolvePromise(promise2: promise2, x: x, resolve: resolve, reject: reject)
                        } catch {
                            reject(error)
                        }
                    }
                })
                
                self.rejectCallbackList.append({
                    
                    DispatchQueue.main.async {
                        do {
                            let x = onRejected?(self.error)
                            try self.resolvePromise(promise2: promise2, x: x, resolve: resolve, reject: reject)
                        } catch {
                            reject(error)
                        }
                    }
                })
            }
        }
        return promise2
    }
    
    func onFail(_ reject:Reject?) -> Promise{
        return self.then(nil, reject)
    }
    
    func done(_ resolve:Resolve?) -> Promise{
        return self.then(resolve, nil)
    }
}
</code></pre>
