---
title: 'Scale-out 시 Scheduling 중복 실행 막기'
excerpt: "Spring Event와 Queue를 이용해 스케줄링 제어"

categories:
  - java
tags:
  - java
last_modified_at: 2023-09-18T19:14:00-05:00
---

# 이 글을 쓰는 이유

저희 서비스에서는 주기적으로 충전기의 상태와 정보를 업데이트하거나, 통계를 저장하는 스케줄링 작업이 있습니다.
지금의 저희 서버는 단일 서버로 구성되어있어 문제가 없지만, 만약 **서버를 scale-out** 하게 된다면 어떻게 될까요?

**똑같은 schedule이 중복**되어 실행될 것입니다. 그렇다고 어떤 서버는 schedule을 동작하지 않도록 하고, 어떤 서버는 schedule을 동작하도록 한다면 스케줄이 동작하는 서버가 다운된다면 동작하는
서버의 다운타임만큼 저희 서버의 데이터를 최신화할 수 없고, 최신화가 중요한 저희 서비스에서는 사용자의 불만을 초래할 수 있습니다.

## 구현해보기

Schedule 정보를 어떻게 다른 환경에서 같이 공유하여 관리할 수 있을까요?
간단히 생각하면 Local 환경이 아닌, Global 환경에서 정보를 관리하면 될 것 같습니다.

따라서 Schedule의 정보를 저장할 수 있는 테이블을 아래의 Entity 의 필드와 같이 생성해보겠습니다.

```java
@Entity
public class ScheduleTask extends BaseEntity {

  @Id
  private String id;

  private String jobName;

  @Enumerated(EnumType.STRING)
  private JobStatus status;
}
```

먼저 id는 해당 스케줄을 구분할 수 있는 id여야 할 것입니다. 가장 쉽게 정할 수 있는 id는 스케줄의 **job 이름**과,
Schedule으로 등록한 **시간**을 조합하여 생성한다면 unique하고 분산 환경에서도 쉽게 구분할 수 있는 id가 될 것 입니다.

그리고 아래와 같은 Business Logic 있다고 가정하겠습니다.
```java
@Service
public class BusinessLogic {

  private final ApplicationEventPublisher applicationEventPublisher;

  @Scheduled(cron = "0/2 * * * * *")
  public void complexJob() {
    log.info("복잡한 Job 시작");
  }

  @Scheduled(cron = "0/4 * * * * *")
  public void moreComplexJob() {
    log.info("좀 더 복잡한 Job 시작");
    try {
      Thread.sleep(3000);
    } catch (InterruptedException e) {
      throw new RuntimeException(e);
    }
  }
}
```
하나는 매 2초마다 실행 후 바로 종료되고, 하나는 매 4초마다 실행 후 3초의 대기와 종료되는 메서드입니다.  
이런 스케줄은 어떻게 동작할까요? 저는 당연히 2초와 4초마다 해당 메서드가 실행될 줄 알았습니다.

로그를 살펴보면 아래와 같은 결과가 발생했습니다.
![log](https://github.com/drunkenhw/comments/assets/106640954/5e275085-fce6-43ae-88ca-d3f9c484b6f3)
복잡한 job이 2번 실행될 때, 좀 더 복잡한 job이 1번 실행되는 걸 볼 수 있습니다. 예상했던 결과입니다. 

하지만 실행된 시간을 살펴보겠습니다.
![log-with-time](https://github.com/drunkenhw/comments/assets/106640954/abbe2c65-c26b-46ba-a4e3-fc0f4e5a6612)

분명 매 2초와 4초마다 실행하기 때문에 작업 시간이 2의 배수가 되어야할텐데

34, 36, 36, **39**, 40, 40, **43**, 44, 44, **47**초 로 점점 작업이 밀리는 것을 확인할 수 있습니다.

왜 그럴까요? 스프링 공식 문서에서는 아래와 같이 설명하고 있습니다.

> A ThreadPoolTaskScheduler can also be auto-configured if need to be associated to scheduled task execution (using @EnableScheduling for instance). The thread pool uses one thread by default and its settings can be fine-tuned using the spring.task.scheduling namespace, as shown in the following example:


[참고 - 스프링 공식 문서](https://docs.spring.io/spring-boot/docs/current/reference/htmlsingle/#features.task-execution-and-scheduling)

스프링의 Schedule은 Default로 하나의 싱글 스레드에서 동작하기 때문입니다.
그렇기 때문에 매번 작업이 밀려 원하는 시간에 동작하지 않는 현상이 발생할 수 있습니다. 

하지만 Schedule을 분산 환경에서 구분하기 위해서는 job이 실행된 시간이 중요하기 때문에 이렇게 작업이 밀려버린다면 구분을 할 수 없게 됩니다. 
따라서 Schedule Thread Pool Size를 늘리도록 하겠습니다.

```java
@Configuration
public class ScheduleConfig implements SchedulingConfigurer {
    @Override
    public void configureTasks(ScheduledTaskRegistrar taskRegistrar) {
        ThreadPoolTaskScheduler taskScheduler = new ThreadPoolTaskScheduler();
        taskScheduler.setPoolSize(10);
        taskScheduler.setThreadNamePrefix("schedule-task-");
        taskScheduler.initialize();
        taskRegistrar.setTaskScheduler(taskScheduler);
    }
}
```
SchedulingConfigurer 를 구현하여 Thread Pool size를 일단 10개로 정의했습니다.

![success](https://github.com/drunkenhw/comments/assets/106640954/14b225bc-297e-4e7d-b196-23d779f635c0)
스레드 풀을 늘렸더니 위와 같이 2의 배수의 시간에 정확히 작동이 되는 것을 확인할 수 있습니다.

하지만 이렇게 여러 작업을 동시에 실행된다면 데이터베이스에 병목현상이 발생되어 오히려 작업이 더 느리게 끝날 수도 있다고 생각했습니다.

그래서 해당 부분의 실행을 관리하는 클래스를 생성하여 해당 클래스에서 Schedule의 작업을 관리하도록 구현했습니다.

```java
@Service
public class BusinessLogic {

    private final ApplicationEventPublisher applicationEventPublisher;

    @Scheduled(cron = "0/2 * * * * *")
    public void complexJobSchedule() {
        applicationEventPublisher.publishEvent(new SchedulingEvent(this::complexJob, "complexJob", LocalDateTime.now()));
    }

    @Scheduled(cron = "0/4 * * * * *")
    public void moreComplexJobSchedule() {
        applicationEventPublisher.publishEvent(new SchedulingEvent(this::moreComplexJob, "moreComplexJob", LocalDateTime.now()));
    }
}
```
로직이 있는 BusinessLogic 서비스에서 스케줄의 시간마다 실행해야할 메서드를 Event로 발행합니다.

```java
@Component
public class ScheduleService {

  private final ExecutorService executorService = Executors.newFixedThreadPool(1);
  private final Queue<SchedulingEvent> scheduleTasks = new ConcurrentLinkedQueue<>();
  private final AtomicBoolean isRunning = new AtomicBoolean(false);

  @EventListener
  public void addTask(SchedulingEvent schedulingEvent) {
    scheduleTasks.add(schedulingEvent);
  }

  @Scheduled(cron = "0/1 * * * * *")
  public void polling() {
    if (!scheduleTasks.isEmpty() || isRunning.compareAndSet(false, true)) {
      SchedulingEvent schedulingEvent = scheduleTasks.poll();
      executorService.execute(() -> execute(schedulingEvent));
    }
  }
}
```
그리고 위와 같은 스케줄을 관리하는 서비스에서는 Schedule Event를 받아 실행하도록 만들었습니다. 해당 클래스에서는 ThreadPool을 새로 생성하여, schedule의 스레드에 영향을 받지 않도록 구현했습니다.

그리고 1초마다 실행되는 스케줄을 만들어 queue에 작업이 있는지, 현재 작업 중인지 확인하여 그렇지 않다면 queue에서 작업을 꺼내 실행하도록 만들었습니다.

거의 구현이 끝나갑니다. 이제는 해당 Schedule의 데이터를 저장하고, 작업이 실패했을 시에 다시 작업을 하기 위한 기능만 구현하면 될 것 같습니다.

```java
@Component
public class ScheduleService {
    
    ...
  
    private void execute(SchedulingEvent schedulingEvent) {
        String jobId = schedulingEvent.jobId();
        LocalDateTime executionTime = schedulingEvent.executionTime();

        if (isJobInProgressOrDone(jobId)) {
            log.info("작업이 실행중입니다. {} {}", executionTime, jobId);
            return;
        }
        ScheduleTask entity = new ScheduleTask(jobId, executionTime, JobStatus.RUNNING);
        scheduleTaskJdbcRepository.save(entity);

        try {
            schedulingEvent.runnable().run();
            scheduleTaskJdbcRepository.updateById(entity.getId(), JobStatus.DONE);
        } catch (Exception e) {
            log.error("{} 작업 실행 중 에러가 발생했습니다.", jobId);
            scheduleTaskJdbcRepository.updateById(entity.getId(), JobStatus.ERROR);
            tasks.add(schedulingEvent);
        }
    }

    private boolean isJobInProgressOrDone(String jobId) {
        Optional<ScheduleTask> taskOptional = scheduleTaskRepository.findById(jobId);
        if (taskOptional.isPresent()) {
            ScheduleTask scheduleTask = taskOptional.get();
            return scheduleTask.getStatus() == JobStatus.RUNNING || scheduleTask.getStatus() == JobStatus.DONE;
        }
        return false;
    }
}
```
이 부분은 간단하게 구현할 수 있습니다. 위와 같이 작업의 실행 시간과, job의 이름으로 데이터베이스에서 조회하고, 없다면 작업을 실행하고
있다면 작업이 ERROR 인지 확인하여 작업을 실행해주면 될 것 같습니다.

![complete](https://github.com/drunkenhw/comments/assets/106640954/3ff855db-ff8e-4aa4-8b47-ed5b2ff6dd64)

위와 같이 두 개의 서버를 동시에 띄웠을 때에도 스케줄이 잘 작동하는 것을 확인할 수 있습니다.

## 결론
스케줄을 이렇게 구현할 수도 있지만 환경이 된다면 Message Queue를 사용하는 것이 어떨까요?


혹시 틀린 부분이 있다면 지적 부탁드립니다.