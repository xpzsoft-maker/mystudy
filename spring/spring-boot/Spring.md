# Spring
## 1. 动态获取bean
```java
public final class TaskInitiator
        implements  ApplicationContextAware {

    private ApplicationContext applicationContext;
    @Override
    public void setApplicationContext(ApplicationContext applicationContext)
            throws BeansException {
        this.applicationContext = applicationContext;
    }

    public <T> T getBeanByClass(Class<T> type) {
        return (T) applicationContext.getBean(type);
    }
}
```