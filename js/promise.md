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
      this.value = value
      this.status = 'fulfilled'
      this.onFulfilledCallbacks.forEach(item => item(value))
    }
    this.reject = reason => {
      this.reason = reason
      this.status = 'rejected'
      this.onRejectedCallbacks.forEach(item => item(reason))
    }

    try {
      excutor(this.resolve, this.reject)
    } catch (error) {
      this.reject(error)
    }
  }

  then = (onFulfilled, onRejected) => {
    if (this.status === 'pending') {
      return new MPromise((resolve, reject) => {
        this.onFulfilledCallbacks.push(() => {
          let x = onFulfilled(this.value)
          x instanceof MPromise ? x.then((resolve, reject)) : resolve(x)
        })
        this.onRejectedCallbacks.push(() => {
          let x = onRejected(this.reason)
          x instanceof MPromise ? x.then((resolve, reject)) : resolve(x)
        })
      })
    }
    if (this.status === 'fulfilled') {
      return new MPromise((resolve, reject) => {
        try {
          let x = onFulfilled(this.value)
          x instanceof MPromise ? x.then((resolve, reject)) : resolve(x)
        } catch (error) {
          reject(error)
        }
      })
    }
    if (this.status === 'rejected') {
      return new MPromise((resolve, reject) => {
        try {
          let x = onRejected(this.reason)
          x instanceof MPromise ? x.then((resolve, reject)) : resolve(x)
        } catch (error) {
          reject(error)
        }
      })
    }
  }

  catch = fn => {
    return this.then(null, fn)
  }
}

const mp = new MPromise((resolve, reject) => {
  setTimeout(() => {
    reject(456)
  }, 1000)
})

mp.then(res => {
  console.log(res)
}).catch(r => {
  console.log(r)
})
```
