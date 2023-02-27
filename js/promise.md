```javascript
class MPromise {
  constructor(excutor) {
    this.status = 'pending'
    this.value = null
    this.reason = null
    // 缓存数据
    this.onFulfilledCallbacks = []
    this.onRejectedCallbacks = []

    this.resolve = value => {
      if (this.status === 'pending') {
        this.value = value
        this.status = 'fulfilled'
        this.onFulfilledCallbacks.forEach(item => item(value))
      }
    }
    this.reject = reason => {
      if (this.status === 'pending') {
        this.reason = reason
        this.status = 'rejected'
        this.onRejectedCallbacks.forEach(item => item(reason))
      }
    }

    try {
      excutor(this.resolve, this.reject)
    } catch (error) {
      this.reject(error)
    }
  }

  then = (onFulfilled, onRejected) => {
    if (this.status === 'pending') {
      this.onFulfilledCallbacks.push(onFulfilled)
      this.onRejectedCallbacks.push(onRejected)
    }
  }
}

const mp = new MPromise((resolve, reject) => {
  setTimeout(() => {
    resolve(123)
  }, 0)
})

mp.then(res => {
  console.log(res)
})
```
