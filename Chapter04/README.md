
## 第4章 Activiti6.0引擎配置
### 4-1  本章概述
### 4-2  创建流程引擎配置-config_samples

1. 新建Maven空项目activiti-samples
2. 给项目新增Module，名称为config  
3. 新增自定义脚手架  
![](images/0402_Add_Archetype.png)
4. 根据脚手架创建模块  
![](images/0403_Create_From_Archetype.png)
5. 新建测试文件，路径 activiti6-samples\config\src\test\java\com\coderdream\config，名称 ConfigTest.java：  
![](images/0401_Project_Structrue.png)
- 代码清单：ConfigTest.java
```java
@Test
public void testConfig1() {
    ProcessEngineConfiguration configuration = ProcessEngineConfiguration
            .createProcessEngineConfigurationFromResourceDefault();

    LOGGER.info("configuration = {}", configuration);
}

@Test
public void testConfig2() {
    ProcessEngineConfiguration configuration = ProcessEngineConfiguration
            .createStandaloneProcessEngineConfiguration();

    LOGGER.info("configuration = {}", configuration);
}
```

- 输出结果：
```
configuration = org.activiti.engine.impl.cfg.StandaloneInMemProcessEngineConfiguration@2beee7ff
configuration = org.activiti.engine.impl.cfg.StandaloneProcessEngineConfiguration@366647c2
```
- 流程引擎配置的继承关系                                                                                                                                                                    
![](images/0404_ProcessEngineConfiguration.png)

### 4-3  创建流程引擎配置-archetype
### 4-4  数据库配置-dbconfig
### 4-5  数据库配置-dbconfig_code

```java
@Test
public void testConfig1() {
    ProcessEngineConfiguration configuration = ProcessEngineConfiguration
            .createProcessEngineConfigurationFromResourceDefault();

    LOGGER.info("configuration = {}", configuration);
    ProcessEngine processEngine = configuration.buildProcessEngine();
    LOGGER.info("获取流程引擎 {}", processEngine.getName());
    processEngine.close();
}
```


```
Loading XML bean definitions from class path resource [activiti.cfg.xml]
configuration = org.activiti.engine.impl.cfg.StandaloneInMemProcessEngineConfiguration@6a6afff2
Activiti 5 compatibility handler implementation not found or error during instantiation : org.activiti.compatibility.DefaultActiviti5CompatibilityHandler. Activiti 5 backwards compatibility disabled.
performing create on engine with resource org/activiti/db/create/activiti.h2.create.engine.sql
performing create on history with resource org/activiti/db/create/activiti.h2.create.history.sql
performing create on identity with resource org/activiti/db/create/activiti.h2.create.identity.sql
ProcessEngine default created
获取流程引擎 default
performing drop on engine with resource org/activiti/db/drop/activiti.h2.drop.engine.sql
performing drop on history with resource org/activiti/db/drop/activiti.h2.drop.history.sql
performing drop on identity with resource org/activiti/db/drop/activiti.h2.drop.identity.sql

Process finished with exit code 0
```


### 4-6  日志记录配置-logging

![](images/0406_01.jpg)

![](images/0406_02.jpg)

![](images/0406_03.jpg)

![](images/0406_04.jpg)

![](images/0406_05.jpg)

![](images/0406_06.jpg)

### 4-7  日志记录配置-logging_mdc

步骤：
1. 创建代理；
2. 修改流程文件；
3. 修改logback文件
3. 打开MDC


- 代码清单：MDCErrorDelegate.java
```java
public class MDCErrorDelegate implements JavaDelegate {

    /** logger */
    private static final Logger LOGGER = LoggerFactory.getLogger(MDCErrorDelegate.class);

    @Override
    public void execute(DelegateExecution execution) {
        LOGGER.info("run MDCErrorDelegate");
        throw new RuntimeException("only test");
    }
}

```
- 代码清单：my-process.bpmn20.xml
```xml
<?xml version="1.0" encoding="UTF-8"?>

<definitions xmlns="http://www.omg.org/spec/BPMN/20100524/MODEL"
	xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance" xmlns:activiti="http://activiti.org/bpmn"
	xmlns:bpmndi="http://www.omg.org/spec/BPMN/20100524/DI" xmlns:omgdc="http://www.omg.org/spec/DD/20100524/DC"
	xmlns:omgdi="http://www.omg.org/spec/DD/20100524/DI" typeLanguage="http://www.w3.org/2001/XMLSchema"
	expressionLanguage="http://www.w3.org/1999/XPath" targetNamespace="http://www.activiti.org/test">

	<process id="my-process">

		<startEvent id="start" />
		<sequenceFlow id="flow1" sourceRef="start" targetRef="someTask" />
		
		<!-- <userTask id="someTask" name="Activiti is awesome!" /> -->
		<serviceTask id="someTask" activiti:class="com.coderdream.delegate.MDCErrorDelegate" />

		<sequenceFlow id="flow2" sourceRef="someTask" targetRef="end" />

		<endEvent id="end" />

	</process>

</definitions>
```

- 代码清单（片段）：logback.xml
```xml
    <!--
     - MDC 配置:
     - ProcessDefinitionId: 流程定义 id
     - executionId:
     - mdcProcessInstanceId: 流程实例 id
     - mdcBusinessKey: 业务 key
     -->
    <property name="mdc" value="%d{HH:mm:ss.SSS} [%thread] [%level] %msg 
	ProcessDefinitionId=%X{mdcProcessDefinitionID}    
	executionId=%X{mdcExecutionId} 
	mdcProcessInstanceId=%X{mdcProcessInstanceId} 
	mdcBusinessKey=%X{mdcBusinessKey} 
	%logger{10}.%M:%L%n"/>

    <!-- 控制台输出 -->
    <appender name="stdout" class="ch.qos.logback.core.ConsoleAppender">
        <encoder>
            <pattern>${mdc}</pattern>
            <charset>${encoding}</charset>
        </encoder>
    </appender>
```


- 代码清单：ConfigMDCTest.java
```java
public class ConfigMDCTest {

    /** logger */
    private static final Logger LOGGER = LoggerFactory.getLogger(ConfigMDCTest.class);

    @Rule
    public ActivitiRule activitiRule = new ActivitiRule();

    @Test
    @Deployment(resources = {"my-process.bpmn20.xml"})
    public void test() {
        // 打开MDC开关
        LogMDC.setMDCEnabled(true);
        ProcessInstance processInstance = activitiRule.getRuntimeService()
                .startProcessInstanceByKey("my-process");
        assertNotNull(processInstance);

        Task task = activitiRule.getTaskService().createTaskQuery().singleResult();
        assertEquals("Activiti is awesome!", task.getName());

        // 继续执行流程
        activitiRule.getTaskService().complete(task.getId());
    }

}
```

- 执行结果
```
15:32:05.953 [main] [ERROR] Error while closing command context ProcessDefinitionId=my-process:1:3     
executionId=5 mdcProcessInstanceId= mdcBusinessKey= o.a.e.i.i.CommandContext.logException:122
java.lang.RuntimeException: only test
	at com.coderdream.delegate.MDCErrorDelegate.execute(MDCErrorDelegate.java:16)
```

**拦截器方式**  

1. 新增拦截器
2. 新增配置文件
3. 新增测试方法及相关配置

- 代码清单1：MDCCommandInvoker.java
```java
public class MDCCommandInvoker extends DebugCommandInvoker {

    // 参考DebugCommandInvoker代码
    @Override
    public void executeOperation(Runnable runnable) {
        // 保存默认状态
        boolean mdcEnabled = LogMDC.isMDCEnabled();
        // 修改状态
        LogMDC.setMDCEnabled(true);
        if (runnable instanceof AbstractOperation) {
            AbstractOperation operation = (AbstractOperation) runnable;

            if (operation.getExecution() != null) {
                // 记录日志
                LogMDC.putMDCExecution(operation.getExecution());
            }

        }

        super.executeOperation(runnable);
        // 清理上下文信息
        LogMDC.clear();
        // 还原为默认状态
        if (!mdcEnabled) {
            LogMDC.setMDCEnabled(false);
        }
    }
}
```

- 代码清单2：activiti_mdc.cfg.xml
```xml
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans" 
       xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
       xsi:schemaLocation="http://www.springframework.org/schema/beans   
	http://www.springframework.org/schema/beans/spring-beans.xsd">

  <bean id="processEngineConfiguration" class="org.activiti.engine.impl.cfg.StandaloneInMemProcessEngineConfiguration">
    <property name="commandInvoker" ref="commandInvoker" />
  </bean>

  <bean id="commandInvoker" class="com.coderdream.interceptor.MDCCommandInvoker" />

</beans>

```

- 代码清单3：my-process_mdcerror.bpmn20.xml
```xml
<?xml version="1.0" encoding="UTF-8"?>

<definitions xmlns="http://www.omg.org/spec/BPMN/20100524/MODEL"
	xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance" xmlns:activiti="http://activiti.org/bpmn"
	xmlns:bpmndi="http://www.omg.org/spec/BPMN/20100524/DI" xmlns:omgdc="http://www.omg.org/spec/DD/20100524/DC"
	xmlns:omgdi="http://www.omg.org/spec/DD/20100524/DI" typeLanguage="http://www.w3.org/2001/XMLSchema"
	expressionLanguage="http://www.w3.org/1999/XPath" targetNamespace="http://www.activiti.org/test">

	<process id="my-process">

		<startEvent id="start" />
		<sequenceFlow id="flow1" sourceRef="start" targetRef="someTask" />
		
		<userTask id="someTask" name="Activiti is awesome!" />

		<sequenceFlow id="flow2" sourceRef="someTask" targetRef="end" />

		<endEvent id="end" />

	</process>

</definitions>
```

- 代码清单4：ConfigMDCTest.java
```java
    @Rule
    public ActivitiRule activitiRule = new ActivitiRule("activiti_mdc.cfg.xml");

    @Test
    @Deployment(resources = {"my-process_mdcerror.bpmn20.xml"})
    public void test2() {
        ProcessInstance processInstance = activitiRule.getRuntimeService()
                .startProcessInstanceByKey("my-process");
        assertNotNull(processInstance);

        Task task = activitiRule.getTaskService().createTaskQuery().singleResult();
        assertEquals("Activiti is awesome!", task.getName());

        // 继续执行流程
        activitiRule.getTaskService().complete(task.getId());
    }
```


- 运行结果
```
15:54:38.046 [main] [INFO] performing create on engine with resource org/activiti/db/create/activiti.h2.create.engine.sql 
ProcessDefinitionId=     executionId= mdcProcessInstanceId= mdcBusinessKey= o.a.e.i.d.DbSqlSession.executeSchemaResource:1147
15:54:38.100 [main] [INFO] performing create on history with resource org/activiti/db/create/activiti.h2.create.history.sql 
ProcessDefinitionId=     executionId= mdcProcessInstanceId= mdcBusinessKey= o.a.e.i.d.DbSqlSession.executeSchemaResource:1147
15:54:38.104 [main] [INFO] performing create on identity with resource org/activiti/db/create/activiti.h2.create.identity.sql 
ProcessDefinitionId=     executionId= mdcProcessInstanceId= mdcBusinessKey= o.a.e.i.d.DbSqlSession.executeSchemaResource:1147
15:54:38.108 [main] [INFO] ProcessEngine default created ProcessDefinitionId=     
executionId= mdcProcessInstanceId= mdcBusinessKey= o.a.e.i.ProcessEngineImpl.<init>:87
15:54:38.303 [main] [INFO] Execution tree while executing operation class org.activiti.engine.impl.agenda.ContinueProcessOperation : 
ProcessDefinitionId=my-process:1:3     executionId=5 mdcProcessInstanceId= mdcBusinessKey= o.a.e.i.i.DebugCommandInvoker.executeOperation:33
15:54:38.307 [main] [INFO] 
4 (process instance)
└── 5 : start (StartEvent, parent id 4 (active)
 ProcessDefinitionId=my-process:1:3     executionId=5 mdcProcessInstanceId= mdcBusinessKey= o.a.e.i.i.DebugCommandInvoker.executeOperation:34
15:54:38.310 [main] [INFO] Execution tree while executing operation class org.activiti.engine.impl.agenda.TakeOutgoingSequenceFlowsOperation : 
ProcessDefinitionId=my-process:1:3     executionId=5 mdcProcessInstanceId= mdcBusinessKey= o.a.e.i.i.DebugCommandInvoker.executeOperation:33
15:54:38.311 [main] [INFO] 
4 (process instance)
└── 5 : start (StartEvent, parent id 4 (active)
 ProcessDefinitionId=my-process:1:3     executionId=5 mdcProcessInstanceId= mdcBusinessKey= o.a.e.i.i.DebugCommandInvoker.executeOperation:34
15:54:38.312 [main] [INFO] Execution tree while executing operation class org.activiti.engine.impl.agenda.ContinueProcessOperation : 
ProcessDefinitionId=my-process:1:3     executionId=5 mdcProcessInstanceId= mdcBusinessKey= o.a.e.i.i.DebugCommandInvoker.executeOperation:33
15:54:38.312 [main] [INFO] 
4 (process instance)
└── 5 : start -> someTask, parent id 4 (active)
 ProcessDefinitionId=my-process:1:3     executionId=5 mdcProcessInstanceId= mdcBusinessKey= o.a.e.i.i.DebugCommandInvoker.executeOperation:34
15:54:38.313 [main] [INFO] Execution tree while executing operation class org.activiti.engine.impl.agenda.ContinueProcessOperation : 
ProcessDefinitionId=my-process:1:3     executionId=5 mdcProcessInstanceId= mdcBusinessKey= o.a.e.i.i.DebugCommandInvoker.executeOperation:33
15:54:38.313 [main] [INFO] 
4 (process instance)
└── 5 : someTask (UserTask, parent id 4 (active)
 ProcessDefinitionId=my-process:1:3     executionId=5 mdcProcessInstanceId= mdcBusinessKey= o.a.e.i.i.DebugCommandInvoker.executeOperation:34
15:54:38.389 [main] [INFO] Execution tree while executing operation class org.activiti.engine.impl.agenda.TriggerExecutionOperation : 
ProcessDefinitionId=my-process:1:3     executionId=5 mdcProcessInstanceId= mdcBusinessKey= o.a.e.i.i.DebugCommandInvoker.executeOperation:33
15:54:38.391 [main] [INFO] 
4 (process instance)
└── 5 : someTask (UserTask, parent id 4 (active)
 ProcessDefinitionId=my-process:1:3     executionId=5 mdcProcessInstanceId= mdcBusinessKey= o.a.e.i.i.DebugCommandInvoker.executeOperation:34
15:54:38.392 [main] [INFO] Execution tree while executing operation class org.activiti.engine.impl.agenda.TakeOutgoingSequenceFlowsOperation : 
ProcessDefinitionId=my-process:1:3     executionId=5 mdcProcessInstanceId= mdcBusinessKey= o.a.e.i.i.DebugCommandInvoker.executeOperation:33
15:54:38.392 [main] [INFO] 
4 (process instance)
└── 5 : someTask (UserTask, parent id 4 (active)
 ProcessDefinitionId=my-process:1:3     executionId=5 mdcProcessInstanceId= mdcBusinessKey= o.a.e.i.i.DebugCommandInvoker.executeOperation:34
15:54:38.393 [main] [INFO] Execution tree while executing operation class org.activiti.engine.impl.agenda.ContinueProcessOperation : 
ProcessDefinitionId=my-process:1:3     executionId=5 mdcProcessInstanceId= mdcBusinessKey= o.a.e.i.i.DebugCommandInvoker.executeOperation:33
15:54:38.394 [main] [INFO] 
4 (process instance)
└── 5 : someTask -> end, parent id 4 (active)
 ProcessDefinitionId=my-process:1:3     executionId=5 mdcProcessInstanceId= mdcBusinessKey= o.a.e.i.i.DebugCommandInvoker.executeOperation:34
15:54:38.394 [main] [INFO] Execution tree while executing operation class org.activiti.engine.impl.agenda.ContinueProcessOperation : 
ProcessDefinitionId=my-process:1:3     executionId=5 mdcProcessInstanceId= mdcBusinessKey= o.a.e.i.i.DebugCommandInvoker.executeOperation:33
15:54:38.394 [main] [INFO] 
4 (process instance)
└── 5 : end (EndEvent, parent id 4 (active)
 ProcessDefinitionId=my-process:1:3     executionId=5 mdcProcessInstanceId= mdcBusinessKey= o.a.e.i.i.DebugCommandInvoker.executeOperation:34
15:54:38.394 [main] [INFO] Execution tree while executing operation class org.activiti.engine.impl.agenda.TakeOutgoingSequenceFlowsOperation : 
ProcessDefinitionId=my-process:1:3     executionId=5 mdcProcessInstanceId= mdcBusinessKey= o.a.e.i.i.DebugCommandInvoker.executeOperation:33
15:54:38.395 [main] [INFO] 
4 (process instance)
└── 5 : end (EndEvent, parent id 4 (active)
 ProcessDefinitionId=my-process:1:3     executionId=5 mdcProcessInstanceId= mdcBusinessKey= o.a.e.i.i.DebugCommandInvoker.executeOperation:34
15:54:38.396 [main] [INFO] Execution tree while executing operation class org.activiti.engine.impl.agenda.EndExecutionOperation : 
ProcessDefinitionId=my-process:1:3     executionId=5 mdcProcessInstanceId= mdcBusinessKey= o.a.e.i.i.DebugCommandInvoker.executeOperation:33
15:54:38.396 [main] [INFO] 
4 (process instance)
└── 5 : end (EndEvent, parent id 4 (active)
 ProcessDefinitionId=my-process:1:3     executionId=5 mdcProcessInstanceId= mdcBusinessKey= o.a.e.i.i.DebugCommandInvoker.executeOperation:34

Process finished with exit code 0

```


### 4-8  历史记录配置-history-1
### 4-9  历史记录配置-history-2
### 4-10  事件处理及监听器配置-eventlog
### 4-11  事件处理及监听器配置-eventLinstener-1
### 4-12  事件处理及监听器配置-eventLinstener-2
### 4-13  命令拦截器配置-command-1
### 4-14  命令拦截器配置-command-2
### 4-15  作业执行器配置-job-1
### 4-16  作业执行器配置-job-2
### 4-17  Activiti与spring集成-1
### 4-18  Activiti与spring集成-2
### 4-19  Activiti与spring集成-3

## 参考网站：

1. [fhqscfss/activiti-test](https://github.com/fhqscfss/activiti-test)
2. [chandler1142/activiti-study](https://github.com/chandler1142/activiti-study)
3. [shengjie8329/activiti-helloword](https://github.com/shengjie8329/activiti-helloword)
4. [DestinyWang/activiti-sample](https://github.com/DestinyWang/activiti-sample)