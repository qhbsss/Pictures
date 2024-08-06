# 事务



错误用法，事务不会执行，Idea提示`@Transactional 自调用(实际上是目标对象内的方法调用目标对象的另一个方法)在运行时不会导致实际的事务`

```
private void computeLayoutBySubnetId(List<ResourceRecalculationEvent> events, String subnetId) throws OSSException {
        savedHandler(newGroups, context);
}

@Transactional(rollbackFor = Exception.class)
public void savedHandler(List<GroupInfo> newGroups, TopoContext context) {
    // dao 操作
}
```

这段代码使用了Spring事务，Spring为当前类生成了一个代理类，然后在执行相关方法时，会判断这个方法有没有@Transactional注解，如果有的话，则会开启一个事务。



目标对象内部的自我调用将无法实施切面中的增强，如图所示



解决方法：
1、将事务方法放到另一个类中进行调用，即符合了在对象之间调用的条件。
2、改成非注解方式调用

```
public void unSavedHandler(List<GroupInfo> newGroups, TopoContext context) {
    DefaultTransactionDefinition def = new DefaultTransactionDefinition();
    def.setPropagationBehavior(TransactionDefinition.PROPAGATION_REQUIRED);
    TransactionStatus status = transactionManager.getTransaction(def);

    try {
        // dao 操作
        transactionManager.commit(status);
    } catch (Exception e) {
        transactionManager.rollback(status);
    }
}
```
