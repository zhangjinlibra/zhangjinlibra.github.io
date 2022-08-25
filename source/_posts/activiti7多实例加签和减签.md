---
title: activiti7多实例加签和减签
categories: activiti7
tags: 加签减签
---

> Activit7中没有加签的操作, 为了实现自定的加签和减签操作就需要程序猿自己来实现对应的命令
> 下面是多实例节点初始化的代码跟踪步骤

##### 流程跟踪

###### 大致流程
1. 完成当前任务节点, 如果节点行为是可触发的, 则触发节点离开能力`TriggerableActivityBehavior.trigger()`
2. 获取当前任务节点的下一个连接线, 并设置为Execution的当前执行元素
3. 获取连接线的下一个节点元素, 并设置为Execution的当前执行元素
4. 执行当前元素的行为`ActivityBehavior.execute()`

###### 代码流程
```
// 1. CompleteTaskCmd.executeTaskComplete(), 触发任务节点完成行为
Context.getAgenda().planTriggerExecutionOperation(executionEntity);

// 2. TriggerExecutionOperation.run(), 真正触发代码
((TriggerableActivityBehavior) activityBehavior).trigger(execution, null, null);

// 3. 执行UserTaskActivityBehavior.trigger(DelegateExecution, String, Object)
leave(execution);// 执行节点离开之后的操作

// 4. 当前节点不是多实例, 执行上面的代码
if (!hasLoopCharacteristics()) {
  // 如果不是多实例执行
  super.leave(execution);
} else if (hasMultiInstanceCharacteristics()) {
  // 节点包含多实例执行
  multiInstanceActivityBehavior.leave(execution);
}

// 5. 流程继续向下走, 执行节点流出操作
bpmnActivityBehavior.performDefaultOutgoingBehavior((ExecutionEntity) execution);

// 6. 取出下一个节点, 向下执行
TakeOutgoingSequenceFlowsOperation.run()
// 取出下个节点, 流程继续向下走
execution.setCurrentFlowElement(sequenceFlow);
execution.setActive(true);
outgoingExecutions.add((ExecutionEntity) execution);
Context.getAgenda().planContinueProcessOperation(outgoingExecution);

// 7. 流程继续执行ContinueProcessOperation.run()
// 取出当前节点
FlowElement currentFlowElement = getCurrentFlowElement(execution);
if (currentFlowElement instanceof FlowNode) {
    continueThroughFlowNode((FlowNode) currentFlowElement);
} else if (currentFlowElement instanceof SequenceFlow) {
    // 当前节点是连接线, 走这个分支, 
    continueThroughSequenceFlow((SequenceFlow) currentFlowElement);
}

protected void continueThroughSequenceFlow(SequenceFlow sequenceFlow) {
// 取出下面节点
FlowElement targetFlowElement = sequenceFlow.getTargetFlowElement();
execution.setCurrentFlowElement(targetFlowElement);
// 执行流程继续操作
Context.getAgenda().planContinueProcessOperation(execution);
}

// 8. ContinueProcessOperation.run(), 继续执行代码, 现在执行才是上一个节点完成后的下一个任务节点, 就是多实例节点
if (currentFlowElement instanceof FlowNode) {
    // UserTask执行该分支
    continueThroughFlowNode((FlowNode) currentFlowElement);
}

// 下面是continueThroughFlowNode()方法的代码片段
if (flowNode instanceof Activity && ((Activity) flowNode).hasMultiInstanceLoopCharacteristics()) {
    // 初始化多实例节点
    executeMultiInstanceSynchronous(flowNode);
}

// 下面是executeMultiInstanceSynchronous()方法的代码片段
// 获取多实例处理行为
ActivityBehavior activityBehavior = (ActivityBehavior) flowNode.getBehavior();
// 执行多实例节点能力
executeActivityBehavior(activityBehavior,
                            flowNode);
// 调用子类的能力
activityBehavior.execute(execution);
                            
// 9. ParallelMultiInstanceBehavior.execute()
// 执行, 多实例节点的能力全部在这个类中
MultiInstanceActivityBehavior.execute();
// 创建会签多实例
ParallelMultiInstanceBehavior.createInstances(execution);// // 创建子类执行流
// ParallelGatewayActivityBehavior执行execute()方法
ParallelMultiInstanceBehavior.createInstances()// 创建实例对象
// 创建子执行流的具体的执行代码
Context.getAgenda().planContinueMultiInstanceOperation((ExecutionEntity) execution);
```

###### 加签实现
1. 创建多实例执行流程
2. 初始化子执行流

```
@Override
public Execution execute(CommandContext commandContext) {
	ExecutionEntityManager executionEntityManager = commandContext.getExecutionEntityManager();

	// 根据执行实例ID获取当前执行实例
	String executionId = task.getExecutionId();
	ExecutionEntity currentExecutionEntity = executionEntityManager.findById(executionId);

	// 获取当前执行实例的父实例, 父实例处实现加签操作
	ExecutionEntity miExecution = currentExecutionEntity.getParent();

	// 获取BpmnModel
	String processDefinitionId = task.getProcessDefinitionId();
	BpmnModel bpmnModel = ProcessDefinitionUtil.getBpmnModel(processDefinitionId);

	// 获取当前Activity
	String taskDefinitionKey = task.getTaskDefinitionKey();
	Activity miActivityElement = (Activity) bpmnModel.getFlowElement(taskDefinitionKey);

	// 判断当前执行实例的节点是否是多实例节点
	MultiInstanceLoopCharacteristics multiInstanceLoopCharacteristics = miActivityElement.getLoopCharacteristics();

	// 非多实例节点不能加签
	if (multiInstanceLoopCharacteristics == null) {
		throw new ActivitiException("No multi instance execution can not executed");
	}
	// 串行多实例不能加签
	if (multiInstanceLoopCharacteristics.isSequential()) {
		throw new ActivitiException("Sequential multi instance can not executed");
	}

	// 创建新的子实例
	ExecutionEntity childExecution = executionEntityManager.createChildExecution(miExecution);
	UserTask flowElement = (UserTask) miExecution.getCurrentFlowElement();
	// flowElement.setAssignee(assignee); 并发问题, 需在下面的变量中设置
	childExecution.setActive(true);
	childExecution.setScope(false);
	childExecution.setCurrentFlowElement(flowElement);

	// 设置多实例流程变量
	Integer nrOfInstances = (Integer) miExecution.getVariableLocal(EActiviti.NUMBER_OF_INSTANCES);
	Integer nrOfActiveInstances = (Integer) miExecution.getVariableLocal(EActiviti.NUMBER_OF_ACTIVE_INSTANCES);
	miExecution.setVariableLocal(EActiviti.NUMBER_OF_INSTANCES, nrOfInstances + 1);
	miExecution.setVariableLocal(EActiviti.NUMBER_OF_ACTIVE_INSTANCES, nrOfActiveInstances + 1);

	childExecution.setVariableLocal(EActiviti.LOOP_COUNTER, nrOfInstances);
    childExecution.setVariableLocal(FlowUtil.MULTIINSTANCE_ELEMENT_VARIABLE, assignee);
    commandContext.getAgenda().planContinueMultiInstanceOperation(childExecution);
	return childExecution;
}
```

###### 减签实现
```
@Override
public Void execute(CommandContext commandContext) {
	ExecutionEntityManager executionEntityManager = commandContext.getExecutionEntityManager();

	// 根据执行实例ID获取当前执行实例
	String executionId = task.getExecutionId();
	ExecutionEntity currentExecutionEntity = executionEntityManager.findById(executionId);

	// 获取当前执行实例的父实例, 父实例处实现加签操作
	ExecutionEntity miExecution = currentExecutionEntity.getParent();
	String processDefinitionId = task.getProcessDefinitionId();
	BpmnModel bpmnModel = ProcessDefinitionUtil.getBpmnModel(processDefinitionId);
	String taskDefinitionKey = task.getTaskDefinitionKey();
	Activity miActivityElement = (Activity) bpmnModel.getFlowElement(taskDefinitionKey);
	MultiInstanceLoopCharacteristics multiInstanceLoopCharacteristics = miActivityElement.getLoopCharacteristics();

	if (miActivityElement.getLoopCharacteristics() == null) {
		throw new ActivitiException("No multi instance execution found for execution id " + executionId);
	}

	if (!(miActivityElement.getBehavior() instanceof MultiInstanceActivityBehavior)) {
		throw new ActivitiException("No multi instance behavior found for execution id " + executionId);
	}

	if (multiInstanceLoopCharacteristics.isSequential()) {
		throw new ActivitiException("Sequential multi instance can not executed " + executionId);
	}

	// 删除执行流程
	executionEntityManager.deleteChildExecutions(currentExecutionEntity, "Delete MI Execution", false);
	executionEntityManager.deleteExecutionAndRelatedData(currentExecutionEntity, "Delete MI Execution", false);

	Integer numberOfCompletedInstances = (Integer) miExecution.getVariable(EActiviti.NUMBER_OF_COMPLETED_INSTANCES);
	miExecution.setVariableLocal(EActiviti.NUMBER_OF_COMPLETED_INSTANCES, numberOfCompletedInstances);

	ParallelMultiInstanceBehavior prallelMultiInstanceBehavior = (ParallelMultiInstanceBehavior) miActivityElement
			.getBehavior();
	prallelMultiInstanceBehavior.leave(currentExecutionEntity);

	return null;
}
```