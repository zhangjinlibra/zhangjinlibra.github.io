---
title: activiti7任务回退
categories: activiti7
tags: 任务回退
---

#### 引言
> 最近公司需要搭建一个类似钉钉、飞书的流程系统，小公司自行实现流程框架比较吃力，于是选择了Activiti7。那么问题来了，一些定制化的业务，比如：节点回退、加签、减签等Activiti没有默认的实现，这就需要自己来实现。刚开始接触Activiti7也是各种头大，经过一个星期啃源码，终于入了门。下面是节点回退的实现思路和一些代码做个分享

#### 思路
1. 根据BpmnModel查询上一个任务节点，如果是单节点任务，完成当前任务，任务流转到上一个节点
2. 如果上一个节点是多节点任务（排他网关或者是多个支线流入）但是不是并行任务（并行网关、包含网关），查询历史任务，找出上历史任务中上一个完成的任务，完成当前任务，任务流转到上一个任务
3. 如果上一个节点是并行网关、包含网关，这类情况直接禁止回退就行。钉钉、飞书不会涉及到这类的情况

#### 实现
```java
/**
 * 回退到上一个任务节点命令 <br>
 * <b>上个节点是并行网关时, 不能回退</b>
 * 
 * @author zhangjin
 *
 */
@Slf4j
public class BackOneStepCmd implements Command<Void>, Serializable {
	private static final long serialVersionUID = 1L;

	// 当前任务
	private Task task;
	private String backReason;

	public BackOneStepCmd(Task task) {
		this.task = Objects.requireNonNull(task, "Task can not be null");
	}

	public BackOneStepCmd(Task task, String deleteReason) {
		this(task);
		this.backReason = deleteReason;
	}

	@Override
	public Void execute(CommandContext commandContext) {
		// 查询BpmnModel
		String processDefinitionId = task.getProcessDefinitionId();
		BpmnModel bpmnModel = ProcessDefinitionUtil.getBpmnModel(processDefinitionId);

		// 当前节点
		String activityId = task.getTaskDefinitionKey();
		UserTask userTask = (UserTask) bpmnModel.getFlowElement(activityId);

		// 检测是否可以回退
		// 当前节点流入是并行, 包含网关时禁止回退
		if (ActivitiUtil.checkCurrentFlowNodeIsParallelIncomingBaseProcessDefinition(userTask)) {
			throw new RuntimeException("Current node is parallel gateway, can not execute back cmd");
		}

		// 查询上一个UserTask
		UserTask lastUserTask = ActivitiUtil.getLastUserTask((TaskEntity) task, userTask);
		log.info("back one step to {}", lastUserTask.getId());

		// 建立回退方向
		SequenceFlow backStepSequenceFlow = new SequenceFlow();
		backStepSequenceFlow.setId(FlowNodeIDFactory.randomSequenceFlowId());
		backStepSequenceFlow.setSourceFlowElement(userTask);
		backStepSequenceFlow.setTargetFlowElement(lastUserTask);

		// 并发问题
		// 记录当前节点的原活动方向
		// List<SequenceFlow> originalOutgoingFlows =
		// ListUtil.toList(currentUserTask.getOutgoingFlows());
		// 设置新的流程走向
		// currentUserTask.setOutgoingFlows(ListUtil.toList(backStepSequenceFlow));
		// 完成任务
		// taskService.complete(task.getId());
		// 还原源活动方向
		// currentUserTask.setOutgoingFlows(originalOutgoingFlows);

		// 解决并发流向出错问题
		// 查询任务, 删除的任务不需要触发任何事件
		TaskEntityManager taskEntityManager = commandContext.getTaskEntityManager();
		TaskEntity taskEntity = taskEntityManager.findById(task.getId());

		// 挂起的任务需要先激活
		if (taskEntity.getDelegationState() != null
				&& taskEntity.getDelegationState().equals(DelegationState.PENDING)) {
			throw new ActivitiException("A delegated task cannot be completed, but should be resolved instead.");
		}

		// 完成当前任务
		taskEntityManager.deleteTask(taskEntity, backReason, false, false);
		ExecutionEntityManager executionEntityManager = commandContext.getExecutionEntityManager();
		ExecutionEntity executionEntity = executionEntityManager.findById(taskEntity.getExecutionId());

		// 检测当前任务节点类型
		MultiInstanceLoopCharacteristics multiInstanceLoopCharacteristics = userTask.getLoopCharacteristics();
		if (multiInstanceLoopCharacteristics != null) {
			if (multiInstanceLoopCharacteristics.isSequential()) {
				// 序签提前结束
				leaveSequentialMultiInstance(executionEntity, backStepSequenceFlow);
			} else {
				// 会签或签提前结束
				leaveParallelMultiInstance(executionEntity, backStepSequenceFlow);
			}
		} else {
			// 一般任务节点
			leaveUserTask(executionEntity, backStepSequenceFlow);
		}
		return null;
	}

	/**
	 * 当前节点流转
	 * 
	 * @param executionEntity
	 * @param backStepSequenceFlow
	 */
	protected void leaveUserTask(ExecutionEntity executionEntity, SequenceFlow backStepSequenceFlow) {
		executionEntity.setCurrentFlowElement(backStepSequenceFlow);
		Context.getAgenda().planTakeOutgoingSequenceFlowsOperation(executionEntity, true);
	}

	/**
	 * 平行节点流转
	 * 
	 * @param childExecution
	 * @param backStepSequenceFlow
	 */
	protected void leaveParallelMultiInstance(ExecutionEntity childExecution, SequenceFlow backStepSequenceFlow) {
		Context.getCommandContext().getHistoryManager().recordActivityEnd(childExecution, null);
		ExecutionEntity multiInstanceRootExecution = getMultiInstanceRootExecution(childExecution);

		// 清除变量和子执行线
		childExecution.removeVariableLocal(EActiviti.LOOP_COUNTER);
		multiInstanceRootExecution.removeVariableLocal(EActiviti.NUMBER_OF_COMPLETED_INSTANCES);
		multiInstanceRootExecution.removeVariableLocal(EActiviti.NUMBER_OF_ACTIVE_INSTANCES);
		multiInstanceRootExecution.removeVariableLocal(EActiviti.NUMBER_OF_INSTANCES);
		this.deleteChildExecutions((ExecutionEntity) multiInstanceRootExecution, false, Context.getCommandContext());

		// 流转到新的节点
		multiInstanceRootExecution.setScope(false);
		multiInstanceRootExecution.setMultiInstanceRoot(false);
		multiInstanceRootExecution.setActive(true);
		this.leaveUserTask(multiInstanceRootExecution, backStepSequenceFlow);
	}

	/**
	 * 序签节点流转
	 * 
	 * @param childExecution
	 * @param backStepSequenceFlow
	 */
	protected void leaveSequentialMultiInstance(ExecutionEntity childExecution, SequenceFlow backStepSequenceFlow) {
		Context.getCommandContext().getHistoryManager().recordActivityEnd(childExecution, null);
		ExecutionEntity multiInstanceRootExecution = getMultiInstanceRootExecution(childExecution);

		// 清除变量和子执行线
		childExecution.removeVariableLocal(EActiviti.LOOP_COUNTER);
		multiInstanceRootExecution.removeVariableLocal(EActiviti.NUMBER_OF_COMPLETED_INSTANCES);
		multiInstanceRootExecution.removeVariableLocal(EActiviti.NUMBER_OF_ACTIVE_INSTANCES);
		multiInstanceRootExecution.removeVariableLocal(EActiviti.NUMBER_OF_INSTANCES);
		Context.getCommandContext().getExecutionEntityManager()
				.deleteChildExecutions((ExecutionEntity) multiInstanceRootExecution, "MI_END", false);

		// 流转到新的节点
		multiInstanceRootExecution.setScope(false);
		multiInstanceRootExecution.setMultiInstanceRoot(false);
		multiInstanceRootExecution.setActive(true);
		this.leaveUserTask(multiInstanceRootExecution, backStepSequenceFlow);
	}

	// TODO: can the ExecutionManager.deleteChildExecution not be used?
	protected void deleteChildExecutions(ExecutionEntity parentExecution, boolean deleteExecution,
			CommandContext commandContext) {
		// Delete all child executions
		ExecutionEntityManager executionEntityManager = commandContext.getExecutionEntityManager();
		Collection<ExecutionEntity> childExecutions = executionEntityManager
				.findChildExecutionsByParentExecutionId(parentExecution.getId());
		if (CollectionUtil.isNotEmpty(childExecutions)) {
			for (ExecutionEntity childExecution : childExecutions) {
				deleteChildExecutions(childExecution, true, commandContext);
			}
		}

		if (deleteExecution) {
			executionEntityManager.deleteExecutionAndRelatedData(parentExecution, null, false);
		}
	}

	protected ExecutionEntity getMultiInstanceRootExecution(ExecutionEntity executionEntity) {
		ExecutionEntity multiInstanceRootExecution = null;
		ExecutionEntity currentExecution = executionEntity;
		while (currentExecution != null && multiInstanceRootExecution == null && currentExecution.getParent() != null) {
			if (currentExecution.isMultiInstanceRoot()) {
				multiInstanceRootExecution = currentExecution;
			} else {
				currentExecution = currentExecution.getParent();
			}
		}
		return multiInstanceRootExecution;
	}

}
```
##### 其他相关的类
```java
public class ActivitiUtil {

	/**
	 * 基于流程定义查询任务上一个任务节点
	 * 
	 * @param userTask
	 * @return
	 */
	public static List<UserTask> getLastUserTaskBaseProcessDefinition(FlowNode userTask) {
		if (Objects.isNull(userTask)) {
			return null;
		}

		List<SequenceFlow> sequenceFlows = userTask.getIncomingFlows();
		List<UserTask> lastUserTasks = new ArrayList<UserTask>();
		if (CollectionUtil.isNotEmpty(sequenceFlows)) {
			for (SequenceFlow sequenceFlow : sequenceFlows) {
				FlowNode flowNode = (FlowNode) sequenceFlow.getSourceFlowElement();
				if (flowNode instanceof UserTask) {
					lastUserTasks.add((UserTask) flowNode);
				} else {
					lastUserTasks.addAll(getLastUserTaskBaseProcessDefinition(flowNode));
				}
			}
		}

		return lastUserTasks;
	}

	/**
	 * 基于流程定义和历史任务获取当前节点的上一个任务节点
	 * 
	 * @param taskEntity
	 * @param userTask
	 * @return
	 */
	public static UserTask getLastUserTask(TaskEntity taskEntity, FlowNode userTask) {
		UserTask lastUserTask = null;
		List<UserTask> lastUserTasks = getLastUserTaskBaseProcessDefinition(userTask);
		BpmnModel bpmnModel = ProcessDefinitionUtil.getBpmnModel(taskEntity.getProcessDefinitionId());
		if (CollectionUtil.isNotEmpty(lastUserTasks)) {
			if (lastUserTasks.size() == 1) {
				lastUserTask = lastUserTasks.get(0);
			} else {// 多分支流入情况处理
				List<String> lastUserTaskDefinitionKeys = lastUserTasks.stream().map(UserTask::getId)
						.collect(Collectors.toList());

				// 查询上一个完成任务的节点
				String executionId = taskEntity.getExecutionId();
				List<HistoricTaskInstance> historicTaskInstances = SpringContext.getBean(HistoryService.class)//
						.createHistoricTaskInstanceQuery()//
						.executionId(executionId)//
						.finished()//
						.orderByTaskCreateTime().desc()//
						.list();

				if (CollectionUtil.isNotEmpty(historicTaskInstances)) {
					HistoricTaskInstance historicTaskInstance = historicTaskInstances.stream()
							.filter(item -> lastUserTaskDefinitionKeys.contains(item.getTaskDefinitionKey()))
							.findFirst().orElse(null);
					if (Objects.nonNull(historicTaskInstance)) {
						String lastActivityId = historicTaskInstance.getTaskDefinitionKey();
						lastUserTask = (UserTask) bpmnModel.getFlowElement(lastActivityId);
					}
				}
			}
		}

		if (Objects.isNull(lastUserTask)) {
			throw new FlowableException("Last task is null");
		}
		return lastUserTask;
	}

	/**
	 * 基于流程定义检查当前节点流入是否是并行网关, 包含网关
	 * 
	 * @param userTask
	 * @return
	 */
	public static boolean checkCurrentFlowNodeIsParallelIncomingBaseProcessDefinition(FlowNode userTask) {
		if (Objects.nonNull(userTask)) {
			List<SequenceFlow> sequenceFlows = userTask.getIncomingFlows();
			if (CollectionUtil.isNotEmpty(sequenceFlows)) {
				for (SequenceFlow sequenceFlow : sequenceFlows) {
					FlowNode flowNode = (FlowNode) sequenceFlow.getSourceFlowElement();
					if (flowNode instanceof InclusiveGateway || flowNode instanceof ParallelGateway) {
						return true;
					}
				}
			}
		}
		return false;
	}

}
```

```java
/**
 * activiti的内置变量和属性名称
 *
 */
public interface EActiviti {

	// 内置变量
	// 多实例执行变量_ACTIVIT_NULL_ASSIGNEE
	String NUMBER_OF_INSTANCES = "nrOfInstances";
	String NUMBER_OF_ACTIVE_INSTANCES = "nrOfActiveInstances";
	String NUMBER_OF_COMPLETED_INSTANCES = "nrOfCompletedInstances";
	// 多实例分支执行变量
	String LOOP_COUNTER = "loopCounter";

	// 内置属性
	String BUSINESS_KEY = "businessKey";
	String VARIABLES = "variables";
	String INITIATOR = "initiator";

	// 候选用户组前缀
	String CANDIDATE_GROUPS_PREFIX = "_GROUPS:";
	// 候选用户前缀
	String CANDIDATE_USERS_PREFIX = "_USERS:";
	// 节点空执行人
	String ACTIVITI_NULL_ASSIGNEE = "_ACTIVIT_NULL_ASSIGNEE";
	// 自动通过机器人
	String ACTIVITI_AUTO_PASS_ROBOT = "_ACTIVITI_AUTO_PASS_ROBOT";
	// 自动拒绝机器人
	String ACTIVITI_AUTO_REFUSE_ROBOT = "_ACTIVITI_AUTO_REFUSE_ROBOT";
	// 节点自动skip表达式
	String ACTIVITI_SKIP_EXPRESSION = "_ACTIVITI_SKIP_EXPRESSION_ENABLED";
}
```

```java
/**
 * 流程定义节点id生成器
 */
public class FlowNodeIDFactory {
        // 基于雪花算法的id
	public static String randomSequenceFlowId() {
		return IDFactory.snowflakeId();
	}
}
```