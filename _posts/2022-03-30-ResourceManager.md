# Resource Manager

Hadoop version is 2.7

## Overview

![ResourceManager](/assets/img/rm/RM.png)

Overview of the process that an application was submitted to RM.
There are several steps as following.

- Submit
  - Client submit the application to the ResourceManager, ResourceManager will create an RMApp object to track all status and report back to Client.
- Create
  - RMApp will create an RMAttempt, which means the first attempt to complete the job.
- Into the Pending Ordering Queue.
  - The RMAttempt will be put into pending Ordering Queue. At the process, there will be several check like resource limit or ACLs.
- Into Ordering Queue.
  - RM will put the RMAttempt into the Ordering Queue after RMAttempt meet the all requirement. When the RMAttempt is in the ordering queue, it will generate a request to run an AM container. Before RM give a container to RMAttempt, RMAttempt will waiting in the ordering Queue.
- Heart Beat.
  - NodeManager will send a Heart beat event to RM periodically to report health status and remaining resource. 
- Assign Container to this Node.
  - When Scheduler(A thread in RM) receive the node heart beat event, scheduler will give the resource to the RMAttempt, which means RMAttempt could launch a container in this Node.
- Launch AM Container
  - RMAttempt will always launch the AM(application master) container first.
- AM register to RM.
  - After AM is running, it will register back to RM.

After that, AM will allocate containers for this RMAttempt.

## Activate Application
This part, let's talk about the activating application (Put RMAttempt from PendingOrderingQueue to OrderingPolicyQueue).

![Queue](/assets/img/rm/Queue.png)
**Put RMAttempt into PendingOrderingPolicy**

For this steps, RM will check the Queue name / Queue acls and so on. RM will not check the Resource Limit at this step.

**Put RMAttempt from PendingOrderingPolicyQueue to OrderingPolicyQueue**

For this steps, There are several limitation need to be met.
- Calculate AM Resource Limit: AMLimit = Queue_Capacity * MaxAMPercentage.
  - AMNeed + AMUserd > AMLimit. ---> Reject to be put in OrderingPolicyQueue
- Calculate User Limit AM
  - effectiveUserLimit $$ = Max(userLimit, \frac{1}{ActiveUser + PendingUser})$$
  - UserAMLimit = Queue_Capacity * effectiveUserLimit * AMPercentage * Factor.
  - UserAMLimit > AMLimit ---> Reject to be put in OrderingPolicyQueue.

If resource requirement is met, then RMAttempt will be moved from PendingOrderingQueue to OrderingPolicy. Then RMAttempt wait for NodeUpdateEvent to get Container.

![Parameters](/assets/img/rm/Parameters.png)

**Related Metrics**

![QueueMetric](/assets/img/rm/QueueMetric.png)

## Node Update

![NodeUpdate](/assets/img/rm/NodeUpdate.png)

1. How to choose the Child Queue.  (Reference:  Class PriorityUtilizationQueueOrderingPolicy)
   - Queue which has access to this node partition go first.
      - If partition is NO_Label, return equal.
      - If QueueA has access, but QueueB not, QueueA go first.
   - Queue with high priority &  low usage ratio go first. ($$usageRation = \frac{usedResource}{QueueGarenteedResource}$$)
      - If priority is equal. Queue with Low UsageRatio go first.
      - If priority isn’t equal.
        - If both Queue are under or over guaranteed resource. Queue with high priority go first.  
        - If not,  Queue with low usageRation go first.
   - Queue with high capacity go first.

2. Queue Traverse Method.
- The Scheduler will traverse using DFS from root. The method to rank all child queue is above.

3. How to order Application.  (Reference: Class FifoOrderingPolicy)
   - Only Traverse Ordering Policy Queue.
   - Application with high priority go first.
   - First in First out

**Related Metric**
- SchedulerEventQueueSize:
  - Ticket: https://issues.apache.org/jira/browse/YARN-10771
  

## Assign Container

**How to choose Application to assign container**
- Can assign to this Queue
  - LeafQueueLimit =  min(Parent.limit - Parent.used + Leaf.used,   Leaf.max)
  - If QueueUsed > LeafQueueLimit --> Can’t assign to this Queue 
- Can assign to this User
  - Calculate User Limit:
    - currentCapacity = consumed < queueCapacity ? queueCapacity : consumed + miniAllocation
    - userLimit = $$max⁡({\frac{currentCapacity}{\#activeUsers}, currentCapacity \times UserLimitPercentage})$$
    - maxUserLimit = $$currentCapacity \times UserLimitFactor$$
    - finalUserLimit = $$min({userLimit,  maxUserLimit})$$
  - If UserUsed > finalUserLimit --> Can’t assign to this Users.

**How to Avoid useless traverse**

![AvoidUselessTraverse](/assets/img/rm/Avoid.png)

- AM Send ResourceRequest
  - ① Application AM request Resource, and send to the ApplicationMasterService.
  - ② Application will add resource request and increase LeafQueue’s pending Resource. 
  - ③ LeafQueue’s pending Resource increased. (Field is QueueUsage)
  - ④ ParentQueue’s pending Resource will increased as well.
- Scheduler traverse the Tree
  - Check whether child queues have pending resource request.
    - If not, just ignore and traverse next child queue.
    - If so, traverse this queue.


## Handle Node Heart Beat

![HeartBeat](/assets/img/rm/HeartBeat.png)

1. NodeManager sends Heart Beat to ResourceTrackerService.
2. ResourceTrackerService will check whether it's a valid request.
  - Check RequestID = LastRequestID + 1
  - After that, ResourceTrackerService will reply the NodeManager, to tell NodeManager that its HeartBeat has been received.
3. ResourceTrackerService will generate a status update event.
4. RMNode Handler will handle status update event.
  - Check RMNode.nextHeartBeat = true. 
  - If false, means the last heart beat has not been processed,Then just drop this event.
5. Send NodeUpdateEvent to Scheduler (capacity-scheduler), this NodeUpdateEvent will be added in the EventQueue.
6. Handle NodeUpdateEvent, details at section "Node Update"
7. Assign Container to NodeManager.

Here is a obersvation.
- NodeUpdateProcessSpeed > NodeUpdateGenerateSpeed => EventQueue size --> 0
- NodeUpdateProcessSpeed < NodeUpdateGenerateSpeed => EventQueue size == NodeManager Num