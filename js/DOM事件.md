# DOM 事件

#### 事件级别

DOM0:
element.onclick = function () {}

DOM2:
element.addEventListener('click', function (){}, false)

DOM3:
element.addEventListener('keyup', function (){}, false)

#### 事件模式

#### 事件流

捕获阶段 Capture Phase; 从上到下, 层层传递, 直到目标接收
目标阶段 Target Phase; 确认目标, 进行处理
冒泡阶段 Bubbling Phase; 处理结束, 往上传递.

#### 自定义事件

```javascript
const ele = document.querySelector('.el')
const event = new Event('test')
ele.addEventListener('test', function () {
  console.log(22)
})
ele.dispatchEvent(event)
```
