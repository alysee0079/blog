# icon 处理

1. 使用阿里iconfont图标库, 可以上传有色(自带 fill 属性)和无色图标(没有 fill 属性);

2. 有色图标因为自带 fill 属性, 所有无法使用 color 覆盖颜色, 如果要覆盖颜色, (1).需要手动删除 fill 属性; (2).使用svgo-loader自动删除 fill 属性; (3). 上传图标时去色成为无色图标;

3. fill: currentColor: 使用当前元素的 color 属性作为内容颜色, 如果是无色图标想要自动逸颜色需要设置, 有色图标无需设置(本身自带);