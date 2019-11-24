## Spring BeanFactoryPostProcessor 原理

`BeanFactoryPostProcessor` BeanFactory 的后置处理器；在 BeanFactory 标准初始化之后调用，所有 Bean 定义已经保存加载到 beanFactory 中但是 bean 的实例还未创建，