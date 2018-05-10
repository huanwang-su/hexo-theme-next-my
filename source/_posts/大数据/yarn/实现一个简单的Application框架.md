---
title: yarn实现一个简单的Application框架
date: 2018/5/10 17:08:48
category:
- 大数据
- yarn
tag:
- yarn
comments: true  
---

#### 主要接口

涉及到的核心接口如下：

- Client<->RM: 
  YarnClient,封装了ApplicationClientProtocol协议。
- AM<->RM: 
  AMRMClientAsync,AMRMClientAsync.CallbackHandler，封装了ApplicationMasterProtocol协议。
- AM<->NM: 
  NMClientAsync,NMClientAsync.CallbackHandler，封装了ContainerManagementProtocol协议。

#### 实现一个简单Application

首先使用配置conf初始化一个YARN客户端：

```
YarnClient yarnClient = YarnClient.createYarnClient();
yarnClient.init(conf);
yarnClient.start();
```

初始化后，创建一个application：

```
YarnClientApplication app = yarnClient.createApplication();
GetNewApplicationResponse appResponse = app.getNewApplicationResponse();
```

返回的Response中包含了一些集群信息，例如资源的最大最小容量，你要根据这些信息确保你的应用在申请资源的时候设置合理的值。

提交作业的核心是定义好ApplicationSubmissionContext：

```java
ApplicationSubmissionContext appContext = app.getApplicationSubmissionContext();
ApplicationId appId = appContext.getApplicationId();

appContext.setKeepContainersAcrossApplicationAttempts(keepContainers);
appContext.setApplicationName(appName);

//为Application Master设置本地资源 例如jar包，本地文件等
Map<String, LocalResource> localResources = new HashMap<String, LocalResource>();

LOG.info("Copy App Master jar from local filesystem and add to local environment");
// 复制application master的jar包到本地环境（运行AM的container）
// Create a local resource to point to the destination jar path
FileSystem fs = FileSystem.get(conf);
addToLocalResources(fs, appMasterJar, appMasterJarPath, appId.toString(),
    localResources, null);

// Set the log4j properties if needed
if (!log4jPropFile.isEmpty()) {
  addToLocalResources(fs, log4jPropFile, log4jPath, appId.toString(),
      localResources, null);
}

// The shell script has to be made available on the final container(s)
// where it will be executed.
// To do this, we need to first copy into the filesystem that is visible
// to the yarn framework.
// We do not need to set this as a local resource for the application
// master as the application master does not need it.
String hdfsShellScriptLocation = "";
long hdfsShellScriptLen = 0;
long hdfsShellScriptTimestamp = 0;
if (!shellScriptPath.isEmpty()) {
  Path shellSrc = new Path(shellScriptPath);
  String shellPathSuffix =
      appName + "/" + appId.toString() + "/" + SCRIPT_PATH;
  Path shellDst =
      new Path(fs.getHomeDirectory(), shellPathSuffix);
  fs.copyFromLocalFile(false, true, shellSrc, shellDst);
  hdfsShellScriptLocation = shellDst.toUri().toString();
  FileStatus shellFileStatus = fs.getFileStatus(shellDst);
  hdfsShellScriptLen = shellFileStatus.getLen();
  hdfsShellScriptTimestamp = shellFileStatus.getModificationTime();
}

if (!shellCommand.isEmpty()) {
  addToLocalResources(fs, null, shellCommandPath, appId.toString(),
      localResources, shellCommand);
}

if (shellArgs.length > 0) {
  addToLocalResources(fs, null, shellArgsPath, appId.toString(),
      localResources, StringUtils.join(shellArgs, " "));
}

// 设置AM运行的环境变量
LOG.info("Set the environment for the application master");
Map<String, String> env = new HashMap<String, String>();

// put location of shell script into env
// using the env info, the application master will create the correct local resource for the
// eventual containers that will be launched to execute the shell scripts
env.put(DSConstants.DISTRIBUTEDSHELLSCRIPTLOCATION, hdfsShellScriptLocation);
env.put(DSConstants.DISTRIBUTEDSHELLSCRIPTTIMESTAMP, Long.toString(hdfsShellScriptTimestamp));
env.put(DSConstants.DISTRIBUTEDSHELLSCRIPTLEN, Long.toString(hdfsShellScriptLen));

// Add AppMaster.jar location to classpath
// At some point we should not be required to add
// the hadoop specific classpaths to the env.
// It should be provided out of the box.
// For now setting all required classpaths including
// the classpath to "." for the application jar
StringBuilder classPathEnv = new StringBuilder(Environment.CLASSPATH.$$())
  .append(ApplicationConstants.CLASS_PATH_SEPARATOR).append("./*");
for (String c : conf.getStrings(
    YarnConfiguration.YARN_APPLICATION_CLASSPATH,
    YarnConfiguration.DEFAULT_YARN_CROSS_PLATFORM_APPLICATION_CLASSPATH)) {
  classPathEnv.append(ApplicationConstants.CLASS_PATH_SEPARATOR);
  classPathEnv.append(c.trim());
}
classPathEnv.append(ApplicationConstants.CLASS_PATH_SEPARATOR).append(
  "./log4j.properties");

// Set the necessary command to execute the application master
Vector<CharSequence> vargs = new Vector<CharSequence>(30);
// Set java executable command
LOG.info("Setting up app master command");
vargs.add(Environment.JAVA_HOME.$$() + "/bin/java");
// Set Xmx based on am memory size
vargs.add("-Xmx" + amMemory + "m");
// 设置application maser的Main类
vargs.add(appMasterMainClass);
// Set params for Application Master
vargs.add("--container_memory " + String.valueOf(containerMemory));
vargs.add("--container_vcores " + String.valueOf(containerVirtualCores));
vargs.add("--num_containers " + String.valueOf(numContainers));
vargs.add("--priority " + String.valueOf(shellCmdPriority));

for (Map.Entry<String, String> entry : shellEnv.entrySet()) {
  vargs.add("--shell_env " + entry.getKey() + "=" + entry.getValue());
}
if (debugFlag) {
  vargs.add("--debug");
}

vargs.add("1>" + ApplicationConstants.LOG_DIR_EXPANSION_VAR + "/AppMaster.stdout");
vargs.add("2>" + ApplicationConstants.LOG_DIR_EXPANSION_VAR + "/AppMaster.stderr");

// Get final commmand
StringBuilder command = new StringBuilder();
for (CharSequence str : vargs) {
  command.append(str).append(" ");
}

LOG.info("Completed setting up app master command " + command.toString());
List<String> commands = new ArrayList<String>();
commands.add(command.toString());

// Set up the container launch context for the application master
ContainerLaunchContext amContainer = ContainerLaunchContext.newInstance(
  localResources, env, commands, null, null, null);

// Set up resource type requirements
// For now, both memory and vcores are supported, so we set memory and
// vcores requirements
Resource capability = Resource.newInstance(amMemory, amVCores);
appContext.setResource(capability);

// Service data is a binary blob that can be passed to the application
// Not needed in this scenario
// amContainer.setServiceData(serviceData);

// Setup security tokens
if (UserGroupInformation.isSecurityEnabled()) {
  // Note: Credentials class is marked as LimitedPrivate for HDFS and MapReduce
  Credentials credentials = new Credentials();
  String tokenRenewer = conf.get(YarnConfiguration.RM_PRINCIPAL);
  if (tokenRenewer == null | | tokenRenewer.length() == 0) {
    throw new IOException(
      "Can't get Master Kerberos principal for the RM to use as renewer");
  }

  // For now, only getting tokens for the default file-system.
  final Token<?> tokens[] =
      fs.addDelegationTokens(tokenRenewer, credentials);
  if (tokens != null) {
    for (Token<?> token : tokens) {
      LOG.info("Got dt for " + fs.getUri() + "; " + token);
    }
  }
  DataOutputBuffer dob = new DataOutputBuffer();
  credentials.writeTokenStorageToStream(dob);
  ByteBuffer fsTokens = ByteBuffer.wrap(dob.getData(), 0, dob.getLength());
  amContainer.setTokens(fsTokens);
}

appContext.setAMContainerSpec(amContainer);
```

设置好Context之后，提交作业：

```
// 设置application master的优先级
Priority pri = Priority.newInstance(amPriority);
appContext.setPriority(pri);

// 设置应用程序要提交到RM的哪个队列queue
appContext.setQueue(amQueue);

yarnClient.submitApplication(appContext);
```

提交之后，可以有多种方式与RM保持通信：

```
// 获取appId对应的状态报告
ApplicationReport report = yarnClient.getApplicationReport(appId);
```

也可以杀死应用程序：

```
yarnClient.killApplication(appId);
```

#### 实现ApplicationMaster的关键步骤

实现AM，其实就是实现应用程序运行中的主要步骤，就是上面介绍的8个步骤。

当RM分配一个Container并且启动了AM之后，AM可以获取到一些参数，例如container id，作业提交信息以及container的宿主信息等。与RM的所有交互都必须使用ApplicationAttempId，这个值可以从ContainerId获取：

```
Map<String, String> envs = System.getenv();
String containerIdString =
    envs.get(ApplicationConstants.AM_CONTAINER_ID_ENV);
if (containerIdString == null) {
  // container id should always be set in the env by the framework
  throw new IllegalArgumentException(
      "ContainerId not set in the environment");
}
ContainerId containerId = ConverterUtils.toContainerId(containerIdString);
ApplicationAttemptId appAttemptID = containerId.getApplicationAttemptId();
```

AM初始化之后，我们可以启动两个客户端，一个与RM通信，一个与NM通信，并且为客户端设置自己的回调处理方法：

```
AMRMClientAsync.CallbackHandler allocListener = new RMCallbackHandler();
amRMClient = AMRMClientAsync.createAMRMClientAsync(1000, allocListener);
amRMClient.init(conf);
amRMClient.start();

containerListener = createNMCallbackHandler();
nmClientAsync = new NMClientAsyncImpl(containerListener);
nmClientAsync.init(conf);
nmClientAsync.start();
```

AM需要向ResourceManager发送心跳以免RM认为我们挂掉，超期时间通过 YarnConfiguration.RM_AM_EXPIRY_INTERVAL_MS设置，默认为YarnConfiguration.DEFAULT_RM_AM_EXPIRY_INTERVAL_MS。为了发送心跳，首先得向RM注册自己：

```
// Register self with ResourceManager
// This will start heartbeating to the RM
appMasterHostname = NetUtils.getHostname();
RegisterApplicationMasterResponse response = amRMClient
    .registerApplicationMaster(appMasterHostname, appMasterRpcPort, appMasterTrackingUrl);
```

返回结果中包含资源的最大容量，可以用来check我们的资源请求是否合理。

```
// Dump out information about cluster capability as seen by the
// resource manager
int maxMem = response.getMaximumResourceCapability().getMemory();
LOG.info("Max mem capabililty of resources in this cluster " + maxMem);

int maxVCores = response.getMaximumResourceCapability().getVirtualCores();
LOG.info("Max vcores capabililty of resources in this cluster " + maxVCores);

// A resource ask cannot exceed the max.
if (containerMemory > maxMem) {
  LOG.info("Container memory specified above max threshold of cluster."
      + " Using max value." + ", specified=" + containerMemory + ", max="
      + maxMem);
  containerMemory = maxMem;
}

if (containerVirtualCores > maxVCores) {
  LOG.info("Container virtual cores specified above max threshold of  cluster."
    + " Using max value." + ", specified=" + containerVirtualCores + ", max="
    + maxVCores);
  containerVirtualCores = maxVCores;
}
List<Container> previousAMRunningContainers =
    response.getContainersFromPreviousAttempts();
LOG.info("Received " + previousAMRunningContainers.size()
        + " previous AM's running containers on AM registration.");
```

接着我们可以根据需要向RM申请资源了：

```
List<Container> previousAMRunningContainers =
    response.getContainersFromPreviousAttempts();
List<Container> previousAMRunningContainers =
    response.getContainersFromPreviousAttempts();
LOG.info("Received " + previousAMRunningContainers.size()
    + " previous AM's running containers on AM registration.");

int numTotalContainersToRequest =
    numTotalContainers - previousAMRunningContainers.size();
// Setup ask for containers from RM
// Send request for containers to RM
// Until we get our fully allocated quota, we keep on polling RM for
// containers
// Keep looping until all the containers are launched and shell script
// executed on them ( regardless of success/failure).
for (int i = 0; i < numTotalContainersToRequest; ++i) {
  ContainerRequest containerAsk = setupContainerAskForRM();
  amRMClient.addContainerRequest(containerAsk);
}
```

setupContainerAskForRM方法中，需要设置资源容量以及优先级：

```
private ContainerRequest setupContainerAskForRM() {
  // setup requirements for hosts
  // using * as any host will do for the distributed shell app
  // set the priority for the request
  Priority pri = Priority.newInstance(requestPriority);

  // Set up resource type requirements
  // For now, memory and CPU are supported so we set memory and cpu requirements
  Resource capability = Resource.newInstance(containerMemory,
    containerVirtualCores);

  ContainerRequest request = new ContainerRequest(capability, null, null,
      pri);
  LOG.info("Requested container ask: " + request.toString());
  return request;
}
```

资源申请提交后，由于分配时异步的，因此需要设置好回调，通过实现AMRMClientAsync.CallbackHandler来完成：

```
@Override
public void onContainersAllocated(List<Container> allocatedContainers) {
  LOG.info("Got response from RM for container ask, allocatedCnt="
      + allocatedContainers.size());
  numAllocatedContainers.addAndGet(allocatedContainers.size());
  for (Container allocatedContainer : allocatedContainers) {
    LaunchContainerRunnable runnableLaunchContainer =
        new LaunchContainerRunnable(allocatedContainer, containerListener);
    Thread launchThread = new Thread(runnableLaunchContainer);

    // launch and start the container on a separate thread to keep
    // the main thread unblocked
    // as all containers may not be allocated at one go.
    launchThreads.add(launchThread);
    launchThread.start();
  }
}
```

同时也要通过心跳发送进度：

```
@Override
public float getProgress() {
  // set progress to deliver to RM on next heartbeat
  float progress = (float) numCompletedContainers.get()
      / numTotalContainers;
  return progress;
}	
```

上面的Container启动线程在NM上启动container，在启动之前，需要准备好ContainerLaunchContext：

```
// Set the necessary command to execute on the allocated container
Vector<CharSequence> vargs = new Vector<CharSequence>(5);

// Set executable command
vargs.add(shellCommand);
// Set shell script path
if (!scriptPath.isEmpty()) {
  vargs.add(Shell.WINDOWS ? ExecBatScripStringtPath
    : ExecShellStringPath);
}

// Set args for the shell command if any
vargs.add(shellArgs);
// Add log redirect params
vargs.add("1>" + ApplicationConstants.LOG_DIR_EXPANSION_VAR + "/stdout");
vargs.add("2>" + ApplicationConstants.LOG_DIR_EXPANSION_VAR + "/stderr");

// Get final commmand
StringBuilder command = new StringBuilder();
for (CharSequence str : vargs) {
  command.append(str).append(" ");
}

List<String> commands = new ArrayList<String>();
commands.add(command.toString());

// Set up ContainerLaunchContext, setting local resource, environment,
// command and token for constructor.

// Note for tokens: Set up tokens for the container too. Today, for normal
// shell commands, the container in distribute-shell doesn't need any
// tokens. We are populating them mainly for NodeManagers to be able to
// download anyfiles in the distributed file-system. The tokens are
// otherwise also useful in cases, for e.g., when one is running a
// "hadoop dfs" command inside the distributed shell.
ContainerLaunchContext ctx = ContainerLaunchContext.newInstance(
  localResources, shellEnv, commands, null, allTokens.duplicate(), null);
containerListener.addContainer(container.getId(), container);
```

准备好之后，通过NMClientAsync启动：

```
nmClientAsync.startContainerAsync(container, ctx);
```

NMClientAsync及其相应的Callback，会处理容器发布的各种事件，例如启动、停止，状态更新等。

当ApplicationMaster确认自己的任务完成后，向RM注销并关闭客户端：

```
try {
  amRMClient.unregisterApplicationMaster(appStatus, appMessage, null);
} catch (YarnException ex) {
  LOG.error("Failed to unregister application", ex);
} catch (IOException e) {
  LOG.error("Failed to unregister application", e);
}

amRMClient.stop();
```

