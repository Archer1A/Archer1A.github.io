# BeanDefinition 介绍

  BeanDefinition 是一个接口,在Spring 中存在三种实现： **RootBeanDefinition**、 **ChildBeanDefinition**和
**GenericBeanDefinition**。三种实现均继承了AbstractBeanDefinition,其中BeanDefinition是配置文件`<Bean>`
的元素标签在容器中的内部表现形式。`<Bean>`元素标签拥有class、scope、lazy-init等配置属性，BeanDefinition同
样提供了beanClass、scope、lazyInit属性，BeanDefinition和`<Bean>`中的属性是一一对应的。其中RootBeanDefinition
是最常见的实现类，它对应一般性的`<bean>`元素标签。
  Spring通过BeanDefinition将配置文件中的`<bean>`配置信息转换为容器的内部表示，并将这些BeanDefinition注册到
BeanDefinitionRegistry中。Spring容器的BeanDefinitionRegistry就像Spring配置信息的内存数据库，主要以map的形
式存储，后续操作直接从BeanDefinitionRegistry中读取配置信息。

继承树：
![继承树](../pic/Beandefinition.png) 