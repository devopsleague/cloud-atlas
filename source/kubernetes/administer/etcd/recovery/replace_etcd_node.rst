.. _replace_etcd_node:

===============
替换etcd节点
===============

当替换一个etcd节点，数显移除被替换节点然后在加入替代它的节点是非常重要的。etcd采用基于投票模式(quorum model)的分布式一致性；要求(n/2)+1成员必须达成一致(agree on a proposal)，然后才能提交内容到集群。这些提议(proposals)包括key-value更新，以及成员关系改变。这种投票模式避免了潜在到脑裂不一致。但是，如果永久性quorum丢失则是灾难性的。

之所以不应该在下线一个etcd node之前就加入新的节点，是因为新加入的节点会导致集群错误配置和不能新加入节点：

当3成员的etcd cluster出现一个宕机节点，此时etcd集群依然能够处理业务，因为quorum数量是2，而2个成员依然是法定多数。然而，添加新成员到3成员的集群就会要求quorum增加到3，因为在4个节点的集群要求3个投票才能决定。

由于quorum增加，这个扩展节点实际上不能起到冗灾作用。因为只要一个节点故障，就会导致集群出现配置错误并且再也不能增加修复节点。这种情况下，没有任何方法恢复quorum：因为此时etcd集群是4个members，2个member宕机(最初宕机的节点和新增加的节点或任意节点)和2个member运行，此时无法获得多数票判决，所以etcd默认会拒绝任何试图添加的用于故障恢复的节点，而且由于只有2个member活着，也不能满足quorum要求，所以集群就无法更新数据。

所以，在没有将故障节点移除之前就贸然添加新节点，存在巨大的风险：新加入的成员节点绝对不能配置错误，如果新加入成员配置错误，就会导致整个etcd集群被摧毁。基于这个原因，你在替换故障etcd节点时，应该遵循操作规范：即首先移除故障节点，然后再添加新的etcd节点，以确保quorum数量不变化。这样即使新加入节点不能正常工作，依然可以移除这个新节点(因为quorum保持在live members是可控的)。

参考
======

- `etcd FAQ - Should I add a member before removing an unhealthy member? <https://etcd.io/docs/v3.4.0/faq/#should-i-add-a-member-before-removing-an-unhealthy-member>`_
