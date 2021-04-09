##### RN优化

- 首屏渲染优化：处理JS Bundle包大小、文件压缩、缓存

- UI更新优化

  - 减少更新或者合并多个更新
  - 提高组件响应速度：
    - setNativeProps直接在底层更新Native组件属性（其实没有解决JS端与Native端的数据同步问题）
    - 立即执行更新回调
  - 动画优化
    - 通过使用Annimated类库，一次性把更新发送到Native端，由Native端自己负责更新
    - 把一些耗时操作放到动画与UI更新之后执行
    - 如果可以尽量不要用动画

- ## shouldComponentUpdate

  - 判断值是否变化来决定是否要刷新

-  PureComponent

  - 会自动检查组件是否需要重新渲染，减少了应用中的渲染次数

- 把只需要更新的参数放在setState()中

- 使用FlatList来加载列表

- ### removeClippedSubviews大列表设置可以增加性能，但是有可能出现问题，如内容无法显示

- ### initialNumToRender默认先初始化列表的个数，让列表先展示出来一部分，再去渲染其他部分

> 桥接
>
> JS桥接原生：先定义一个类继承自ReactContextBaseJavaModule，然后重写getName()方法，返回这个类的标识，然后可以定义自己的方法加上@ReactMethod，然后就可以在JS端调用了，调用前需要引入NativeModeules
>
> JS使用原生UI组件：先定义一个类继承自ViewGroupManager，然后重写getName()方法，返回这个类的标识，然后重写createViewInstance()返回自定义的原生组件，创建原生prop方法添加@ReactProp，然后在JS中import requireNativeComponent，然后requireNativeComponent("标识")，然后就可以直接以标签的方式使用了

##### Flutter



