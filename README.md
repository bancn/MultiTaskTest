## 线程模型

在阅读此部分之前，强烈建议不熟悉Java线程的读者先通读官方的Java并发教程。

IntelliJ平台是一个高度并发的环境。代码在多个线程中同时执行。一般来说，与普通的Swing应用程序类似，线程可以分为两大类：

1. **事件调度线程（EDT）**：也称为UI线程。它的主要目的是处理UI事件（例如响应按钮点击或更新UI），但平台也用它来写入数据。EDT从事件队列中执行事件。在EDT上执行的操作必须尽可能快，以免阻塞队列中的其他事件并冻结UI。在运行的应用程序中，只有一个EDT。
2. **后台线程（BGT）**：用于执行长时间运行和耗费资源的操作或后台任务。

在IntelliJ平台中，可以在后台线程（BGT）和事件调度线程（EDT）之间进行双向切换。这意味着你可以从后台线程将操作调度到EDT执行，也可以从EDT将操作移到后台线程执行。

具体来说：

- **从后台线程切换到EDT**：你可以使用`invokeLater()`方法来调度操作在EDT上执行。这种方法可以从后台线程或其他EDT操作中调用，以确保某个操作在EDT上执行（例如更新用户界面）。
- **从EDT切换到后台线程**：你可以启动后台进程来处理耗时的任务，从而避免阻塞EDT。这是为了确保长时间的计算或数据处理不会冻结用户界面。

这种机制允许开发者根据任务的性质选择合适的线程来执行操作，以确保应用程序的响应性和性能。



目标版本为2024.1及以上的插件应使用协程调度器在线程之间切换。

### 读者-写者锁

IntelliJ平台的数据结构（如程序结构接口、虚拟文件系统或项目根模型）不是线程安全的。访问它们需要一种同步机制，以确保所有线程看到的数据是一致和最新的。这是通过一个应用程序范围内的读者-写者（RW）锁实现的，线程需要在读取或写入数据模型之前获得该锁。

如果一个线程需要访问数据模型，它必须获得以下锁之一：

- **读锁**：允许线程读取数据。可以与其他读锁和写意图锁同时从任何线程获取。如果另一线程持有写锁，则不能获取。
- **写意图锁**：允许线程读取数据，并可能升级到写锁。可以与其他读锁同时从任何线程获取。如果另一线程持有写锁或写意图锁，则不能获取。
- **写锁**：允许线程读取和写入数据。只能从EDT中获取，并且不能与其他任何锁同时获取。

以下表格显示了锁之间的兼容性：

| 锁类型   | 读锁 | 写意图锁 | 写锁 |
| -------- | ---- | -------- | ---- |
| 读锁     | +    | +        | -    |
| 写意图锁 | +    | -        | -    |
| 写锁     | -    | -        | -    |

由此可得以下结论：

- 多个线程可以同时读取数据
- 一旦一个线程获取了写锁，其他线程无法读取或写入数据

在代码中显式获取和释放锁是冗长且容易出错的，插件中不应这样做。IntelliJ平台在EDT上隐式启用写意图锁，并提供了API来在读锁或写锁下访问数据。

### 锁和EDT

虽然理论上可以从任何线程获取所有类型的锁，但平台隐式获取写意图锁，并仅允许在EDT上获取写锁。这意味着数据写入只能在EDT上进行。

众所周知，仅在EDT上写入数据可能导致UI冻结。当前正在进行的工作旨在允许从任何线程写入数据。有关此行为的历史原因，请参阅当前平台版本。

在EDT上隐式获取写意图锁的范围取决于平台版本：

- **2023.3及以上版本**：当操作在EDT上通过`Application.invokeLater()`调用时，自动获取写意图锁。

### 访问数据

IntelliJ平台提供了一个简单的API来在读锁或写锁下访问数据，即读写操作。

读写操作允许在锁下执行一段代码，在操作开始前自动获取锁，并在操作结束后释放锁。

#### 最小化锁定范围

始终尝试仅将所需的操作封装在读/写操作中，尽量减少持有锁的时间。如果读操作本身很长，考虑使用读操作可取消技术以避免阻塞写锁和EDT。

### 读操作

#### API

- `Application.runReadAction()`:

  ```
  PsiFile psiFile = ApplicationManager.getApplication()
      .runReadAction((Computable<PsiFile>)() -> {
        // read and return PsiFile
      });
  ```

- `ReadAction.run()`或`ReadAction.compute()`:

  ```
  PsiFile psiFile = ReadAction.compute(() -> {
    // read and return PsiFile
  });
  ```



#### 规则

- **2023.3及以上版本**：允许从任何线程读取数据。
- 通过`Application.invokeLater()`在EDT上调用的读取数据不需要显式的读操作，因为允许读取数据的写意图锁已隐式获取。
- 在所有其他情况下，读取操作需要用读操作API方法之一封装。

#### 对象有效性

在多个连续的读操作之间，不保证读对象的有效性。在开始读操作时，检查PSI/VFS/项目/模块是否仍然有效。

在执行第一个和第二个读操作之间，另一个线程可能会使虚拟文件无效：

### 写操作

#### API

- `Application.runWriteAction()`:

  ```
  ApplicationManager.getApplication().runWriteAction(() -> {
    // write data
  });
  ```

- `WriteAction.run()`或`WriteAction.compute()`:

  ```
  WriteAction.run(() -> {
    // write data
  });
  ```

  

#### 规则

- **2023.3及以上版本**：写入数据仅允许在EDT上通过`Application.invokeLater()`调用。

写操作必须始终使用一个API方法封装在写操作中。

### 调用EDT上的操作和模态性

在EDT上写入数据的操作应通过`Application.invokeLater()`调用，因为它允许为安排的操作指定模态状态（ModalityState）。`SwingUtilities.invokeLater()`和类似API不支持这一点。

模态状态（ModalityState）表示活动模态对话框的堆栈，并用于调用`Application.invokeLater()`以确保计划的可运行对象可以在给定的模态状态下执行，即在相同或子集的模态对话框存在时。

要更好地理解模态状态解决的问题，考虑以下场景：

1. 用户操作开始。
2. 同时，另一个操作在EDT上通过`SwingUtilities.invokeLater()`（不支持模态状态）安排。
3. 第1步的操作显示一个询问“是/否”的对话框。
4. 在显示对话框时，第2步的操作开始并修改数据模型，使PSI失效。
5. 用户在对话框中点击“是”或“否”，并根据答案执行一些代码。
6. 现在，基于用户答案执行的代码必须处理已更改的数据模型，而这些代码并未准备好处理这种情况。例如，它可能是要执行在PSI中已失效的更改。

使用模态状态可以解决这个问题：

1. 用户操作开始。
2. 同时，另一个操作在EDT上通过`Application.invokeLater()`（支持模态状态）安排。操作使用`ModalityState.defaultModalityState()`安排（请参见下表中的其他辅助方法）。
3. 第1步的操作显示一个询问“是/否”的对话框。这将一个模态对话框添加到模态状态堆栈中。
4. 在显示对话框时，安排的操作等待，因为它是以“较低”的模态状态安排的，而当前状态有额外的对话框。
5. 用户在对话框中点击“是”或“否”，并根据答案执行一些代码。
6. 代码在显示对话框之前的相同数据状态下执行。
7. 第1步的操作现在执行，而不会干扰用户的操作。

### 读操作可取消性

后台线程（BGT）不应长时间持有读锁。原因是如果EDT需要执行写操作（例如，用户在编辑器中输入某些内容），则必须尽快获取写锁。否则，UI将在所有BGT释放其读操作之前冻结。以下图表展示了这个问题：

有时，可能需要运行长时间的读操作，并且无法加快速度。在这种情况下，建议的方法是在即将进行写操作时取消读操作，并稍后从头重新启动该读操作：

这种情况下，EDT不会被阻塞，从而避免了UI冻结。尽管读操作的总执行时间会因为多次尝试而变长，但不影响UI响应更为重要。

取消方法广泛应用于IntelliJ平台的各种领域：编辑器高亮显示、代码补全、“转到类/文件/…”操作等。有关更多详细信息，请阅读后台进程部分。

#### 可取消读操作API

目标版本为2024.1及以上的插件应使用Kotlin协程读操作API中的允许写操作的读操作。

要运行可取消的读操作，请使用以下API之一：

- `ReadAction.nonBlocking()` 返回 `NonBlockingReadAction` (NBRA)。NBRA会自动处理重新启动操作。
- `ReadAction.computeCancellable()` 在当前线程中立即计算结果，或者如果有正在运行或请求的写操作，则抛出异常。

在这两种情况下，当读操作开始并在此期间发生写操作时，读操作将被标记为已取消。读操作必须足够频繁地检查取消以触发实际取消。尽管取消机制可能在底层有所不同（进度API或Kotlin协程），但取消处理规则在两种情况下都是相同的。

始终在每个读操作开始时检查对象是否仍然有效，并检查整个操作是否仍然有意义。对于`ReadAction.nonBlocking()`，使用`expireWith()`或`expireWhen()`来实现。

如果NBRA需要访问基于文件的索引（例如，进行任何项目范围内的PSI分析、解析引用或执行依赖于索引的其他任务），请使用`ReadAction.nonBlocking(…).inSmartMode()`。

### 避免UI冻结

#### 不要在EDT上执行长时间操作

特别是，不要遍历VFS、解析PSI、解析引用或查询索引。

平台本身仍然会在某些情况下调用此类耗时代码（例如，在`AnAction.update()`中解析），但这些问题正在解决中。与此同时，尝试加快插件中的相关操作将是普遍有益的，并且也将提高后台高亮显示性能。

#### 操作更新

对于`AnAction`的实现，插件作者应特别查看`AnAction.getActionUpdateThread()`的文档，因为它描述了操作的线程处理方式。

#### 最小化写操作范围

写操作目前必须在EDT上进行。为了加快速度，应尽可能将其移动到写操作之外的准备步骤中，然后可以在后台或NBRA中调用。

### 慢操作上的断言

一些长时间操作由`SlowOperations.assertSlowOperationsAreAllowed()`报告。根据其Javadoc，必须将它们移到BGT中。这可以通过Javadoc中提到的技术、后台进程、`Application.executeOnPooledThread()`或协程（推荐用于目标为2024.1及以上版本的插件）来实现。请注意，此断言在IDE EAP版本、内部模式或开发实例中启用，常规用户在IDE中看不到它们。这将在未来发生变化，因此修复这些异常是必要的。

### 事件监听器

监听器不应执行任何耗时操作。理想情况下，它们只应清除某些缓存。

还可以安排事件的后台处理。在这种情况下，请准备好在后台处理开始之前可能会传递一些新的事件——因此，此时世界可能已发生变化，甚至在后台处理期间也是如此。考虑使用`MergingUpdateQueue`和NBRA来缓解这些问题。

### VFS事件

可以在后台使用`AsyncFileListener`预处理大量VFS事件。



## 后台进程

后台进程是指在后台线程上执行的耗时计算任务。IntelliJ平台广泛使用后台进程，并为插件提供了两种主要运行它们的方法：

1. **Progress API**：允许取消任务并跟踪其进度。
2. **Application.executeOnPooledThread()**方法：用于运行不需要取消或进度跟踪的简单后台任务。

**注意**：自2024.1起，建议插件使用Kotlin协程，这是更高效的解决方案，并且内置了取消机制。有关协程的更多信息，请参见执行上下文。

### Progress API

Progress API允许在后台线程（BGT）上运行具有模态（对话框）、非模态（在状态栏中可见）或不可见进度的进程。它还允许取消进程并跟踪进度（以完成工作的比例或文本形式显示）。

关键类包括：

- **ProgressManager**：提供执行和管理后台进程的方法。
- **ProgressIndicator**：与运行中的进程相关联的对象。它允许取消进程，并可选择跟踪其进度。可以随时通过`ProgressManager.getProgressIndicator()`获取当前线程的指示器。

常用的`ProgressIndicator`实现包括：

- **EmptyProgressIndicator**：不可见（忽略与文本/比例相关的方法），仅用于取消跟踪。记住其创建时的模态状态。
- **ProgressIndicatorBase**：不可见，但可以通过子类化显示。存储文本/比例，并允许在UI中检索和显示。默认情况下为非模态。
- **ProgressWindow**：可见的进度，可以是模态或后台。通常不直接创建，而是通过`ProgressManager.run`方法内部实例化。
- **ProgressWrapper**：包装现有的进度指示器，通常用于在另一个线程中使用相同的取消策略。使用`SensitiveProgressWrapper`允许该线程的指示器独立于主线程取消。

**任务（Task）**：封装要执行的操作。有关后台、模态和其他基本任务类的信息，请参见`Task`的内部子类。

### 开始后台进程

通过排队运行封装在`Task`中的后台进程。例如：

```
java复制代码new Task.Backgroundable(project, "Synchronizing data", true) {
  public void run(ProgressIndicator indicator) {
    // 操作内容
  }
}
  .setCancelText("Stop loading")
  .queue();
```

`ProgressManager`还允许使用多个`run*()`方法运行未封装在`Task`中的`Runnable`和`Computable`。例如：

```
java复制代码ProgressManager.getInstance().runProcessWithProgressSynchronously(
    () -> {
      // 操作内容
    },
    "Synchronizing data", true, project
);
```

### 取消任务

Progress API的最重要功能是能够在计算结果变得不相关时取消进程。取消可以由用户执行（按取消按钮）或在代码中当当前操作由于项目中的某些更改而变得过时时触发。示例：

**用户取消操作（例如，查找符号用法时）**：

1. 用户在大型项目中触发“查找用法”操作。
2. 结果正在计算并逐渐显示给用户。
3. 用户看到了他们感兴趣的位置或意识到不再需要这些结果。
4. 用户点击状态栏中的取消按钮，操作被取消。

**代码取消操作（例如，代码补全时）**：

1. 用户在编辑器中输入一个字母。
2. 开始计算代码补全的结果。
3. 用户输入另一个字母。
4. 步骤2中开始的计算现在已过时，被取消以开始新的输入计算。

在插件代码中准备处理取消请求对于节省CPU资源和保持IDE响应速度至关重要。

### 请求取消

可以通过调用`ProgressIndicator.cancel()`将进程标记为已取消。这个方法由启动进程的基础架构调用，例如，当点击取消按钮时，或负责调用代码补全的代码。

`cancel()`方法将进程标记为已取消，实际的取消操作由运行的进程自行决定。请参阅下文了解如何处理取消。

### 处理取消

取消通过在运行的进程代码中调用`ProgressIndicator.checkCanceled()`来处理，如果当前上下文中没有指示器实例，则使用`ProgressManager.checkCanceled()`。

如果进程被标记为取消，那么调用`checkCanceled()`会抛出一个特殊的未经检查的`ProcessCanceledException`（PCE），然后进行实际取消。这个异常不表示任何错误，仅用于方便地处理取消。它允许在调用堆栈的深处取消进程，而无需在每个级别处理取消。

`PCE`由启动进程的基础架构处理，绝不应被记录或忽略。如果因为某种原因捕获到它，必须重新抛出。使用检查`Plugin DevKit | Code | 'ProcessCanceledException' handled incorrectly (2023.3)`。

所有使用PSI或其他类型后台进程的代码都必须准备好在任何时候抛出`PCE`。

运行的操作应该足够频繁地调用`checkCanceled()`，以保证进程能够顺利取消。PSI内部有许多`checkCanceled()`调用。如果进程进行耗时的非PSI活动，请插入显式的`checkCanceled()`调用，以便它能够频繁地发生，例如在每N次循环迭代时。使用检查`Plugin DevKit | Code | Cancellation check in loops (2023.1)`。

### 禁用ProcessCanceledException

在开发的内部模式下，可以禁用从`checkCanceled()`抛出的PCE（例如，在调试代码时）：

- **2023.2及以上版本**：`Tools | Internal Actions | Skip Window Deactivation Events`
- **早期版本**：`Tools | Internal Actions | Disable ProcessCanceledException`

### 进度跟踪

通过`ProgressIndicator`或`ProgressManager`（如果当前上下文中没有指示器实例）向用户显示进度。

要报告进度，请使用以下方法：

- `setText(String)`：设置进度条上方的文本。
- `setText2(String)`：设置进度条下方的文本。
- `setFraction(double)`：设置进度比例：一个介于0.0（无进度）到1.0（全部完成）之间的数字，反映已完成工作的比例。仅适用于确定的指示器。比例应为用户提供剩余时间的粗略估计。如果无法做到这一点，请考虑使进度不可确定。
- `setIndeterminate(boolean)`：将进度标记为不可确定（用于无法估计工作量的进程）或确定（用于可以显示工作完成比例的进程）。

通过这些方法，可以有效地向用户展示当前后台进程的状态和进度，有助于提升用户体验。

## 消息机制

IntelliJ平台的消息机制是一种发布-订阅模式的实现，提供了诸如层次结构广播和特殊嵌套事件处理等额外功能（嵌套事件是从另一个事件的回调中直接或间接触发的事件）。

所有可用的监听器/主题列在IntelliJ平台扩展点和监听器列表的监听器部分。

### 设计

以下部分描述了消息传递API的主要组件：

#### Topic（主题）

Topic类作为消息传递基础设施的终端点。客户端可以订阅特定总线内的特定主题，并向该总线内的主题发送消息。为了明确相应的消息总线，Topic字段声明应使用`@Topic.AppLevel`和/或`@Topic.ProjectLevel`注解。

**主要属性：**
- **显示名称**：用于日志记录/监控的可读名称。
- **广播方向**：见广播部分的更多细节，默认值为TO_CHILDREN。
- **监听器类**：特定主题的业务接口。订阅者在消息传递基础设施中注册该接口的实现。发布者随后检索符合该接口的对象，并调用这些实现中的任何方法。消息传递基础设施负责通过在注册的实现回调上调用相同的方法和参数，将消息分派给该主题的所有订阅者。

#### Message Bus（消息总线）

MessageBus是消息系统的核心，主要用于以下场景：
- **订阅者**：创建连接并订阅。
- **发布者**：发布消息。

#### Connection（连接）

Connection由`MessageBusConnection`类表示，管理特定客户端在特定总线中的所有订阅。

**连接存储主题-处理器映射**：当接收到目标主题的消息时，调用的回调（在同一连接中每个主题不允许超过一个处理器）。

可以指定默认处理器并订阅目标主题，而无需显式提供回调。在存储主题-处理器映射时，连接将使用该默认处理器。

可以显式释放获取的资源（参见`disconnect()`）。此外，它可以连接到标准的半自动处置（Disposable）。

### 消息API的使用

以下示例假设一个项目级别的主题。

#### 定义业务接口和主题

创建具有业务方法的接口，并绑定到该业务接口的主题字段：

```java
public interface ChangeActionNotifier {

  @Topic.ProjectLevel
  Topic<ChangeActionNotifier> CHANGE_ACTION_TOPIC =
      Topic.create("custom name", ChangeActionNotifier.class);

  void beforeAction(Context context);
  void afterAction(Context context);
}
```

#### 订阅主题

获取消息总线引用，创建与总线的连接并进行订阅。

对于2019.3或更高版本，尽可能使用声明式注册。

```java
project.getMessageBus().connect().subscribe(
    ChangeActionNotifier.CHANGE_ACTION_TOPIC,
    new ChangeActionNotifier() {
        @Override
        public void beforeAction(Context context) {
          // 处理'操作前'事件
        }
        @Override
        public void afterAction(Context context) {
          // 处理'操作后'事件
        }
});
```

`MessageBus`实例可以通过`ComponentManager.getMessageBus()`获取。许多标准接口都实现了返回消息总线的功能，例如`Application.getMessageBus()`和`Project.getMessageBus()`。

#### 发布消息

获取消息总线引用，请求总线发布特定主题的发布者并调用发布者上的目标方法。消息传递将相同的方法调用在目标处理器上。

```java
public void doChange(Context context) {
  ChangeActionNotifier publisher = project.getMessageBus()
      .syncPublisher(ChangeActionNotifier.CHANGE_ACTION_TOPIC);
  publisher.beforeAction(context);
  try {
    // 执行操作
  } finally {
    publisher.afterAction(context);
  }
}
```

### 广播

消息总线可以组织成层次结构。此外，IntelliJ平台已经有了这样的结构：

- 应用程序总线
- 项目总线
- 模块总线

这样可以通知一个消息总线上的订阅者关于另一个消息总线发送的消息。

#### 示例设置

假设一个简单的层次结构（应用程序总线是项目总线的父级），三个订阅者订阅相同的主题。

如果`topic1`定义广播方向为TO_CHILDREN，那么我们得到以下结果：
1. 消息通过应用程序总线发送到`topic1`。
2. `handler1`被通知消息。
3. 消息被传递给项目总线中相同主题的订阅者（`handler2`和`handler3`）。

广播的主要好处是管理绑定到子总线但对父总线级别事件感兴趣的订阅者。在上面的例子中，我们可能希望有特定于项目的功能来响应应用程序级别的事件。我们需要做的就是在项目总线内订阅目标主题。在应用程序级别将不会存储项目级别订阅者的硬引用，即我们避免了在项目重新打开时的内存泄漏。

#### 嵌套消息

嵌套消息是指在另一个消息处理期间发送（直接或间接）的消息。IntelliJ平台的消息传递基础设施保证所有发送到特定主题的消息都按发送顺序交付。

#### 小贴士和技巧

**简化监听器管理**  
消息传递基础设施非常轻量，因此可以在本地子系统中重用它，以减轻订阅者构造的负担。以下是需要做的事情：
1. 定义业务接口。
2. 创建共享消息总线和使用上述接口的主题（共享意味着主体或订阅者知道它们）。

**避免订阅者共享数据修改**  
在某些情况下，当两个订阅者尝试修改同一个文档时会出现问题。每次文档更改都是通过以下步骤执行的：
1. 向所有文档监听器发送“更改前”事件，其中一些在处理过程中发布新消息；
2. 实际更改执行；
3. 向所有文档监听器发送“更改后”事件。

如遇到这种问题，需要仔细设计和管理消息传递和数据同步机制，以避免潜在的冲突和数据不一致问题。



## IDE 支持框架

#### 日志记录

如果您的插件直接使用了log4j库，请注意它在IntelliJ平台2022.1版本中已被移除。有关迁移的说明，请参阅相应的博客文章。

IntelliJ平台使用`Logger`抽象类来屏蔽底层的日志实现和配置。插件应获取专用的`Logger`实例：

```
java复制代码import com.intellij.openapi.diagnostic.Logger;

public class MyPluginClass {
  private static final Logger LOG = Logger.getInstance(MyPluginClass.class);

  public void someMethod() {
    LOG.info("someMethod() was called");
  }
}
```

默认情况下，所有级别为INFO及以上的消息都会写入日志输出文件`idea.log`。要为特定类别启用DEBUG/TRACE日志记录，请使用`Help | Diagnostic Tools | Debug Log Settings`。

要定位日志文件，请选择`Help | Show Log in Finder/Explorer`操作。当内部模式启用时，可以使用`Help | Open Log in Editor`打开当前运行的IDE日志文件。

要为特定安装定位日志文件，请参阅此[知识库文章](https://www.jetbrains.com/help/idea/locations-of-log-files.html)。有关开发实例沙盒目录的信息，请参阅Development Instance Sandbox Directory。

有关如何在测试期间启用DEBUG/TRACE级别日志记录以及获取失败测试的单独日志，请参阅Testing FAQ。

要为报告致命错误提供附加上下文，请使用带有附加信息的`Logger.error()`方法（参见`CoreAttachmentFactory`和`AttachmentFactory`）。

#### 错误报告

IDE将显示自身捕获的致命错误以及IDE Fatal Errors对话框中错误级别的日志消息：

- 对于IDE平台：在EAP版本或内部模式运行时
- 对于第三方插件：始终

对于后者，报告默认是禁用的——相反，有一个选项可以禁用引发异常的插件。

要让用户将这些错误报告给供应商，插件可以实现自定义的`ErrorReportSubmitter`并注册在`com.intellij.errorHandler`扩展点。有关现有实现的更多信息，请参阅IntelliJ Platform Explorer——从预填充基于Web的问题跟踪表单到完全自动化的日志监控系统提交。这篇[教程](https://www.jetbrains.com/help/idea/error-reporting.html)还提供了使用Sentry的工作解决方案。

要禁用状态栏中的红色感叹号通知图标，请调用`Help | Edit Custom Properties...`并在打开的`idea.properties`中添加`idea.fatal.error.notification=disabled`。

#### 运行时信息

`ApplicationInfo`提供有关IDE版本和供应商的信息。注意：要限制兼容性，请通过`plugin.xml`声明IDEs和版本。

要获取操作系统和Java虚拟机的信息，请使用`SystemInfo`。

有关相关配置目录的信息，请参阅`PathManager`。

要获取唯一的安装UUID，请使用`PermanentInstallationID`。有关付费插件的信息，请参阅Marketplace文档。

#### 上下文帮助

要为插件功能（例如对话框）显示自定义的基于Web的上下文帮助，请提供`WebHelpProvider`并注册在`com.intellij.webHelpProvider`扩展点。

#### 一次性任务运行

使用`RunOnceUtil`在项目/应用程序中准确运行一次任务。

#### 应用程序事件

可以通过`AppLifecycleListener`监听器跟踪应用程序生命周期事件。有关更多信息，请参阅Application Startup and Project and Application Close。

要接收“应用程序焦点/失焦”事件的通知，请注册`ApplicationActivationListener`监听器。

要请求重启IDE，请使用`Application.restart()`。

#### 启动浏览器

使用`BrowserLauncher`。

#### 在系统文件管理器中打开文件

使用`RevealFileAction.openFile()`或`openDirectory()`。

#### 主题更改

使用`LafManagerListener`主题接收更改通知（例如刷新UI）。

#### 电源节能模式

可以启用`File | Power Save Mode`以限制笔记本电脑上的高功耗功能。使用`PowerSaveMode`服务和`PowerSaveMode.Listener`主题相应地禁用插件中的这些功能。

#### 插件管理

可以通过`PluginManagerCore.isPluginInstalled()`检查已安装的插件。

#### 插件建议

对于特定功能（例如文件类型、Facet等），IDE会自动建议安装匹配的插件。有关详细信息，请参阅Marketplace文档中的插件推荐。

要建议其他相关插件，请使用`PluginsAdvertiser.installAndEnable()`。

#### 插件弃用

要建议用新插件替换当前安装的已弃用插件，请实现`PluginReplacement`并注册在`com.intellij.pluginReplacement`扩展点。

#### 网络

使用`HttpConnectionUtils`（2024.3）以使用平台网络设置（例如代理）。对于更早的版本，请参阅`HttpConfigurable`。





## 用户界面组件

#### 概览

IntelliJ平台包含大量自定义的Swing组件。使用这些组件可以确保您的插件在外观和功能上与IDE的其余部分保持一致，并且通常可以减少与使用默认Swing组件相比的代码量。

#### 检查现有UI

使用UI Inspector工具可以定位底层的Swing组件实现，或在运行时检查现有的UI。

对于对话框或设置等UI表单，建议使用Kotlin UI DSL（IntelliJ平台2021.3+）。使用UI Designer插件与Kotlin不受支持。

有关编写UI相关文本的指南，请参阅UI Guidelines中的Writing Short and Clear部分。

在使用Figma设计UI时，请参阅UI Kit。

#### 特别值得注意的组件

1. **菜单和工具栏**：使用Actions构建。
2. **工具窗口**：提供了多种用于显示和管理插件功能的窗口。
3. **对话框**：用于与用户交互的模态和非模态对话框。
4. **弹出窗口**：用于显示临时信息或选项的弹出窗口。
5. **通知**：用于显示系统消息或插件消息的通知框。
6. **文件和类选择器**：用户选择文件或类的对话框。
7. **编辑器组件**：如文本编辑器和代码编辑器。
8. **列表和树控件**：用于显示和管理列表和树状结构的数据。
9. **状态栏小部件**：显示在IDE状态栏中的小部件。
10. **表格（TableView）**：用于显示和操作表格数据。
11. **拖放助手**：帮助实现拖放操作的组件。
12. **其他Swing组件**：各种辅助的UI组件。



### 工具窗口

工具窗口是IDE的子窗口，用于显示信息。这些窗口通常有自己的工具栏（称为工具窗口栏），位于主窗口的外边缘，包含一个或多个工具窗口按钮，这些按钮可以激活显示在主IDE窗口左侧、底部和右侧的面板。

每一侧都有两个工具窗口组，主组和次组，每次只能激活一个组中的一个工具窗口。

每个工具窗口可以显示多个标签（在API中称为“内容”）。例如，运行工具窗口为每个活动的运行配置显示一个标签，而与版本控制相关的工具窗口根据项目中使用的版本控制系统显示一组固定的标签。

在插件中使用工具窗口有两种主要场景：使用声明式设置时，工具窗口按钮始终可见，用户可以随时激活它并与插件功能进行交互；或者，使用编程设置时，工具窗口用于显示特定操作的结果，操作完成后可以关闭。

#### 声明式设置

工具窗口在`plugin.xml`中使用`com.intellij.toolWindow`扩展点进行注册。扩展点属性指定显示工具窗口按钮所需的所有数据：

- `id`属性（必填）：工具窗口的标识符，对应显示在工具窗口按钮上的文本。要提供本地化文本，请在资源包中指定匹配的`toolwindow.stripe.[id]`消息键（空格用`_`转义）。

- `icon`：显示在工具窗口按钮上的图标（13x13像素，灰色和单色；参见UI指南中的工具窗口和图标处理部分）。

- `anchor`：工具窗口显示的屏幕侧边（“left”（默认）、“right”或“bottom”）。

- `secondary`属性：指定工具窗口是显示在主组还是次组中。

- `factoryClass`属性（必填）：实现`ToolWindowFactory`的类。

当用户点击工具窗口按钮时，工厂类的`createToolWindowContent()`方法被调用，初始化工具窗口的UI。这一过程确保未使用的工具窗口不会对启动时间或内存使用造成任何开销：如果用户不与工具窗口交互，则不会加载或执行任何插件代码。

#### 条件显示

如果插件的工具窗口不应在所有项目中显示：

- **2023.3及更高版本**：实现`ToolWindowFactory.isApplicableAsync(Project)`。

- **2021.1及更高版本**：实现`ToolWindowFactory.isApplicable(Project)`。

- **2019.3及更早版本**：需要手动处理。

注意，条件只会在项目加载时评估一次。要在用户与项目交互时动态显示和隐藏工具窗口，请使用编程设置进行工具窗口注册。

#### 编程设置

对于仅在调用特定操作后显示的工具窗口，请使用`ToolWindowManager.registerToolWindow(String, RegisterToolWindowTaskBuilder)`。

在安排与工具窗口相关的EDT任务时，请始终使用`ToolWindowManager.invokeLater()`，而不是使用普通的`Application.invokeLater()`（参见线程模型）。

#### 内容（标签）

显示许多工具窗口的内容需要访问索引。因此，在构建索引时，工具窗口通常被禁用，除非`ToolWindowFactory`被标记为`dumb aware`。

如前所述，工具窗口可以包含多个内容（标签）。要管理工具窗口的内容，请调用`ToolWindow.getContentManager()`。要添加内容（标签），首先通过调用`ContentManager.getFactory().createContent()`创建内容，然后使用`ContentManager.addContent()`将其添加到工具窗口中。使用`Content.setDisposer()`注册关联的`Disposable`（参见Disposer和Disposable）。

参考`SimpleToolWindowPanel`作为一个方便的基类，支持工具栏和垂直/水平布局。

#### 关闭标签

插件可以控制用户是否允许关闭标签，既可以全局设置，也可以按内容设置。前者通过将`canCloseContents`参数传递给`registerToolWindow()`函数，或在`plugin.xml`中指定`canCloseContents="true"`来完成。默认值为`false`；除非明确设置了`canCloseContents`，否则调用`ContentManager`内容上的`setClosable(true)`将被忽略。

如果一般情况下启用了关闭标签功能，插件可以通过调用`Content.setCloseable(false)`来禁用特定标签的关闭。

#### 工具窗口常见问题

**访问工具窗口**：使用`ToolWindowManager.getToolWindow()`并指定注册时使用的id。

**工具窗口通知**：`ToolWindowManager.notifyByBalloon()`允许为给定的工具窗口显示通知。

**事件**：项目级别主题`ToolWindowManagerListener`允许监听工具窗口的注册/显示事件（参见监听器）。

这些功能和组件使得开发人员可以更好地在IntelliJ IDEA中实现和管理工具窗口，从而提升插件的用户体验和功能集成。



### 对话框

#### DialogWrapper

`DialogWrapper`是用于在IntelliJ平台中显示所有模态对话框（以及一些非模态对话框）的基类。它提供以下功能：

- **按钮布局**：包括平台特定的OK/Cancel按钮顺序，以及macOS特有的帮助按钮。
- **上下文帮助**。
- **记住对话框的尺寸**。
- **非模态验证**：当对话框中输入的数据无效时显示错误消息文本。
- **键盘快捷键**：
  - `Esc` 关闭对话框
  - `左/右箭头` 切换按钮
  - `Y/N` 用于对话框中的Yes/No操作（如果存在）
- **可选的“不再询问”复选框**。

此外，还提供了一种类似DSL的API通过`DialogBuilder`使用。

#### 使用方式

使用`DialogWrapper`类创建对话框时，请遵循以下步骤：

1. 调用基类构造函数，并提供显示对话框的项目或父组件。
2. 调用`setTitle()`方法设置对话框的标题。
3. 从对话框类的构造函数中调用`init()`方法。
4. 实现`createCenterPanel()`方法，返回组成对话框主要内容的组件。

可选步骤：

- 重写`getPreferredFocusedComponent()`方法，返回对话框首次显示时应聚焦的组件。
- 重写`getDimensionServiceKey()`方法，返回用于持久化对话框尺寸的标识符。
- 重写`getHelpId()`方法，返回与对话框相关的上下文帮助主题（参见上下文帮助）。

使用Kotlin UI DSL提供对话框的内容（参见示例）。或者，当使用Java时，可以将`DialogWrapper`类与GUI Designer表单结合使用。在这种情况下，将GUI Designer表单绑定到继承自`DialogWrapper`的类上，将表单的顶级面板绑定到一个字段，并从`createCenterPanel()`方法返回该字段。

有关在对话框中排列UI控件的建议，请参阅UI Guidelines中的布局主题。

可以使用UI Inspector在运行时检查现有对话框，例如查找UI组件的底层实现。

要显示对话框，调用`show()`方法，然后使用`getExitCode()`方法检查对话框是如何关闭的（参见`DialogWrapper#OK_EXIT_CODE`, `CANCEL_EXIT_CODE`, `CLOSE_EXIT_CODE`）。`showAndGet()`方法可以组合这两个调用。

要自定义对话框中显示的按钮（替换标准的OK/Cancel/Help按钮），重写`createActions()`或`createLeftActions()`方法。这两个方法都返回一个Swing Action对象的数组。如果一个按钮关闭对话框，请使用`DialogWrapperExitAction`作为动作的基类。使用`action.putValue(DialogWrapper.DEFAULT_ACTION, true)`设置默认按钮。

#### 输入验证

有关输入验证的更多信息，请参见UI Guidelines中的验证错误主题。

要验证对话框中输入的数据，重写`doValidate()`方法。该方法将自动由计时器调用。如果当前输入的数据有效，则返回`null`。否则，返回一个`ValidationInfo`对象，其中包含错误消息和一个可选的与无效数据关联的组件。指定组件时，将在其旁边显示错误图标，当用户尝试执行OK操作时，焦点将移动到该组件。

#### 示例

`DialogWrapper`的最小示例：

```java
public class SampleDialogWrapper extends DialogWrapper {

  public SampleDialogWrapper() {
    super(true); // 使用当前窗口作为父级
    setTitle("Test DialogWrapper");
    init();
  }

  @Nullable
  @Override
  protected JComponent createCenterPanel() {
    JPanel dialogPanel = new JPanel(new BorderLayout());

    JLabel label = new JLabel("Testing");
    label.setPreferredSize(new Dimension(100, 100));
    dialogPanel.add(label, BorderLayout.CENTER);

    return dialogPanel;
  }
}
```

在用户点击按钮时显示`SampleDialogWrapper`对话框：

```java
JButton testButton = new JButton();
testButton.addActionListener(actionEvent -> {
  if (new SampleDialogWrapper().showAndGet()) {
    // 用户点击了OK
  }
});
```

以上内容展示了如何在IntelliJ平台中使用`DialogWrapper`创建和管理对话框，包括自定义对话框的行为和外观。



### 弹出窗口

IntelliJ平台的用户界面广泛使用弹出窗口（Popups）——这是一种半模态窗口，没有显式的关闭按钮，当失去焦点时会自动消失。在您的插件中使用这些控件可以确保插件的用户体验与IDE的其他部分一致。

弹出窗口可以选择性地显示标题，可以移动和调整大小（并支持记住其尺寸），并且可以嵌套（当选择某个项目时显示另一个弹出窗口）。

`JBPopupFactory`接口允许创建显示不同类型组件的弹出窗口，具体取决于您的需求。最常用的方法包括：

#### 常用方法

1. **createComponentPopupBuilder()**
   - **描述**：通用方法，允许显示任何Swing组件。
   - **示例**：`IntentionPreviewPopupUpdateProcessor`创建一个渲染意图预览的弹出窗口。

2. **createPopupChooserBuilder()**
   - **描述**：用于从一个简单的`java.util.List`中选择一个或多个项目。
   - **示例**：`ShowMessageHistoryAction`创建一个弹出窗口，显示提交消息文本区域中的最近提交消息历史。

3. **createConfirmation()**
   - **描述**：用于在两个选项之间选择，并根据选择的选项执行不同的操作。
   - **示例**：`VariableInplaceRenamer`在内联重命名操作中提供无效变量名后，创建确认弹出窗口。

4. **createActionGroupPopup()**
   - **描述**：显示来自操作组的操作，并执行用户选择的操作。
   - **示例**：`ShowRecentFindUsagesGroup`通过编辑/查找用法/最近用法调用，显示最近用法组的弹出窗口。

#### 操作组

操作组弹出窗口支持不同的键盘选择方式，除了正常的箭头键之外。通过传递`JBPopupFactory.ActionSelectionAid`枚举中的常量，您可以选择是否通过按下对应于顺序号的键、输入部分文本（快速搜索）或按下助记符字符来选择一个操作。对于具有固定项目集的弹出窗口，建议使用顺序编号；对于具有可变且可能大量项目的弹出窗口，通常快速搜索效果最佳。

#### 列表弹出窗口

如果需要创建比简单`JList`更灵活的列表式弹出窗口，但不想将可能的选择表示为操作组中的操作，可以直接使用`ListPopupStep`接口和`JBPopupFactory.createListPopup()`方法。通常不需要实现整个接口；相反，可以从`BaseListPopupStep`类派生。关键方法是`getTextFor()`（返回要显示的项目文本）和`onChosen()`（在选择项目时调用）。通过从`onChosen()`方法返回一个新的弹出步骤，可以实现分层（嵌套）弹出窗口。

#### 显示弹出窗口

创建弹出窗口后，需要通过调用其中一个`show()`方法来显示它。可以通过调用`showInBestPositionFor()`让IntelliJ平台根据上下文自动选择位置，也可以通过`showUnderneathOf()`和`showInCenterOf()`等方法明确指定位置。

`show()`方法会立即返回，而不会等待弹出窗口关闭。

如果需要在弹出窗口关闭时执行某些操作，可以使用`addListener()`方法附加监听器，重写弹出内容的方法（如`PopupStep.onChosen()`），或在弹出窗口内的组件上附加事件处理程序。

这些功能使得开发者能够灵活地创建和管理弹出窗口，从而提供丰富的用户交互体验。





### 通知

IntelliJ平台的设计原则之一是避免使用模态消息框来通知用户错误和其他需要注意的情况。作为替代方案，IntelliJ平台提供了多种非模态通知的UI选项。

#### 对话框

在对话框中工作时，推荐使用`DialogWrapper.doValidate()`方法检查输入的有效性，而不是在按下OK按钮时通过模态对话框通知用户无效数据。这一点在对话框部分有详细描述。

#### 编辑器提示

对于从编辑器调用的操作（如重构、导航操作和各种代码洞察功能），最好使用`HintManager`类来通知用户无法执行某个操作。其`showErrorHint()`方法会在编辑器上方显示一个浮动的提示，当用户开始执行编辑器中的其他操作时，该提示会自动隐藏。`HintManager`的其他方法可用于在编辑器上方显示其他类型的非模态通知提示。

#### 编辑器横幅

有关UI参考，请参见UI指南中的横幅部分。

出现在文件编辑器顶部的通知是要求用户采取重要操作的好方法，否则可能会妨碍他们的体验（例如，缺少SDK，设置/项目配置需要用户输入）。

使用`com.intellij.editorNotificationProvider`扩展点注册一个`EditorNotifications.Provider`的实现。如果不需要访问索引，可以将其标记为dumb aware。

常用的UI实现是`EditorNotificationPanel`。

#### "Got It" 通知

通过`GotItTooltip`来突出显示重要的新功能或更改功能。有关概述，请参见UI指南中的Got It Tooltip。

#### 顶级通知（气泡）

最通用的显示非模态通知的方法是使用`Notifications`类。

其主要有两个优势：

1. 用户可以在`Settings | Appearance & Behavior | Notifications`中控制每种通知类型的显示方式。
2. 所有显示的通知都会集中在事件日志工具窗口中，用户可以稍后查看。

有关UI参考，请参见UI指南中的气泡部分。

有关特定工具窗口的气泡通知，请参见工具窗口通知。

用于显示通知的具体方法是`Notifications.Bus.notify()`。如果已知当前的项目，请使用带有项目参数的重载方法，以便通知显示在与该项目相关的框架中。

通知的文本可以包括HTML标签用于呈现目的。使用`Notification.addAction(AnAction)`在内容下添加链接，为方便起见，可以使用`NotificationAction`。

通知构造函数的`groupId`参数指定了通知类型。用户可以在`Settings | Appearance & Behavior | Notifications`下选择对应每种通知类型的显示类型。

要指定首选的显示类型，您需要使用`NotificationGroup`来创建通知。

#### 设置步骤

##### 2020.3及更高版本

`NotificationGroup`在`plugin.xml`中使用`com.intellij.notificationGroup`扩展点注册。使用`key`提供本地化的组显示名称。

```xml
<extensions defaultExtensionNs="com.intellij">
  <notificationGroup id="Custom Notification Group"
                     displayType="BALLOON"
                     key="notification.group.name"/>
</extensions>
```

注册的实例可以通过其id获取。对于期望通知组id的参数，提供了代码提示。

```java
public class MyNotifier {

  public static void notifyError(Project project, String content) {
    NotificationGroupManager.getInstance()
        .getNotificationGroup("Custom Notification Group")
        .createNotification(content, NotificationType.ERROR)
        .notify(project);
  }

}
```

这段代码展示了如何创建和管理通知，使得插件能够在IntelliJ平台中以一致的方式向用户展示重要信息和错误。

### 文件和类选择器

#### 文件选择器

##### 通过对话框

要让用户选择文件、目录或多个文件，可以使用`FileChooser.chooseFiles()`方法。此方法有多个重载。最佳使用方法是返回`void`且接收一个回调作为参数的版本，该回调接收所选文件的列表。只有这个重载会在macOS上显示本地文件打开对话框。

`FileChooserDescriptor`类允许您控制可以选择哪些文件。其构造函数参数指定是否可以选择文件和/或目录，以及是否允许多选（参见`FileChooserDescriptorFactory`的常见变体）。

对于更精细的选择控制，您可以重写`isFileSelectable()`方法。您还可以通过重写`getIcon()`、`getName()`和`getComment()`方法来自定义文件的展示。请注意，macOS原生文件选择器不支持大多数自定义功能，因此如果您依赖这些功能，需要使用显示标准IntelliJ平台对话框的`chooseFiles()`的重载。

##### 通过文本字段

使用文件选择器的常见方式是使用带有省略号按钮（...）的文本字段来输入路径，以显示文件选择器。要创建这样的控件，请使用`TextFieldWithBrowseButton`组件，并调用其`addBrowseFolderListener()`方法来设置文件选择器。作为附加功能，这将启用在文本框中输入路径时的文件名补全功能。

##### 通过树状结构

另一种选择文件的UI，是通过`TreeFileChooserFactory`类实现的，这种方式更适合通过输入文件名来选择文件的场景。

该API显示的对话框有两个标签页：
1. 一个显示项目结构
2. 另一个显示类似于导航|文件弹出窗口的文件列表。

要显示对话框，调用从`createFileChooser()`返回的选择器上的`showDialog()`，然后调用`getSelectedFile()`以获取用户的选择。

#### 类和包选择器

如果您希望用户选择一个Java类，可以使用`TreeClassChooserFactory`类。其不同的方法允许您指定类的来源范围，限制选择特定类的子类或接口的实现，以及包括或排除内部类。

要选择一个Java包，可以使用`PackageChooserDialog`类。

### 编辑器组件

#### EditorTextField

与Swing的`JTextArea`相比，IntelliJ平台的编辑器组件具有许多优势，如语法高亮、代码补全、代码折叠等功能。编辑器通常显示在编辑器标签页中，但也可以嵌入对话框或工具窗口中。这通过`EditorTextField`组件实现。

`EditorTextField`可以指定以下属性：

- 根据文本字段中的文本解析的文件类型；
- 文本字段是只读还是可编辑的；
- 文本字段是单行还是多行。

通过子类化和重写`createEditor()`以及应用`EditorCustomization`可以进行进一步的自定义。已有几个常见的自定义实现，包括：

- `SpellCheckingEditorCustomization`：禁用拼写检查
- `HorizontalScrollBarEditorCustomization`：启用/禁用水平滚动条
- `ErrorStripeEditorCustomization`：启用/禁用右侧的错误条纹

`EditorTextField`有多个子类，可根据需要使用附加功能。如果您希望在对话框中使用编辑器作为输入字段，可以考虑使用`LanguageTextField`，因为它提供了更方便的API。

#### 提供补全

如果希望在编辑器中添加自动补全功能，可以使用`TextFieldWithCompletion`。构造函数接受一个实现`TextCompletionProvider`的类作为参数，以提供补全选项。使用`TextFieldCompletionProvider`创建自己的提供者。为此，重写`addCompletionVariants()`并使用`CompletionResultSet.addElement()`添加补全选项。

参见`TextFieldCompletionProviderDumbAware`了解如何在索引阶段提供补全。

更多关于补全的信息，请参见[代码补全](https://www.jetbrains.com/help/idea/auto-completing-code.html)。

#### Java

如果您的插件依赖于Java功能并面向2019.2或更高版本，请参阅Java的相关文档。

`EditorTextField`的一个常见用例是输入Java类或包的名称。实现步骤如下：

1. 使用`JavaCodeFragmentFactory.createReferenceCodeFragment()`创建表示类或包名称的代码片段；
2. 调用`PsiDocumentManager.getDocument()`获取与代码片段对应的文档；
3. 将返回的文档传递给`EditorTextField`构造函数或其`setDocument()`方法。

示例代码：

```java
PsiFile psiFile = PsiDocumentManager.getInstance(project)
        .getPsiFile(editor.getDocument());
PsiElement element =
        psiFile.findElementAt(editor.getCaretModel().getOffset());

PsiExpressionCodeFragment code =
        JavaCodeFragmentFactory.getInstance(project)
        .createExpressionCodeFragment("", element, null, true);

Document document =
        PsiDocumentManager.getInstance(project).getDocument(code);

EditorTextField editorTextField =
        new EditorTextField(document, project, JavaFileType.INSTANCE);
```

#### 提示

- 当创建多个字段时，需要两个单独的文档。这可以通过使用`PsiExpressionCodeFragment`的不同实例来实现。
- `setText()`不再适用于输入字段。然而，`createExpressionCodeFragment()`接受字段的文本作为参数。可以替换空字符串并创建一个新文档来代替`setText()`。
- 在GUI构建器中，`JTextField`实例可以通过右键单击替换为自定义替换组件。确保使用“Custom Create”以便初始化代码正常工作。

这些功能使开发者能够在插件中实现丰富的编辑器功能，与IntelliJ平台的原生编辑器体验一致。



### 列表和树控件

#### JBList 和 Tree

在需要使用标准的Swing `JList`组件时，建议使用`JBList`类作为替代。`JBList`在`JList`的基础上增加了以下功能：

- **完整文本提示**：当列表项的文本无法完全显示时，会绘制一个工具提示显示完整文本。
- **空列表提示**：当列表为空时，在列表框中间绘制一条灰色的文本消息。可以通过调用`getEmptyText().setText()`来自定义文本内容。
- **忙碌图标**：在列表框右上角绘制一个忙碌图标，表示后台操作正在进行中。可以通过调用`setPaintBusy()`启用此功能。

类似地，`Tree`类提供了标准`JTree`类的替代。除了`JBList`的功能外，它还支持宽选择绘制（Mac风格）和拖放自动滚动。

#### ColoredListCellRenderer 和 ColoredTreeCellRenderer

如果需要自定义列表框或树项的展示，建议使用`ColoredListCellRenderer`或`ColoredTreeCellRenderer`作为单元格渲染器。这些类允许通过调用`append()`方法将多个具有不同属性的文本片段组合起来，并通过调用`setIcon()`为项设置一个可选的图标。渲染器会自动处理选中项的正确文本颜色设置以及其他平台特定的渲染细节。

#### ListSpeedSearch 和 TreeSpeedSearch

为了方便使用键盘选择列表框或树中的项，可以使用`ListSpeedSearch`和`TreeSpeedSearch`为其安装快速搜索处理程序。这可以通过简单地调用`new ListSpeedSearch(list)`或`new TreeSpeedSearch(tree)`来完成。要自定义用于定位元素的文本，可以重写`getElementText()`方法。或者，可以传递一个函数来将项目转换为字符串。函数需要作为`elementTextDelegate`传递给`ListSpeedSearch`构造函数，或作为`toString`传递给`TreeSpeedSearch`构造函数。

#### ToolbarDecorator

在插件开发中，常见的任务是显示一个列表或树，用户可以在其中添加、删除、编辑或重新排序项目。`ToolbarDecorator`类极大地方便了这项任务的实现。此类提供了一个带有项目操作的工具栏，并且如果基础列表模型支持，则自动启用列表框中的拖放重新排序功能。工具栏的位置取决于IDE运行的平台，可能在列表的上方或下方。

使用工具栏装饰器的步骤：

1. 如果需要支持列表框中项目的删除和重新排序，请确保您的列表模型实现了`EditableModel`接口。`CollectionListModel`是一个实现此接口的便捷模型类。

2. 调用`ToolbarDecorator.createDecorator()`创建一个装饰器实例。

3. 如果需要支持添加和/或删除项目，调用`setAddAction()`和/或`setRemoveAction()`。

4. 如果需要除标准按钮外的其他按钮，调用`addExtraAction()`或`setActionGroup()`。

5. 调用`createPanel()`并将其返回的组件添加到您的面板中。

这些控件和工具使开发者能够轻松创建和管理丰富的列表和树视图，从而提升插件的用户体验。

### 状态栏小部件

IntelliJ平台允许插件通过添加自定义小部件来扩展IDE的状态栏。状态栏小部件是一些小型UI元素，用于向用户提供当前文件、项目、IDE等相关的有用信息和设置。例如，状态栏中可以显示当前文件的编码或项目的当前版本控制系统（VCS）分支。

由于状态栏的显著展示位置和空间限制，状态栏小部件应仅用于始终显示的相关信息或设置。

#### 状态栏小部件的扩展

扩展状态栏以添加新小部件的起点是`StatusBarWidgetFactory`接口，该接口在`com.intellij.statusBarWidgetFactory`扩展点中注册。注意：在`plugin.xml`注册时必须提供`id`属性，并与`StatusBarWidgetFactory.getId()`的返回值匹配。

如果小部件提供与编辑器文件相关的信息或功能，可以考虑继承`StatusBarEditorBasedWidgetFactory`类。

每个小部件工厂从`createWidget()`返回一个新小部件。要控制小部件的销毁，请实现`disposeWidget()`，如果只想销毁它，请使用`Disposer.dispose(widget)`。

任何小部件都必须实现`StatusBarWidget`接口。

#### 重用IntelliJ平台的实现

您可以扩展以下两个类之一：

- `EditorBasedWidget`
- `EditorBasedStatusBarPopup`

##### EditorBasedWidget

`EditorBasedWidget`是基本的小部件实现。要实现它，重写`ID()`方法，该方法返回小部件的唯一ID。稍后可能需要此标识符来获取小部件实例。

使用以下现有的预定义小部件外观选项之一：

- `com.intellij.openapi.wm.StatusBarWidget.IconPresentation`：仅包含图标的小部件。
  - 示例：`PowerSaveStatusWidgetFactory`

- `com.intellij.openapi.wm.StatusBarWidget.TextPresentation`：仅包含文本的小部件。
  - 示例：`PositionPanel`

- `com.intellij.openapi.wm.StatusBarWidget.MultipleTextValuesPresentation`：包含文本和弹出窗口的小部件。
  - 示例：`DvcsStatusWidget`

注意，这些外观选项不能组合使用，例如无法同时显示图标和文本。

要使用选定的外观，从`getPresentation()`返回一个实现上述接口之一的类。

要创建包含自定义内容的小部件，应该实现`CustomStatusBarWidget`接口。重写`getComponent()`方法，返回自定义小部件的组件以显示。

示例：`MemoryUsagePanel`

##### EditorBasedStatusBarPopup

`EditorBasedStatusBarPopup`是所有带有动作列表弹出窗口的小部件的基础。例如，当前文件的编码小部件。

要显示的组件从`createComponent()`返回。每次更新小部件时，IDE都会调用`updateComponent()`更新此组件。在`updateComponent()`的实现中，可以描述小部件根据当前状态如何变化。

实现`getWidgetState()`方法以返回小部件的当前状态。当小部件更新时，此状态将传递给`updateComponent()`。该方法接受当前在编辑器中打开的文件。要创建自己的状态类，可以从`EditorBasedStatusBarPopup.WidgetState`继承。

实现`ID()`方法，返回小部件的唯一ID。稍后可能需要此标识符来获取小部件实例。

实现`createInstance()`方法，返回新的小部件实例。

最后，实现`createPopup()`方法，该方法返回当用户点击小部件时显示的弹出窗口。

可以使用`registerCustomListeners()`注册自定义监听器，以便在小部件更新时收到通知。

要更新小部件，请使用`update()`。

##### 在LightEdit模式下显示小部件

默认情况下，小部件在LightEdit模式下不显示。要显示小部件，请在工厂中实现`LightEditCompatible`接口。

通过这些接口和类，开发者可以在状态栏中添加功能丰富且视觉简洁的小部件，为用户提供即时的信息和操作入口。



### 各种Swing组件

#### 消息框（Messages）

`Messages`类提供了一种显示简单消息框、输入对话框（带有文本框的模态对话框）以及选择对话框（带有组合框的模态对话框）的方法。该类中不同方法的功能从名称上应该就可以看出。当在macOS上运行时，`Messages`类显示的消息框使用原生UI。

`showCheckboxMessageDialog()`函数提供了一种在消息中实现“不再显示此消息”复选框的简单方法。

注意：建议尽可能使用非模态通知而不是模态消息框。有关详细信息，请参阅通知部分。

#### JBSplitter

`JBSplitter`类是JetBrains对标准`JSplitPane`类的替代。与其他一些JetBrains增强的Swing组件不同，它不是直接替换的，具有不同的API。然而，为了实现一致的用户体验，建议使用`JBSplitter`而不是标准`JSplitPane`。

要向分割器中添加组件，请调用`setFirstComponent()`和`setSecondComponent()`方法。

`JBSplitter`支持自动记住分割比例。要启用此功能，请调用`setSplitterProportionKey()`方法，并传递存储比例的ID。

#### JBTabs

`JBTabs`类是JetBrains的标签控件实现，用于编辑器标签和其他一些组件。与标准Swing标签相比，它具有显著不同的外观和感觉，在macOS平台上看起来不太原生，因此开发人员可以根据需要选择使用哪种标签控件更合适。

#### 工具栏

有关概述，请参阅UI指南中的工具栏部分。

“从操作构建UI”涵盖了创建基于`AnAction`的工具栏的内容。

这些组件使开发者能够创建和管理丰富的用户界面元素，提供直观的用户体验和一致的视觉风格。



### 图标使用指南

#### 平台图标 vs 自定义图标

IntelliJ平台插件广泛使用图标。插件主要需要图标用于动作、定制组件渲染器、工具窗口等。

插件Logo（代表插件本身）的要求与插件内使用的图标不同。更多信息请参见插件Logo部分。

**平台图标 vs 自定义图标**

插件应尽可能重复使用现有的平台图标。

使用Icons列表浏览现有图标。平台图标位于`AllIcons`中。来自插件的图标位于相应的`<PLUGIN_NAME>Icons`类中（例如，`GithubIcons`）。

如果需要自定义图标，请参阅详细的设计指南。

#### 图标组织

参考`Action Basics`示例插件。

对于基于Gradle的项目，图标应放置在资源目录中。如果项目基于DevKit，推荐的做法是将图标放置在标记为资源根的专用源根中，例如`icons`或`resources`。

如果图标仅在`plugin.xml`属性或元素中引用，或者在`@Presentation`图标属性中引用，则可以通过路径引用它们。如果图标在代码和/或XML中多次引用，则将它们组织在图标持有类中会更方便。

#### 图标类

在顶级包中定义一个名为`icons`的类/接口，持有图标常量作为静态字段：

```java
package icons;

public interface MyIcons {
  Icon MyAction = IconLoader.getIcon("/icons/myAction.png", MyIcons.class);
  Icon MyToolWindow = IconLoader.getIcon("/icons/myToolWindow.png", MyIcons.class);
}
```

`IconLoader`的`getIcon()`方法可用于访问图标。传递给`IconLoader.getIcon()`的图标路径必须以`/`开头。

从2021.2开始，`Icons`类不再需要位于`icons`包中，但可以使用插件的包名：`icons.MyIcons` → `com.example.plugin.MyIcons`。

#### 使用图标

在`plugin.xml`中使用`icon`属性定义的图标，例如`<action>`或扩展点中的图标，以及`@Presentation`的`icon`属性中的图标，可以通过两种方式引用：

- 通过图标文件路径
- 通过图标持有类中的图标常量

要通过路径引用图标，请提供相对于资源目录的路径，例如，对于位于`my-plugin/src/main/resources/icons`目录中的图标：

```xml
<actions>
  <action icon="/icons/myAction.svg" ... />
</actions>

<extensions defaultExtensionNs="com.intellij">
  <toolWindow icon="/icons/myToolWindow.svg" ... />
</extensions>
```

在图标持有类的情况下，引用图标常量。如果类位于顶级`icons`包中，名称`icons`将自动加前缀，不必指定。在自定义包中放置类的情况下，必须提供完整的包名，例如：

```xml
<actions>
  <!-- 引用顶级 'icons' 包中的类的图标 -->
  <action icon="MyIcons.MyAction" ... />
</actions>

<extensions defaultExtensionNs="com.intellij">
  <!-- 引用自定义包中的类的图标 -->
  <toolWindow icon="com.example.plugin.MyIcons.MyToolWindow" ... />
</extensions>
```

#### 图标格式

IntelliJ平台支持Retina显示屏，并且有一个捆绑的暗主题Darcula。因此，每个图标都应有一个针对Retina设备和Darcula主题的专用变体。如果原始图标在Darcula下看起来不错，可以省略暗变体。

根据使用情况，所需的图标大小如下表所示：

| 使用场景             | 图标大小（像素） |
| -------------------- | ---------------- |
| 节点、动作、文件类型 | 16x16            |
| 工具窗口             | 13x13            |
| 编辑器槽             | 12x12            |
| 编辑器槽（新UI）     | 14x14            |

##### SVG 格式

自2018.2以来，支持SVG（可缩放矢量图形）图标。

由于SVG图标可以任意缩放，它们在HiDPI环境或结合更大屏幕字体使用时效果更佳（例如，在演示模式下）。

提供基准尺寸，表示以1倍缩放比例渲染的图像大小（用户空间）。尺寸通过`width`和`height`属性设置，不需要单位。如果未指定，默认为16x16像素。

最小的SVG图标文件示例：

```xml
<svg xmlns="http://www.w3.org/2000/svg" width="16" height="16">
  <rect width="100%" height="100%" fill="green"/>
</svg>
```

用于PNG图标的命名规则（参见下文）仍然适用。

然而，SVG图标的@2x版本应提供相同的基准尺寸。这样的图标图形可以通过双精度表达更多细节。如果图标图形足够简单，可以在任何缩放比例下完美渲染，那么可以省略@2x版本。

##### PNG 格式（已弃用）

如果您的插件目标是2018.2+，请使用SVG图标。

所有图标文件必须放置在同一目录中，遵循以下命名模式（将`.png`替换为`.svg`以使用SVG图标）：

| 主题/分辨率      | 文件名模式           | 大小      |
| ---------------- | -------------------- | --------- |
| 默认             | iconName.png         | W x H     |
| Darcula          | iconName_dark.png    | W x H     |
| 默认 + Retina    | iconName@2x.png      | 2*W x 2*H |
| Darcula + Retina | iconName@2x_dark.png | 2*W x 2*H |

`IconLoader`类将根据当前环境加载最佳匹配的图标。

以下是`toolWindowStructure.png`图标表示的示例：

| 主题/分辨率      | 文件名                          | 图标                     |
| ---------------- | ------------------------------- | ------------------------ |
| 默认             | toolWindowStructure.png         | 工具窗口结构             |
| Darcula          | toolWindowStructure_dark.png    | 工具窗口结构，暗         |
| 默认 + Retina    | toolWindowStructure@2x.png      | 工具窗口结构，Retina     |
| Darcula + Retina | toolWindowStructure@2x_dark.png | 工具窗口结构，Retina，暗 |

##### 动画图标

动画图标是一种显示插件正在执行某些长时间操作（例如，当插件加载数据时）的方法。

任何动画图标都是一组带有延迟的帧循环。

要创建新的动画图标，请使用`AnimatedIcon`。要创建帧按相同延迟连续的图标，请使用接受延迟和图标的构造函数：

```java
AnimatedIcon icon = new AnimatedIcon(
    500,
    AllIcons.Ide.Macro.Recording_1,
    AllIcons.Ide.Macro.Recording_2);
```

要从不同延迟的帧创建图标，请使用`AnimatedIcon.Frame`。每个帧代表一个图标，以及到下一帧的延迟。

使用预定义的`AnimatedIcon.Default`加载图标以指示长时间的过程。此图标有一个更大的`AnimatedIcon.Big`版本。

将`true`添加到列表、表格和树组件的`AnimatedIcon.ANIMATION_IN_RENDERER_ALLOWED`客户端属性中，以自动重新绘制动画图标。有关详细信息，请参阅`ANIMATION_IN_RENDERER_ALLOWED`的Javadoc。

#### 图标工具提示

通过`com.intellij.iconDescriptionBundle`扩展点注册资源包，以自动为所有`SimpleColoredComponent`渲染器提供工具提示。

在资源包中创建`icon.<icon-path>.tooltip`键，其中`<icon-path>`是带有前导斜杠的图标路径，去掉`.svg`，并将斜杠替换为点（例如，`/nodes/class.svg` → `icon.nodes.class.tooltip`）。

### 新UI图标

有关指南和概述，请参阅新UI图标指南。

要完全支持新UI，插件必须提供附加的专用图标和映射信息。这样可以根据用户选择同时支持两个UI变体。

#### 设置

在图标根目录中创建一个新的`expui`目录。

将新UI的所有图标复制到此目录中。

在资源根目录中创建一个空的`$PluginName$IconMappings.json`映射文件。

通过`com.intellij.iconMapper`扩展点在`plugin.xml`中注册`$PluginName$IconMappings.json`。

来自Maven插件的示例设置：

- 图标资源根目录：`images`
- 映射文件：`MavenIconMappings.json`
- 扩展点注册（`<iconMapper mappingFile="MavenIconMappings.json"/>`）：`plugin.xml`

#### 映射条目

所有新UI图标必须在`$PluginName$IconMappings.json`映射

### 用户界面常见问题

#### 检查现有的UI

使用**UI Inspector**可以在运行时定位底层的Swing组件实现或检查现有的UI。

#### 有用的类

- **com.intellij.ui** 包
- **com.intellij.util.ui** 包

#### 颜色

始终使用**JBColor**而不是普通的`java.awt.Color`（通过插件DevKit的检查插件高亮显示）。自定义颜色必须通过当前主题设置的`JBColor.namedColor()`检索。有关如何公开相应的元数据，请参阅[Exposing Theme Metadata](https://www.jetbrains.org/intellij/sdk/docs/reference_guide/ui_themes.html)。

如果需要从一个地方检索颜色并在另一个地方使用，不要只检索一次并使用检索到的值。相反，使用`JBColor.lazy()`并传递一个lambda表达式来检索颜色。这个lambda表达式需要足够快速和安全，因为它每次检索颜色时都会被调用，例如，用于绘制。这样做可以确保如果颜色在源头发生变化，例如由于主题或方案更改，它将被正确更新。

通用的UI颜色（例如用于绘制边框的颜色）可以通过`UIUtil`和`JBUI`访问。在`JBColor`、`Gray`和`LightColors`中定义了一些硬编码颜色。

`ColorUtil`允许调整现有颜色。

#### 文本

参见[UI Guidelines: Data Formats](https://www.jetbrains.org/intellij/sdk/docs/reference_guide/ui_themes.html)了解数据格式的使用。

使用**NaturalComparator**进行"自然"排序。

`StringUtil`包含许多用于UI文本处理的有用方法：

- `unpluralize()`/`pluralize()`：使用英语规则进行单复数转换。
- `formatDuration()`：格式化时长，例如：2 m 3 s 456 ms。
- `formatFileSize()`：格式化文件大小，例如：1.23 KB。
- `escapeLineBreak()`及相关方法：转义特殊字符。
- `shortenTextWithEllipsis()`和`shortenPathWithEllipsis()`：生成以'…'结尾的缩写UI文本。
- `quote()`和`unquoteString()`：为值添加引号，例如：`Usages of "$value$": 218 found`。

关于插件国际化的信息，请参见[Internationalization](https://www.jetbrains.org/intellij/sdk/docs/basics/internationalization.html)。

要生成本地化消息，请参见[NlsMessages](https://www.jetbrains.org/intellij/sdk/docs/basics/internationalization.html)。

#### "最近使用"条目

要存储和检索"最近使用"的条目值（例如，过滤值），请使用**RecentsManager**。

#### 当前主题：亮色或暗色？

要确定当前主题的样式，请使用`JBColor.isBright()`。

#### 边框和内边距

参见[UI Guidelines: Layout](https://www.jetbrains.org/intellij/sdk/docs/reference_guide/ui_themes.html)了解布局。

始终通过**JBUI.Borders**和**JBUI.Insets**中的工厂方法创建边框和内边距，这些方法创建DPI感知的实例。使用标准的DPI不感知实例（通过插件DevKit的检查插件高亮显示）可能会导致UI布局问题。

如果在空边框中使用DPI感知的内边距（`JBUI.Borders.empty()`），那么如果由于IDE缩放操作或其他原因导致缩放更改，内边距将自动更新。如果在其他地方使用内边距，则需要在组件的`updateUI()`方法中手动调用`JBInsets.update()`以相应更新内边距。

#### 有什么缺失的吗？

如果您感兴趣的话题未涵盖在上述部分中，请通过页面底部的"Was this page helpful?"反馈表或其他渠道告知我们。

请具体说明您想要添加的话题及其原因，并留下您的电子邮件以便我们需要更多详细信息时联系您。感谢您的反馈！

这些指南和最佳实践帮助开发者在IntelliJ平台上创建一致且美观的用户界面。

### 嵌入式浏览器 (JCEF)

JCEF（Java Chromium Embedded Framework）是CEF的Java移植版本。自2020.1作为实验功能引入，2020.2起默认启用。JCEF允许在Swing应用程序中嵌入基于Chromium的浏览器。

#### JCEF的应用场景

- 渲染HTML内容
- 预览生成的HTML（例如，从Markdown生成）
- 创建自定义的基于Web的组件（例如，图表预览、图像浏览器等）

建议在默认的IntelliJ平台UI框架（Swing）中实现UI。只有在插件需要显示HTML文档或标准UI创建方法不足时，才考虑使用JCEF。

JCEF取代了以前在IDE中用于渲染Web内容的JavaFX。

#### 启用JCEF

- **2020.2及更高版本**：JCEF默认启用，无需额外操作。

#### 在插件中使用JCEF

`JBCefApp`是IntelliJ平台API中暴露的核心JCEF类，负责初始化JCEF上下文和管理其生命周期。

无需显式初始化`JBCefApp`，在调用`JBCefApp.getInstance()`或创建浏览器或客户端对象时会自动完成。

在使用JCEF API之前，需要检查JCEF是否在运行的IDE中受支持。可以通过调用`JBCefApp.isSupported()`来完成：

```java
if (JBCefApp.isSupported()) {
  // 使用JCEF
} else {
  // 可选的回退到无浏览器的解决方案
}
```

JCEF可能不受支持的情况包括：

- IDE使用了不包含JCEF的替代JDK启动。
- 版本与运行的IDE不兼容。

#### 浏览器

JCEF浏览器由`JBCefBrowser`类表示，负责在实际的基于Chromium的浏览器中加载和渲染请求的文档。

可以使用`JBCefBrowser`类的构造函数或`JBCefBrowserBuilder`创建JCEF浏览器。构造函数适用于默认客户端和默认选项的浏览器。构建器方法允许使用自定义客户端和配置其他选项。

#### 添加浏览器到UI

`JBCefBrowser.getComponent()`暴露嵌入实际浏览器的UI组件。该组件是Swing `JComponent`的一个实例，可以添加到插件UI中：

```java
// 假设 'JPanel myPanel' 是工具窗口UI的一部分
JBCefBrowser browser = new JBCefBrowser();
myPanel.add(browser.getComponent());
```

#### 加载文档

要在浏览器中加载文档，请使用`JBCefBrowserBase.load*()`方法。这些方法可以从EDT和后台线程调用。可以设置初始URL（传递给构造函数或构建器），该URL将在浏览器创建和初始化时加载。

#### 浏览器客户端

浏览器客户端提供了设置与各种浏览器事件相关的处理程序的接口，例如：

- HTML文档加载完成
- 控制台消息打印
- 浏览器获得焦点

处理程序允许在插件代码中对这些事件作出反应并改变浏览器的行为。每个浏览器与一个客户端绑定，一个客户端可以与多个浏览器实例共享。

浏览器客户端由`JBCefClient`表示，这是`JCEF`的`CefClient`的包装器。`JBCefClient`允许注册多个相同类型的处理程序，这在`CefClient`中是不可能的。要访问底层的`CefClient`及其API，请调用`JBCefClient.getCefClient()`。

#### 创建和访问客户端

如果创建`JBCefBrowser`实例时未传递特定客户端，则会绑定到默认客户端。默认客户端会在关联的浏览器实例销毁时自动销毁。

对于更高级的用例，可以通过调用`JBCefApp.createClient()`创建自定义客户端并注册所需的处理程序。自定义客户端必须在插件代码中显式销毁。

要访问与浏览器关联的客户端，请调用`JBCefBrowser.getJBCefClient()`。

#### 事件处理程序

JCEF API提供了各种事件处理程序接口，允许处理浏览器发出的广泛事件。示例处理程序包括：

- `CefLoadHandler` - 处理浏览器加载事件。
- `CefDisplayHandler` - 处理与浏览器显示状态相关的事件。
- `CefContextMenuHandler` - 处理上下文菜单事件。
- `CefDownloadHandler` - 处理文件下载事件。

每个处理程序接口，JCEF API都提供了一个适配器类，可以扩展它以避免实现未使用的方法，例如，`CefLoadHandlerAdapter`。

#### 执行JavaScript

JCEF API允许从插件代码在嵌入式浏览器中执行JavaScript代码。JavaScript可用于操作DOM、创建所需功能的函数、注入样式等。

在最简单的情况下，可以使用`JBCefBrowser.getCefBrowser().executeJavaScript()`执行JavaScript代码，例如：

```java
browser.getCefBrowser().executeJavaScript(
    "alert('Hello World!')",
    url, lineNumber
);
```

上述代码将在嵌入式浏览器中执行，并显示包含“Hello World!”消息的警报框。`url`和`lineNumber`参数用于浏览器中的错误报告，如果脚本抛出错误，它们有助于调试。

#### 从JavaScript执行插件代码

JCEF不提供从插件代码直接访问DOM的功能（未来可能会改变），异步与JavaScript的通信通过回调机制实现。它允许通过JavaScript从嵌入式浏览器中执行插件代码，例如，当点击按钮或链接时，按下快捷键时，调用JavaScript函数时等。

JavaScript查询回调由`JBCefJSQuery`表示。它是绑定到特定浏览器的对象，持有实现所需插件行为的处理程序集。

示例代码展示如何在编辑器中打开本地文件链接和在外部浏览器中打开外部链接：

```java
JBCefJSQuery openLinkQuery = JBCefJSQuery.create(browser); // 1
openLinkQuery.addHandler((link) -> { // 2
    if (LinkUtil.isExternal(link)) {
      BrowserUtil.browse(link);
    } else {
      EditorUtil.openFileInEditor(link);
    }
    return null; // 3
});

browser.getCefBrowser().executeJavaScript( // 4
    "window.openLink = function(link) {" +
      openLinkQuery.inject("link") + // 5
      "};",
    browser.getCefBrowser().getURL(), 0
);

browser.getCefBrowser().executeJavaScript( // 6
    """
    document.addEventListener('click', function (e) {
      const link = e.target.closest('a').href;
      if (link) {
        window.openLink(link);
      }
    });""",
    browser.getCefBrowser().getURL(), 0
);
```

1. 创建`JBCefJSQuery`实例。确保传递的浏览器实例是`JBCefBrowserBase`类型（可能需要转换）。
2. 添加一个实现插件代码的处理程序。示例实现根据链接是本地还是外部在编辑器中打开链接或在外部浏览器中打开。
3. 处理程序可以选择返回`JBCefJSQuery.Response`对象，包含插件代码端发生的成功或错误信息。如果需要，可以在浏览器中处理。
4. 执行JavaScript，创建自定义`openLink`函数。
5. 注入调用第2步中实现的插件代码的JavaScript代码。每次调用`openLink`函数时，会调用`openLinkQuery`添加的处理程序。
6. 执行JavaScript，在浏览器中注册点击事件监听器。每次点击`a`元素时，监听器会调用第4步中定义的`openLink`函数，传递点击链接的`href`值。

#### 从插件分发中加载资源

如果插件功能实现了基于Web的UI，插件可以在其分发中提供HTML、CSS和JavaScript文件，或根据某些配置动态生成这些文件。此类资源无法轻松地被浏览器访问。可以通过实现适当的请求处理程序，将它们在预定义的URL下提供给浏览器。

这种方法需要实现`CefRequestHandler`和`CefResourceRequestHandler`，将资源路径映射到资源提供者。

提供此类资源的示例是用于显示SVG文件的Image Viewer组件，见`JCefImageViewer`及相关类的实现细节。





### 颜色方案管理

IntelliJ IDEA 12.1 中的颜色方案管理进行了更改，以简化方案设计师的工作，并使不同编程语言的方案看起来同样好，即使它们没有专门为这些语言设计。以前，语言插件使用了与黑暗主题不兼容的固定默认颜色。

新的实现允许指定一组与特定语言无关的标准文本属性的依赖关系。方案设计师仍然可以根据需要设置特定于语言的属性，但这是可选的。新的颜色方案使用新的 `.icls`（Idea CoLor Scheme）扩展名，以避免与旧平台版本的兼容性问题：如果只设置了标准属性，它们不会被12.1之前的版本使用，从而导致不同的高亮颜色。

#### 插件开发者

##### 文本属性键依赖关系

指定高亮文本属性的最简单和最佳方法是依赖于 `DefaultLanguageHighlighterColors` 中定义的标准键之一：

```java
static final TextAttributesKey MY_KEYWORD =
  TextAttributesKey.createTextAttributesKey("MY_KEYWORD", DefaultLanguageHighlighterColors.KEYWORD);
```

颜色方案管理器将首先搜索由 `MY_KEYWORD` 键指定的文本属性。如果未明确定义这些属性或所有属性为空（未定义），它将使用 `DEFAULT_KEYWORD` 键进行搜索。如果两者都未定义，它将进一步回退到默认方案。

文本属性键可以链式定义，例如，您可以定义另一个键如下：

```java
static final TextAttributesKey MY_PREDEFINED_SYMBOL =
  TextAttributesKey.createTextAttributesKey("MY_PREDEFINED_SYMBOL", MY_KEYWORD);
```

规则相同：如果无法通过 `MY_PREDEFINED_SYMBOL` 键找到文本属性或它们为空，颜色方案管理器将搜索 `MY_KEYWORD`，如果未找到（空），将进一步查找 `DEFAULT_KEYWORD`。

强烈不建议使用固定的默认属性。

如果您不确定要使用哪个基本键，最好选择最通用的一个，例如 `DefaultLanguageHighlighterColors.IDENTIFIER`。请记住，使用固定的默认属性将迫使方案设计师显式设置该元素的颜色。否则，其默认颜色可能与颜色方案视觉冲突。如果方案设计师没有语言插件，他将无法解决这个问题。

##### 为特定方案提供属性

语言插件可以为 "Default" 和 "Darcula" 捆绑方案或基本上任何其他已知名称的方案提供默认文本属性。这可以通过在 `plugin.xml` 中添加 `com.intellij.additionalTextAttributes` 扩展点来完成，提供包含所需文本属性的文件的名称，例如：

```xml
<extensions defaultExtensionNs="com.intellij">
  ...
  <additionalTextAttributes
      scheme="Default"
      file="colorSchemes/MyLangDefault.xml"/>
  ...
</extensions>
```

这告诉IDE文件 `MyLangDefault.xml` 必须在 `colorSchemes` 下的资源中搜索。请注意，路径不应以反斜杠开头，其全限定名（在我们的例子中是 `colorSchemes/MyLangDefault.xml`）必须是唯一的，以避免不同提供者之间的命名冲突。因此，强烈建议添加语言前缀，例如 "MyLang"。

文件本身是包含所需属性的颜色方案的提取，例如：

```xml
<?xml version='1.0'?>
<list>
  <option name="MY_VAR">
    <value>
      <option name="FOREGROUND" value="660000"/>
    </value>
  </option>
  <option name="MY_SPECIAL_CHAR">
    <value>
      <option name="FOREGROUND" value="008000"/>
      <option name="BACKGROUND" value="e3fcff"/>
      <option name="FONT_TYPE" value="1"/>
    </value>
  </option>
</list>
```

注意：当通过“另存为...”复制方案时，所有属性，包括扩展点中定义的属性，将被复制到新方案中。方案设计师可能需要检查这些复制的属性是否与他们的颜色方案冲突，尽管在这种情况下，插件已安装，并且不应该引起任何问题。无论如何，尽量坚持简单的键依赖，如果可能的话（注意它在 "Darcula" 上表现良好），仅在必要时提供显式属性。

#### 方案设计师

##### 创建新方案的典型工作流程

1. 选择将用作基础的方案，例如 "Default"。
2. 点击“另存为...”，并为新方案命名。
3. 首先在“常规”部分设置属性，然后继续设置“语言默认”。
4. 检查所有语言，并在必要时调整特定语言的文本属性。在大多数情况下，这可能是不必要的，但有两种情况可能需要额外的操作：
   - 有一个过时的插件不使用新的颜色方案管理API，因此不使用语言默认中设置的属性。理想情况下，必须为语言插件创建一个报告，以便其作者最终修复它。
   - 插件有意设置了一些默认颜色，如果方案是从默认方案创建的，这些颜色将被复制到新创建的方案中。这可以通过重置所有属性来解决，以恢复对语言默认的继承（见下文），或通过设置其他适合方案的颜色来解决。

第一种方式是首选的，因为它需要更少的努力来以后更改颜色方案。

##### 文本属性继承

对于许多没有任何值的语言文本属性，将有一行表示属性继承自特定部分/属性，例如语言默认中的关键词。如果元素设置了任何属性，则仅使用这些属性。基础元素的所有属性将被忽略。要恢复继承，请取消选中所有框并点击应用。

这些指南和最佳实践帮助开发者在IntelliJ平台上管理和设计颜色方案，确保了在各种编程语言和主题下的一致性和可读性。



### 暴露主题元数据

所有可用于自定义主题的UI自定义键必须定义在一个专用的`*.themeMetadata.json`文件中，并通过`com.intellij.themeMetadataProvider`扩展点注册。

#### 示例

以下是一个暴露插件UI自定义键的最小示例，展示了所需的所有细节。

**plugin.xml**:

```xml
<idea-plugin>
  <extensions defaultExtensionNs="com.intellij">
    <themeMetadataProvider path="/META-INF/MyPlugin.themeMetadata.json"/>
  </extensions>
</idea-plugin>
```

**MyPlugin.themeMetadata.json**:

```json
{
  "name": "My Plugin",
  "fixed": false,
  "ui": [
    {
      "key": "MyComponent.border",
      "description": "The border for my component. Not used anymore.",
      "deprecated": true,
      "source": "com.example.MyComponent"
    }
  ]
}
```

#### 属性说明

- **name**: 人类可读的名称，例如插件名称。
- **fixed**: 指定元数据是否描述外部元素，例如UI库。默认值为`false`。
- **ui**: 列出所有自定义键的根元素。
  - **key**: 自定义键名（请参阅键命名方案）。
  - **description**: 为编辑 `*.theme.json` 文件的主题作者显示的描述。
  - **deprecated**: 如果键已弃用，设置为`true`。强烈建议在说明中提供解释和/或替代方案。
  - **source**: 底层UI组件实现的全限定名称，例如`javax.swing.JPasswordField`。
  - **since**: 暴露此UI自定义键的版本号，例如 `2021.1`。

建议始终提供说明条目，以便主题作者了解使用场景。不要移除现有的键，而是将它们标记为弃用，以帮助主题作者升级现有的主题。

颜色键可以通过 `JBColor.namedColor()` 使用，为明亮和黑暗主题提供默认值：

```java
private static final JBColor SECTION_HEADER_FOREGROUND =
    JBColor.namedColor(
      "Plugins.SectionHeader.foreground",
      new JBColor(0x787878, 0x999999)
    );
```

其他键可以通过 `javax.swing.UIManager#getXXX()` 方法获取。

#### 键命名方案

所有键必须遵循以下命名模式：

```
Object[.SubObject].[state][Part]Property
```

##### 属性

| 属性        | 说明                                                         | 示例                                    |
| ----------- | ------------------------------------------------------------ | --------------------------------------- |
| foreground  | 文本颜色。                                                   | Label.foreground                        |
| background  | 具有文本对象的背景色。                                       | Label.background                        |
| <part>Color | 具有单一颜色的对象（没有前景色/背景色）。不要单独使用“Color”一词，始终与“part”一词一起使用。 | Popup.borderColor, Group.separatorColor |

##### 状态

| 状态     | 说明                                                     | 示例                                    |
| -------- | -------------------------------------------------------- | --------------------------------------- |
| Active   | 启用组件，默认状态。省略此词。默认状态不需要显式命名。   | Notification.background                 |
| Inactive | 启用的组件，可能被认为是交互的但实际上不是。             | Tree.inactiveBackground                 |
| Focused  | 当前聚焦的组件。                                         | Button.focusedBorderColor               |
| Selected | 选中的选项卡或其他具有相等意义的选定和未选中状态的控件。 | ToolWindow.HeaderTab.selectedBackground |
| Hover    | 悬停状态。                                               | Link.hoverForeground                    |
| Pressed  | 按下状态。                                               | Link.pressedForeground                  |
| Error    | 错误状态。                                               | ValidationTooltip.errorBackground       |
| Warning  | 警告状态。                                               | ValidationTooltip.warningBorderColor    |
| Success  | 成功状态。                                               |                                         |
| Disabled | 不可用组件。                                             | Label.disabledForeground                |

##### 部分

部分是组件的内部元素，例如组合框中的箭头按钮。如果其属性与父对象不同，请为部分创建一个单独的键。

##### 子对象

当为以下情况创建键时，使用子对象：

1. 实现变体。通常具有与父对象相似的一组UI属性键。例子：
   - 默认按钮：Button.Default.background
   - 工具窗口通知：Notification.ToolWindow.errorBackground
2. 复杂组件的内部小组件，具有自己的UI和行为。例如：
   - 工具窗口选项卡：ToolWindow.HeaderTab.inactiveBackground
   - 弹出窗口底部的提示文本：Popup.Advertiser.background

##### 渐变色

如果组件具有渐变色，请为渐变的开始和结束添加“start”和“end”字样。例如：

- Button.startBorderColor / Button.endBorderColor
- SearchMatch.startBackground / SearchMatch.endBackground

##### 大小写

对象和子对象首字母大写。属性使用小写驼峰式。

##### 禁止使用

不要单独使用"Color"一词，替代用`<Part>Color`。不要使用"Outline"，替代用`borderColor`。

### 结论

通过遵循这些命名方案和建议，开发者可以创建具有一致性的可自定义UI，同时为主题作者提供灵活性和清晰的指导。这些方案旨在确保在不同主题和外观下的可视化一致性和易用性。



# 动作 (Actions)

动作系统允许插件将其项目添加到基于 IntelliJ 平台的 IDE 菜单和工具栏。例如，某些动作类负责“文件”菜单中的“打开文件...”项目和工具栏中的“打开...”按钮。

在 IntelliJ 平台中，动作需要代码实现并且必须注册。动作实现决定了动作在何种上下文中可用，以及在 UI 中选择时的功能。注册决定了动作在 IDE UI 中的显示位置。一旦实现并注册，一个动作将响应用户手势收到来自 IntelliJ 平台的回调。

[创建动作教程](https://plugins.jetbrains.com/docs/intellij/creating-actions.html) 描述了向插件添加自定义动作的过程。[分组动作教程](https://plugins.jetbrains.com/docs/intellij/grouping-actions.html) 演示了三种可以包含动作的组。

### 动作实现

一个动作是继承自抽象类 `AnAction` 的类。对于在哑模式下可用的动作，扩展 `DumbAwareAction`。有关详细信息，请参见下文的有用动作基类。

IntelliJ 平台在用户与菜单项或工具栏按钮交互时调用动作的方法。

#### 主实现重写

每个 IntelliJ 平台动作都应该重写 `AnAction.update()`，并且必须重写 `AnAction.actionPerformed()`。

**AnAction.update()**  
动作的 `AnAction.update()` 方法由 IntelliJ 平台框架调用，以更新动作状态。动作的状态（启用、可见）决定了该动作在 UI 中是否可用。此方法传递的 `AnActionEvent` 对象包含有关动作当前上下文的信息。

动作通过更改与事件上下文关联的 `Presentation` 对象的状态来启用。实现者必须确保更改表示和可用性状态处理所有变体和状态转换；否则，给定的动作将会“卡住”。

**AnAction.getActionUpdateThread()**  
`AnAction.getActionUpdateThread()` 返回一个 `ActionUpdateThread`，该线程指定 `update()` 方法是在线程 (BGT) 还是事件分发线程 (EDT) 上调用。首选方法是在 BGT 上运行更新，这样可以保证对 PSI、虚拟文件系统 (VFS) 或项目模型的应用范围内的读取访问。更新会话在 BGT 上运行的动作不应直接访问 Swing 组件层次结构。相反，指定在 EDT 上运行其更新的动作不得访问 PSI、VFS 或项目数据，但可以访问 Swing 组件和其他 UI 模型。

所有可访问的数据都由 `DataContext` 提供。如有必要，可以使用 `AnActionEvent.getUpdateSession()` 访问 `UpdateSession`，然后调用 `UpdateSession.compute()` 以在 EDT 上运行函数。

**AnAction.actionPerformed()**  
动作的 `AnAction.actionPerformed()` 方法由 IntelliJ 平台调用（如果可用且用户选择了该动作）。此方法是动作的主要实现部分：它包含在动作被调用时执行的代码。`actionPerformed()` 方法还接收一个 `AnActionEvent` 作为参数，用于访问项目、文件、选择等上下文数据。

### 动作分组

动作组将动作组织成逻辑 UI 结构，这些结构可以包含其他组。一个动作组可以形成一个工具栏或菜单。组的子组可以形成菜单的子菜单。

动作可以包含在多个组中，因此可以出现在 UI 中的不同位置。一个动作在 UI 中的每个位置必须有一个唯一的标识符。



### Presentation

在 IntelliJ 平台的动作系统中，**Presentation**（表示）对象是用来封装动作在用户界面中的视觉表示和状态的。它包括以下属性：

- **文本**：在菜单、工具栏或其他 UI 元素中显示的动作标签。
- **描述**：对动作功能的简要说明，通常用作工具提示。
- **图标**：与该动作相关的图形表示。
- **启用状态**：指示该动作当前是否可用。
- **可见状态**：指示该动作当前是否在 UI 中可见。

每个 `AnAction`（动作类）在不同的上下文中可以有不同的表示。例如，同一个动作在工具栏和菜单中可能会显示不同的文本或图标。表示对象对于提供一致且用户友好的体验至关重要，它通过适应当前的上下文和用户交互来调整动作的外观和可用性。

通常在动作的 `update()` 方法中操作 `Presentation` 对象，以根据当前应用程序的状态动态更改动作的状态（启用/禁用）或外观。例如，如果某个动作仅在有打开的项目时可用，则 `update()` 方法可以在没有项目打开时禁用该动作，通过调用 `Presentation.setEnabled(false)` 实现。

可以通过传递给动作方法的 `AnActionEvent` 对象访问 `Presentation` 对象，从而使动作能够动态调整其 UI 表现。

每个动作在界面中的不同位置都会创建一个新的 `Presentation` 对象。因此，同一个动作在不同的用户界面位置中可以有不同的文本或图标。不同位置的 `Presentation` 是通过复制 `AnAction.getTemplatePresentation()` 方法返回的模板 `Presentation` 来创建的。这种设计允许动作在不同的上下文中展示不同的外观和行为。

#### The compact Attribute

一个组的 `compact` 属性决定了该组中的动作在禁用时是否可见。有关如何为一个组设置 `compact` 属性的解释，请参阅 `plugin.xml` 中的动作注册部分。如果一个菜单组的 `compact` 属性设置为 `true`，那么菜单中的一个动作只有在其状态为启用且可见时才会显示。相反，如果 `compact` 属性为 `false`，那么即使该动作的状态是禁用但可见，它仍然会在菜单中显示。例如，像 `Tools` 这样的菜单组设置了 `compact` 属性，所以如果某个动作没有启用，就无法显示在 `Tools` 菜单中。

以下是 `compact` 属性设置对菜单项可见性和灰显外观的影响：

| Host Menu `compact` Setting | Action Enabled | Visibility Enabled | Menu Item Visible? | Menu Item Appears Gray? |
| --------------------------- | -------------- | ------------------ | ------------------ | ----------------------- |
| `T`                         | `F`            | `T`                | `F`                | N/A                     |
| `T`                         | `T`            | `T`                | `T`                | `T`                     |
| `F`                         | `F`            | `T`                | `T`                | `T`                     |
| `F`                         | `T`            | `T`                | `T`                | `F`                     |

在所有其他情况下，`compact`、可见性和启用状态的组合不会出现灰显外观，因为菜单项不可见。

### 例子

可以参考动作分组教程中的例子，了解如何创建动作组并设置 `compact` 属性。



### 注册动作

注册动作有两种主要方式：在插件的 `plugin.xml` 文件中的 `<actions>` 部分列出动作，或者通过代码注册。

#### 在 plugin.xml 中注册动作

在 `plugin.xml` 中注册动作的示例如下，展示了 `<actions>` 部分中使用的所有元素和属性，并描述了每个元素的含义。

```xml
<actions>

  <action
      id="VssIntegration.GarbageCollection"
      class="com.example.impl.CollectGarbage"
      text="Garbage Collector: Collect _Garbage"
      description="Run garbage collector"
      icon="icons/garbage.png">

    <override-text place="MainMenu" text="Collect _Garbage"/>
    <override-text place="EditorPopup" use-text-of-place="MainMenu"/>

    <synonym text="GC"/>

    <add-to-group
        group-id="ToolsMenu"
        relative-to-action="GenerateJavadoc"
        anchor="after"/>

    <keyboard-shortcut
        keymap="$default"
        first-keystroke="control alt G"
        second-keystroke="C"/>

    <keyboard-shortcut
        keymap="Mac OS X"
        first-keystroke="control alt G"
        second-keystroke="C"
        remove="true"/>

    <keyboard-shortcut
        keymap="Mac OS X 10.5+"
        first-keystroke="control alt G"
        second-keystroke="C"
        replace-all="true"/>

    <mouse-shortcut
        keymap="$default"
        keystroke="control button3 doubleClick"/>
  </action>

  <action
      id="sdk.action.PopupDialogAction"
      class="sdk.action.PopupDialogAction"
      icon="SdkIcons.Sdk_default_icon"/>

  <group
      class="com.example.impl.MyActionGroup"
      id="TestActionGroup"
      text="Test Group"
      description="Group with test actions"
      icon="icons/testGroup.png"
      popup="true"
      compact="true">

    <action
        id="VssIntegration.TestAction"
        class="com.example.impl.TestAction"
        text="My Test Action"
        description="My test action"/>

    <separator/>

    <group id="TestActionSubGroup"/>

    <reference ref="EditorCopy"/>

    <add-to-group
        group-id="MainMenu"
        relative-to-action="HelpMenu"
        anchor="before"/>
  </group>
</actions>
```

#### 通过代码注册动作

要通过代码注册动作，需要两个步骤：

1. 将继承自 `AnAction` 的类的实例传递给 `ActionManager` 的 `registerAction()` 方法，以将动作与 ID 关联。
2. 动作需要添加到一个或多个组中。要通过 ID 获取动作组的实例，需要调用 `ActionManager.getAction()` 并将返回的值转换为 `DefaultActionGroup`。

### 从动作构建 UI

如果插件需要在其用户界面中包含由一组动作构建的工具栏或弹出菜单，可以通过调用 `ActionManager.createActionPopupMenu()` 和 `createActionToolbar()` 方法来创建这些对象。要从这些对象中获取 Swing 组件，请调用相应的 `getComponent()` 方法。

如果将动作工具栏附加到特定组件（例如工具窗口中的面板），请调用 `ActionToolbar.setTargetComponent()` 并传递相关组件的实例作为参数。设置目标可确保工具栏按钮的状态取决于相关组件的状态，而不是 IDE 框架中的当前焦点位置。

### 有用的动作基类

- **ToggleAction**: 用于具有“选中”/“按下”状态的动作（例如，带复选框的菜单项、工具栏动作按钮）。
- **BackAction** 和 **ForwardAction**: 提供历史记录导航。
- **EmptyAction**: 用于保留动作 ID，使其在设置中可见。

### 程序化执行动作

有时需要以程序化方式执行动作，例如执行实现了我们需要的逻辑的动作，而这个实现我们无法控制。可以通过 `ActionUtils.invokeAction()` 实现动作的程序化执行。

应尽可能避免程序化执行动作。如果程序化执行的动作在您的控制之下，请将其逻辑提取到服务或实用程序类中并直接调用它。



### 创建自定义动作

在 IntelliJ 平台中，插件可以通过自定义动作（actions）来扩展现有的 IDE 菜单和工具栏，或者添加新的菜单和工具栏。插件的动作会响应用户与 IDE 的交互，但是这些动作必须首先被定义并注册到 IntelliJ 平台中。

以下是如何创建一个自定义动作的步骤，使用 SDK 示例代码 `action_basics` 作为参考。

#### 创建自定义动作

自定义动作是通过扩展抽象类 `AnAction` 来实现的。扩展 `AnAction` 的类应该重写 `AnAction.update()` 方法，且必须重写 `AnAction.actionPerformed()` 方法。

- `update()` 方法实现用于启用或禁用动作的代码。
- `actionPerformed()` 方法实现当用户调用该动作时执行的代码。

从 2022.3 版本开始，必须实现 `AnAction.getActionUpdateThread()` 方法。

以下是 `PopupDialogAction` 的示例，它扩展了 `AnAction`，这是 `action_basics` 示例中的代码：

```java
public class PopupDialogAction extends AnAction {

  @Override
  public void update(@NotNull AnActionEvent event) {
    // 使用事件，评估上下文，
    // 并启用或禁用动作。
  }

  @Override
  public void actionPerformed(@NotNull AnActionEvent event) {
    // 使用事件，实现一个动作。
    // 例如，创建并显示一个对话框。
  }

  // 当目标为 2022.3 或更高版本时，重写 getActionUpdateThread() 方法！

}
```

`AnAction` 类不应该有任何类字段。这种限制是为了防止内存泄漏。有关详细原因，请参阅动作实现。

在这一步中，`update()` 方法默认始终启用该动作。`actionPerformed()` 方法的实现目前为空。这些方法将在下面的步骤中进行完整实现。

在实现这些方法之前，我们需要将 `PopupDialogAction` 注册到 IntelliJ 平台中。

#### 注册自定义动作

动作可以通过代码或在插件配置文件的 `<actions>` 部分中声明注册。此部分描述了使用 IDE 工具（“新动作”表单）在 `plugin.xml` 文件中添加声明，然后手动调整注册属性。有关动作注册的更全面解释，请参阅本指南中的注册动作部分。

##### 使用新动作表单注册动作

IntelliJ IDEA 中有一个嵌入的检查器，可以识别未注册的动作。验证该检查器是否已启用，路径为 `Settings | Editor | Inspections | Plugin DevKit | Code | Component/Action not registered`。下面是 `PopupDialogAction` 类的示例：

“动作从未使用”检查器

要注册 `PopupDialogAction` 并设置其基本属性，请按 `Alt+Shift+Enter`。填写新动作表单以设置 `PopupDialogAction` 的参数：

**新动作表单**：

表单中的字段有：

- **Action ID**：每个动作必须有唯一的 ID。如果动作类仅在 IDE UI 的一个地方使用，则类的全限定名（FQN）是 ID 的一个好的默认值。如果动作类在多个地方使用，则需要为每个 ID 添加后缀。
- **Class Name**：动作的 FQN 实现类。如果同一个动作在多个地方使用，则实现类的 FQN 可以与不同的 Action ID 一起重复使用。
- **Name**：菜单中显示的文本。
- **Description**：显示的提示文本。
- **Add to Group**：动作添加到的动作组（菜单或工具栏）。点击列表中的组并输入会触发搜索，如“ToolsMenu”。
- **Anchor**：菜单动作在工具菜单中相对于其他动作的位置。

在这种情况下，`PopupDialogAction` 将出现在工具菜单中，它会被放置在顶部，并且没有快捷键。

填写完新动作表单并应用更改后，插件的 `plugin.xml` 文件中的 `<actions>` 部分将包含：

```xml
<actions>
  <action
      id="org.intellij.sdk.action.PopupDialogAction"
      class="org.intellij.sdk.action.PopupDialogAction"
      text="Popup Dialog Action"
      description="SDK action example">
    <add-to-group group-id="ToolsMenu" anchor="first"/>
  </action>
</actions>
```

`<action>` 元素声明了来自新动作表单的 Action ID (`id`)、Class Name (`class`)、Name (`text`) 和 Description。`<add-to-group>` 元素声明动作将出现在何处，并反映表单中的条目名称。

此声明是足够的，但在下一节中将讨论添加更多属性。

#### 手动设置注册属性

可以手动将动作声明添加到 `plugin.xml` 文件。有关声明元素和属性的详尽列表，请参阅 `plugin.xml` 中的注册动作。可以通过从新动作表单中选择或直接在 `plugin.xml` 文件中编辑注册声明来添加属性。

`PopupDialogAction` 的 `<action>` 声明在 `action_basics` 插件的 `plugin.xml` 文件中。它还包含图标的属性，并封装了文本覆盖、键盘和鼠标快捷方式以及动作应添加到的菜单组的声明元素。

完整声明如下：

```xml
<action
    id="org.intellij.sdk.action.PopupDialogAction"
    class="org.intellij.sdk.action.PopupDialogAction"
    text="Action Basics Plugin: Popup Dialog Action"
    description="SDK action example"
    icon="SdkIcons.Sdk_default_icon">
  <override-text place="MainMenu" text="Popup Dialog Action"/>
  <keyboard-shortcut
      keymap="$default"
      first-keystroke="control alt A"
      second-keystroke="C"/>
  <mouse-shortcut
      keymap="$default"
      keystroke="control button3 doubleClick"/>
  <add-to-group group-id="ToolsMenu" anchor="first"/>
</action>
```

### 测试自定义动作

完成上述步骤后，编译并运行插件，即可在工具菜单中看到新创建的动作条目：

#### 扩展 `actionPerformed()` 方法

在 `PopupDialogAction.actionPerformed()` 方法中添加代码，使该动作执行一些有用的功能。以下代码示例通过从 `AnActionEvent` 输入参数中获取信息并构造一个消息对话框来演示。对话框显示一个通用图标、从调用菜单动作中获取的消息和标题属性。然而，这个方法中的代码还可以操作项目、调用检查、更改文件内容等。

```java
@Override
public void actionPerformed(@NotNull AnActionEvent event) {
  // 使用事件，创建并显示一个对话框
  Project currentProject = event.getProject();
  StringBuilder message =
      new StringBuilder(event.getPresentation().getText() + " Selected!");
  // 如果编辑器中选择了一个元素，添加关于它的信息。
  Navigatable selectedElement = event.getData(CommonDataKeys.NAVIGATABLE);
  if (selectedElement != null) {
    message.append("\nSelected Element: ").append(selectedElement);
  }
  String title = event.getPresentation().getDescription();
  Messages.showMessageDialog(
      currentProject,
      message.toString(),
      title,
      Messages.getInformationIcon());
}
```

#### 扩展 `update()` 方法

在 `PopupDialogAction.update()` 中添加代码可以更细粒度地控制动作的可见性和可用性。可以根据上下文动态更改动作的状态和(或)显示。

此方法需要执行非常快。有关此限制的更多信息，请参阅覆盖 `AnAction.update()` 方法中的警告。

例如，`update()` 方法依赖于 `Project` 对象的可用性。这意味着用户必须在 IDE 中至少打开一个项目，`PopupDialogAction` 才能可用。因此，`update()` 方法会在 `Project` 对象未定义的上下文中禁用该动作。

通过在 `Presentation` 对象上设置启用和可见性状态来确定动作的可用性。设置动作的启用和可见性会产生一致的行为，尽管可能存在主菜单设置的不同，如在“动作分组”中讨论的那样。

```java
@Override
public void update(AnActionEvent e) {
  // 根据项目是否打开来设置可用性
  Project project = e.getProject();
  e.getPresentation().setEnabledAndVisible(project != null);
}
```

`update()` 方法不会在启用 `PopupDialogAction` 之前检查是否有 `Navigatable` 对象的可用性。这种检查是没有必要的，因为在 `actionPerformed()` 中使用 `Navigatable` 对象是机会性的。有关如何从 `AnActionEvent` 输入参数中获取信息的更多信息，请参阅确定动作上下文。

`AnAction.getActionUpdateThread()` 方法是在 IntelliJ 平台 2022.3 及更高版本中引入的，用于指定 `update()` 方法在哪个线程上运行。这个方法的返回值类型是 `ActionUpdateThread`，它定义了 `update()` 方法是在线程BGT（Background Thread，后台线程）还是EDT（Event Dispatch Thread，事件分发线程）上运行。

### getActionUpdateThread

`getActionUpdateThread()` 方法决定了动作的更新逻辑是否应该在后台线程执行还是在EDT线程执行。选择合适的线程对于确保动作的性能和响应速度非常重要，特别是当动作需要访问不同类型的数据或执行复杂的计算时。

### 方法定义

```java
@NotNull
public ActionUpdateThread getActionUpdateThread() {
    return ActionUpdateThread.BGT; // 或 ActionUpdateThread.EDT
}
```

### 返回值选项

- **ActionUpdateThread.BGT**: 如果动作的更新逻辑需要访问大型数据集、进行文件系统操作、访问PSI（Program Structure Interface）或虚拟文件系统（VFS）等，这种情况下应选择BGT。BGT可以保证应用程序范围内对这些数据结构的只读访问，从而避免对UI响应时间的影响。然而，在BGT上运行的更新逻辑不能直接访问Swing组件或UI模型。

- **ActionUpdateThread.EDT**: 如果动作的更新逻辑需要直接与Swing组件或UI模型交互，应选择EDT。EDT用于处理所有的UI事件和更新，因此在这个线程上进行的操作必须非常快，以免影响UI的响应速度。在EDT上执行的代码不应访问大型数据集或执行阻塞操作。

### 具体应用

选择 `ActionUpdateThread.BGT` 还是 `ActionUpdateThread.EDT` 取决于 `update()` 方法中的操作类型：

- **使用BGT**:
  - 访问PSI元素（例如Java代码结构）来检查文件状态。
  - 读取虚拟文件系统（VFS）中的文件和文件夹信息。
  - 任何其他可能导致UI阻塞的繁重操作。

- **使用EDT**:
  - 更新或查询Swing组件的状态。
  - 直接与用户界面元素交互。
  - 轻量级的检查和逻辑判断，例如简单的布尔判断。

### 在 `AnActionEvent` 中使用 `UpdateSession`

当确实需要从BGT切换到EDT（或反之），可以使用 `AnActionEvent.getUpdateSession()` 来访问 `UpdateSession`，然后调用 `UpdateSession.compute()` 在另一个线程上执行某个函数。这样可以确保在不同线程之间安全地执行相关操作。

### 重要性

- **性能优化**: 正确地选择 `getActionUpdateThread()` 可以避免UI线程的阻塞，提升IDE的响应速度。
- **线程安全**: 在合适的线程上执行更新逻辑可以避免数据竞争和不一致问题。

总结，`AnAction.getActionUpdateThread()` 方法是IntelliJ平台提供的一个重要工具，用于指定动作更新逻辑的执行线程，从而帮助开发者优化插件性能并确保线程安全。



### 创建动作组

在IntelliJ平台中，可以通过创建动作组（Action Groups）来组织多个动作（Actions），这在动作过多或需要将多个相关动作分组以简化菜单时特别有用。本教程将演示如何将动作添加到现有组，创建新的动作组，以及带有可变动作集的动作组。本文使用的示例代码来自`action_basics`代码示例。

### 简单动作组

在第一个示例中，动作组将作为顶级菜单项出现，动作以下拉菜单项的形式呈现。这个组基于IntelliJ平台的默认实现。

#### 创建简单组

可以通过在插件的`plugin.xml`文件的`<actions>`部分中添加`<group>`元素来注册分组。以下示例中`<group>`元素中没有类属性，因为IntelliJ平台框架将提供组的默认实现类。这个默认实现用于组中的动作集在运行时不变化的情况，即大多数情况下。`id`属性必须是唯一的，因此最好采用插件ID或包名作为命名习惯。

`popup`属性决定动作组中的动作是否放在子菜单中。`icon`属性指定要显示的图标的完全限定名（FQN）。由于未指定`compact`属性，因此此组将支持子菜单。有关这些属性的更多信息，请参阅[Registering Actions in plugin.xml](https://plugins.jetbrains.com/docs/intellij/registering-actions.html)。

```xml
<group
    id="org.intellij.sdk.action.GroupedActions"
    text="Static Grouped Actions"
    popup="true"
    icon="SdkIcons.Sdk_default_icon"/>
```

#### 将动作组绑定到UI组件

以下示例展示了如何使用`<add-to-group>`元素将自定义动作组相对于`Tools`菜单中的某个条目放置。`relative-to-action`属性引用了`PopupDialogAction`的动作ID，这个动作定义在同一个`plugin.xml`文件中。此组放置在动作`PopupDialogAction`的条目之后，如[Creating Actions](https://plugins.jetbrains.com/docs/intellij/creating-actions.html)教程中所述。

```xml
<group
    id="org.intellij.sdk.action.GroupedActions"
    text="Static Grouped Actions"
    popup="true"
    icon="SdkIcons.Sdk_default_icon">
  <add-to-group
      group-id="ToolsMenu"
      anchor="after"
      relative-to-action="org.intellij.sdk.action.PopupDialogAction"/>
</group>
```

#### 向静态分组动作添加新动作

`PopupDialogAction`的实现将被重用并注册到新创建的静态组中。为重用的`PopupDialogAction`实现设置的`id`属性为`org.intellij.sdk.action.GroupPopDialogAction`。这个值将此`<action>`条目与先前在[Creating Actions](https://plugins.jetbrains.com/docs/intellij/creating-actions.html)教程中使用的ID区分开来。一个唯一的ID支持在多个菜单或组中重用动作类。此组中的动作将在菜单中显示为“A Group Action”。

```xml
<group
    id="org.intellij.sdk.action.GroupedActions"
    text="Static Grouped Actions"
    popup="true"
    icon="SdkIcons.Sdk_default_icon">
  <add-to-group
      group-id="ToolsMenu"
      anchor="after"
      relative-to-action="org.intellij.sdk.action.PopupDialogAction"/>
  <action
      class="org.intellij.sdk.action.PopupDialogAction"
      id="org.intellij.sdk.action.GroupPopDialogAction"
      text="A Group Action"
      description="SDK static grouped action example"
      icon="SdkIcons.Sdk_default_icon">
  </action>
</group>
```

在完成上述步骤后，动作组及其内容将显示在`Tools`菜单中。底层的`PopupDialogAction`实现被重用，并在`Tools`菜单中以两种方式显示：

1. 作为顶级菜单项`Tools | Pop Dialog Action`，其动作ID为`org.intellij.sdk.action.PopupDialogAction`，如[Creating Actions](https://plugins.jetbrains.com/docs/intellij/creating-actions.html)教程中所述。
2. 作为菜单项`Tools | Static Grouped Actions | A Group Action`，其动作ID为`org.intellij.sdk.action.GroupPopDialogAction`。

### 自定义动作组类的实现

在某些情况下，动作组的具体行为需要依赖于上下文。解决方案类似于使单个动作条目依赖于上下文。

#### 扩展DefaultActionGroup

`DefaultActionGroup`是`ActionGroup`的一个实现类。`DefaultActionGroup`类用于将子动作和它们之间的分隔符添加到组中。如果组中的动作集在运行时不变化，则使用此类。

例如，可以在`action_basics`代码示例中扩展`DefaultActionGroup`来创建`CustomDefaultActionGroup`类：

```java
public class CustomDefaultActionGroup extends DefaultActionGroup {
  @Override
  public void update(AnActionEvent event) {
    // 根据用户是否在编辑启用/禁用动作...
  }
}
```

#### 注册自定义动作组

与静态动作组的情况一样，动作组应该在`plugin.xml`文件的`<actions>`部分中声明。例如，`action_basics`插件。为了演示，此实现将使用本地化。

以下`<group>`元素声明展示了：

- 在`<actions>`部分外部的可选资源包声明，用于本地化动作。
- `<group>`元素中的类属性告诉IntelliJ平台框架使用`CustomDefaultActionGroup`而不是默认实现。
- 将组的`popup`属性设置为允许子菜单。
- 在`<group>`声明中省略了`text`和`description`属性，改为使用本地化资源包定义它们。
- 组没有图标属性；`CustomDefaultActionGroup`的实现将在动作上下文中添加图标。
- `<add-to-group>`元素指定将组添加到现有的`EditorPopupMenu`的第一个位置。

```xml
<resource-bundle>messages.BasicActionsBundle</resource-bundle>

<actions>
  <group
      id="org.intellij.sdk.action.CustomDefaultActionGroup"
      class="org.intellij.sdk.action.CustomDefaultActionGroup"
      popup="true">
    <add-to-group group-id="EditorPopupMenu" anchor="first"/>
  </group>
</actions>
```

#### 将动作添加到自定义组

与静态分组动作中的情况一样，可以在`<group>`元素中添加`PopupDialogAction`动作。以下是`<action>`元素的声明：

- `<action>`元素中的类属性具有相同的FQN以重用此动作实现。
- `id`属性是唯一的，以区别于动作系统中实现的其他用途。
- 在`<action>`声明中省略了`text`和`description`属性；它们改为使用本地化资源包定义。
- 为此动作声明了SDK图标。

```xml
<group
    id="org.intellij.sdk.action.CustomDefaultActionGroup"
    class="org.intellij.sdk.action.CustomDefaultActionGroup"
    popup="true"
    icon="SdkIcons.Sdk_default_icon">
  <add-to-group group-id="EditorPopupMenu" anchor="first"/>
  <action
      id="org.intellij.sdk.action.CustomGroupedAction"
      class="org.intellij.sdk.action.PopupDialogAction"
      icon="SdkIcons.Sdk_default_icon"/>
</group>
```

现在，必须在资源包`BasicActionsBundle.properties`文件中根据[Localizing Actions and Groups](https://plugins.jetbrains.com/docs/intellij/localizing-actions.html)提供`text`和`description`属性的翻译。注意，有两组`text`和`description`翻译，一组用于动作，一组用于组。如果使用了`<override-text>`属性，则可能还有另一组翻译。

```properties
action.org.intellij.sdk.action.CustomGroupedAction.text=A Popup Action [EN]
action.org.intellij.sdk.action.CustomGroupedAction.description=SDK popup grouped action example [EN]
group.org.intellij.sdk.action.CustomDefaultActionGroup.text=Popup Grouped Actions [EN]
group.org.intellij.sdk.action.CustomDefaultActionGroup.description=Custom defaultActionGroup demo [EN]
```

#### 为自定义组提供特定行为

覆盖`CustomDefaultActionGroup.update()`方法，使得该组只有在有可用的编辑器实例时才可见。此外，还添加了一个自定义图标，以演示组图标可以根据动作上下文进行更改：

```java
public class CustomDefaultActionGroup extends DefaultActionGroup {
  @Override
  public void update(AnActionEvent event) {
    // 根据用户是否在编辑启用/禁用动作
    Editor editor = event.getData(CommonDataKeys.EDITOR);
    event.getPresentation().setEnabled(editor != null);
    // 此时可以设置组的图标。
    event.getPresentation().setIcon(SdkIcons.Sdk_default_icon);
  }
}
```

编译并运行上述代码示例后，打开编辑器中的文件并右键单击，编辑器上下文菜单将弹出，包含新动作组在第一个位置。注意，组和动作来自资源文件，所有内容都有后缀“ [EN]”。新组还将有一个图标。

### 具有动态动作集的动作组

如果自定义组的动作集取决于上下

文而有所变化，则该组必须扩展`ActionGroup`。此时，动作组中的动作集是动态定义的。

#### 创建可变动作组

要创建具有可变数量动作的组，扩展`ActionGroup`。例如，在`action_basics`类`DynamicActionGroup`代码中：

```java
public class DynamicActionGroup extends ActionGroup {
}
```

#### 注册可变动作组

要注册动态菜单组，需要在`plugin.xml`的`<actions>`部分中放置`<group>`属性。启用后，此组显示在`Tools`菜单中`Static Grouped Actions`下方的位置：

```xml
<group
    id="org.intellij.sdk.action.DynamicActionGroup"
    class="org.intellij.sdk.action.DynamicActionGroup"
    popup="true"
    text="Dynamically Grouped Actions"
    description="SDK dynamically grouped action example"
    icon="SdkIcons.Sdk_default_icon">
  <add-to-group
      group-id="ToolsMenu"
      anchor="after"
      relative-to-action="org.intellij.sdk.action.GroupedActions"/>
</group>
```

如果`<group>`元素的类属性命名的类派生自`ActionGroup`，则该组中的任何静态`<action>`声明都会抛出异常。对于静态定义的组，请使用`DefaultActionGroup`。

#### 向动态组添加子动作

要向`DynamicActionGroup`添加动作，应从`DynamicActionGroup.getChildren()`方法返回一个非空的`AnAction`实例数组。这里再次重用`PopupDialogAction`的实现。这个用例就是`PopupDialogAction`覆盖构造函数的原因：

```java
public class DynamicActionGroup extends ActionGroup {
  @NotNull
  @Override
  public AnAction[] getChildren(AnActionEvent event) {
    return new AnAction[]{
        new PopupDialogAction(
            "Action Added at Runtime",
            "Dynamic Action Demo",
            SdkIcons.Sdk_default_icon)
    };
  }
}
```

在提供了`DynamicActionGroup`的实现并使其返回非空的动作数组后，`Tools`菜单的第三个位置将包含一个新的动作组。



## 持久化模型（Persistence Model）

IntelliJ平台的持久化模型用于存储各种信息。例如，运行配置和设置就是使用持久化模型存储的。

根据持久化的数据类型，有两种不同的方法：

1. **持久化组件状态（Persisting State of Components）**
2. **持久化敏感数据（Persisting Sensitive Data）**

#### 1. 持久化组件状态（Persisting State of Components）

在IntelliJ平台中，持久化组件状态是通过`PersistentStateComponent`接口来实现的。该接口允许插件开发者将组件的状态序列化为XML格式，并在需要时恢复这些状态。这种方法通常用于保存用户设置、UI状态等。

##### 实现步骤：
1. **定义状态类**：定义一个POJO类来保存需要持久化的数据。这些类通常使用公共字段，或者使用getter和setter方法。

2. **实现PersistentStateComponent接口**：创建一个类实现`PersistentStateComponent<T>`接口，其中`T`是状态类。

3. **注册组件**：在`plugin.xml`中使用`<applicationService>`或`<projectService>`元素注册持久化组件。

**示例：**

```java
public class MySettingsState {
    public String someSetting = "default";
    // Other fields...
}

public class MySettings implements PersistentStateComponent<MySettingsState> {
    private MySettingsState mySettingsState = new MySettingsState();

    @Nullable
    @Override
    public MySettingsState getState() {
        return mySettingsState;
    }

    @Override
    public void loadState(@NotNull MySettingsState state) {
        XmlSerializerUtil.copyBean(state, mySettingsState);
    }
}
```

**注册组件：**
```xml
<projectService serviceImplementation="com.example.MySettings"/>
```

#### 2. 持久化敏感数据（Persisting Sensitive Data）

对于敏感数据（如密码、API密钥等），直接以明文形式存储是不安全的。IntelliJ平台提供了安全存储敏感数据的方法，即通过使用`PasswordSafe`和`CredentialAttributes`类。

##### 使用方法：
1. **定义凭证属性**：创建一个`CredentialAttributes`对象，用于标识存储的凭证（通常使用`serviceName`和`userName`作为标识符）。

2. **存储和检索凭证**：使用`PasswordSafe`类的`setPassword`和`getPassword`方法来存储和检索凭证。

**示例：**

```java
CredentialAttributes attributes = new CredentialAttributes("com.example.myService", "myUsername");
PasswordSafe.getInstance().setPassword(attributes, "mySecretPassword");

// Retrieve the password
String password = PasswordSafe.getInstance().getPassword(attributes);
```

#### 总结

持久化模型在IntelliJ平台中广泛应用于保存用户的设置和状态。通过`PersistentStateComponent`接口可以方便地保存和恢复组件的状态，而对于敏感数据，使用`PasswordSafe`来保证数据的安全性。使用这些持久化技术，可以确保插件在重启IDE或跨设备使用时能正确地保留用户的设置和状态。



## 组件状态持久化

IntelliJ 平台提供了一种 API，允许组件或服务在 IDE 重启之间持久化其状态。该 API 允许持久化简单的键值对条目以及复杂的状态类。

对于持久化敏感数据（如密码），请参见“敏感数据持久化”。

### 使用 PersistentStateComponent

`PersistentStateComponent` 接口允许持久化状态类，并提供最大程度的灵活性来定义要持久化的值、它们的格式和存储位置。

要使用它：

1. 标记一个服务（分别用于存储项目或应用数据的项目级或应用级服务）为实现 `PersistentStateComponent` 接口。
2. 定义状态类。
3. 使用 `@State` 指定存储位置。

请注意，扩展点的实例不能通过实现 `PersistentStateComponent` 来持久化它们的状态。如果扩展点需要持久化状态，请定义一个单独的服务来管理该状态。

### 实现 PersistentStateComponent 接口

`PersistentStateComponent` 的实现必须使用状态类类型参数化。状态类可以是一个单独的类，也可以是实现 `PersistentStateComponent` 的类。

#### 使用独立状态类的持久化组件

在这种情况下，状态类实例通常存储为 `PersistentStateComponent` 类中的一个字段。当状态从存储中加载时，它会被分配给状态字段（见 `loadState()`）：

```java
@Service
@State(...)
class MySettings implements PersistentStateComponent<MySettings.State> {

  static class State {
    public String value;
  }

  private State myState = new State();

  @Override
  public State getState() {
    return myState;
  }

  @Override
  public void loadState(State state) {
    myState = state;
  }
}
```

使用独立的状态类是推荐的方法。

#### 持久化组件作为状态类

在这种情况下，`getState()` 返回组件本身，而 `loadState()` 将从存储加载的状态的属性复制到组件实例：

```java
@Service
@State(...)
class MySettings implements PersistentStateComponent<MySettings> {

  public String stateValue;

  @Override
  public MySettings getState() {
    return this;
  }

  @Override
  public void loadState(MySettings state) {
    XmlSerializerUtil.copyBean(state, this);
  }
}
```

### 实现状态类

`PersistentStateComponent` 的实现通过将公共字段、注释的私有字段（参见自定义持久化值的 XML 格式）和 bean 属性序列化为 XML 格式来工作。

要排除某个公共字段或 bean 属性不被序列化，可以使用 `@Transient` 注解该字段或 getter。

请注意，状态类必须有一个默认构造函数。它应该返回组件的默认状态：即 XML 文件中尚未持久化的状态。

状态类应该有一个 `equals()` 方法，但如果未实现，则按字段比较状态对象。

可以持久化的值类型包括：

- 数字（基本类型，如 `int` 和包装类型，如 `Integer`）
- 布尔值
- 字符串
- 集合
- 映射
- 枚举

对于其他类型，可以扩展 `Converter`。请参阅下面的示例。

### 定义存储位置

要指定持久化值的确切存储位置，请将 `@State` 注解添加到 `PersistentStateComponent` 类。

它有以下字段：

- `name`（必需）：指定状态的名称（XML 中根标签的名称）。
- `storages`：一个或多个 `@Storage` 注解，用于指定存储位置。对于项目级值，这个是可选的——标准项目文件将被使用。
- `reloadable`（可选）：如果设置为 `false`，当 XML 文件被外部更改且状态已更改时，需要完全重新加载项目（或应用程序）。

指定 `@Storage` 注解的最简单方法如下：

- `@Storage(StoragePathMacros.WORKSPACE_FILE)`：对于存储在项目工作空间文件中的值（仅项目级组件）。
- `@Storage("yourName.xml")`：如果组件是项目级的，对于 .ipr 文件的项目，自动使用标准项目文件，无需指定任何内容。

对于应用级存储，强烈建议使用自定义文件。使用 `other.xml` 已被弃用。

在规划存储位置时，请考虑其预期用途。项目级自定义文件应优先用于存储插件设置。要存储缓存的值，请使用 `@Storage(StoragePathMacros.CACHE_FILE)`。参见 `StoragePathMacros` 以了解常用的宏。

`@Storage` 注解的 `roamingType` 参数指定了设置共享时的漫游类型：

- `RoamingType.DEFAULT`：设置被共享
- `RoamingType.PER_OS`：设置按操作系统共享
- `RoamingType.DISABLED`：设置共享被禁用

如果多个组件在同一个文件中存储状态，它们必须具有相同的 `roamingType` 属性值。

### 在 IDE 安装之间共享设置

可以在不同的 IDE 安装之间共享组件的持久化状态。这使得用户可以在每个开发机器上拥有相同的设置，或者在团队内共享他们的设置。

可以通过以下功能共享设置：

- **Settings Sync 插件**，允许在 JetBrains 服务器上同步设置。用户可以选择要同步的设置类别。
- **Settings Repository 插件**，允许在用户创建和配置的 Git 存储库中同步设置。
- **导出设置**功能，允许手动导入和导出设置。

只有在这些插件已安装并启用的情况下，才能通过 Settings Sync 或 Settings Repository 插件同步。

是否将特定组件的状态设为可共享的决定应谨慎做出。只有那些不特定于某台机器的设置应被共享，例如，不应共享用户特定目录的路径。如果一个组件包含可共享和不可共享的数据，则应将其拆分为两个独立的组件。

### Settings Sync 插件

要将插件的组件状态包含在 Settings Sync 插件的同步中，必须满足以下要求：

- 通过 `@Storage` 注解的 `roamingType` 属性定义 `RoamingType`，并且不等于 `DISABLED`。
- 通过 `@State` 注解的 `category` 属性定义 `SettingsCategory`，并且不等于 `OTHER`。
- 没有其他 `PersistentStateComponent` 存储在相同的 XML 文件中，并具有不同的 `RoamingType`。

如果组件状态与操作系统相关，则 `@Storage` 注解的 `roamingType` 必须设置为 `RoamingType.PER_OS`。

请注意，`other.xml` 文件不可漫游，并且在 `@Storage` 注解中声明它将禁用组件状态的漫游。建议为组件使用单独的 XML 文件，或者使用另一个现有的存储文件。

### Settings Repository 插件和导出设置功能

Settings Repository 插件从 2022.3 版开始不再捆绑，并且将不再维护。

持久化组件可以通过 Settings Repository 插件和导出设置功能进行共享，具体取决于 `@Storage` 注解的 `roamingType`。有关详细信息，请参见定义存储位置。

### 自定义持久化值的 XML 格式

如果需要使用默认的 bean 序列化但需要自定义 XML 中的存储格式（例如，为了与插件的先前版本或外部定义的 XML 格式兼容），请使用 `@Tag`、`@Attribute`、`@Property`、`@MapAnnotation`、`@XMap` 和 `@XCollection` 注解。

如果要序列化的状态无法映射到 JavaBean，则可以使用 `org.jdom.Element` 作为状态类。在这种情况下，使用 `getState()` 方法构建具有任意结构的 XML 元素，然后将其直接保存在状态 XML 文件中。在 `loadState()` 方法中，使用任何自定义逻辑反序列化 JDOM 元素树。这不推荐使用，除非万不得已。

要禁用存储值中的路径宏（PathMacro）的展开，请实现 `PathMacroFilter` 并在 `com.intellij.pathMacroFilter` 扩展点中注册。

### 迁移持久化值

如果基础持久化模型或存储格式已更改，则 `ConverterProvider` 可以提供 `ProjectConverter`，其 `getAdditionalAffectedFiles()` 方法返回要迁移的受影响文件并执行持久化值的程序化迁移。

### 持久化组件生命周期

`PersistentStateComponent.loadState()` 方法在组件创建后被调用（仅当组件的某个非默认状态已持久化时），以及 XML 文件被外部更改后（例如，如果项目文件从版本控制系统中更新）。在后一种情况下，组件负责根据更改的状态更新 UI 和其他相关组件。

`PersistentStateComponent.getState()` 方法在每次保存设置时被调用（例如，在窗口失去焦点或关闭 IDE 时）。如果从 `getState()` 返回的状态等于默认状态（通过默认构造函数创建状态类获得），则 XML 中不会持久化任何

内容。否则，返回的状态会被序列化为 XML 并存储起来。

### 使用 PropertiesComponent 进行简单的非漫游持久化

如果插件需要持久化一些简单的值，最简单的方法是使用 `PropertiesComponent` 服务。它可以在工作空间文件中保存应用级和项目级的值。对于 `PropertiesComponent`，漫游是禁用的，因此只能用于临时的、非漫游的属性。

使用 `PropertiesComponent.getInstance()` 方法存储应用级值，使用 `PropertiesComponent.getInstance(Project)` 方法存储项目级值。

由于所有插件共享相同的命名空间，强烈建议使用前缀键名（例如，使用插件 ID `com.example.myCustomSetting`）。

### 旧的 API（JDOMExternalizable）

旧的组件使用 `JDOMExternalizable` 接口来持久化状态。它使用 `readExternal()` 方法从 JDOM 元素读取状态，并使用 `writeExternal()` 将状态写入。

实现可以手动将状态存储在属性和子元素中，或者使用 `DefaultJDOMExternalizer` 类自动存储所有公共字段的值。

组件将其状态保存到以下文件：

- 项目级：项目 (.ipr) 文件。如果在 `plugin.xml` 文件中设置了 `workspace` 选项为 `true`，则使用工作空间 (.iws) 文件。
- 模块级：模块 (.iml) 文件。



## 持久化敏感数据

IntelliJ 平台提供了一个凭据存储 API，用于安全地存储敏感的用户数据，如密码、服务器 URL 等。

### 如何使用

要处理凭据，可以使用 `PasswordSafe`。

#### 常用的实用方法

以下是创建凭据属性的一个示例方法：

```java
private CredentialAttributes createCredentialAttributes(String key) {
  return new CredentialAttributes(
    CredentialAttributesKt.generateServiceName("MySystem", key)
  );
}
```

#### 检索已存储的凭据

要检索已存储的凭据，可以使用以下代码：

```java
String key = null; // 例如 serverURL, accountID
CredentialAttributes attributes = createCredentialAttributes(key);
PasswordSafe passwordSafe = PasswordSafe.getInstance();

Credentials credentials = passwordSafe.get(attributes);
if (credentials != null) {
  String password = credentials.getPasswordAsString();
}

// 或者只获取密码
String password = passwordSafe.getPassword(attributes);
```

#### 存储凭据

要存储新的凭据：

```java
CredentialAttributes attributes = createCredentialAttributes(key);
Credentials credentials = new Credentials(username, password);
PasswordSafe.getInstance().set(attributes, credentials);
```

要移除已存储的凭据，可以将 `credentials` 参数设置为 `null`。

### 存储方式

默认的存储格式根据操作系统的不同而不同：

- **Windows**：使用 KeePass 格式的文件
- **macOS**：使用安全框架的钥匙串
- **Linux**：使用 libsecret 的 Secret Service API

用户可以在“设置” > “外观与行为” > “系统设置” > “密码”中覆盖默认行为。

这提供了一个安全、可靠的方式来存储和管理敏感信息，确保用户数据的安全性和隐私保护。



## 设置指南

最后修改时间：2024年7月31日

设置在IntelliJ平台基础上的IDE中持久存储控制行为和外观的状态。在本页面中，"设置"（Settings）一词在某些平台上与“偏好设置”（Preferences）同义。

插件可以创建和存储设置，以捕获其配置，并利用IntelliJ平台的持久化模型。这些自定义设置的用户界面（UI）可以添加到IDE的设置对话框中。

设置可以影响不同级别的范围。本文件描述了如何在项目级别和应用程序（或全局，IDE）级别添加自定义设置。

参见[设置教程](https://www.jetbrains.com/help/idea/settings-tutorial.html)获取创建简单自定义设置的分步说明。

参见[检查设置](https://www.jetbrains.com/help/idea/inspecting-settings.html)以了解如何在IDE实例中收集设置对话框的信息。

## 设置的扩展点
自定义设置实现通过plugin.xml文件中的扩展点（EP）声明，具体取决于设置的级别。许多属性在EP声明中是共享的。

应用程序和项目设置通常基于Configurable接口提供实现，因为它们没有运行时依赖项。有关更多信息，请参见[设置扩展点的实现](https://www.jetbrains.com/help/idea/settings-extension-points.html)。

出于性能原因，建议使用插件.xml描述符中的EP元素中的属性尽可能多地声明'设置'实现的信息。如果未声明，则必须加载组件以从实现中检索它，这会降低UI响应速度。

### 声明应用程序设置
应用程序级别的设置使用`com.intellij.applicationConfigurable`扩展点进行声明。

下面是一个`<applicationConfigurable>` EP声明的示例。声明表明设置是工具设置组的子项，实现的FQN是`com.example.ApplicationSettingsConfigurable`，唯一ID与实现的完全限定名称（FQN）相同，显示给用户的（非本地化）标题是"My Application Settings"。有关更多信息，请参见[设置声明属性](https://www.jetbrains.com/help/idea/settings-declaration-attributes.html)。

```xml
<extensions defaultExtensionNs="com.intellij">
  <applicationConfigurable
      parentId="tools"
      instance="com.example.ApplicationSettingsConfigurable"
      id="com.example.ApplicationSettingsConfigurable"
      displayName="My Application Settings"/>
</extensions>
```

### 声明项目设置
项目级别的设置使用`com.intellij.projectConfigurable`扩展点进行声明。

下面是一个`<projectConfigurable>` EP声明的示例。与上述应用程序设置示例类似，但包括额外的`nonDefaultProject`属性，指示这些设置不适用于默认项目。详见[设置声明属性](https://www.jetbrains.com/help/idea/settings-declaration-attributes.html)。

```xml
<extensions defaultExtensionNs="com.intellij">
  <projectConfigurable
      parentId="tools"
      instance="com.example.ProjectSettingsConfigurable"
      id="com.example.ProjectSettingsConfigurable"
      displayName="My Project Settings"
      nonDefaultProject="true"/>
</extensions>
```

## 设置声明属性
建议读者查阅Configurable的Javadoc注释，因为这些属性信息也适用于ConfigurableProvider以及Configurable类。本节对这些注释提供了一些补充说明。

### 属性表
下表列出了`com.intellij.applicationConfigurable`和`com.intellij.projectConfigurable`扩展点支持的属性：

| 属性                | 必须的 | 值                                                           | 实现依据                              |
| ------------------- | ------ | ------------------------------------------------------------ | ------------------------------------- |
| `instance`          | 是 (1) | 实现的FQN。详见[Configurable接口](https://www.jetbrains.com/help/idea/configurable-interface.html)。 | `Configurable`                        |
| `provider`          | 是 (1) | 实现的FQN。详见[ConfigurableProvider类](https://www.jetbrains.com/help/idea/configurable-provider.html)。 | `ConfigurableProvider`                |
| `nonDefaultProject` | 是     | 仅适用于`com.intellij.projectConfigurable`扩展点。 `true`表示显示所有项目的设置，`false`表示仅显示默认项目的设置。 | `Configurable`                        |
| `displayName`       | 是 (2) | 显示给用户的非本地化设置名称，需要在设置对话框左侧菜单中显示。要使用本地化的可见名称，省略`displayName`并使用`key`和`bundle`属性。 | `Configurable` `ConfigurableProvider` |
| `key`和`bundle`     | 是 (2) | 可见的设置名称的本地化键和资源束。要使用非本地化的可见名称，省略`key`和`bundle`并使用`displayName`。 | `Configurable` `ConfigurableProvider` |
| `id`                | 是     | 该实现的唯一FQN标识符。FQN应基于插件ID以确保唯一性。         | `Configurable` `ConfigurableProvider` |
| `parentId`          | 是     | 用于创建设置层次结构的属性。指定组件作为指定`parentId`组件的子组件，通常用于将设置面板放置在设置对话框菜单中。可接受的`parentId`值见下文。 | `Configurable` `ConfigurableProvider` |
| `groupWeight`       | 否     | 指定组件在父级可配置组件中的分组顺序，默认权重为0，表示最低顺序。如果组中的一个子项或父组件的权重非零，则所有子项按权重降序排列。如果权重相等，则组件按显示名称升序排列。 | `Configurable` `ConfigurableProvider` |
| `dynamic`           | 否     | 该组件的子组件是通过调用`getConfigurables()`方法动态计算的。 不推荐使用，因为它需要在构建设置树时加载其他类。若可能，使用XML属性代替。 | `Configurable.Composite`              |
| `childrenEPName`    | 否     | 指定用于计算该组件的子组件的扩展点的FQN名称。                | `Configurable`                        |

### 属性说明
1. `instance`或`provider`之一必须根据实现进行指定。
2. `displayName`或`key`和`bundle`之一必须根据显示的设置名称是否本地化进行指定。

## `parentId`属性的值
下表列出了所有设置组及其相应的`parentId`属性的值。有关支持的所有属性，请参见上一节。

| 组别                         | `parentId`值  | 详情                                                         |
| ---------------------------- | ------------- | ------------------------------------------------------------ |
| 外观与行为                   | `appearance`  | 包含设置以个性化IDE外观，如更改主题和字体大小。还包括设置以自定义行为，如键映射、配置插件和系统设置等。 |
| 构建、执行、部署             | `build`       | 包含设置以配置项目与不同构建工具的集成，修改默认编译器设置，管理服务器访问配置，自定义调试器行为等。 |
| 构建集成                     | `build.tools` | `build`的子组。此子组配置项目与构建工具如Maven、Gradle或Gant的集成。 |
| 编辑器                       | `editor`      | 包含设置以个性化源代码外观，如字体、高亮样式、缩进等。还包括设置以自定义编辑器的外观，如行号、插入符位置、标签、源代码检查、设置模板和文件编码。 |
| 语言和框架                   | `language`    | 包含与项目中使用的特定语言框架和技术相关的设置。             |
| 第三方设置                   | `tools`       | 包含设置以配置与第三方应用程序的集成，指定SSH终端连接设置，管理服务器证书和任务，配置图表布局等。 |
| 超级父级                     | `root`        | 所有现有组的不可见父级。除了基于IntelliJ平台构建的IDEs或广泛的设置套件外，不使用该组。不应将设置放在此组中。 |
| 默认组（不推荐）             | `default`     | 如果未设置`parentId`或`groupId`属性，则组件会添加到默认设置组。这是不期望的；请参见默认组说明。 |
| 其他（不推荐使用）           | `other`       | IntelliJ平台不再使用此组。请勿使用此组。请使用`tools`组代替。 |
| 项目相关的设置（不推荐使用） | `project`     | IntelliJ平台不再使用此组。它曾被用来存储一些项目相关的设置。请勿使用此组。 |

## 设置扩展点的实现
`com.intellij.projectConfigurable`和`com.intellij.applicationConfigurable`扩展点的实现可以基于以下两种方式之一：

1. **Configurable接口**：提供一个带有Swing表单的可命名可配置组件。大多数设置提供者都基于Configurable接口或其子接口

。
2. **ConfigurableProvider类**：根据运行时条件隐藏设置对话框中的可配置组件。

### Configurable接口
intellij-community代码库中的许多设置实现了Configurable或其子类型，如SearchableConfigurable。建议读者查看Configurable的Javadoc注释。

#### 构造函数
实现必须满足多个构造函数的要求：

- 使用`applicationConfigurable` EP声明的应用程序设置实现必须具有无参数的默认构造函数。
- 使用`projectConfigurable` EP声明的项目设置实现必须声明一个带有`Project`类型的单个参数的构造函数。

从2020.2版本开始，不允许构造函数注入（除了`Project`）。

对于正确使用EP声明的Configurable实现，只有当用户在设置对话框菜单中选择相应的显示名称时，IntelliJ平台才会调用实现的构造函数。

#### IntelliJ平台与Configurable的交互
Configurable实现的实例化记录在接口文件中。这里简要回顾一些高层次的要点：

- `Configurable.reset()`方法在`Configurable.createComponent()`之后立即被调用。在构造函数或`createComponent()`中初始化设置值是不必要的。
- 一旦实例化，Configurable实例的生命周期将继续，无论实现的设置是否发生变化，或者用户是否在设置对话框菜单中选择了不同的条目。
- Configurable实例的生命周期在设置对话框中选择确定或取消时结束。当设置对话框关闭时，将调用实例的`Configurable.disposeUIResources()`。

### Configurable标记接口
基于Configurable的实现可以实现标记接口，这为实现提供了额外的灵活性。

- `Configurable.NoScroll`：不要向表单添加滚动条。默认情况下，插件的设置组件放置在可滚动窗格中。然而，设置面板可能有一个`JTree`，它需要自己的`JScrollPane`。因此，`NoScroll`接口应被用来删除外部`JScrollPane`。
- `Configurable.NoMargin`：不要向表单添加空边框。默认情况下，为插件的设置组件添加了空边框。
- `Configurable.Beta`（2022.3）：在设置树中的设置页面标题旁边添加Beta标签。

### 基于Configurable的其他接口
IntelliJ平台中有一些专门处理特定类型设置的类。这些子类型基于`com.intellij.openapi.options.ConfigurableEP`。例如，`Settings | Editor | General | Appearance`允许通过`EditorSmartKeysConfigurableEP`在`com.intellij.editorSmartKeysConfigurable`扩展点中添加设置。

### 示例
IntelliJ平台中可以作为参考的Configurable的现有实现包括：

- `ConsoleConfigurable`（应用程序配置）
- `AutoImportOptionsConfigurable`（项目配置）

### ConfigurableProvider类
`ConfigurableProvider`类只有在运行时条件满足时才提供`Configurable`实现。IntelliJ平台首先调用`ConfigurableProvider.canCreateConfigurable()`，该方法评估运行时条件以确定设置更改是否在当前上下文中有意义。如果设置有意义，`canCreateConfigurable()`返回true。在这种情况下，IntelliJ平台调用`ConfigurableProvider.createConfigurable()`，该方法返回其设置实现的`Configurable`实例。

通过选择在某些情况下不提供配置实现，`ConfigurableProvider`选择退出设置显示和修改过程。使用`ConfigurableProvider`作为设置实现的基础是通过EP声明中的属性进行声明的。

# 自定义设置组
最后修改时间：2024年7月31日

如[设置扩展点](https://www.jetbrains.com/help/idea/settings-extension-points.html)中所述，自定义设置可以声明为现有父组（如工具组）的子组。这些父组是基于IntelliJ平台的IDE中的现有设置类别。

但是，如果自定义设置足够丰富，需要多个层次呢？例如，一个自定义设置实现有多个子设置实现。扩展点声明可以创建这种多层设置层次结构。

参见[检查设置](https://www.jetbrains.com/help/idea/inspecting-settings.html)以了解如何在IDE实例中为设置对话框收集信息。

## 父子设置关系的扩展点
可以通过实现或扩展点声明的多种方式在设置组中创建父子关系。但是，在实现中创建这些关系会有性能损失，因为必须实例化对象来确定关系。本节描述了在`com.intellij.projectConfigurable`或`com.intellij.applicationConfigurable`扩展点中声明更复杂的父子关系的语法。

一个应用程序可配置项可以是项目可配置项的父项。

有两种方式使用`com.intellij.projectConfigurable`扩展点或`com.intellij.applicationConfigurable`扩展点来声明父子关系。第一种方法是使用由一个属性值联系在一起的单独的扩展点声明。第二种方法是使用嵌套声明。

### 使用单独的扩展点进行父子设置
声明父子关系的一种方法是使用两个单独的声明。无论父设置声明是否在同一插件中，都可以使用这种形式。如果知道父项的`id`属性，插件可以将设置添加为该父项的子项。

例如，下面是两个项目设置的声明。第一个被添加到工具组，第二个被添加到父项的`id`中。第二个子`<projectConfigurable>`添加了一个后缀（servers）到父项的`id`。

```xml
<extensions defaultExtensionNs="com.intellij">
  <projectConfigurable
      parentId="tools"
      id="com.intellij.sdk.tasks"
      displayName="Tasks"
      nonDefaultProject="true"
      instance="com.intellij.sdk.TaskConfigurable"/>

  <projectConfigurable
      parentId="com.intellij.sdk.tasks"
      id="com.intellij.sdk.tasks.servers"
      displayName="Servers"
      nonDefaultProject="true"
      instance="com.intellij.sdk.TaskRepositoriesConfigurable"/>
</extensions>
```

有关后缀`id`的详细信息，请参见[父子设置扩展点的属性](https://www.jetbrains.com/help/idea/settings-declaration-attributes.html#attributes)。

### 使用嵌套扩展点进行父子设置
另一种方式是使用`configurable`属性。这种方法在`com.intellij.projectConfigurable`或`com.intellij.applicationConfigurable`扩展点中嵌套子设置声明。

当使用`configurable`时，子项没有`parentId`，因为嵌套已经隐含了这一点。与使用单独的扩展点声明一样，子项的`id`属性也有格式限制——后缀（servers）被添加。有关详细信息，请参见[父子设置扩展点的属性](https://www.jetbrains.com/help/idea/settings-declaration-attributes.html#attributes)。

下面的例子展示了一个嵌套的`configurable`声明：

```xml
<extensions defaultExtensionNs="com.intellij">
  <projectConfigurable
        parentId="tools"
        id="com.intellij.sdk.tasks"
        displayName="Tasks"
        nonDefaultProject="true"
        instance="com.intellij.sdk.TaskConfigurable"/>
    <configurable
        id="com.intellij.sdk.tasks.servers"
        displayName="Servers"
        nonDefaultProject="true"
        instance="com.intellij.sdk.TaskRepositoriesConfigurable"/>
  </projectConfigurable>
</extensions>
```

在上面的父`<projectConfigurable>`扩展点声明中，可以添加更多的`<configurable>`声明作为兄弟设置。

## 父子设置扩展点的属性
在声明子设置扩展点时只有一个独特的属性。其他属性与[设置声明属性](https://www.jetbrains.com/help/idea/settings-declaration-attributes.html)中讨论的属性相同。

对于父项的子项，`id`属性变成复合格式：

| 属性 | 必须的 | 值                                                           |
| ---- | ------ | ------------------------------------------------------------ |
| `id` | 是     | 基于`com.intellij.openapi.options.Configurable`实现的复合FQN，格式为：`XX.YY`，其中：XX - 父设置组件的FQN基础ID，YY - 在其他兄弟项中唯一的后缀。 |

所有子项共享父项的`id`作为其自身`id`的基础。所有子项的`id`后缀在兄弟项中是唯一的。

## 父子设置的实现
实现可以基于`Configurable`、`ConfigurableProvider`或其子类型。有关创建设置实现的更多信息，请参见[设置扩展点的实现](https://www.jetbrains.com/help/idea/settings-extension-points.html)。

### Configurable标记接口
`Configurable.Composite`接口表示一个可配置组件具有子组件。首选的方法是在扩展点声明中指定子组件。使用Composite接口会带来加载子类时构建设置Swing组件树的性能损失。



## 自定义设置教程

最后修改时间：2024年7月31日

如[设置指南](https://www.jetbrains.com/help/idea/settings-guide.html)所讨论的，插件可以向基于IntelliJ平台的IDE添加设置。IDE会响应用户选择的设置来显示这些设置。自定义设置的显示和功能与IDE原生设置类似。

## 自定义设置实现概述
使用SDK代码示例`settings`，本教程说明了创建自定义应用程序级别设置的步骤。许多IntelliJ平台的设置实现使用较少的类，但为了清晰起见，这个示例将功能分为三个类：

1. **AppSettingsConfigurable** 类似于MVC模型中的控制器——它与其他两个设置类和IntelliJ平台交互，
2. **AppSettings** 类似于模型，因为它持久存储设置，
3. **AppSettingsComponent** 类似于视图，因为它显示和捕获对设置值的编辑。

项目设置的实现结构相同，但在`Configurable`实现和扩展点（EP）声明中有一些小的差异。

参见Kotlin中的设置示例[MarkdownSettings](https://www.jetbrains.com/help/idea/markdown-settings.html)和[MarkdownSettingsConfigurable](https://www.jetbrains.com/help/idea/markdown-settings-configurable.html)类。

### AppSettings 类
`AppSettings`类持久存储自定义设置。它基于IntelliJ平台的持久化模型。

#### 声明 AppSettings
给定没有使用轻量服务的情况下，持久数据类必须在`plugin.xml`文件中声明为服务EP。如果这是项目设置，则使用`com.intellij.projectService` EP。然而，由于这些是应用程序设置，因此使用`com.intellij.applicationService` EP并提供实现类的完全限定名（FQN）：

```xml
<extensions defaultExtensionNs="com.intellij">
  <applicationService
      serviceImplementation="org.intellij.sdk.settings.AppSettings"/>
</extensions>
```

#### 创建 AppSettings 实现
如[实现PersistentStateComponent接口](https://www.jetbrains.com/help/idea/persistentstatecomponent.html)中所讨论的，`AppSettings`使用实现`PersistentStateComponent`的模式，并使用单独的状态类进行参数化：

```java
@State(
    name = "org.intellij.sdk.settings.AppSettings",
    storages = @Storage("SdkSettingsPlugin.xml")
)
final class AppSettings implements PersistentStateComponent<AppSettings.State> {

  static class State {
    @NonNls
    public String userId = "John Smith";
    public boolean ideaStatus = false;
  }

  private State myState = new State();

  static AppSettings getInstance() {
    return ApplicationManager.getApplication().getService(AppSettings.class);
  }

  @Override
  public State getState() {
    return myState;
  }

  @Override
  public void loadState(@NotNull State state) {
    myState = state;
  }
}
```

#### @Storage 注解
`@State`注解位于类声明之上，用于定义数据存储位置。对于`AppSettings`，数据名称参数是类的FQN。使用FQN是最佳实践，并且如果自定义数据存储在标准项目或工作区文件中，则是必需的。

`storages`参数使用`@Storage`注解定义了`AppSettings`数据的自定义文件名。在这种情况下，文件位于IDE的配置目录的选项目录中。

#### 持久状态类
`AppSettings`实现包含一个内部状态类，具有两个公共字段：一个字符串和一个布尔值。从概念上讲，这些字段分别保存用户名和是否为IntelliJ IDEA用户。有关PersistentStateComponent如何序列化公共字段的更多信息，请参见[实现状态类](https://www.jetbrains.com/help/idea/persistentstatecomponent.html#state-class)。

#### AppSettings 方法
由于这个类的字段非常有限和简单，因此为了简化没有使用封装。所需的功能只需要覆盖两个方法：当加载新组件状态时调用（`PersistentStateComponent.loadState()`）和保存状态时调用（`PersistentStateComponent.getState()`）。有关这些方法的更多信息，请参见[PersistentStateComponent](https://www.jetbrains.com/help/idea/persistentstatecomponent.html)。

添加了一个静态方便方法——`AppSettings.getInstance()`——允许`AppSettingsConfigurable`轻松获取`AppSetting`的引用。

### AppSettingsComponent 类
`AppSettingsComponent`的角色是为IDE设置对话框提供一个`JPanel`来显示自定义设置。`AppSettingsComponent`具有一个`JPanel`，并负责其生命周期。`AppSettingsComponent`由`AppSettingsConfigurable`实例化。

#### 创建 AppSettingsComponent 实现
`AppSettingsComponent`定义了一个包含`JBTextField`和`JBCheckBox`的`JPanel`，用于保存和显示映射到`AppSettings.State`数据字段的数据：

```java
/**
 * Supports creating and managing a {@link JPanel} for the Settings Dialog.
 */
public class AppSettingsComponent {

  private final JPanel myMainPanel;
  private final JBTextField myUserNameText = new JBTextField();
  private final JBCheckBox myIdeaUserStatus = new JBCheckBox("IntelliJ IDEA user");

  public AppSettingsComponent() {
    myMainPanel = FormBuilder.createFormBuilder()
        .addLabeledComponent(new JBLabel("User name:"), myUserNameText, 1, false)
        .addComponent(myIdeaUserStatus, 1)
        .addComponentFillVertically(new JPanel(), 0)
        .getPanel();
  }

  public JPanel getPanel() {
    return myMainPanel;
  }

  public JComponent getPreferredFocusedComponent() {
    return myUserNameText;
  }

  @NotNull
  public String getUserNameText() {
    return myUserNameText.getText();
  }

  public void setUserNameText(@NotNull String newText) {
    myUserNameText.setText(newText);
  }

  public boolean getIdeaUserStatus() {
    return myIdeaUserStatus.isSelected();
  }

  public void setIdeaUserStatus(boolean newStatus) {
    myIdeaUserStatus.setSelected(newStatus);
  }
}
```

#### AppSettingsComponent 方法
构造函数使用方便的`FormBuilder`构建了`JPanel`并保存对`JPanel`的引用。该类的其余部分是封装`JPanel`中使用的UI组件的简单访问器和修改器。

### AppSettingsConfigurable 类
`AppSettingsConfigurable`的方法由IntelliJ平台调用，而`AppSettingsConfigurable`依次与`AppSettingsComponent`和`AppSettings`交互。

#### 声明 AppSettingsConfigurable
如[声明应用程序设置](https://www.jetbrains.com/help/idea/settings-declaration.html#application)中所述，`com.intellij.applicationConfigurable`用作EP。有关此声明的说明，请参见[声明应用程序设置](https://www.jetbrains.com/help/idea/settings-declaration.html#application)：

```xml
<extensions defaultExtensionNs="com.intellij">
  <applicationConfigurable
      parentId="tools"
      instance="org.intellij.sdk.settings.AppSettingsConfigurable"
      id="org.intellij.sdk.settings.AppSettingsConfigurable"
      displayName="SDK: Application Settings Example"/>
</extensions>
```

#### 创建 AppSettingsConfigurable 实现
`AppSettingsConfigurable`类实现了`Configurable`接口。该类有一个字段保存对`AppSettingsComponent`的引用。

```java
/**
 * Provides controller functionality for application settings.
 */
final class AppSettingsConfigurable implements Configurable {

  private AppSettingsComponent mySettingsComponent;

  // A default constructor with no arguments is required because
  // this implementation is registered as an applicationConfigurable

  @Nls(capitalization = Nls.Capitalization.Title)
  @Override
  public String getDisplayName() {
    return "SDK: Application Settings Example";
  }

  @Override
  public JComponent getPreferredFocusedComponent() {
    return mySettingsComponent.getPreferredFocusedComponent();
  }

  @Nullable
  @Override
  public JComponent createComponent() {
    mySettingsComponent = new AppSettingsComponent();
    return mySettingsComponent.getPanel();
  }

  @Override
  public boolean isModified() {
    AppSettings.State state =
        Objects.requireNonNull(AppSettings.getInstance().getState());
    return !mySettingsComponent.getUserNameText().equals(state.userId) ||
        mySettingsComponent.getIdeaUserStatus() != state.ideaStatus;
  }

  @Override
  public void apply() {
    AppSettings.State state =
        Objects.requireNonNull(AppSettings.getInstance().getState());
    state.userId = mySettingsComponent.getUserNameText();
    state.ideaStatus = mySettingsComponent.getIdeaUserStatus();
  }

  @Override
  public void reset() {
    AppSettings.State state =
        Objects.requireNonNull(AppSettings.getInstance().getState());
    mySettingsComponent.setUserNameText(state.userId);
    mySettingsComponent.setIdeaUserStatus(state.ideaStatus);
  }

  @Override
  public void disposeUIResources() {
    mySettingsComponent = null;
  }
}
```

#### AppSettingsConfigurable 方法
该类中的所有方法都是`Configurable`接口中的方法重写。读者被鼓励查看`Configurable`方法的Javadoc注释。还请查看[与Configurable的IntelliJ平台交互](



# 虚拟文件系统 (Virtual File System, VFS)

虚拟文件系统（VFS）是IntelliJ平台的一个组件，用于封装大部分与文件交互的活动，这些文件以虚拟文件的形式表示。VFS的主要目的是提供一个通用的API，用于处理文件，无论这些文件的实际位置是在磁盘、存档文件、HTTP服务器等。

## VFS的主要功能

1. **提供通用的文件操作API**：无论文件的实际位置在哪里，VFS提供了统一的接口来进行文件操作。

2. **跟踪文件修改**：VFS可以跟踪文件的修改，并在检测到更改时提供文件内容的旧版本和新版本。

3. **关联附加数据**：允许在VFS中的文件上关联额外的持久性数据。

为了实现后两项功能，VFS管理一个持久化的用户硬盘内容快照。这个快照只存储那些通过VFS API至少请求过一次的文件，并异步更新以匹配磁盘上的变化。

该快照是应用级别的，而不是项目级别的——因此，如果某些文件（例如JDK中的一个类）被多个项目引用，则其内容只会在VFS中存储一个副本。

所有的VFS访问操作都通过快照进行。如果通过VFS API请求了一些信息，而快照中没有该信息，则它会从磁盘加载并存储到快照中。如果快照中有该信息，则返回快照中的数据。文件的内容和目录中的文件列表仅在特定信息被访问时存储在快照中。否则，只存储文件的元数据，如名称、长度、时间戳、属性等。

这意味着，IntelliJ平台UI中显示的文件系统状态和文件内容来自快照，这可能不总是与磁盘上的实际内容相符。例如，在某些情况下，删除的文件可能仍然在UI中可见，直到IntelliJ平台检测到删除为止。

## VFS刷新操作

刷新操作用于将VFS的部分状态与实际磁盘内容同步。刷新操作通常是异步触发的。所有通过VFS进行的写操作都是同步的，即内容会立即保存到磁盘。

刷新操作由IntelliJ平台或插件代码显式调用——即当文件在IDE运行时在磁盘上发生变化时，VFS不会立即检测到该变化。VFS会在下一次刷新操作期间更新，其中包括该文件。

IntelliJ平台在启动时会异步刷新整个项目的内容。默认情况下，当用户从其他应用程序切换到IDE时，它会执行刷新操作，但用户可以在“设置 | 外观和行为 | 系统设置 | 同步外部更改”中关闭此功能。

在Windows、Mac和Linux上，启动了一个本地文件监视器进程，该进程接收文件系统的文件更改通知，并将其报告给IntelliJ平台。如果文件监视器可用，刷新操作只会检查被报告为已更改的文件。如果没有文件监视器，刷新操作会遍历刷新范围内的所有目录和文件。

可以通过内部操作“工具 | 内部操作 | VFS | 显示监视的VFS根目录”来查看当前项目的所有注册根目录。

## 同步和异步刷新

从调用者的角度来看，刷新操作可以是同步的或异步的。实际上，刷新操作根据自己的线程策略执行。同步标志仅意味着调用线程将在刷新操作（很可能在不同的线程上运行）完成之前被阻塞。

无论是同步还是异步刷新，都可以从任何线程启动。如果从后台线程启动刷新，调用线程不能持有读操作，否则会导致死锁。有关线程模型和读/写操作的更多详细信息，请参见IntelliJ平台架构概述。

在几乎所有情况下，都强烈建议使用异步刷新。如果需要在刷新完成后执行某些代码，可以将代码作为`postRunnable`参数传递给其中一个刷新方法：

- `RefreshQueue.createSession()`
- `VirtualFile.refresh()`

在某些情况下，同步刷新可能会导致死锁，具体取决于哪个线程持有锁。

## 虚拟文件系统事件

虚拟文件系统中发生的所有更改，无论是由于刷新操作还是用户操作引起的，都会报告为虚拟文件系统事件。VFS事件总是在事件分发线程中并在写操作中触发。

监听VFS事件的最有效方法是实现`BulkFileListener`并将其订阅到`VirtualFileManager.VFS_CHANGES`主题。非阻塞变体`AsyncFileListener`也可以在2019.2或更高版本中使用。有关实现详细信息，请参见如何在VFS更改时收到通知。

VFS监听器是应用级别的，会接收所有用户打开的项目中的更改事件。您可能需要过滤掉与您的任务无关的事件（例如，通过`ProjectFileIndex.isInContent()`）。

VFS事件在每次更改之前和之后都会发送，您可以在之前的事件中访问文件的旧内容。请注意，由刷新引起的事件是在更改已经在磁盘上发生之后发送的。因此，当您处理`beforeFileDeletion`事件时，例如，该文件已经从磁盘上删除。然而，它仍然存在于VFS快照中，您可以使用VFS API访问其最后的内容。

请注意，刷新操作仅针对快照中已加载的文件发送更改事件。例如，如果您访问了一个目录的`VirtualFile`，但从未使用`VirtualFile.getChildren()`加载其内容，则在该目录中创建文件时，您可能不会收到`fileCreated`通知。

如果您只使用`VirtualFile.findChild()`加载了目录中的一个文件，您将收到对该文件的更改通知，但可能不会收到同一目录中其他文件的创建/删除通知。

## 虚拟文件 (Virtual Files)

虚拟文件 (VirtualFile, VF) 是 IntelliJ 平台中用于表示虚拟文件系统 (VFS) 中文件的抽象概念。它不仅限于本地文件系统中的文件，还可以表示 JAR 文件中的类、从版本控制仓库中加载的旧文件版本等。

## 虚拟文件的主要特性

1. **多种文件系统支持**：虚拟文件不仅可以是本地文件，还可以是多种可插拔文件系统实现中的文件，如归档文件、版本控制中的文件等。
2. **二进制内容处理**：VFS 层只处理二进制内容，内容被视为字节流。编码和行分隔符等概念在更高的系统层次上处理。

## 如何获取虚拟文件？

虚拟文件可以通过以下不同的上下文获取：

- **Action事件**：使用 `AnActionEvent.getData(PlatformDataKeys.VIRTUAL_FILE)` 获取当前选择的虚拟文件，或使用 `AnActionEvent.getData(PlatformDataKeys.VIRTUAL_FILE_ARRAY)` 获取多个选择的文件。
- **文档对象**：使用 `FileDocumentManager.getFile()` 从文档获取文件。
- **PSI 文件**：使用 `PsiFile.getVirtualFile()` 获取，注意如果 PSI 文件只存在于内存中可能返回 `null`。
- **文件名**：使用 `FilenameIndex.getVirtualFilesByName()` 通过文件名获取文件。
- **本地文件系统路径**：使用 `LocalFileSystem.findFileByIoFile()` 或 `VirtualFileManager.findFileByNioPath()` 来获取文件。

## 可以对虚拟文件进行哪些操作？

虚拟文件支持常见的文件操作，如遍历文件系统、获取文件内容、重命名、移动或删除。对于递归遍历，建议使用 `VfsUtilCore.iterateChildrenRecursively()` 以防止由于递归符号链接而导致的无限循环。

## 虚拟文件的来源

VFS 通过自上而下扫描文件系统逐渐构建起来，通常从项目根目录开始。VFS 刷新操作可以检测文件系统中新出现的文件。可以通过 `VirtualFileManager.syncRefresh()` 或 `VirtualFile.refresh()` 编程地启动刷新操作。当文件系统监视器收到文件系统更改通知时，也会触发VFS刷新。

## 虚拟文件的生命周期

在IDE进程的整个生命周期中，磁盘上的特定文件由等价的 `VirtualFile` 实例表示。这些实例可以被垃圾回收。如果文件被删除，其对应的 `VirtualFile` 实例将变为无效（`isValid()` 返回 `false`），并且对它的操作会导致异常。

## 如何创建虚拟文件？

通常不需要直接创建虚拟文件。通常是通过 PSI API 或常规的 `java.io.File` API 来创建文件。如果需要通过 VFS 创建文件，可以使用 `VirtualFile.createChildData()` 创建 `VirtualFile` 实例，然后使用 `VirtualFile.setBinaryContent()` 向文件写入数据。

## 如何接收VFS更改的通知？

可以实现 `BulkFileListener` 并订阅 `VirtualFileManager.VFS_CHANGES` 消息总线主题。例如：

```java
project.getMessageBus().connect().subscribe(VirtualFileManager.VFS_CHANGES,
    new BulkFileListener() {
      @Override
      public void after(@NotNull List<? extends VFileEvent> events) {
        // 处理事件
      }
    });
```

从 2019.2 版本开始，还可以使用非阻塞的替代方法 `AsyncFileListener`。

## 是否有用于分析和操作虚拟文件的工具？

`VfsUtil` 和 `VfsUtilCore` 提供了用于分析虚拟文件系统中文件的实用方法。对于存储大量虚拟文件，可以使用 `VfsUtilCore.createCompactVirtualFileSet()`。使用 `ProjectLocator` 可以找到包含给定虚拟文件的项目。

## 如何扩展VFS？

要提供替代文件系统实现（例如 FTP 文件系统），可以实现 `VirtualFileSystem` 类（通常还需要实现 `VirtualFile`），并通过 `com.intellij.virtualFileSystem` 扩展点注册你的实现。

要挂钩到本地文件系统中执行的操作（例如，当开发需要自定义重命名/移动处理的版本控制系统集成时），可以实现 `LocalFileOperationsHandler` 并通过 `LocalFileSystem.registerAuxiliaryFileOperationsHandler()` 注册它。

## 工作中与VFS相关的规则

请参阅虚拟文件系统的详细描述和使用指南，以更好地了解VFS的架构和使用规范。

# 文档 (Documents)

在 IntelliJ 平台中，文档 (Document) 是一段可编辑的 Unicode 字符序列，通常对应于虚拟文件的文本内容。文档中的换行符总是规范化为 `\n`，而 IntelliJ 平台在加载和保存文档时会透明地处理编码和换行符的转换。

## 如何获取文档？

根据不同的上下文，可以使用以下 API 来获取文档：

- **Action 事件**：使用 `AnActionEvent.getData(CommonDataKeys.EDITOR).getDocument()` 获取当前编辑器中的文档。
- **PSI 文件**：使用 `PsiDocumentManager.getDocument()` 获取对应的文档，或者使用 `PsiDocumentManager.getCachedDocument()` 获取已缓存的文档。
- **虚拟文件**：使用 `FileDocumentManager.getDocument()` 强制从磁盘加载文档内容，或者使用 `FileDocumentManager.getCachedDocument()` 获取已缓存的文档。

## 可以对文档进行哪些操作？

可以在“纯文本”级别上执行任何访问或修改文件内容的操作，这意味着操作的是字符序列而不是 PSI 元素的树结构。

## 文档从何而来？

文档实例是在某些操作需要访问文件的文本内容时创建的，例如构建文件的 PSI 结构。此外，可以临时创建不链接到任何虚拟文件的文档实例，例如在对话框中表示文本编辑器字段的内容。

## 文档的生命周期有多长？

文档实例被虚拟文件实例弱引用。因此，如果没有人引用未修改的文档实例，它可以被垃圾回收，并且如果稍后重新访问文档内容，将创建一个新的实例。

将文档引用存储在插件的长期数据结构中会导致内存泄漏。

## 如何创建文档？

如果要在磁盘上创建新文件，请不要创建文档，而是创建 PSI 文件并获取其文档（见如何创建 PSI 文件？）。要创建一个不绑定到任何东西的文档实例，请使用 `EditorFactory.createDocument()`。

## 如何接收文档更改的通知？

- 使用 `Document.addDocumentListener()` 可以接收特定文档实例的更改通知。
- 使用 `EditorFactory.getEventMulticaster().addDocumentListener()` 可以接收所有打开文档的更改通知。
- 注册 `FileDocumentManagerListener` 或订阅 `AppTopics.FILE_DOCUMENT_SYNC` 消息总线，以接收文档保存或从磁盘重新加载的通知。

## 操作文档的规则是什么？

文档操作遵循一般的读/写操作规则（参见线程模型）。此外，任何修改文档内容的操作都必须包装在一个命令中（`CommandProcessor.executeCommand()`）。`executeCommand()` 调用是可以嵌套的，最外层的 `executeCommand()` 调用将添加到撤销栈中。如果在一个命令中修改了多个文档，撤销该命令时系统会默认向用户显示确认对话框。

如果与文档对应的文件是只读的（例如，未从版本控制系统中检出），则文档修改将失败。因此，在修改文档之前，必须调用 `ReadonlyStatusHandler.ensureFilesWritable()` 以确保文件可写。

所有传递给文档修改方法（如 `setText()`、`insertString()`、`replaceString()`）的文本字符串必须只使用 `\n` 作为行分隔符。

## 有哪些实用工具可以用于文档操作？

`DocumentUtil` 包含用于文档处理的实用方法。它允许获取特定行的文本偏移量等信息。这对于需要关于给定 `PsiElement` 的文本位置/偏移信息时特别有用。



# 编辑器 (Editors)

在 IntelliJ 平台中，编辑器是处理文本的主要工具。编辑器不仅支持基本的文本编辑功能，还提供了许多高级功能，如语法高亮、代码折叠、自动补全等。以下内容介绍如何与 IntelliJ 平台的编辑器交互和进行扩展。



# Working with Text﻿

[Edit page](https://github.com/JetBrains/intellij-sdk-docs/edit/main/topics/tutorials/editor_basics/working_with_text.md)Last modified: 31 July 2024

This tutorial shows how to use actions to access a caret placed in a document open in an editor. Using information about the caret, replace selected text in a document with a string.

The approach in this tutorial relies heavily on creating and registering actions. To review the fundamentals of creating and registering actions, refer to the [Actions Tutorial](https://plugins.jetbrains.com/docs/intellij/action-system.html).

Multiple examples are used from the [editor_basics](https://github.com/JetBrains/intellij-sdk-code-samples/tree/main/editor_basics) plugin code sample from the IntelliJ Platform SDK. It may be helpful to open that project in an IntelliJ Platform-based IDE, build the project, run it, select some text in the editor, and invoke the Editor Replace Text menu item on the editor context menu.

![Editor Basics Menu](https://plugins.jetbrains.com/docs/intellij/images/basics.png)

## Creating a New Menu Action﻿

In this example, we access the `Editor` from an action. The source code for the Java class in this example is [EditorIllustrationAction](https://github.com/JetBrains/intellij-sdk-code-samples/tree/main/editor_basics/src/main/java/org/intellij/sdk/editor/EditorIllustrationAction.java).

To register the action, we must add the corresponding elements to the [``](https://plugins.jetbrains.com/docs/intellij/plugin-configuration-file.html#idea-plugin__actions) section of the plugin configuration file [plugin.xml](https://github.com/JetBrains/intellij-sdk-code-samples/tree/main/editor_basics/src/main/resources/META-INF/plugin.xml). For more information, refer to the [Registering Actions](https://plugins.jetbrains.com/docs/intellij/working-with-custom-actions.html#registering-a-custom-action) section of the Actions Tutorial. The `EditorIllustrationAction` action is registered in the group `EditorPopupMenu` so it will be available from the context menu when focus is on the editor:

```markup
<action
    id="EditorBasics.EditorIllustrationAction"
    class="org.intellij.sdk.editor.EditorIllustrationAction"
    text="Editor Replace Text"
    description="Replaces selected text with 'Replacement'."
    icon="SdkIcons.Sdk_default_icon">
  <add-to-group group-id="EditorPopupMenu" anchor="first"/>
</action>
```



## Defining the Menu Action's Visibility﻿

To determine conditions by which the action will be visible and available requires `EditorIllustrationAction` to override the `AnAction.update()` method. For more information, refer to [Extending the Update Method](https://plugins.jetbrains.com/docs/intellij/working-with-custom-actions.html#extending-the-update-method) section of the Actions Tutorial.

To work with a selected part of the text, it's reasonable to make the menu action available only when the following requirements are met:

- There is a [`Project`](https://github.com/JetBrains/intellij-community/tree/idea/241.18034.62/platform/core-api/src/com/intellij/openapi/project/Project.java) object,
- There is an instance of [`Editor`](https://github.com/JetBrains/intellij-community/tree/idea/241.18034.62/platform/editor-ui-api/src/com/intellij/openapi/editor/Editor.java) available,
- There is a text selection in `Editor`.

Additional steps will show how to check these conditions through obtaining instances of `Project` and `Editor` objects, and how to show or hide the action's menu items based on them.

### Getting an Instance of the Active Editor from an Action Event﻿

Using the [`AnActionEvent`](https://github.com/JetBrains/intellij-community/tree/idea/241.18034.62/platform/editor-ui-api/src/com/intellij/openapi/actionSystem/AnActionEvent.java) event passed into the `update` method, a reference to an instance of the `Editor` can be obtained by calling `getData(CommonDataKeys.EDITOR)`. Similarly, to obtain a project reference, we use the `getProject()` method.

```java
public class EditorIllustrationAction extends AnAction {
  @Override
  public void update(@NotNull AnActionEvent event) {
    // Get required data keys
    Project project = event.getProject();
    Editor editor = event.getData(CommonDataKeys.EDITOR);
    // ...
  }
}
```



Note: There are other ways to access an `Editor` instance:

- If a [`DataContext`](https://github.com/JetBrains/intellij-community/tree/idea/241.18034.62/platform/core-ui/src/openapi/actionSystem/DataContext.java) object is available: `CommonDataKeys.EDITOR.getData(context);`
- If only a `Project` object is available, use `FileEditorManager.getInstance(project).getSelectedTextEditor()`

### Obtaining a Caret Model and Selection﻿

After making sure a project is open, and an instance of the `Editor` is obtained, we need to check if any selection is available. The [`SelectionModel`](https://github.com/JetBrains/intellij-community/tree/idea/241.18034.62/platform/editor-ui-api/src/com/intellij/openapi/editor/SelectionModel.java) interface is accessed from the `Editor` object. Determining whether some text is selected is accomplished by calling the `SelectionModel.hasSelection()` method. Here's how the `EditorIllustrationAction.update(AnActionEvent event)` method should look:

```java
public class EditorIllustrationAction extends AnAction {
  @Override
  public void update(@NotNull AnActionEvent event) {
    // Get required data keys
    Project project = event.getProject();
    Editor editor = event.getData(CommonDataKeys.EDITOR);

    // Set visibility only in the case of
    // existing project editor, and selection
    event.getPresentation().setEnabledAndVisible(project != null
        && editor != null && editor.getSelectionModel().hasSelection());
  }
}
```



Note: `Editor` also allows access to different models of text representation. The model classes are located in [editor](https://github.com/JetBrains/intellij-community/tree/idea/241.18034.62/platform/editor-ui-api/src/com/intellij/openapi/editor), and include:

- [`CaretModel`](https://github.com/JetBrains/intellij-community/tree/idea/241.18034.62/platform/editor-ui-api/src/com/intellij/openapi/editor/CaretModel.java),
- [`FoldingModel`](https://github.com/JetBrains/intellij-community/tree/idea/241.18034.62/platform/editor-ui-api/src/com/intellij/openapi/editor/FoldingModel.java),
- [`IndentsModel`](https://github.com/JetBrains/intellij-community/tree/idea/241.18034.62/platform/editor-ui-api/src/com/intellij/openapi/editor/IndentsModel.java),
- [`ScrollingModel`](https://github.com/JetBrains/intellij-community/tree/idea/241.18034.62/platform/editor-ui-api/src/com/intellij/openapi/editor/ScrollingModel.java),
- [`SoftWrapModel`](https://github.com/JetBrains/intellij-community/tree/idea/241.18034.62/platform/editor-ui-api/src/com/intellij/openapi/editor/SoftWrapModel.java)

## Safely Replacing Selected Text in the Document﻿

Based on the evaluation of conditions by `EditorIllustrationAction.update()`, the `EditorIllustrationAction` action menu item is visible. To make the menu item do something, the `EditorIllustrationAction` class must override the `AnAction.actionPerformed()` method. As explained below, this will require the `EditorIllustrationAction.actionPerformed()` method to:

- Gain access to the document.
- Get the character locations defining the selection.
- Safely replace the contents of the selection.

Modifying the selected text requires an instance of the [`Document`](https://github.com/JetBrains/intellij-community/tree/idea/241.18034.62/platform/core-api/src/com/intellij/openapi/editor/Document.java) object, which is accessed from the `Editor` object. The [Document](https://plugins.jetbrains.com/docs/intellij/documents.html) represents the contents of a text file loaded into memory and opened in an IntelliJ Platform-based IDE editor. An instance of the `Document` will be used later when a text replacement is performed.

The text replacement will also require information about where the selection is in the document, which is provided by the primary `Caret` object, obtained from the `CaretModel`. Selection information is measured in terms of [Offset](https://plugins.jetbrains.com/docs/intellij/coordinates-system.html#caret-offset), the count of characters from the beginning of the document to a caret location.

Text replacement could be done by calling the `Document` object's `replaceString()` method. However, safely replacing the text requires the `Document` to be locked and any changes performed in a write action. See the [Threading Issues](https://plugins.jetbrains.com/docs/intellij/threading-model.html) section to learn more about synchronization issues and changes safety on the IntelliJ Platform. This example changes the document within a [`WriteCommandAction`](https://github.com/JetBrains/intellij-community/tree/idea/241.18034.62/platform/core-api/src/com/intellij/openapi/command/WriteCommandAction.java).

The complete `EditorIllustrationAction.actionPerformed()` method is shown below:

- Note the selection in the document is replaced by a string using a method on the `Document` object, but the method call is wrapped in a write action.
- After the document change, the new text is de-selected by a call to the primary caret.

```java
public class EditorIllustrationAction extends AnAction {
  @Override
  public void actionPerformed(@NotNull AnActionEvent event) {
    // Get all the required data from data keys
    Editor editor = event.getRequiredData(CommonDataKeys.EDITOR);
    Project project = event.getRequiredData(CommonDataKeys.PROJECT);
    Document document = editor.getDocument();

    // Work off of the primary caret to get the selection info
    Caret primaryCaret = editor.getCaretModel().getPrimaryCaret();
    int start = primaryCaret.getSelectionStart();
    int end = primaryCaret.getSelectionEnd();

    // Replace the selection with a fixed string.
    // Must do this document change in a write action context.
    WriteCommandAction.runWriteCommandAction(project, () ->
        document.replaceString(start, end, "editor_basics")
    );

    // De-select the text range that was just replaced
    primaryCaret.removeSelection();
  }
}
```



# 编辑器坐标系统：位置和偏移量

在前面的教程“处理文本”中，我们展示了如何使用操作来访问在编辑器中打开的文档中的插入符（Caret）。示例使用插入符的信息来替换文档中选定的文本。

每个插入符都有一组描述其在多个坐标系统中的位置的属性。本教程将介绍如何获取有关编辑器中插入符的信息。

## 编辑器基础代码示例

在本教程中，我们使用了 editor_basics 代码示例来探索插入符的位置。特别是，通过 editor_basics 添加到编辑器上下文菜单中的“Caret Position”操作来获取当前插入符位置的信息。该操作也可以通过键盘快捷键启动。

菜单操作背后的 Java 类的源代码是 `EditorAreaIllustration`。讨论的重点将是 `EditorAreaIllustration.actionPerformed()` 方法。有关创建操作类的更多信息，请参阅操作教程。

## 从 CaretModel 和 Caret 对象获取插入符位置

可以通过获取 `CaretModel` 对象的实例来访问插入符的属性。与“处理文本”教程中的方法相同，使用 `AnActionEvent` 获取 `Editor` 对象。`Editor` 对象提供了对 `CaretModel` 对象的访问，如下所示：

```java
public class EditorAreaIllustration extends AnAction {
  @Override
  public void actionPerformed(@NotNull AnActionEvent event) {
    // 获取编辑器和插入符模型的访问权限。update() 验证了编辑器的存在。
    Editor editor = event.getRequiredData(CommonDataKeys.EDITOR);
    CaretModel caretModel = editor.getCaretModel();
  }
}
```

## 编辑器坐标系统

当文档打开时，编辑器为文档中的行和列分配一个内部的、基于零的坐标系统。文档中的第一行和每行的第一个字符被分配为零位置。文档中的每个字符都分配一个偏移量（Offset），这是从文件开头到该字符的字符数。这些逻辑位置坐标用于描述插入符位置的行和列号。注意，逻辑位置坐标系统不同于编辑器的 UI，它是基于一的，而不是零。

逻辑位置坐标和本教程中讨论的其他坐标系统可以用来描述编辑器中的任何位置，而不仅仅是插入符。例如，用于代码提示的提示位置就是用这些坐标来描述的，例如 `HintManager.getHintPosition()`。在编辑器中显示的自定义视觉元素（称为 Inlay 对象）也是以这些坐标系统表示的。

下图显示了应用于一些示例内容的逻辑位置坐标系统。红框中的字符“s”表示将光标放置在该字符上。它的插入符位置是第 1 行，第 9 列，偏移量为 28。下面将进一步讨论插入符偏移量。

![Editor Coordinates](https://resources.jetbrains.com/help/img/idea/2022.1/EditorCoordinates.png)

## 插入符逻辑位置

插入符的逻辑位置是在编辑器中的零基（行和列）位置。可以通过该插入符的 `LogicalPosition` 对象获取逻辑位置信息。

逻辑位置行号忽略了改变文档在编辑器中呈现方式的设置效果。这些设置的示例包括代码（行）折叠和软换行。这些效果意味着无论编辑器中的一行或多行是折叠还是软换行，插入符的逻辑位置行号都不会改变。

在下面的 Java 文件示例中，逻辑位置行号 1-3 被折叠到行 0。插入符（蓝色块）被放置在“public”中的字母“p”上。使用 editor_basics 的 `Caret Position` 操作检查插入符时，它被报告为逻辑位置 (5,0) - 即第 5 行，第 0 个字符 - 该行的第一个字符。这意味着插入符的逻辑位置不会因代码折叠而改变。

![Caret Logical Position with Folding](https://resources.jetbrains.com/help/img/idea/2022.1/CaretLogicalPosition.png)

## 插入符视觉位置

插入符的视觉位置与逻辑位置不同，因为它考虑了编辑器呈现设置，如代码折叠和软换行。因此，视觉位置的行数是文档中可以在编辑器中显示的行数。因此，视觉位置不能唯一地映射到逻辑位置或相应的文档行。

例如，软换行会影响后续行的视觉位置。在下面的图像中，软换行应用于逻辑行三。将插入符放置在前述测试中的相同字符位置，逻辑位置未发生变化。然而，视觉位置行号增加了一行！每行的注释说明了软换行部分的逻辑行三被评估为视觉位置行四，就像它是一个单独的行一样。

![Caret Visual Position with Soft-Wrap](https://resources.jetbrains.com/help/img/idea/2022.1/CaretVisualPosition.png)

## 插入符列位置

列位置是从逻辑行开头到该行当前插入符位置的字符数。字符是用零基编号系统计数的，所以行的第一个字符编号为零。注意，列位置不同于编辑器 UI，后者使用基于一的编号方案。

列位置包括：

- 空白字符，如制表符。制表符可以占据多个列，最多可以是编辑器设置的制表符大小。
- 由插入符选择的字符。

## 插入符偏移

插入符的偏移量是从文档开头到插入符位置的字符数。插入符偏移量总是以逻辑位置计算的。插入符偏移量包括：

- 文档中的第一个（第 0 个）字符。
- 空白字符，包括换行和制表符。
- 如果 IDE 设置允许，则换行后的任何字符。（设置 | 编辑器 | 常规 | 虚拟空间）
- 由插入符选择的字符。

下面的示例演示了将插入符放置在逻辑第一行第一个字符处的偏移量。注意偏移量为 22，比第 0 行的可见字符数多一个，以及第 1 行的第一个字符。这种表面上的不一致实际上是正确的，因为偏移量包括第 0 行的换行符。

## 显示插入符位置

为了显示插入符的逻辑和视觉位置以及偏移量的值，使用 `Messages.showInfoMessage()` 方法在执行操作时以通知的形式显示这些信息。

```java
public class EditorAreaIllustration extends AnAction {

  public void actionPerformed(@NotNull AnActionEvent event) {
    // 获取编辑器和插入符模型的访问权限。
    Editor editor = event.getRequiredData(CommonDataKeys.EDITOR);
    CaretModel caretModel = editor.getCaretModel();

    // 获取主插入符以确保获取正确的插入符。
    Caret primaryCaret = caretModel.getPrimaryCaret();
    // 获取插入符信息
    LogicalPosition logicalPos = primaryCaret.getLogicalPosition();
    VisualPosition visualPos = primaryCaret.getVisualPosition();
    int caretOffset = primaryCaret.getOffset();

    // 构建并显示插入符报告。
    String report = logicalPos.toString() + "\n" +
        visualPos.toString() + "\n" +
        "Offset: " + caretOffset;
    Messages.showInfoMessage(report, "Caret Parameters Inside The Editor");
  }

}
```

## 总结

本文介绍了如何在 IntelliJ 平台中使用插入符坐标系统来访问编辑器中的文本位置。我们讨论了如何通过不同的坐标系统表示插入符的位置，并展示了如何获取和使用这些信息来执行文本操作。这些技术对于需要处理文本位置和偏移量的插件开发者来说非常有用。

## 处理编辑器事件

在上一教程“编辑器坐标系统”中，我们讨论了如何在编辑器窗口中使用插入符的坐标系统。插入符位置通过逻辑位置、视觉位置和偏移量进行讨论。本教程介绍了编辑器操作系统，该系统处理由编辑器中的按键事件激活的操作。我们将使用 `editor_basics` 代码示例中的两个类来说明：

- 使用 IntelliJ 平台的 `EditorActionHandler` 操作插入符。
- 创建和注册自定义 `TypedActionHandler` 拦截按键并更改文档。

### 使用 IntelliJ 平台的 `EditorActionHandler`

在本教程的这一部分中，我们将使用 `editor_basics` 代码示例来演示克隆现有的插入符。自定义操作类将使用 `EditorActionManager` 访问特定的 `EditorActionHandler` 以进行插入符克隆。`editor_basics` 代码示例在编辑器上下文菜单中添加了一个“Editor Add Caret”菜单项：

#### 创建菜单操作类

Java 操作类的源代码是 `EditorHandlerIllustration`，它是 `AnAction` 的子类。有关创建操作类的更多信息，请参阅操作教程，其中详细介绍了该主题。

`EditorHandlerIllustration` 操作在 `editor_basic` 的 `plugin.xml` 文件中注册。注意，此操作类被注册为在编辑器上下文菜单中出现。

```xml
<actions>
  <action
      id="EditorBasics.EditorHandlerIllustration"
      class="org.intellij.sdk.editor.EditorHandlerIllustration"
      text="Editor Add Caret"
      description="Adds a second caret below the existing one."
      icon="SdkIcons.Sdk_default_icon">
    <add-to-group group-id="EditorPopupMenu" anchor="first"/>
  </action>
</actions>
```

#### 设置操作菜单条目的可见性

在什么条件下 `EditorHandlerIllustration` 操作应该能够克隆插入符？只有在 `EditorHandlerIllustration.update()` 方法中满足以下条件时：

- 项目已打开，
- 有可用的编辑器，
- 编辑器中至少有一个活动的插入符。

确保 `Project` 和 `Editor` 对象可用后，使用 `Editor` 对象验证编辑器中是否至少有一个插入符：

```java
public class EditorHandlerIllustration extends AnAction {
  @Override
  public void update(@NotNull AnActionEvent event) {
    Project project = event.getProject();
    Editor editor = event.getData(CommonDataKeys.EDITOR);

    // 确保至少有一个插入符可用
    boolean menuAllowed = false;
    if (editor != null && project != null) {
      // 确保编辑器中的插入符列表不为空
      menuAllowed = !editor.getCaretModel().getAllCarets().isEmpty();
    }
    event.getPresentation().setEnabledAndVisible(menuAllowed);
  }
}
```

#### 获取正确的 `EditorActionHandler`

当 `EditorHandlerIllustration.actionPerformed()` 方法克隆插入符时，应使用合适的 IntelliJ 平台 `EditorActionHandler`。需要一个 `EditorActionManager` 实例来获取正确的 `EditorActionHandler`。`EditorActionManager` 类提供了一个静态方法来完成此操作。

要从 `EditorActionManager` 请求正确的 `EditorActionHandler`，请参阅 `IdeActions` 接口，以获取传递给 `EditorActionManager.getActionHandler()` 方法的正确常量。对于在主插入符下方克隆插入符的操作，常量是 `ACTION_EDITOR_CLONE_CARET_BELOW`。基于该常量，`EditorActionManager` 返回一个 `CloneCaretActionHandler` 实例，这是 `EditorActionHandler` 的子类。

```java
// EditorHandlerIllustration.actionPerformed() 的片段
EditorActionManager actionManager = EditorActionManager.getInstance();
EditorActionHandler actionHandler =
    actionManager.getActionHandler(IdeActions.ACTION_EDITOR_CLONE_CARET_BELOW);
```

#### 使用 `EditorActionHandler` 克隆插入符

要克隆插入符，只需调用 `EditorActionHandler.execute()` 方法，并传递适当的上下文即可。

```java
public class EditorHandlerIllustration extends AnAction {
  @Override
  public void actionPerformed(@NotNull AnActionEvent event) {
    Editor editor = event.getRequiredData(CommonDataKeys.EDITOR);
    EditorActionManager actionManager = EditorActionManager.getInstance();
    EditorActionHandler actionHandler =
        actionManager.getActionHandler(IdeActions.ACTION_EDITOR_CLONE_CARET_BELOW);
    actionHandler.execute(editor,
        editor.getCaretModel().getPrimaryCaret(), event.getDataContext());
  }
}
```

### 创建自定义 `TypedActionHandler`

`TypedActionHandler` 接口是处理来自编辑器的按键事件的基础类。自定义实现的类注册为处理编辑器按键事件，并为每个按键事件接收回调。以下步骤说明如何使用 `TypedActionHandler` 自定义编辑器接收按键事件时的行为。

#### 实现自定义 `TypedActionHandler` 类

首先，创建一个基于 `TypedActionHandler` 的子类，例如 `MyTypedHandler`。该类重写 `TypedActionHandler.execute()` 方法，该方法是编辑器按键事件的回调。

#### 实现按键事件处理逻辑

在 `MyTypedHandler` 中重写 `TypedActionHandler.execute()` 方法，以实现按键事件的处理逻辑。每次按键时，当编辑器工具窗口具有焦点时，都会调用此方法。

在以下示例中，每当发生按键事件时，`MyTypedHandler.execute()` 方法在零插入符偏移位置插入“editor_basics\n”。如“处理文本”中所述，文档的安全修改必须在写操作上下文中进行。因此，尽管 `Document` 接口上的方法执行了字符串插入，但写操作确保了上下文的稳定性。

```java
final class MyTypedHandler implements TypedActionHandler {
  @Override
  public void execute(@NotNull Editor editor,
                      char c,
                      @NotNull DataContext dataContext) {
    Document document = editor.getDocument();
    Project project = editor.getProject();
    Runnable runnable = () -> document.insertString(0, "editor_basics\n");
    WriteCommandAction.runWriteCommandAction(project, runnable);
  }
}
```

#### 注册自定义 `TypedActionHandler`

自定义的 `TypedActionHandler` 实现必须注册，以替换现有的键入处理程序以接收编辑器按键事件。注册是通过 `TypedAction` 类完成的。

如下面的代码片段所示，使用 `EditorActionManager` 获取 `TypedAction` 类的访问权限。使用 `TypedAction.setupHandler()` 方法注册自定义的 `MyTypedHandler` 类：

```java
public class EditorHandlerIllustration extends AnAction {
  static {
    EditorActionManager actionManager = EditorActionManager.getInstance();
    TypedAction typedAction = actionManager.getTypedAction();
    typedAction.setupHandler(new MyTypedHandler());
  }
}
```

将注册代码放置在 `EditorHandlerIllustration` 类中在某种程度上是任意的，因为 `MyTypedHandler` 的注册与 `EditorHandlerIllustration` 类无关。然而，`EditorHandlerIllustration` 类很方便，因为作为一个操作，它会在应用程序启动时实例化。在实例化时，`EditorHandlerIllustration` 中的静态代码块被评估。在 `editor_basics` 代码示例中，任何 `AnAction` 派生类都可以用于注册 `MyTypedHandler`。

# 文本选择

在 IntelliJ 平台 IDE 中，扩展和收缩选择的操作允许根据源代码的结构调整所选文本。这使得选择表达式、代码块、函数定义等变得容易，甚至可以选择整行或 Javadoc 注释中的标签。当实现自定义语言时，IntelliJ 平台提供了此扩展点（EP）的基本实现，允许您根据 PSI 结构选择代码和选择整行。虽然这些通常足以提供良好的用户体验，但有时为用户提供其他可选区域是有利的。

## 扩展/收缩文本选择

通过实现 `ExtendWordSelectionHandler` 并在 `plugin.xml` 中的 `com.intellij.extendWordSelectionHandler` EP 中注册它，您可以提供扩展或收缩选择时使用的其他文本范围。对于您希望提供其他文本范围的 PSI 元素，从 `canSelect(PsiElement)` 返回 true。对于这些元素，IntelliJ 平台将调用 `select(PsiElement, CharSequence, int, Editor)`，您可以在此处计算其他文本范围并将其作为 `List<TextRange>` 返回。

### 方法概述

此 EP 有两个需要实现的方法：

- `canSelect(PsiElement)`: 从光标所在的 PSI 元素开始，遍历每个父元素，对每个 PSI 元素调用此方法。对于特定元素返回 true，表示应包括进一步的文本范围。
  
- `select(PsiElement, CharSequence, int, Editor)`: 返回感兴趣的 PSI 元素内的文本范围。

### 示例用例

假设自定义语言开发者有一个函数调用 `f(a, b)`，其中函数调用节点有两个参数作为子元素。如果光标位于参数 `a`，扩展选择首先会选择参数 `a` 本身，下一步则扩展覆盖整个函数调用。然而，可能希望选择所有参数的列表作为中间步骤。这可以通过以下方式实现：

1. 创建一个实现 `ExtendWordSelectionHandler` 接口的类，并在 `plugin.xml` 中的 `com.intellij.extendWordSelectionHandler` EP 中注册它。

2. `canSelect(PsiElement)` 方法应对函数调用节点返回 true。这表示将为函数调用节点调用 `select(PsiElement, CharSequence, int, Editor)`。

3. 当调用 `select()` 方法时，可以使用函数调用 PSI 元素或编辑器文本提取跨越参数 `a` 和 `b` 的文本范围，并将其作为包含一个元素的 `List<TextRange>` 返回。

### 进一步的见解和调试

查看其他实现是理解这个 EP 工作方式的有效方法。您可以查看 `DocTagSelectioner` 以获得进一步的见解，它提供了选择 Javadoc 注释中像 `@param` 这样的标签名的能力。此外，IntelliJ 平台浏览器提供了包含 `com.intellij.extendWordSelectionHandler` EP 实现的开源插件列表。

在调试时，可以在 IntelliJ 平台中的一些重要位置设置断点。当用户调用扩展选择时，它由 `SelectWordHandler` 处理。然而，大部分工作是在 `SelectWordUtil` 内完成的，`processElement()` 会检查哪些此 EP 的实现适用于当前的 PSI 元素。如果其中一个从其 `canSelect()` 方法返回 true，则在 `askSelectioner()` 函数中提取其他文本范围。这些地方是设置断点和调查调试的好位置。

## 多个插入符（Multiple Carets）

在 IntelliJ 平台中，编辑器支持同时存在多个插入符（也称为光标或插入点）。这意味着用户可以在文档的多个位置同时进行文本编辑操作，如导航、插入、删除等。这一功能对提高编程效率特别有帮助，尤其是在需要同时编辑多个相似代码块时。

### 核心功能

核心多插入符功能由 `CaretModel` 提供。`CaretModel` 允许访问当前存在的所有插入符，并提供添加和移除插入符的方法。文本选择的相关操作由 `SelectionModel` 处理。

在多插入符模式下，大多数操作都会独立应用于每个插入符，每个插入符都有其关联的选区（可以为空）。当多个插入符在某些操作后位于相同的视觉位置时，它们会合并为一个插入符，并且它们的选区也会合并。类似地，当多个插入符的选区发生重叠时，也只会保留一个插入符，并合并选区。

### 主插入符

在多个插入符的场景中，**主插入符**（Primary Caret）是一个特别重要的概念。非多插入符感知的操作以及需要单点文档上下文的操作（例如代码补全）将只作用于主插入符。当前，最最近创建或移动的插入符被视为主插入符。

### 编辑器操作

#### `EditorAction` 和 `EditorActionHandler`

在执行编辑器操作时，`EditorActionHandler` 会接收一个额外的参数：当前操作涉及的插入符实例（`caret`），如果没有特定的插入符上下文，则该参数为 `null`。通常，处理程序会将此参数传递给委派处理程序（如果有），以保持一致的上下文。

如果处理程序需要实现多插入符功能，可以在覆盖的 `doExecute` 方法中显式处理。对于只需要在每个插入符上调用 `doExecute` 方法的处理程序，可以在 `EditorActionHandler` 构造函数中传递一个参数，以确保在没有特定插入符上下文的情况下为每个插入符调用 `doExecute` 方法。

#### 编辑器操作委托

平台中提供了以下一些委托：

- `EnterHandlerDelegate`
- `BackspaceHandlerDelegate`
- `JoinLinesHandlerDelegate`
- `EditorNavigationDelegate`
- `SmartEnterProcessor`
- `CommentCompleteHandler`
- `StatementUpDownMover`
- `CodeBlockProvider`

这些处理程序已经支持多插入符，不需要额外的修改。

#### 打字操作

`TypedActionHandler` 和 `TypedHandlerDelegate` 处理打字事件。`TypedActionHandler` 和 `TypedHandlerDelegate` 的实现每次键入字符时只会被调用一次。如果需要支持多插入符，必须在实现中显式处理。

可以使用 `EditorModificationUtil.typeInStringAtCaretHonorMultipleCarets()` 方法来在所有插入符位置插入相同的文本或移动所有插入符。

#### 代码智能感知操作

继承自 `CodeInsightAction` 的现有操作仅适用于主插入符。要支持多插入符，可以继承 `MultiCaretCodeInsightAction`。在这种情况下，每个插入符可能有不同的编辑器和 PSI 实例，因此不能使用旧的 API。

以上是 IntelliJ 平台中多个插入符相关的功能和扩展点的概述。在开发插件或自定义编辑器行为时，理解这些概念和相关的 API 是非常重要的。





# 项目

## 项目结构

### 项目及其组件

在 IntelliJ 平台中，项目（Project）是组织源代码、库和构建指令的基本单元。所有使用 IntelliJ 平台 SDK 的操作都在项目的上下文中进行。项目可以包含多个模块和库，根据项目的逻辑和功能需求，您可以创建单模块或多模块项目。

#### 项目（Project）

项目是将所有源代码、库和构建指令打包成一个单一的组织单位。它定义了一组被称为模块（Modules）和库（Libraries）的集合。

#### 模块（Module）

模块是一个独立的功能单元，可以单独运行、测试和调试。模块包括源代码、构建脚本、单元测试、部署描述符等。在项目中，每个模块可以使用特定的 SDK，或者继承项目级别定义的 SDK。一个模块可以依赖项目的其他模块。

#### 库（Library）

库是模块依赖的已编译代码的归档文件（如 JAR 文件）。IntelliJ 平台支持三种类型的库：

- **模块库（Module Library）**：库类只在此模块中可见，库信息记录在模块的 `.iml` 文件中。
- **项目库（Project Library）**：库类在项目内可见，库信息记录在项目的 `.ipr` 文件或 `.idea/libraries` 中。
- **全局库（Global Library）**：库信息记录在 `~/.IntelliJIdea/config/options` 目录中的 `applicationLibraries.xml` 文件中。全局库类似于项目库，但在不同的项目中可见。

#### 软件开发工具包（SDK）

每个项目都使用一个软件开发工具包（SDK）。对于 Java 项目，SDK 通常称为 JDK（Java 开发工具包）。SDK 决定了用于构建项目的 API 库。如果项目是多模块的，默认情况下，项目 SDK 是所有模块共享的。也可以为每个模块配置单独的 SDK。

#### Facet

Facet 表示特定框架/技术相关的模块配置。一个模块可以有多个 Facet。例如，Spring 特定的配置存储在 Spring Facet 中。

以上是 IntelliJ 平台项目结构的简要介绍。了解这些概念对于有效地组织和管理项目非常重要，特别是在使用 IntelliJ IDEA 或其他基于 IntelliJ 平台的 IDE 时。

## 项目

在 IntelliJ 平台中，项目（Project）是管理源代码、库和构建指令的基本单位。项目的配置数据存储在 XML 文件中，具体文件列表取决于所选择的项目格式。

### 项目配置数据存储

- **文件格式项目**：旧版本项目的核心信息（如组件模块的位置、编译器设置等）存储在 `$project_name$.ipr` 文件中。有关项目包含的模块信息存储在 `$module_name$.iml` 文件中。每个模块创建一个模块文件。

- **目录格式项目**：项目和工作区设置存储在 `.idea` 目录下的多个 XML 文件中。每个 XML 文件负责其自身的一组设置，可以通过其名称识别，如 `projectCodeStyle.xml`、`encodings.xml`、`vcs.xml` 等。与文件格式项目类似，.iml 文件描述模块。

无需直接访问项目文件即可加载或保存设置，详情请参阅[组件状态的持久化](https://www.jetbrains.com/help/idea/persisting-state-of-components.html)。

### 项目实例的获取

在多个上下文中可以获得 Project 实例：

- **动作（Action）**：`AnActionEvent.getProject()` 或 `DataContext.getData(CommonDataKeys.PROJECT)`
- **编辑器**：`Editor.getProject()`
- **模块**：`Module.getProject()`
- **PSI**：`PsiElement.getProject()`
- **测试**：`IdeaProjectTestFixture.getProject()`

此外，还可以在通用上下文中检索项目：

- **从 VirtualFile 获取项目**：`ProjectLocator.guessProjectForFile(VirtualFile)` 返回包含指定文件的任何项目。`ProjectLocator.getProjectsForFile(VirtualFile)` 返回指定文件所属的项目列表。
- **当前打开的项目列表**：`ProjectManager.getOpenProjects()`

### 其他相关操作

#### 获取项目中所有模块的源根列表

使用 `ProjectRootManager.getContentSourceRoots()` 方法，例如：

```java
String projectName = project.getName();
VirtualFile[] vFiles = ProjectRootManager.getInstance(project).getContentSourceRoots();
String sourceRootsList = Arrays.stream(vFiles)
    .map(VirtualFile::getUrl)
    .collect(Collectors.joining("\n"));
Messages.showInfoMessage("Source roots for the " + projectName +
    " plugin:\n" + sourceRootsList, "Project Properties");
```

#### 检查文件是否属于项目

使用 `ProjectFileIndex` 获取此信息：

```java
ProjectFileIndex projectFileIndex = ProjectRootManager.getInstance(project).getFileIndex();
```

#### 获取文件或目录所属的内容或源根

使用 `ProjectFileIndex.getContentRootForFile()` 和 `ProjectFileIndex.getSourceRootForFile()` 方法。例如：

```java
VirtualFile moduleContentRoot = ProjectRootManager.getInstance(project)
    .getFileIndex().getContentRootForFile(virtualFileOrDirectory);
VirtualFile moduleSourceRoot = ProjectRootManager.getInstance(project)
    .getFileIndex().getSourceRootForFile(virtualFileOrDirectory);
```

#### 检查文件或目录是否与项目库相关

`ProjectFileIndex` 接口提供了一些方法来检查指定的文件是否属于项目库的类或库的源：

- `isLibraryClassFile()`：如果指定的 `virtualFile` 是已编译的类文件，返回 `true`。
- `isInLibraryClasses()`：如果指定的 `virtualFileOrDirectory` 属于库类，返回 `true`。
- `isInLibrarySource()`：如果指定的 `virtualFileOrDirectory` 属于库的源，返回 `true`。

#### 获取项目 SDK

项目的所有模块默认使用项目 SDK。也可以为每个模块配置单独的 SDK。

### 修改项目结构

修改项目结构的工具类可以在 `projectModel-impl.openapi` 包中找到。其根子包包含用于处理项目和模块源根的实例和工具，包括 `ModuleRootModificationUtil` 和 `ProjectRootUtil`。项目结构的更改需要在写操作中执行。

### 接收项目结构更改的通知

要接收有关项目结构更改的通知（如模块或库的添加或删除、模块依赖项的更改等），可以使用消息总线和 `ProjectTopics.PROJECT_ROOTS` 主题：

```java
project.getMessageBus().connect().subscribe(
    ProjectTopics.PROJECT_ROOTS,
    new ModuleRootListener() {
      @Override
      public void rootsChanged(@NotNull ModuleRootEvent event) {
        // action
      }
    });
```

对于 2019.3 或更高版本，可以使用声明式注册。

事件仅通知有变化发生；如果需要了解更详细的变化情况，可以保存相关项目结构模型的状态副本，并在变化后进行比较。

## 信任项目（Trusted Project）

在 IntelliJ 平台版本 2021.2.4/2021.3.1 及更高版本中引入了信任项目的概念。当用户首次在 IDE 中打开一个项目时，系统会询问用户是否信任该项目。如果用户选择在安全模式下预览项目，那么任何潜在危险的功能都不会自动或意外地执行。

插件可以通过 Kotlin 的扩展方法 `Project.isTrusted()` 或 Java 的静态方法 `TrustedProjects.isTrusted(Project)` 来检查项目是否被信任。

### 信任状态的改变

一个在安全模式下打开的项目可以在之后变为信任的状态：用户可以点击编辑器顶部通知面板中的“信任项目”链接，或者通过编程方式改变状态，例如，当用户调用一个危险的操作时，该操作会建议切换到信任模式。

因此，插件可以通过应用程序级别的监听器 `TrustStateListener` 订阅信任状态的变化，以启用在安全模式下被禁用的功能。实现 `TrustStateListener.onProjectTrusted()` 方法，或更好地，使用 `TrustedProjects.whenProjectTrusted()` 辅助方法，这些方法接受一个 lambda 表达式，一个用于 Kotlin，另一个用于 Java。

### 功能是否危险？

如果某个功能可能执行恶意代码，并且不明显会执行这些代码，则该功能必须在安全模式下被禁用，并且启用它必须通过确认保护。

#### 示例：

- 打开 IDE 中的文件夹时，可能执行 Gradle 构建脚本，而该脚本可能调用项目内部的恶意代码 => 在安全模式下禁用 Gradle 导入。
- 运行或调试源代码时显然可能执行恶意代码 => 不需要将此操作包装在确认之内。



## 项目向导 - 添加对创建新项目类型的支持

### 项目向导

IntelliJ 平台的项目向导可以通过 RedLine SmallTalk 插件来说明。更多信息请参见 [Project Wizard Tutorial](https://plugins.jetbrains.com/docs/intellij/project-wizard.html)。

### 实现新模块类型

要为特定工具和技术提供支持，通常需要实现某种模块类型，并将其附加到项目中。新的模块类型应从 `ModuleType` 类派生。

### 自定义项目向导

配置自定义项目向导的主要实用工具位于 `lang-api.ide.util.projectWizard` 包中。这些类和接口的主要用途包括：

- 修改配置向导视图
- 向向导添加新步骤
- 提供项目创建的附加设置
- 处理项目创建期间的活动
- 初始环境配置

### 模块类型

要创建新模块类型，请在 `plugin.xml` 中添加扩展点：

```xml
<moduleType
    id="MY_MODULE"
    implementationClass="st.redline.smalltalk.module.MyModuleType"/>
```

自定义模块类型应扩展 `ModuleType`，并使用 `ModuleBuilder` 进行泛型实现。以下示例展示了如何注册和实现自定义模块类型。

### 实现模块构建器

要设置新的模块环境，应扩展 `ModuleBuilder` 类并注册为扩展点，如下所示：

```xml
<extensions defaultExtensionNs="com.intellij">
  <moduleBuilder
      builderClass="org.jetbrains.plugins.ruby.rails.facet.versions.MyModuleBuilder"/>
</extensions>
```

必须实现的功能包括：

1. 通过覆盖 `setupRootModel(ModifiableRootModel modifiableRootModel) throws ConfigurationException` 方法设置新模块的根模型。
2. 获取模块类型：`public abstract ModuleType getModuleType();`

如果您的模块类型基于 Java 模块并打算支持 Java，仅需扩展 `JavaModuleBuilder` 即可，无需注册扩展点。可参考 SmallTalk 模块类型以了解如何派生 `JavaModuleBuilder`。

自 2022.1 版本起，基于 IntelliJ 的 IDE 使用了新的项目向导，如果模块构建器的 `isAvailable()` 方法返回 `false`，模块构建器将不会显示在向导中。

### 实现模块构建器监听器

模块构建器监听器在新模块创建时做出反应，这可能是项目创建过程的一部分，也可能是向现有项目添加新模块。要在模块创建后立即提供特定行为，模块构建器应实现 `ModuleBuilderListener.moduleCreated(Module)` 方法。

例如，在模块创建后立即执行的任务可能包括配置模块根目录、查找并设置 SDK、添加特定的 Facet 等。详情请参考 SmallTalk 自定义模块类型的实现。

### 添加新的向导步骤

通过覆盖 `AbstractModuleBuilder.createWizardSteps(WizardContext, ModulesProvider)` 方法可以向模块向导添加新步骤。如果此方法返回一个非空的 `ModuleWizardStep` 对象数组，则在创建新模块时会按照索引顺序显示新步骤。以下 SmallTalk 项目类型的实现展示了如何创建自定义向导步骤。`RsModuleWizardStep` 类派生自 `ModuleWizardStep`，它包含以下两个方法需要被覆盖：

1. `public JComponent getComponent();` - 定义步骤的外观
2. `public void updateDataModel();` - 将 UI 中的数据提交到 `ModuleBuilder` 和 `WizardContext`

### Facet

Facet 是 IntelliJ 平台中用于存储多种模块特定设置的方式，例如，使某种语言支持或框架在特定模块中可用。要了解更多 Facet 的用户视角信息，请参阅 Facet 文档部分。

### 实现项目结构检测器

要支持从现有源代码导入项目时创建模块，请扩展 `ProjectStructureDetector`。实现 `ProjectStructureDetector.detectRoots()` 方法以检测模块支持的文件。

参考 Smalltalk 项目结构检测器以了解示例实现。

但是，仅仅检测文件还不够，您还需要在适当的情况下为项目创建模块，可以通过实现 `setupProjectStructure()` 方法来完成。例如，如果项目结构中不存在其他模块，则创建一个模块：

```java
@Override
public void setupProjectStructure(@NotNull Collection<DetectedProjectRoot> roots,
                                  @NotNull ProjectDescriptor projectDescriptor,
                                  @NotNull ProjectFromSourcesBuilder builder) {
  List<ModuleDescriptor> modules = projectDescriptor.getModules();
  if (modules.isEmpty()) {
    modules = new ArrayList<>();
    for (DetectedProjectRoot root : roots) {
      modules.add(new ModuleDescriptor(root.getDirectory(),
          MyModuleType.getInstance(), ContainerUtil.emptyList()));
    }
    projectDescriptor.setModules(modules);
  }
}
```

## 支持模块类型

IntelliJ 平台提供了一组标准的模块类型。然而，有时应用程序可能需要一种尚未支持的模块类型。本教程展示了如何注册一个新的模块类型，并将其与项目创建过程和用户界面连接。

本教程中使用的代码示例来自 `module` 和 `project_wizard` 代码示例。

### 先决条件

创建一个空的插件项目。有关详细信息，请参见 [Creating a Plugin Gradle Project](https://plugins.jetbrains.com/docs/intellij/gradle-prerequisites.html) 部分。

项目向导中选择模块类型和创建模块的用户界面是特定于 IntelliJ IDEA 的。

### 注册新模块类型

在 `plugin.xml` 配置文件中添加一个新的 `com.intellij.moduleType` 实现。

```xml
<extensions defaultExtensionNs="com.intellij">
  <moduleType
      id="DEMO_MODULE_TYPE"
      implementationClass="org.intellij.sdk.module.DemoModuleType"/>
</extensions>
```

### 实现 ModuleType 接口

基于 `ModuleType` 创建 `DemoModuleType` 实现。

`getNodeIcon()` 方法应该返回模块类型特定的图标。

```java
final class DemoModuleType extends ModuleType<DemoModuleBuilder> {

  private static final String ID = "DEMO_MODULE_TYPE";

  DemoModuleType() {
    super(ID);
  }

  public static DemoModuleType getInstance() {
    return (DemoModuleType) ModuleTypeManager.getInstance().findByID(ID);
  }

  @NotNull
  @Override
  public DemoModuleBuilder createModuleBuilder() {
    return new DemoModuleBuilder();
  }

  @NotNull
  @Override
  public String getName() {
    return "SDK Module Type";
  }

  @NotNull
  @Override
  public String getDescription() {
    return "Example custom module type";
  }

  @NotNull
  @Override
  public Icon getNodeIcon(@Deprecated boolean b) {
    return SdkIcons.Sdk_default_icon;
  }

  @Override
  public ModuleWizardStep @NotNull [] createWizardSteps(@NotNull WizardContext wizardContext,
                                                        @NotNull DemoModuleBuilder moduleBuilder,
                                                        @NotNull ModulesProvider modulesProvider) {
    return super.createWizardSteps(wizardContext, moduleBuilder, modulesProvider);
  }
}
```

### 实现自定义模块构建器

基于 `ModuleBuilder` 创建 `DemoModuleBuilder`。

```java
public class DemoModuleBuilder extends ModuleBuilder {

  @Override
  public void setupRootModel(@NotNull ModifiableRootModel model) {
    // 实现设置模块根模型的逻辑
  }

  @Override
  public ModuleType<DemoModuleBuilder> getModuleType() {
    return DemoModuleType.getInstance();
  }

  @Nullable
  @Override
  public ModuleWizardStep getCustomOptionsStep(WizardContext context, Disposable parentDisposable) {
    return new DemoModuleWizardStep();
  }

}
```

### 提供自定义向导步骤

提供一个简单的用户界面组件实现，用于项目创建阶段。基于 `ModuleWizardStep` 创建一个通用的 `DemoModuleWizardStep`。

```java
public class DemoModuleWizardStep extends ModuleWizardStep {

  @Override
  public JComponent getComponent() {
    return new JLabel("Provide some setting here");
  }

  @Override
  public void updateDataModel() {
    // 根据 UI 更新模型
  }

}
```

通过以上步骤，您可以创建和注册一个新的模块类型，并将其与 IntelliJ 平台的项目向导集成。



## 框架

本教程展示了如何为项目支持自定义框架类型，并使该框架类型作为UI组件嵌入到项目向导中。教程中的示例主要依赖于 `framework_basics` 代码示例。

请注意，此功能需要依赖于 Java 插件。

### 创建新框架

要使自定义框架可用并可配置，需要扩展 `FrameworkTypeEx` 类。在本例中，创建 `DemoFramework` 类：

```java
final class DemoFramework extends FrameworkTypeEx {
}
```

### 注册框架

新创建的框架类应作为扩展点在 `plugin.xml` 配置文件中注册，添加 `com.intellij.framework.type` 扩展：

```xml
<extensions defaultExtensionNs="com.intellij">
  <framework.type
      implementation="org.intellij.sdk.framework.DemoFramework"/>
</extensions>
```

### 设置必要的属性

框架组件应具有唯一的名称，作为字符串字面量传递给构造函数。最好使用类的完全限定名（FQN）：

```java
final class DemoFramework extends FrameworkTypeEx {

  public static final String FRAMEWORK_ID =
      "org.intellij.sdk.framework.DemoFramework";

  DemoFramework() {
    super(FRAMEWORK_ID);
  }
}
```

`PresentableName` 和 `Icon` 定义了与框架相关的视觉组件的外观：

```java
final class DemoFramework extends FrameworkTypeEx {
  @NotNull
  @Override
  public String getPresentableName() {
    return "SDK Demo Framework";
  }

  @NotNull
  @Override
  public Icon getIcon() {
    return SdkIcons.Sdk_default_icon;
  }
}
```

### 创建启用框架支持的提供者

为了使框架在创建项目的过程中可用，必须实现 `DemoFramework.createProvider()` 方法，返回 `FrameworkSupportInModuleConfigurable` 类型的对象，该对象将框架添加到模块中。在此示例中，框架被添加到任何 `ModuleType`，通常情况下不检查，这通常不是标准的做法。

```java
@NotNull
@Override
public FrameworkSupportInModuleProvider createProvider() {
  return new FrameworkSupportInModuleProvider() {
    @NotNull
    @Override
    public FrameworkTypeEx getFrameworkType() {
      return DemoFramework.this;
    }

    @NotNull
    @Override
    public FrameworkSupportInModuleConfigurable createConfigurable(
        @NotNull FrameworkSupportModel model) {
      return new FrameworkSupportInModuleConfigurable() {

        @Override
        public JComponent createComponent() {
          return new JCheckBox("SDK Extra Option");
        }

        @Override
        public void addSupport(@NotNull Module module,
                               @NotNull ModifiableRootModel model,
                               @NotNull ModifiableModelsProvider provider) {
          // 在这里设置库，生成特定文件，
          // 并实际向模块添加框架支持。
        }
      };
    }

    @Override
    public boolean isEnabledForModuleType(@NotNull ModuleType type) {
      return true;
    }
  };
}
```

这个实现为项目向导添加了一个新的框架选项，使开发者能够在创建新项目时选择支持该框架。它展示了如何通过实现 `FrameworkSupportInModuleProvider` 接口的 `createProvider` 方法，提供框架的设置界面以及相应的框架支持配置。

## 模块（Module）

在 IntelliJ 平台中，模块是一个可以独立运行、测试和调试的功能单元。模块包含源代码、构建脚本、单元测试、部署描述符等。

### 模块的关键组件

1. **内容根目录（Content Roots）**：
   - 这些是存储模块相关文件（如源代码、资源等）的目录。每个目录只能属于一个模块，无法在多个模块之间共享内容根目录。

2. **源根目录（Source Roots）**：
   - 内容根目录下可以有多个源根目录。源根目录可以有不同的类型，如常规源根目录、测试源根目录、资源根目录等。在 IntelliJ IDEA 中，源根目录被用作包层次结构的根目录。直接位于源根目录下的 Java 类将位于根包中。源根目录还可以用于实现更细粒度的依赖检查，例如常规源根目录下的代码不能依赖测试源根目录下的代码。

3. **顺序条目（Order Entries）**：
   - 模块的依赖项，它们以有序列表形式存储。依赖项可以是 SDK、库或其他模块的引用。

4. **面板（Facets）**：
   - 特定框架的配置条目列表。

此外，模块还可以存储其他设置，例如模块特定的 SDK、编译输出路径设置等。插件可以通过创建面板或模块级组件来存储与模块相关的其他数据。

### IntelliJ 平台提供的类和接口

- `Module`
- `ModuleUtil` 和 `ModuleUtilCore`
- `ModuleManager`
- `ModuleRootManager`
- `ModuleRootModel`
- `ModifiableModuleModel`
- `ModifiableRootModel`

### 常见任务

#### 获取项目中包含的模块列表
使用 `ModuleManager.getModules()` 方法。

#### 获取模块的依赖项和类路径
使用 `OrderEnumerator` 类。以下代码片段展示了如何获取模块的类路径（所有依赖项的类根）：

```java
VirtualFile[] roots = ModuleRootManager.getInstance(module).orderEntries().classes().getRoots();
```

#### 获取模块使用的 SDK
使用 `ModuleRootManager.getSdk()` 方法。以下代码片段展示了如何获取指定模块使用的 SDK 的详细信息：

```java
ModuleRootManager moduleRootManager = ModuleRootManager.getInstance(module);
Sdk sdk = moduleRootManager.getSdk();
String jdkInfo = "Module: " + module.getName() +
    " SDK: " + sdk.getName() +
    " SDK version: " + sdk.getVersionString() +
    " SDK home directory: " + sdk.getHomePath();
```

#### 获取模块直接依赖的模块列表
使用 `ModuleRootManager.getDependencies()` 方法获取 `Module` 类型值的数组或 `ModuleRootManager.getDependencyModuleNames()` 获取模块名称的数组。示例代码：

```java
ModuleRootManager moduleRootManager = ModuleRootManager.getInstance(module);
Module[] dependentModules = moduleRootManager.getDependencies();
String[] dependentModulesNames = moduleRootManager.getDependencyModuleNames();
```

#### 获取依赖于此模块的模块列表
使用 `ModuleManager.getModuleDependentModules(module)` 方法。

还可以使用 `ModuleManager.isModuleDependent()` 方法检查一个模块是否依赖于另一个模块：

```java
boolean isDependent = ModuleManager.getInstance(project).isModuleDependent(module1, module2);
```

#### 获取指定文件或 PSI 元素所属的模块
使用静态方法 `ModuleUtil.findModuleForFile()` 获取文件所属的项目模块：

```java
String pathToFile = "/path/to/your/file";
VirtualFile virtualFile = LocalFileSystem.getInstance().findFileByPath(pathToFile);
Module module = ModuleUtil.findModuleForFile(virtualFile, myProject);
String moduleName = module == null ? "Module not found" : module.getName();
```

使用 `ModuleUtil.findModuleForPsiElement()` 获取 PSI 元素所属的项目模块。

#### 存储模块的引用
使用 `ModulePointer` 存储模块的实例或名称引用。模块的删除或重命名将自动跟踪。

#### 访问模块根目录
通过 `ModuleRootManager` 访问模块根目录的信息。示例代码：

```java
VirtualFile[] contentRoots = ModuleRootManager.getInstance(module).getContentRoots();
```

#### 检查文件或目录是否属于模块源根目录
使用 `ProjectFileIndex.getSourceRootForFile()` 方法检查：

```java
VirtualFile moduleSourceRoot = ProjectRootManager.getInstance(project).getFileIndex().getSourceRootForFile(virtualFileOrDirectory);
```

#### Java 编译输出属性
获取 `CompilerModuleExtension` 以访问编译输出路径相关属性。

#### 接收关于模块变化的通知
使用消息总线和 `ProjectTopics.MODULES` 主题：

```java
project.getMessageBus().connect().subscribe(
    ProjectTopics.MODULES,
    new ModuleListener() {
      @Override
      public void moduleAdded(@NotNull Project project, @NotNull Module module) {
        // action
      }
    });
```

2021.3或更高版本还可以使用声明式注册。

## 库（Library）

在 IntelliJ 平台中，库是模块依赖的已编译代码的存档（例如 JAR 文件）。

### IntelliJ 平台支持的三种库类型

1. **模块库（Module Library）**：
   - 该模块库中的类仅对该模块可见，库信息记录在模块的 `.iml` 文件中。

2. **项目库（Project Library）**：
   - 该项目库中的类在整个项目中可见，库信息记录在 `.idea/libraries` 目录或项目的 `.ipr` 文件中。

3. **全局库（Global Library）**：
   - 库信息记录在 `$USER_HOME$/.IntelliJIdea/config/options` 目录中的 `applicationLibraries.xml` 文件中。全局库类似于项目库，但在不同的项目之间可见。

另一类程序化定义的库是预定义库（Predefined Libraries）。

### 访问库和 JAR 文件

`com.intellij.openapi.roots.libraries` 包提供了处理项目库和 JAR 文件的功能。

#### 获取模块依赖的库列表

使用 `OrderEnumerator.forEachLibrary` 方法，如下所示：

```java
List<String> libraryNames = new ArrayList<>();
ModuleRootManager.getInstance(module).orderEntries().forEachLibrary(library -> {
  libraryNames.add(library.getName());
  return true;
});
Messages.showInfoMessage(StringUtil.join(libraryNames, "\n"), "Libraries in Module");
```

该示例代码输出给定模块依赖的库列表。

#### 获取所有库的列表

要管理应用程序级和项目级库的列表，请使用 `LibraryTable`。应用程序级库表的列表可以通过调用 `LibraryTablesRegistrar.getLibraryTable()` 访问，而项目级库表的列表可以通过 `LibraryTablesRegistrar.getLibraryTable(Project)` 访问。一旦你有了 `LibraryTable`，你可以通过调用 `LibraryTable.getLibraries()` 获取其中的库。

要获取给定模块中定义的所有模块库的列表，请使用 `OrderEntryUtil` 中的 API：

```java
OrderEntryUtil.getModuleLibraries(ModuleRootManager.getInstance(module));
```

#### 获取库内容

`Library` 提供了 `getUrls()` 方法，可以用来获取库中包含的源根和类的列表。以下代码片段展示了如何获取这些信息：

```java
StringBuilder roots = new StringBuilder("The " + lib.getName() + " library includes:\n");
roots.append("Sources:\n");
for (String each : lib.getUrls(OrderRootType.SOURCES)) {
  roots.append(each).append("\n");
}
roots.append("Classes:\n");
for (String each : lib.getUrls(OrderRootType.CLASSES)) {
  roots.append(each).append("\n");
}
Messages.showInfoMessage(roots.toString(), "Library Info");
```

#### 创建一个库

要创建一个库，请执行以下步骤：

1. 获取一个写操作。
2. 获取要添加库的库表。根据库的级别，使用以下方法之一：
   - `LibraryTablesRegistrar.getInstance().getLibraryTable()`
   - `LibraryTablesRegistrar.getInstance().getLibraryTable(Project)`
   - `ModuleRootManager.getInstance(module).getModifiableModel().getModuleLibraryTable()`
3. 通过调用 `LibraryTable.createLibrary()` 创建库。
4. 添加库内容（见下文）。
5. 对于模块级库，提交由 `ModuleRootManager.getInstance(module).getModifiableModel()` 返回的可修改模型。

对于模块级库，你还可以使用 `ModuleRootModificationUtil` 类中的简化 API 来通过单一 API 调用添加库。你可以在 `project_model` 代码示例中找到使用这些 API 的例子。

#### 添加内容或修改库

要添加或更改库的根目录，请执行以下步骤：

1. 获取一个写操作。
2. 获取库的可修改模型，使用 `Library.getModifiableModel()`。
3. 使用诸如 `Library.ModifiableModel.addRoot()` 之类的方法进行必要的更改。
4. 使用 `Library.ModifiableModel.commit()` 提交模型。

#### 向模块添加库依赖

使用 `ModuleRootModificationUtil.addDependency(Module, Library)` 在写操作下添加库依赖。

#### 检查文件是否属于库

`ProjectFileIndex` 接口实现了一些方法，可以用来检查指定的文件是否属于项目库类或库源。你可以使用以下方法：

- 检查指定的虚拟文件是否为已编译的类文件：

  ```java
  ProjectFileIndex.isLibraryClassFile(virtualFile)
  ```

- 检查指定的虚拟文件或目录是否属于库类：

  ```java
  ProjectFileIndex.isInLibraryClasses(virtualFileOrDirectory)
  ```

- 检查指定的虚拟文件或目录是否属于库源：

  ```java
  ProjectFileIndex.isInLibrarySource(virtualFileOrDirectory)
  ```

有关库的更多详细信息，请参阅 `plugin_model` 代码示例。

#### 预定义库

扩展点：`com.intellij.additionalLibraryRootsProvider`

`AdditionalLibraryRootsProvider` 允许在不将它们暴露在模型中的情况下在项目中提供合成/预定义库（`SyntheticLibrary`）。默认情况下，它们也在 UI 中隐藏。



好的，我明白了。接下来我会根据您提供的内容，将其详细翻译成中文，并在必要时添加解释。以下是关于 Facet 的翻译和解释：

---

## Facet（框架配置）

**Facet** 是与模块相关的特定框架或技术的配置表示。每个模块可以有多个 Facet，每个 Facet 对应不同的框架或技术。例如，Spring 框架的特定配置就存储在一个 Spring Facet 中。

### Facet 基本示例

要了解如何使用 Facet，可以参考 Facet Basics 示例插件项目。该项目提供了一个实现和使用 Facet 的示例，展示了如何在 IntelliJ 平台中管理这些配置。

### 使用 Facet

#### 管理 Facet

使用 `FacetManager` 类来管理 Facet。这个类提供了创建、搜索和访问与模块相关的 Facet 列表的方法。Facet 管理包括添加、移除或修改模块中的 Facet 配置。

例如，可以使用 `FacetManager` 来查找某个模块中是否存在特定的 Facet，或者添加新的 Facet 来扩展模块的功能。

#### 基于 Facet 的工具窗口

可以通过 `com.intellij.facet.toolWindow` 扩展点注册一个依赖于特定 Facet 的工具窗口。这允许集成专门的工具窗口，为特定 Facet 提供额外的功能或信息。

例如，如果一个模块使用了 Spring 框架，你可以添加一个专门的工具窗口来显示与 Spring 配置相关的信息或工具。

以上是 Facet 的基本概念和使用方法。有关更详细的实现细节，请参考 IntelliJ 平台 SDK 文档和 Facet Basics 示例插件项目。

## 外部系统集成
最后修改日期：2024年7月31日

本页面概述了外部系统子系统的高层次概念。存在多种项目管理系统（如Apache Maven、Gradle、sbt等），IntelliJ平台提供了一个机制来支持这些系统在IDE中的集成。

从集成的角度来看，大多数项目管理系统提供了一套类似的功能：
- 从外部系统配置（如pom.xml、build.gradle.kts等）构建项目
- 提供可用任务的列表
- 允许执行特定任务
- 以及更多

这意味着我们可以将外部系统特定的逻辑与通用的IDE处理分开。外部系统子系统提供了一个简单的API，用于包装外部系统元素并扩展IDE特定的处理逻辑。

### 项目管理
#### 项目数据领域
外部系统包装器需要能够基于给定的外部系统配置构建项目信息。该信息通过以下基本类构建：
- `DataNode`
- `Key`
- `ExternalEntityData`

`DataNode` 类只是目标数据的持有者（数据类型由 `Key` 定义）。多个 `DataNode` 对象可以组织成有向图，每条边表示父子关系。

例如，一个简单的单模块项目可能如下所示：
- `DataNode<ProjectData>`
  - `DataNode<ModuleData>`
    - `DataNode<LibraryData>` (如 JUnit)
    - `DataNode<ContentRootData>`
    - `DataNode<LibraryDependencyData>` (如 JUnit)

IDE 提供了一组内置的 `Key` 和 `ExternalEntityData` 类，但任何外部系统集成或第三方插件开发者都可以通过定义自定义的 `Key` 和 `ExternalEntityData` 来增强项目数据，并将其存储在合适的 `DataNode` 的子节点中。

#### 管理项目数据
基于外部系统配置构建的项目数据处理可以通过 `ProjectDataService` 完成。这是一种知道如何管理特定 `ExternalEntityData` 的策略。例如，当我们想从外部模型导入一个项目时，我们可以从引用项目信息的顶级 `DataNode` 开始，然后使用相应的服务导入其数据。

自定义服务可以通过 `com.intellij.externalProjectDataService` 扩展点注册。

值得注意的是，我们可以在这里将项目解析和管理分开。这意味着特定技术的一组 `DataNode`、`Key` 和 `ProjectDataServices` 可以引入，然后每个外部系统集成可以在必要时使用它构建相应的数据。

#### 从外部模型导入
IntelliJ 平台提供了从外部模型导入项目的 API：
- `ProjectImportBuilder`
- `ProjectImportProvider`

有两个基于模板方法模式的类简化了实现：
- `AbstractExternalProjectImportBuilder`
- `AbstractExternalProjectImportProvider`

请注意，`AbstractExternalProjectImportBuilder` 基于“外部系统设置”控件之上。

具体的实现应分别在 `com.intellij.projectImportBuilder` 和 `com.intellij.projectImportProvider` 扩展点中注册。

例如，对于 Gradle 的项目导入提供者和构建器：
- `JavaGradleProjectImportProvider`
- `JavaGradleProjectImportBuilder`

#### 自动导入
可以配置外部系统集成，以便在外部项目的配置文件修改时自动刷新项目结构。

从 2020.1 版本开始，用户无法禁用自动导入功能。

##### 外部系统管理器的自动导入实现
通过让外部系统的 `ExternalSystemManager` 实现 `ExternalSystemAutoImportAware` 来描述要跟踪的项目的设置文件。

`ExternalSystemAutoImportAware.getAffectedExternalProjectPath()` 方法被频繁调用，因此应尽可能快地返回控制。可以使用辅助类 `CachingExternalSystemAutoImportAware` 进行缓存，即 `ExternalSystemManager` 实现 `ExternalSystemAutoImportAware` 可以拥有一个字段 `new CachingExternalSystemAutoImportAware(new MyExternalSystemAutoImportAware())` 并将 `ExternalSystemAutoImportAware.getAffectedExternalProjectPath()` 调用委托给它。

##### 独立外部系统的自动导入
某些外部系统没有 `ExternalSystemManager`（例如 Maven），但它们也可以使用自动导入核心来跟踪设置文件的更改。为此，实现 `ExternalSystemProjectAware` 接口，该接口描述了要跟踪的设置文件和重新加载项目模型的操作。然后，将实例注册到 `ExternalSystemProjectTracker` 以开始跟踪。

多个 `ExternalSystemProjectAware` 实例可以对应于单个外部系统。这允许根据设置文件集（每个设置文件、每个模块、每个外部项目等）不同地执行项目重新加载。

#### 2020.1+ 重载通知图标
可以为每个外部系统指定重载通知的图标。实现 `ExternalSystemIconProvider` 并通过 `plugin.xml` 中的 `com.intellij.externalIconProvider` 扩展点进行注册。或者，直接将实现 `ExternalSystemIconProvider` 的外部系统设置 `reloadIcon` 字段。

### 设置
所有外部系统设置控件均由 `ExternalSystemSettingsControl` 实现表示。特定的外部系统设置UI包含以下内容：
- 系统常规设置
- 已链接的外部项目列表
- 所选项目的项目级设置

推荐扩展 `AbstractExternalProjectSettingsControl` 来实现项目级设置控件，因为它已经处理了一些设置。

示例：
- `GradleSystemSettingsControl` 处理 `Settings | Build, Execution, Deployment | Build Tools | Gradle` 中的常规设置
- `GradleProjectSettingsControl` 处理 `Settings | Build, Execution, Deployment | Build Tools | Gradle` 中所选 Gradle 项目的设置

类似的方法用于在导入外部项目的UI中提供设置。实现应扩展 `AbstractImportFromExternalSystemControl`，并且它包含目标外部项目路径控件，而不是已链接的外部项目列表。

---

这是对您提供的外部系统集成的详细翻译和解释。如有任何进一步的要求或问题，请随时告知！



# PSI

## 程序结构接口 (PSI)
最后修改日期：2024年7月31日

程序结构接口，通常简称为PSI，是IntelliJ平台中的一层，负责解析文件并创建语法和语义代码模型，该模型为平台的许多功能提供支持。

### PSI文件
PSI文件是一个抽象的代码结构表示，它提供了代码的语法和语义信息。这种表示形式使得各种IDE功能（如代码分析、重构、代码补全等）得以实现。每个PSI文件对应于一个虚拟文件（`VirtualFile`），并且可以提供关于文件内容的详细结构信息。

### 文件视图提供者
文件视图提供者（File View Providers）是PSI系统的一个组件，它管理文件的不同语言视图。换句话说，如果一个文件包含多种语言的代码（例如HTML文件中包含JavaScript代码），文件视图提供者能够创建每种语言的PSI树。它通过使用语言插件中的语言定义来解析文件的内容，并为每种语言创建相应的PSI树。

### PSI元素
PSI元素（PSI Elements）是PSI树的基本构建块。它们表示代码中的最小语法单元，如变量、函数、类、表达式等。每个PSI元素都有自己的类型，标识它在代码中的角色。通过PSI元素，可以查询和修改代码的结构和内容。PSI元素还支持导航、重构等操作，是实现这些IDE功能的核心部分。

### 总结
PSI是IntelliJ平台中一个关键的组件，它负责解析代码文件并创建用于表示代码结构的语法树。通过PSI，开发者可以对代码进行深入的分析、导航和修改，这使得PSI成为IDE功能实现的基础。

如果您有任何特定的功能或进一步的细节需要解释，请告诉我！



## PSI文件
最后修改日期：2024年7月31日

PSI（程序结构接口）文件是表示文件内容的结构化层次的根节点，这个结构是根据特定编程语言来构建的。

### PSI文件类
`PsiFile`类是所有PSI文件的公共基类，而特定语言的文件通常由其子类表示。例如，`PsiJavaFile`类表示Java文件，而`XmlFile`类表示XML文件。

与虚拟文件（Virtual Files）和文档（Documents）不同，这些具有应用范围（即使多个项目打开，每个文件也由相同的VirtualFile实例表示），PSI具有项目范围：如果文件属于同时打开的多个项目，则该文件由多个`PsiFile`实例表示。

### 如何获取PSI文件？
根据不同的上下文，可以使用以下API获取PSI文件：

| 上下文       | API                                                          | 操作                                                         |
| ------------ | ------------------------------------------------------------ | ------------------------------------------------------------ |
| Action       | `AnActionEvent.getData(CommonDataKeys.PSI_FILE)`             | 获取当前上下文中的PSI文件                                    |
| Document     | `PsiDocumentManager.getPsiFile()`                            | 从文档获取对应的PSI文件                                      |
| PSI Element  | `PsiElement.getContainingFile()`                             | 获取包含该PSI元素的PSI文件（如果PSI元素不包含在文件中，则可能返回null） |
| Virtual File | `PsiManager.findFile()` 或 `PsiUtilCore.toPsiFiles()`        | 从虚拟文件获取PSI文件                                        |
| File Name    | `FilenameIndex.getVirtualFilesByName()` 然后通过`PsiManager.findFile()`或`PsiUtilCore.toPsiFiles()`定位 | 通过文件名获取PSI文件                                        |

### 可以对PSI文件做什么？
大多数有趣的修改操作是在单个PSI元素的级别上执行的，而不是在整个文件上。

要遍历文件中的元素，可以使用以下代码：

```java
psiFile.accept(new PsiRecursiveElementWalkingVisitor() {
  // 实现访问者 ...
});
```

参见**Navigating the PSI**了解更多。

### PSI文件的来源？
由于PSI是依赖于语言的，PSI文件是使用语言实例创建的：

```java
LanguageParserDefinitions.INSTANCE
    .forLanguage(MyLanguage.INSTANCE)
    .createFile(fileViewProvider);
```

类似于文档，PSI文件是在访问特定文件的PSI时按需创建的。

### PSI文件的生命周期？
与文档类似，PSI文件是从相应的`VirtualFile`实例中弱引用的，如果没有人引用它们，它们可能会被垃圾回收。

### 如何创建PSI文件？
使用`PsiFileFactory.createFileFromText()`可以创建具有指定内容的内存中的PSI文件。

要将PSI文件保存到磁盘上，可以使用其父目录的`PsiDirectory.add()`方法。

### 如何在PSI文件更改时收到通知？
`PsiManager.addPsiTreeChangeListener()`允许您接收项目中PSI树的所有更改的通知。或者，可以在`com.intellij.psi.treeChangeListener`扩展点中注册`PsiTreeChangeListener`。

请参阅`PsiTreeChangeEvent`的Javadoc，了解处理PSI事件时常见的问题。

### 如何扩展PSI？
可以通过自定义语言插件扩展PSI，以支持其他语言。有关开发自定义语言插件的更多详细信息，请参阅**Custom Language Support**参考指南。

### 操作PSI的规则是什么？
对PSI文件内容的任何更改都会反映在文档中，因此所有处理文档的规则（读/写操作、命令、只读状态处理）都有效。

这就是PSI文件的详细介绍。如果您有其他问题或需要更深入的解释，请告诉我！

## 文件视图提供者（File View Providers）
最后修改日期：2024年7月31日

文件视图提供者（FileViewProvider）管理对单个文件中多个PSI树的访问。例如，一个JSPX页面有一个独立的PSI树用于其中的Java代码（PsiJavaFile）、一个用于XML代码的树（XmlFile）以及一个用于整个JSP的树（JspFile）。

每个PSI树覆盖文件的全部内容，并在找到不同语言内容的地方包含特殊的“外部语言元素”。

一个FileViewProvider实例对应一个VirtualFile（虚拟文件）和一个Document（文档），并可以检索多个PsiFile实例。

### 如何获取FileViewProvider？
根据不同的上下文，可以使用以下API获取FileViewProvider：

| 上下文   | API                                                          |
| -------- | ------------------------------------------------------------ |
| PSI文件  | `PsiFile.getViewProvider()`                                  |
| 虚拟文件 | `PsiManager.getInstance(project).findViewProvider(virtualFile)` |

### 可以对FileViewProvider做什么？
- **获取文件中存在的所有语言的集合：** 使用`fileViewProvider.getLanguages()`方法。
- **获取特定语言的PSI树：** 使用`fileViewProvider.getPsi(language)`方法。例如，要获取XML的PSI树，可以使用`fileViewProvider.getPsi(XMLLanguage.INSTANCE)`。
- **在文件中指定偏移量处找到特定语言的元素：** 使用`fileViewProvider.findElementAt(offset, language)`方法。

### 如何扩展FileViewProvider？
要创建一种包含不同语言的多种交错树的文件类型，插件必须包含对`com.intellij.fileType.fileViewProviderFactory`扩展点的扩展。

实现`FileViewProviderFactory`接口，并在`createFileViewProvider()`方法中返回自定义的FileViewProvider实现。

在`plugin.xml`中注册，如下所示：

```xml
<extensions defaultExtensionNs="com.intellij">
  <fileType.fileViewProviderFactory
      filetype="$FILE_TYPE$"
      implementationClass="com.example.MyFileViewProviderFactory"/>
</extensions>
```

其中`$FILE_TYPE$`指的是正在创建的文件类型（例如，“JSF”）。

这个扩展点允许为特定文件类型创建自定义的视图提供者，以便处理多语言内容或其他复杂的文件内容解析需求。如果您有其他问题或需要进一步的说明，请告诉我！

## PSI 元素
最后修改日期：2024年7月31日

PSI（程序结构接口）文件代表了PSI元素的层次结构（称为PSI树）。单个PSI文件（本身也是一个PSI元素）可能会在特定的编程语言中暴露多个PSI树（参见文件视图提供者）。PSI元素可以有子PSI元素。

PSI元素及其相关操作用于探索源代码的内部结构，这些结构由IntelliJ平台解释。例如，可以使用PSI元素执行代码分析，如代码检查或意图操作。

`PsiElement`类是PSI元素的公共基类。

### 如何获取PSI元素？
根据不同的上下文，可以使用以下API获取PSI元素：

| 上下文                            | API                                                          |
| --------------------------------- | ------------------------------------------------------------ |
| 操作                              | `AnActionEvent.getData(CommonDataKeys.PSI_ELEMENT)`          |
|                                   | **注意**：如果当前打开的编辑器中光标下的元素是引用，这将返回解析该引用的结果。 |
| PSI文件                           | `PsiFile.findElementAt(offset)` - 这将返回指定偏移处的叶子元素，通常是词法分析器的标记。使用`PsiTreeUtil.getParentOfType()`找到精确类型的元素。 |
| PsiRecursiveElementWalkingVisitor |                                                              |
| 引用                              | `PsiReference.resolve()`                                     |

### 可以对PSI元素做什么？
可以参考PSI Cookbook（PSI教程）和修改PSI来了解更多关于PSI元素的操作及其用法。这些资源提供了详细的示例和指南，帮助开发者更好地理解如何利用PSI元素进行代码分析、检查、修改等操作。

如有更多问题或需要进一步的解释，请告知我！

## PSI 导航
最后修改日期：2024年7月31日

在IntelliJ平台的PSI（程序结构接口）中，有三种主要的导航方式：自上而下、自下而上和通过引用。以下是每种导航方式的详细说明：

### 自上而下导航
自上而下导航是指从一个PSI文件或更高级别的元素（例如，一个方法）开始，查找符合特定条件的所有元素（例如，所有的变量声明）。这种导航方式最常见的实现方法是使用Visitor（访问者模式）。

要使用访问者模式，你需要创建一个类（通常是匿名内部类），扩展基础访问者类，重写处理你感兴趣的元素的方法，并将访问者实例传递给`PsiElement.accept()`。

访问者的基础类是语言特定的。例如，如果你需要处理Java文件中的元素，你可以扩展`JavaRecursiveElementVisitor`并重写对应Java元素类型的方法。

以下代码片段展示了如何使用访问者查找所有Java局部变量声明：

```java
file.accept(new JavaRecursiveElementVisitor() {
  @Override
  public void visitLocalVariable(@NotNull PsiLocalVariable variable) {
    super.visitLocalVariable(variable);
    System.out.println("Found a variable at offset " +
         variable.getTextRange().getStartOffset());
  }
});
```

在许多情况下，你也可以使用更具体的API进行自上而下的导航。例如，如果你需要获取Java类中的所有方法列表，可以使用访问者，但更简单的方法是调用`PsiClass.getMethods()`。

`PsiTreeUtil`包含了一些通用的、与语言无关的PSI树导航函数，其中一些（例如`findChildrenOfType()`）执行自上而下的导航。

### 自下而上导航
自下而上导航的起点通常是PSI树中的一个特定元素（例如，解析引用的结果）或一个偏移量。如果你有一个偏移量，可以通过调用`PsiFile.findElementAt()`找到对应的PSI元素。该方法返回树中最低级别的元素（例如，一个标识符），如果你想确定更广泛的上下文，需要向上导航树结构。

在大多数情况下，自下而上导航是通过调用`PsiTreeUtil.getParentOfType()`来实现的。该方法会向上遍历树，直到找到你指定类型的元素。例如，要找到包含的方法，可以调用`PsiTreeUtil.getParentOfType(element, PsiMethod.class)`。

在某些情况下，你还可以使用特定的导航方法。例如，要找到包含某个方法的类，可以调用`PsiMethod.getContainingClass()`。

以下代码片段展示了如何结合这些调用来实现导航：

```java
PsiFile psiFile = anActionEvent.getData(CommonDataKeys.PSI_FILE);
PsiElement element = psiFile.findElementAt(offset);
PsiMethod containingMethod = PsiTreeUtil.getParentOfType(element, PsiMethod.class);
PsiClass containingClass = containingMethod.getContainingClass();
```

### 引用导航
引用导航允许你从元素的用法（例如，方法调用）导航到声明（被调用的方法）和返回。引用导航在单独的主题中有更详细的描述。

要了解导航如何在实践中工作，请参考代码示例。

## PSI 引用
最后修改日期：2024年7月31日

PSI 树中的引用是一个对象，它表示从代码中特定元素的使用到相应声明的链接。解析引用意味着定位到一个特定使用所指向的声明。

最常见的引用类型是由语言语义定义的。例如，考虑一个简单的Java方法：

```java
public void hello(String message) {
  System.out.println(message);
}
```

这个简单的代码片段包含了五个引用。由标识符`String`、`System`、`out`和`println`创建的引用可以解析到JDK中的相应声明：`String`和`System`类，`out`字段和`println`方法。而由`println(message)`中的第二个`message`标识符创建的引用则可以解析到方法头中的`String message`参数声明。

需要注意的是，`String message`本身不是一个引用，因此无法被解析。它是一个声明，它定义了一个名称，而不是引用其他地方定义的名称。

一个引用是实现`PsiReference`接口的类的实例。需要注意的是，引用与PSI元素是不同的。由PSI元素创建的引用可以通过`PsiElement.getReferences()`返回，而引用的底层PSI元素可以通过`PsiReference.getElement()`获得。

要解析引用（定位到被引用的声明），可以调用`PsiReference.resolve()`。理解`PsiReference.getElement()`和`PsiReference.resolve()`之间的区别非常重要。前者方法返回引用的来源，而后者返回其目标。在上面的示例中，对于`message`引用，`getElement()`将返回代码片段第二行的`message`标识符，而`resolve()`将返回第一行（参数列表中的）`message`标识符。

解析引用的过程与解析和生成PSI树的过程不同，且不是在同一时间进行的。此外，解析并不总是成功的。如果当前打开的代码在IDE中无法编译，或者在其他情况下，`PsiReference.resolve()`返回null是正常的——所有与引用相关的代码都必须准备好处理这种情况。

请参阅《缓存繁重计算结果》。

### 贡献的引用
除了由编程语言语义定义的引用外，IDE还识别许多由API和框架语义确定的引用。例如，考虑以下示例：

```java
File file = new File("foo.txt");
```

从Java语法的角度来看，"foo.txt"没有特殊意义——它只是一个字符串文字。然而，在IntelliJ IDEA中打开此示例并且在相同目录中有一个名为"foo.txt"的文件时，可以通过Ctrl/Cmd+点击"foo.txt"来导航到该文件。这是因为IDE识别了`new File(...)`的语义，并为传递给方法的字符串文字贡献了一个引用。

通常，可以为没有自身引用的元素（如字符串文字和注释）贡献引用。引用也经常被贡献给非代码文件，例如XML或JSON。

贡献引用是扩展现有语言的最常见方式之一。例如，您的插件可以为Java代码贡献引用，即使Java PSI是平台的一部分并且不是由您的插件定义的。

实现`PsiReferenceContributor`并在`com.intellij.psi.referenceContributor`扩展点中注册。

属性`language`应该设置为此贡献者适用的语言ID。然后在调用`PsiReferenceRegistrar.registerReferenceProvider()`时使用元素模式来指定贡献引用的确切位置。

请参阅《Reference Contributor 教程》。

### 带有可选或多重解析结果的引用
在最简单的情况下，引用解析为单个元素，如果解析失败，则代码不正确，IDE需要将其标记为错误。然而，有些情况下情况会有所不同。

第一个情况是软引用。考虑上面的`new File("foo.txt")`示例。如果IDE找不到"foo.txt"文件，这并不意味着需要将其标记为错误——可能该文件仅在运行时可用。此类引用从`PsiReference.isSoft()`方法返回true，这样可以在检查/注释中跳过标记它们为错误或使用较低的严重性。

第二种情况是多变量引用。考虑JavaScript程序的情况。JavaScript是一种动态类型语言，因此IDE无法总是准确地确定在特定位置调用的是哪个方法。为了解决这个问题，它提供了一个可以解析为多个可能元素的引用。此类引用实现`PsiPolyVariantReference`接口。

为了解析`PsiPolyVariantReference`，需要调用其`multiResolve()`方法。该调用返回一个`ResolveResult`对象数组。每个对象标识一个PSI元素，并且还指定结果是否有效。例如，如果你有多个Java方法重载，并且调用的参数不匹配任何一个重载，你将得到所有重载的`ResolveResult`对象，并且`isValidResult()`对所有结果返回false。

### 搜索引用
解析引用意味着从使用位置导航到相应的声明。要执行相反方向的导航——从声明到其使用位置——可以执行引用搜索。

使用`ReferencesSearch`执行搜索时，指定要搜索的元素，和可选的其他参数，如需要搜索引用的范围。创建的查询允许一次获取所有结果或逐一迭代结果。后者允许在找到第一个（匹配的）结果时停止处理。

### 实现引用
请参阅指南和相应的教程了解更多信息。

## 修改 PSI
最后修改日期：2024年7月31日

PSI（程序结构接口）是一个读/写的表示，它将源代码表示为对应于源文件结构的元素树。您可以通过添加、替换和删除PSI元素来修改PSI。

要执行这些操作，可以使用如 `PsiElement.add()`、`PsiElement.delete()` 和 `PsiElement.replace()` 等方法，以及其他定义在 `PsiElement` 接口中的方法，这些方法允许您在单个操作中处理多个元素，或者指定需要添加元素的树中的确切位置。

与文档操作一样，PSI修改需要包装在写操作和命令中（并且只能在事件调度线程中执行）。有关命令和写操作的更多信息，请参阅文档文章。

### 创建新的 PSI
通常，添加到树中或用于替换现有PSI元素的PSI元素是从文本创建的。在最一般的情况下，您使用 `PsiFileFactory` 的 `createFileFromText()` 方法创建一个包含您需要添加到树中的代码构造的新文件，或者用作现有元素的替代，遍历生成的树以找到您需要的特定部分，然后将该元素传递给 `add()` 或 `replace()`。请参阅如何创建PSI文件。

大多数语言提供了工厂方法，使您更容易创建特定的代码构造。例如：

- `PsiJavaParserFacade` 类包含如 `createMethodFromText()` 之类的方法，用于从给定的文本创建一个Java方法。
- `SimpleElementFactory.createProperty()` 创建一个 Simple 语言属性。

在实现重构、意图或代码检查修复时，工作代码会组合硬编码的片段和从现有文件中提取的代码片段。对于小的代码片段（单个标识符），可以简单地将现有代码的文本附加到正在构建的代码片段的文本中。在这种情况下，需要确保生成的文本在语法上是正确的。否则，`createFromText()` 方法将抛出异常。

对于较大的代码片段，最好分几个步骤进行修改：

1. 从文本创建一个替换树片段，为用户代码片段留下占位符；
2. 用用户代码片段替换占位符；
3. 用替换树替换原始源文件中的元素。

这确保了用户代码的格式保持不变，并且修改不会引入任何不需要的空白更改。正如在IntelliJ平台API的其他地方一样，传递给 `createFileFromText()` 和其他 `createFromText()` 方法的文本必须仅使用 `\n` 作为行分隔符。

以下示例展示了这种方法的应用，参见 `ComparingStringReferencesInspection` 示例中的快速修复：

```java
public void applyFix(@NotNull Project project, @NotNull ProblemDescriptor descriptor) {
  PsiBinaryExpression binaryExpression = (PsiBinaryExpression) descriptor.getPsiElement();
  IElementType opSign = binaryExpression.getOperationTokenType();
  PsiExpression lExpr = binaryExpression.getLOperand();
  PsiExpression rExpr = binaryExpression.getROperand();
  if (rExpr == null) {
    return;
  }
  PsiElementFactory factory = JavaPsiFacade.getInstance(project).getElementFactory();
  PsiMethodCallExpression equalsCall =
      (PsiMethodCallExpression) factory.createExpressionFromText("a.equals(b)", null);
  PsiExpression qualifierExpression =
      equalsCall.getMethodExpression().getQualifierExpression();
  assert qualifierExpression != null;
  qualifierExpression.replace(lExpr);
  equalsCall.getArgumentList().getExpressions()[0].replace(rExpr);
  PsiExpression result = (PsiExpression) binaryExpression.replace(equalsCall);

  if (opSign == JavaTokenType.NE) {
    PsiPrefixExpression negation =
        (PsiPrefixExpression) factory.createExpressionFromText("!a", null);
    PsiExpression operand = negation.getOperand();
    assert operand != null;
    operand.replace(result);
    result.replace(negation);
  }
}
```

### 维护树结构一致性
PSI修改方法在构建结果树结构的方式上没有限制。例如，当处理一个Java类时，您可以将一个for语句直接添加为 `PsiMethod` 元素的子元素，即使Java解析器从不生成这样的结构（for语句将始终是表示方法体的 `PsiCodeBlock` 的子元素）。生成不正确树结构的修改可能看起来有效，但它们会导致后续的问题和异常。因此，您需要确保通过PSI修改操作构建的结构与解析器解析您创建的代码时生成的结构相同。

为了确保您没有引入不一致，可以在修改PSI的操作的测试中调用 `PsiTestUtil.checkFileStructure()`。此方法确保您构建的结构与解析器生成的结构相同。

### 空白和导入
在使用PSI修改功能时，您不应单独创建文本中的空白节点（空格或换行符）。相反，所有的空白修改都是由格式化程序执行的，它遵循用户选择的代码样式设置。格式化程序会在每个命令结束时自动执行，如果需要，也可以使用 `CodeStyleManager` 类中的 `reformat(PsiElement)` 方法手动执行。

另外，在处理Java代码（或类似导入机制的其他语言代码如Groovy或Python）时，您不应手动创建导入。相反，您应该在生成的代码中插入完全限定的名称，然后调用 `JavaCodeStyleManager` 中的 `shortenClassReferences()` 方法（或工作语言的等效API）。这确保导入按照用户的代码样式设置创建并插入文件的正确位置。

### 结合PSI和文档修改
在某些情况下，您需要执行PSI修改，然后在通过PSI修改的文档上执行操作（例如，启动一个实时模板）。为了完成基于PSI的后处理（如格式化）并提交更改到文档，可以调用 `PsiDocumentManager` 实例的 `doPostponedOperationsAndUnblockDocument()` 方法。

# PSI Cookbook
最后修改日期：2024年7月31日

本页面提供了与PSI（程序结构接口）相关的最常见操作的步骤和方法。

与开发自定义语言插件不同，这里讨论的是如何与现有语言的PSI（如Java）进行交互。

另请参阅PSI性能部分。

## 通用操作
### 如何在不知道路径的情况下查找文件？
可以使用 `FilenameIndex.getFilesByName()` 方法根据文件名查找文件。

### 如何查找特定PSI元素的使用位置？
可以使用 `ReferencesSearch.search()` 方法查找某个特定PSI元素的使用位置。

### 如何重命名一个PSI元素？
可以使用 `RefactoringFactory.createRename()` 方法来重命名一个PSI元素。

### 如何强制重新解析虚拟文件的PSI？
可以使用 `FileContentUtil.reparseFiles()` 方法来重新解析文件的PSI。

## Java特定操作
如果你的插件依赖Java功能并且目标版本是2019.2或更高版本，请参阅Java部分。另外，如果你的插件支持其他JVM语言，可以考虑使用UAST（统一抽象语法树）。

### 如何查找某个类的所有继承者？
可以使用 `ClassInheritorsSearch.search()` 方法查找某个类的所有继承者。

### 如何通过完全限定名查找类？
可以使用 `JavaPsiFacade.findClass()` 方法通过完全限定名查找类。

### 如何通过短名查找类？
可以使用 `PsiShortNamesCache.getClassesByName()` 方法通过短名查找类。

### 如何查找Java类的超类？
可以使用 `PsiClass.getSuperClass()` 方法查找Java类的超类。

### 如何获取Java类所属的包的引用？
```java
PsiJavaFile javaFile = (PsiJavaFile) psiClass.getContainingFile();
PsiPackage psiPackage = JavaPsiFacade.getInstance(project)
    .findPackage(javaFile.getPackageName());
```
或者
```java
PsiUtil.getPackageName()
```

### 如何查找重写特定方法的方法？
可以使用 `OverridingMethodsSearch.search()` 方法查找重写特定方法的方法。

## 2023.2+
### 如何检查JVM库的存在？
可以使用 `JavaLibraryUtil` 提供的专用（且高度缓存的）方法：
- `hasLibraryClass()`：通过已知的库类完全限定名（FQN）检查库的存在
- `hasLibraryJar()`：使用Maven坐标（例如，`io.micronaut:micronaut-core`）检查库的存在

# PSI 性能
最后修改日期：2024年7月31日

参见避免UI卡顿和提升索引性能。

IDE Perf插件提供即时的性能诊断工具，包括专门用于CachedValue指标的视图。

## 避免PsiElement的昂贵方法
在深层次的树结构中，避免使用PsiElement的一些耗时方法。

- `getText()`：遍历给定元素下的整个树并连接字符串，建议考虑使用 `textMatches()`。
- `getTextRange()`、`getContainingFile()` 和 `getProject()`：这些方法会向上遍历到文件根节点，在非常嵌套的树中可能会很耗时。如果仅需要PSI元素的长度，使用 `getTextLength()`。
- `getContainingFile()` 和 `getProject()` 通常可以在每个任务中计算一次，然后存储在字段中或通过参数传递。

另外，诸如 `getText()`、`getNode()` 或 `getTextRange()` 之类的方法需要AST（抽象语法树），获取AST可能是一个相当昂贵的操作，请参见下一节。

## 避免使用多个PSI树/文档
避免将太多已解析的树或文档加载到内存中。理想情况下，只有编辑器中打开的文件的AST节点应该存在于内存中。其他所有内容，即使是用于解析/高亮显示的目的，也可以通过PSI接口访问，但其实现应该使用底层的存根，这样会更少消耗CPU和内存。

如果存根不适合你的情况（例如，你需要的信息很大且/或很少需要，或者你正在为你无法控制其PSI的语言开发插件），你可以创建自定义索引或要点。

为了确保你不会意外加载AST，你可以在生产环境中使用 `AstLoadingFilter`，并在测试中使用 `PsiManagerEx.setAssertOnFileLoadingFilter()`。

同样适用于文档：只有在编辑器中打开的文档应该被加载。通常，你不需要文档内容（因为大部分信息可以从PSI中获取）。如果你确实需要文档，考虑将你需要提供的信息保存到自定义索引或要点中，以便以后更便宜地获取。如果仍然需要文档，请至少确保一次只加载一个文档，并且不要持有强引用，以便GC尽快释放内存。

## 缓存耗时计算的结果
一些方法调用如 `PsiElement.getReference()`（和 `getReferences()`）、`PsiReference.resolve()`（和 `multiResolve()` 等等）或者表达式类型的计算、类型推断结果、控制流图等可能是昂贵的。为了避免多次付出这些代价，可以缓存这些计算的结果并重复使用。通常，使用 `CachedValueManager` 创建的 `CachedValue` 可以很好地满足这个目的。

如果你缓存的信息仅依赖于当前PSI元素的子树（不依赖于解析结果或其他文件），你可以在你的PsiElement实现中将其缓存到字段中，并在重写 `ASTDelegatePsiElement.subtreeChanged()` 时删除缓存。

## 2024.1+
### 使用 ProjectRootManager 作为依赖
平台不再在哑模式结束时递增根变化修改跟踪器。如果缓存的值使用 `ProjectRootManager` 作为依赖（没有 `PsiModificationTracker`），并且同时依赖于索引，则必须添加对 `DumbService` 的依赖。



在技术文档中，"Dumb Mode" 通常指的是一个特定状态，在此状态下，IDE暂时禁用了某些功能，因此它的字面翻译为“哑模式”确实不太合适。我们可以用更贴近原意的术语来翻译，例如“简化模式”或“无智能模式”。以下是调整后的内容：

---

## 简化模式（Dumb Mode）

索引是一个可能耗时的过程。它在后台执行，在此期间，所有IDE功能都受到限制，只能使用不需要索引的功能：基本文本编辑、版本控制等。此限制由DumbService管理。违反此限制会通过IndexNotReadyException报告，详见其文档以了解如何适应调用者。

DumbService提供了API来查询IDE当前是否处于“简化模式”（不允许访问索引）或“智能模式”（所有索引已构建完毕并可以使用）。它还提供了在索引准备好之前延迟代码执行的方法。

## DumbAware API

某些扩展点的实现可以通过实现DumbAware接口来标记为在简化模式下可用。这些扩展点在IntelliJ平台扩展点和监听器列表中标有DumbAware标签。常用的包括CompletionContributor、(External)Annotator和各种运行配置的扩展点。自2024.2版本起，意图和快速修复也包含在内。

对于在简化模式下可用的操作，请扩展DumbAwareAction。

其他API可能会通过扩展PossiblyDumbAware来指示其兼容简化模式。

## 测试

要在测试时切换简化模式，请在IDE以内部模式运行时调用“工具 | 内部操作 | 进入/退出简化模式”。

---

这个翻译更好地反映了“Dumb Mode”在IDE中的作用和意义。如果还有其他问题或需要进一步调整，请告知我。

# 文件索引（File-Based Indexes）

文件索引基于Map/Reduce架构。每个索引都有特定类型的键和特定类型的值。

键是用于从索引中检索数据的标识。例如，在单词索引中，键是单词本身。

值是与键相关联的任意数据。例如，在单词索引中，值可以是一个掩码，指示单词出现在什么上下文中（代码、字符串字面量或注释）。

在最简单的情况下，当需要知道某些数据在哪些文件中存在时，值的类型为`Void`且不会存储在索引中。

当索引实现对文件进行索引时，它接收文件的内容并返回从文件中找到的键到相关值的映射。

访问索引时，需要指定感兴趣的键，并返回包含该键的文件列表，以及每个文件相关的值。

在某些情况下，可以考虑使用Gists作为替代方案。

## 实现文件索引

一个相对简单的文件索引实现是UI设计器绑定表单索引，它存储了GUI设计器`.form`文件的绑定实现类的全限定名（FQN）。

每个特定的索引实现都是一个扩展FileBasedIndexExtension的类，通过`com.intellij.fileBasedIndex`扩展点注册。

一个文件索引的实现包括以下主要部分：

- `getIndexer()` 返回`DataIndexer`的实现，实际负责基于文件内容构建键/值对。
- `getKeyDescriptor()` 返回`KeyDescriptor`，负责比较键并将其存储为序列化的二进制格式。最常用的实现可能是`EnumeratorStringDescriptor`，它被设计用于高效存储标识符。
- `getValueExternalizer()` 返回`DataExternalizer`，负责以二进制格式存储值。
- `getInputFilter()` 允许仅将索引限制在某些文件集上。考虑使用`DefaultFileTypeSpecificInputFilter`。
- `getName()` 返回唯一的索引ID。建议使用完全限定的索引类名，以免与其他定义相同ID的插件冲突，例如`com.example.myplugin.indexing.MyIndex`。
- `getVersion()` 返回索引实现的版本。如果当前版本与用于构建索引的实现版本不同，索引将自动重建。

如果没有值与文件关联（即，值类型为`Void`），可以通过扩展`ScalarIndexExtension`简化实现。如果每个文件只有一个值，则扩展`SingleEntryFileBasedIndexExtension`。

请参阅提高索引性能。

### 实现注意事项

- 值类必须正确实现`equals()`和`hashCode()`，以确保从二进制数据反序列化的值与原始值相等。
- `DataIndexer.map()`返回的数据必须仅依赖于传递给该方法的输入数据，不得依赖任何外部文件。否则，索引在外部数据更改时不会正确更新，导致索引中有过时数据。
- 请在开发期间设置系统属性`intellij.idea.indices.debug`或`intellij.idea.indices.debug.extra.sanity`为`true`，以启用额外的调试断言，以确保正确的索引实现。

## 访问文件索引

通过`FileBasedIndex`类可以访问文件索引。

请注意，在简化模式（Dumb Mode）下，索引访问受到限制。

支持的主要操作包括：

- `getAllKeys()` 和 `processAllKeys()` 允许获取项目中所有文件中找到的所有键的列表。为优化性能，考虑从`FileBasedIndexExtension.traceKeyHashToVirtualFileMapping()`返回`true`（详见其Javadoc）。
- 返回的数据保证包含项目内容中找到的所有键，但也可能包括当前项目中找不到的额外键。
- `getValues()` 允许获取与特定键相关联的所有值，但不包括找到它们的文件。
- `getContainingFiles()` 允许收集找到特定键的所有文件。
- `processValues()` 允许迭代找到特定键的所有文件，并同时访问相关的值。

## 嵌套索引访问

在嵌套调用中访问索引数据（通常来自多个索引）可能会有一些限制。

### 2023.1及更高版本
从2023.1版本开始，嵌套索引访问已被允许。

### 2022.3及更早版本
请不要使用嵌套索引访问，这在某些条件下已知会引起问题，请关注相关问题跟踪。

## 标准索引

IntelliJ平台包含若干标准文件索引。对于插件开发者来说，最有用的索引是：

- **单词索引（Word Index）**
  - 通常，通过使用`PsiSearchHelper`类的辅助方法间接访问单词索引。

- **文件名索引（File Name Index）**
  - `FilenameIndex` 提供了一种快速查找特定文件名的所有文件的方法。

- **文件类型索引（File Type Index）**
  - `FileTypeIndex` 用于快速找到特定文件类型的所有文件。

## 额外的索引根

要添加额外的文件/目录进行索引，实现`IndexableSetContributor`并注册到`com.intellij.indexedRootsProvider`扩展点。

# Stub 索引（Stub Indexes）

## Stub 树

Stub 树是文件 PSI 树的子集，它以紧凑的二进制格式序列化存储。文件的 PSI 树可以由 AST 支持（通过解析文件构建）或从磁盘反序列化的 stub 树支持。这两者之间的切换是透明的。

Stub 树只包含节点的一个子集。通常，它只包含需要从外部文件解析该文件中声明的节点。尝试访问不属于 stub 树的任何节点或执行无法由 stub 树满足的任何操作（例如，访问 PSI 元素的文本）时，将导致文件解析从 stub 切换到 AST 支持。

每个 stub 在 stub 树中只是一个不具备行为的 bean 类。Stub 存储对应 PSI 元素状态的一个子集，如元素的名称、修饰符标志（例如 public 或 final）等。Stub 还保存了指向树中父级的指针以及其子级 stub 的列表。

## 实现

使用 Grammar-Kit 生成语言 PSI 时，请参阅 Stub 索引支持部分以获取有关将语法与 stub 集成的说明。

### Stub 设置

以下步骤每种支持 stub 的语言只需执行一次：

1. 将语言的文件元素类型（从 `ParserDefinition.getFileNodeType()` 返回的元素类型）更改为扩展 `IStubFileElementType` 的类，并重写其 `getExternalId()` 方法（参见下一项）。
2. 在 `plugin.xml` 中，定义 `com.intellij.stubElementTypeHolder` 扩展点，并指定包含语言解析器使用的 `IElementType` 常量的接口。
3. 定义所有 stub 元素类型使用的公共 `externalIdPrefix`（参见添加 stub 元素）。参见 `StubElementTypeHolderEP` 文档中的重要要求。

示例：
- 在 `JavaPsiPlugin.xml` 中注册的 `JavaStubElementTypes`
- 参见 `Angular2MetadataElementTypes` 的 Kotlin 示例

### 添加 Stub 元素

对于需要存储在 stub 树中的每种元素类型，执行以下步骤：

1. 定义一个接口，派生自 `StubElement` 接口（示例）。
2. 提供该接口的实现（示例）。
3. 确保 PSI 元素的接口扩展了 `StubBasedPsiElement`，并以 stub 接口类型为参数（示例）。
4. 确保 PSI 元素的实现类扩展了 `StubBasedPsiElementBase`，并以 stub 接口类型为参数（示例）。提供一个接受 `ASTNode` 的构造函数和一个接受 stub 的构造函数。
5. 创建一个实现 `IStubElementType` 的类，并以 stub 接口和实际的 PSI 元素接口为参数（示例）。实现 `createPsi()` 和 `createStub()` 方法，以从 stub 创建 PSI 和 vice versa。实现 `serialize()` 和 `deserialize()` 方法，以二进制流的形式存储数据。
6. 根据语言通用的 `externalIdPrefix` 重写 `getExternalId()`（参见 Stub 设置）。
7. 对于总是叶子节点的 stub 返回 `isAlwaysLeaf()`（2023.3）。不序列化任何自身数据的“容器” stub 可以实现 `EmptyStubSerializer` 以优化存储（2023.3）。
8. 在解析时使用实现 `IStubElementType` 的类作为元素类型常量（示例）。
9. 确保 PSI 元素接口中的所有方法在适当时访问 stub 数据而不是 PSI 树（示例：`Property.getKey()` 实现）。

默认情况下，如果 PSI 元素扩展了 `StubBasedPsiElement`，则该类型的所有元素都将存储在 stub 树中。要更精确地控制哪些元素存储在 stub 树中，可以重写 `IStubElementType.shouldCreateStub()`，并为不应包括在 stub 树中的元素返回 false。排除并不是递归的：如果返回 false 的元素的某些子元素也是基于 stub 的 PSI 元素，它们将包括在 stub 树中。

### 数据序列化

对于在 stub 中序列化字符串数据（例如元素名称），使用 `StubOutputStream` 的 `writeName()` 和 `readName()`。这些方法确保每个唯一的标识符只在数据流中存储一次，从而减少序列化 stub 树数据的大小。参见 `DataInputOutputUtil`。

要更改 stub 的存储二进制格式（例如，存储一些额外的数据或一些新的元素），请确保提高语言 `IStubFileElementType.getStubVersion()` 返回的 stub 版本。这将导致重建 stub 和 Stub 索引，并避免存储数据格式与尝试加载它的代码之间的不匹配。

确保存储在 stub 树中的所有信息仅依赖于构建 stub 的文件的内容，而不依赖于任何外部文件或其他数据。否则，当外部依赖项更改时，stub 树不会重建，导致 stub 树中的数据陈旧和不正确。

请参阅提高索引性能。

## Stub 索引

在构建 stub 树时，插件可以同时将有关 stub 元素的一些数据放入多个索引中，然后可以使用这些索引通过相应的键找到 PSI 元素。与基于文件的索引不同，stub 索引不支持将自定义数据作为值存储；值始终是 PSI 元素。stub 索引中的键通常是字符串（例如类名）；如果需要，也支持其他数据类型。

一个 stub 索引是一个扩展 `AbstractStubIndex` 的类。在最常见的情况下，当键类型为字符串时，使用更具体的基类，即 `StringStubIndexExtension`。Stub 索引实现类在 `com.intellij.stubIndex` 扩展点中注册。

要将数据放入索引，实现 `IStubElementType.indexStub()`（示例：`JavaClassElementType.indexStub()`）。该方法接受 `IndexSink` 作为参数，并将索引 ID 和每个元素应存储的索引键放入索引中。

## 访问 Stub 索引

要从索引中访问数据，请使用实现管理的单例实例上的以下实例方法：

### 键

`AbstractStubIndex.getAllKeys()`/`processAllKeys()` 返回索引中指定项目的所有键的列表（处理所有键）（例如，项目中找到的所有类名列表）。

注意：这些可能会返回陈旧/过时的数据。请参阅 Elements 以获取/验证给定键的实际存在的元素（例如，在迭代所有键以收集补全变体时）。

### 元素

`StubIndex.getElements()` 返回与特定键对应的 PSI 元素集合（例如，具有指定短名称的类）在指定范围内。

示例：`JavaAnnotationIndex`

## 相关讨论

Stub 创建的生命周期



# 元素模式（Element Patterns）

元素模式提供了一种通用方法来指定对象上的条件。插件作者使用它们来检查 PSI 元素是否符合特定的结构。就像正则表达式用于字符串的匹配测试一样，元素模式用于对 PSI 元素的嵌套结构进行条件检查。IntelliJ 平台内部主要有两个应用场景：

1. **指定在实现自定义语言的补全贡献者时自动补全应出现的位置。**
2. **指定提供进一步引用的 PSI 元素，通过 PSI 引用贡献者实现。**

然而，插件作者很少直接实现 `ElementPattern` 接口。我们建议使用 IntelliJ 平台提供的高级模式类：

| 类                   | 主要内容                                                     | 典型例子                                                     |
| -------------------- | ------------------------------------------------------------ | ------------------------------------------------------------ |
| `StandardPatterns`   | 工厂类，用于创建字符串和字符模式；逻辑操作如 `and`、`or`、`not` | `LogbackReferenceContributor`、`RegExpCompletionContributor` |
| `PlatformPatterns`   | 工厂类，用于创建 PSI-、IElement- 和 VirtualFile-模式         | `FxmlReferencesContributor`、`PyDataclassCompletionContributor` |
| `PsiElementPattern`  | PSI 模式；检查子元素、父元素或邻近的叶子元素                 | `XmlCompletionContributor`                                   |
| `CollectionPattern`  | 过滤和检查集合模式；主要用于提供其他高级模式类的功能         | `PsiElementPattern`                                          |
| `TreeElementPattern` | 特定用于检查（PSI）树结构的模式                              | `PyMetaClassCompletionContributor`                           |
| `StringPattern`      | 检查字符串是否匹配，具有特定长度，特定开始或结尾，或属于特定集合 | `AbstractGradleCompletionContributor`                        |
| `CharPattern`        | 检查字符是否是空格、数字或 Java 标识符部分                   | `CompletionUtil`                                             |

一些 IntelliJ 平台内置语言实现了自己的模式类，并可以提供更多的例子：

- `XmlPatterns` 提供用于 XML 属性、值、实体和文本的模式。
- `PsiJavaPatterns` 提供用于 Java 的文字、字符串、参数和函数/方法参数的模式。
- `DomPatterns` 构建在 `XmlPatterns` 之上，为 DOM-API 提供进一步的模式。

## 例子

元素模式的一个好的起点是[自定义语言支持教程](https://www.jetbrains.org/intellij/sdk/docs/tutorials/custom_language_support_tutorial.html)。它们在教程的补全和引用贡献者部分中使用。然而，IntelliJ 平台源代码提供了许多内置语言（如 JSON、XML、Groovy、Markdown 等）的元素模式示例。查看上表中的引用或搜索高级模式类的用法将提供一个全面的列表，展示了元素模式在生产代码中的使用。

例如，可以在 JavaFX 插件的 `FxmlReferencesContributor` 中找到一个示例，测试给定的 PSI 元素是否是 *.fxml 文件中的 XML 属性值：

```java
XmlAttributeValuePattern attributeValueInFxml =
    XmlPatterns.xmlAttributeValue().inVirtualFile(
        virtualFile().withExtension(JavaFxFileTypeFactory.FXML_EXTENSION)
    );
```

如上所示，元素模式可以堆叠和组合，以创建更复杂的条件。`JsonCompletionContributor` 包含了一个具有更多 PSI 元素要求的示例。

```java
PsiElementPattern.Capture<PsiElement> AFTER_COMMA_OR_BRACKET_IN_ARRAY =
    psiElement()
        .afterLeaf("[", ",")
        .withSuperParent(2, JsonArray.class)
        .andNot(
            psiElement().withParent(JsonStringLiteral.class)
        );
```

上面的模式确保了 PSI 元素：

1. 出现在一个左方括号或逗号之后，这是通过对邻近的叶子元素施加限制来表示的。
2. 有 `JsonArray` 作为二级父元素，这表示 PSI 元素必须在一个 JSON 数组内。
3. 没有 `JsonStringLiteral` 作为父元素，这防止了数组中包含括号或逗号的字符串给出误报匹配。

这个最后的例子表明，即使是简单模式，也需要仔细考虑边界情况。

## 工具和调试

使用元素模式可能很棘手，插件作者需要对底层 PSI 结构有扎实的理解。因此，建议使用 PsiViewer 插件或内置的 PSI 查看器，以验证元素确实具有预期的结构和属性。

### 调试

本节假设插件作者具有如何使用调试器、如何设置断点以及如何设置断点条件的基本理解。

调试元素模式时，插件作者需要记住，元素模式实例化的位置与它们实际使用的位置无关。例如，尽管补全贡献者的模式在注册贡献者时实例化，但这些模式是在键入期间的补全时检查的。因此，找到 IntelliJ 平台中调试元素模式的正确位置是第一步。

然而，在 `ElementPattern.accepts()` 方法中设置断点会产生许多误报，因为元素模式在整个 IDE 中广泛使用。过滤这些误报的一种方法是在断点上使用条件。以下步骤可以帮助你调查模式检查的地点：

1. 在 `ElementPattern.accepts()` 方法中设置断点。
2. 在断点上设置条件，检查模式的字符串表示是否包含可识别的部分。
3. 进行调试，当断点触发时，确保它是正确的模式，并调查调用栈以找到使用模式检查的相关方法。
4. 调试相关方法，例如填充补全变体或查找引用的方法。

## 调试示例

使用上述 Markdown 代码示例，我们注意到 `MarkdownLinkDestinationImpl` 类在元素模式中使用。现在，在以下位置设置一个断点：

```java
com.intellij.patterns.ElementPattern#accepts(
  java.lang.Object,
  com.intellij.util.ProcessingContext
)
```

右键点击断点并设置以下条件：

```java
toString().contains("MarkdownLinkDestinationImpl")
```

现在开始一个调试会话并打开一个 Markdown 文件。当断点命中时，调试工具窗口中的调用栈显示，在 `ReferenceProvidersRegistryImpl` 中的方法 `doGetReferencesFromProviders` 中检查了引用提供者。这为进一步调查提供了一个良好的起点。

# 统一抽象语法树（UAST）

统一抽象语法树（UAST）是针对不同编程语言（面向JVM，Java虚拟机）的PSI的抽象层。它提供了一种统一的API，用于处理通用的语言元素，例如类和方法声明、文字值和控制流操作符。

不同的JVM语言有自己的PSI，但许多IDE功能（如检查、侧边标记、引用注入等）在所有这些语言中都以相同的方式工作。使用UAST可以提供适用于所有支持的JVM语言的功能，实现单一实现。

在现实场景中使用UAST的概述请参考演讲[编写适用于Kotlin的IntelliJ插件](https://www.youtube.com/watch?v=wJ2GLkOetOs)。

## 何时使用UAST？

适用于需要在所有JVM语言中以相同方式工作的插件。

一些已知的例子包括：

- Spring Framework
- Android Studio
- Plugin DevKit

## 支持哪些语言？

- Java: 完全支持
- Kotlin: 完全支持
- Scala: Beta版，但完全支持
- Groovy: 仅支持声明，不支持方法体

## 关于修改PSI？

UAST是一个只读的API。目前有实验性的`UastCodeGenerationPlugin`和`JvmElementActionsFactory`类，但它们目前不建议外部使用。

## 使用UAST

UAST的基本元素是`UElement`。所有常见的基类子接口都位于`uast`模块的`declarations`和`expressions`目录中。

所有这些子接口提供了获取关于通用语法元素信息的方法：`UClass`表示类声明，`UIfExpression`表示条件表达式，等等。

### PSI到UAST的转换

要获取给定的PSI元素的UAST，可以使用`UastFacade`类或`UastContextKt.toUElement()`：

**Java:**
```java
UastContextKt.toUElement(element);
```

**Kotlin:**
```kotlin
UastContextKt.toUElement(element)
```

要将PSI元素转换为特定的`UElement`，可以使用以下方法之一：

简单转换：
```java
UastContextKt.toUElement(element, UCallExpression.class);
```

转换为多种可能的选项之一：
```java
UastFacade.INSTANCE.convertElementWithParent(element, new Class[]{UInjectionHost.class, UReferenceExpression.class});
```

在某些情况下，PSI元素可能表示多个`UElement`。例如，Kotlin中主构造函数的参数同时是`UField`和`UParameter`。需要所有选项时，可以使用：
```java
UastFacade.INSTANCE.convertToAlternatives(element, new Class[]{UField.class, UParameter.class});
```

**注意**：最好转换为特定类型的`UElement`，而不是无类型转换后再强制转换为特定类型。这是因为：
1. 性能原因：带有类型的`toUElement()`失败快速。
2. 可能在某些情况下得到不同的结果：带类型的转换更可预测。

### UAST到PSI的转换

有时需要从`UElement`回到底层语言的源代码。为此，`UElement#sourcePsi`属性返回原始语言的对应PSI元素。

`sourcePsi`是一个“物理”PSI元素，主要用于在原始文件中获取文本范围（例如用于高亮显示）。避免将`sourcePsi`强制转换为特定类，因为这意味着回退到UAST抽象的语言特定PSI。一些`UElement`是“虚拟”的，因此没有`sourcePsi`。对于某些`UElement`，`sourcePsi`可能与从中获取`UElement`的元素不同。

此外，还有`UElement#javaPsi`属性，返回一个“Java样式”的PSI元素。它是一个“虚假”的PSI元素，使不同的JVM语言模拟Java语言，以保持与Java-API的兼容性。例如，当调用`MethodReferencesSearch.search(PsiMethod)`时，只有Java本机提供`PsiMethod`；其他JVM语言因此通过`UMethod#javaPsi`提供“假”的`PsiMethod`。

**注意：**`UElement#javaPsi`仅对Java是物理的。因此，应使用`sourcePsi`来获取文本范围或检查警告/侧边标记放置的锚点元素。

简而言之：

- `sourcePsi`：
  - 是物理的：代表源文件中实际存在的PSI元素。
  - 可用于高亮显示、PSI修改、创建智能指针等。
  - 除非绝对需要，否则不应强制转换（例如，处理特定语言的情况）。

- `javaPsi`：
  - 应仅用于表示JVM可见的声明：`PsiClass`、`PsiMethod`、`PsiField`，获取其名称、类型、参数等，或将其传递给接受Java-PSI声明的方法。
  - 不保证是物理的：可能在源文件中不存在。
  - 不可修改：调用修改方法可能会为非Java语言抛出异常。

**注意：**`sourcePsi`和`javaPsi`都可以转换回`UElement`。

### UAST 访问者

在UAST中，没有统一的方法来获取`UElement`的子元素（尽管可以通过`UElement#uastParent`获取其父元素）。因此，遍历UAST作为树的唯一方法是将`UastVisitor`传递给`UElement.accept()`方法。

**注意：** 在UAST访问者中有一个惯例，如果`visit*()`返回true，访问者将不会传递给子元素。否则，`UastVisitor`将继续深入遍历。

`UastVisitor`可以使用`UastVisitorAdapter`或`UastHintedVisitorAdapter`转换为`PsiElementVisitor`。后者更可取，因为它提供了更好的性能和更可预测的结果。

作为一般规则，如果不需要处理许多不同类型的`UElement`，并且元素的结构不太重要，则建议避免使用`UastVisitor`。在这种情况下，更好的是使用`PsiElementVisitor`遍历PSI树，并将每个遇到的匹配元素显式转换为UAST。

### UAST 性能提示

UAST不是零成本的抽象：对于某些语言，一些方法可能出人意料地昂贵，因此在进行优化时要小心，因为这可能会导致相反的效果。

转换为`UElement`在某些情况下可能需要解析，这对于某些语言来说可能非常昂贵。应仅在必要时执行转换为UAST。例如，将整个`PsiFile`转换为`UFile`并遍历它只是为了收集`UMethod`声明是低效的。相反，遍历`PsiFile`并将每个遇到的匹配元素显式转换为`UMethod`。

### UAST 问题

#### `ULiteralExpression`不应用于字符串

`ULiteralExpression`表示数字、布尔值和字符串等文字值。虽然字符串值也是文字，但`ULiteralExpression`用于处理字符串时并不方便。例如，它不处理Kotlin的字符串插值。处理字符串文字时，请使用`UInjectionHost`。

#### `sourcePsi`和`javaPsi`，`psi`和UElement作为PSI

由于历史原因，`UElement`和`PsiElement`之间的关系很复杂。一些`UElement`实现了`PsiElement`，例如`UMethod`实现了`PsiMethod`。强烈建议不要将`UElement`用作`PsiElement`，并且Plugin DevKit提供了相应的检查（Plugin DevKit | Code | UElement as PsiElement usage）。这种“实现”被认为是已弃用的，未来可能会被移除。

此外，还有`UElement#psi`属性；它返回与`javaPsi`或`sourcePsi`相同的元素。由于很难猜测将返回什么，因此也已弃用。

因此，`sourcePsi`和`javaPsi`应该是从`UElement`获取`PsiElement`的唯一方法。请参阅相应的部分。

### 使用UAST还是PSI

UAST提供了一种统一的方式来表示JVM兼容的声明，如`UMethod`、`UField`、`UClass`等。但同时，所有JVM语言插件都实现了`PsiMethod`、`PsiClass`等，以保持与Java的兼容性。这些实现可以通过`UElement#javaPsi`属性获取。

因此，问题是：“我应该在代码中使用什么来表示Java声明？”答案是：我们鼓励使用`PsiMethod`、`PsiClass`作为Java声明的通用接口，而不鼓励在API中暴露UAST接口。

**注意：** 对于方法体，没有这样的替代品，因此暴露`UExpression`等不是不鼓励的。仍然考虑暴露原始的`PsiElement`。

### UAST/PSI树结构不匹配

UAST是不同语言PSI之上的抽象层，尝试构建统一的树（参见检查UAST树）。这导致UAST和原始语言

之间的树结构可能严重不同，因此不保证保留祖先-后代关系。

例如，以下结果可能不同，不仅在元素数量上，而且在其顺序上也不同：

```java
generateSequence(uElement, UElement::uastParent).mapNotNull { it.sourcePsi }
generateSequence(uElement.sourcePsi) { it.parent }
```

### 在插件中使用UAST

要在插件中使用UAST，请添加对捆绑的Java插件（`com.intellij.java`）的依赖。

### 语言扩展

要注册适用于UAST的扩展，请在plugin.xml中的注册中指定`language="UAST"`。

### 检查UAST树

要检查UAST树，请调用内部操作`Tools | Internal Actions | UAST | Dump UAST Tree (By Each PsiElement)`。

### 检查

使用`AbstractBaseUastLocalInspectionTool`作为基类，并在注册时指定`language="UAST"`。如果检查目标仅是默认类型的子集（`UFile`、`UClass`、`UField`和`UMethod`），请在重载构造函数中指定`UElements`作为提示，以提高性能。

使用`ProblemsHolder.registerUProblem()`扩展函数来注册问题（2023.2）。

### 行标记

使用`UastUtils.getUParentForIdentifier()`或`UAnnotationUtils.getIdentifierAnnotationOwner()`来获取合适的“标识符”元素（有关详细信息，请参见行标记提供者）。



### XML DOM API 详细中文翻译与解释

#### 文章概述
这篇文章主要面向开发自定义Web服务器集成或用于简化XML编辑的插件开发者。它描述了IntelliJ平台中的文档对象模型（DOM），一种与DTD或基于Schema的XML模型进行交互的简便方式。本文涵盖的主题包括：如何操作DOM（读取/写入标签内容、属性和子标签），以及如何通过将UI连接到DOM来实现UI中的简单XML编辑。

假设读者已经熟悉Java、Swing、IntelliJ平台XML PSI（如`XmlTag`、`XmlFile`、`XmlTagValue`等类），以及IntelliJ平台插件开发基础（应用程序和项目组件、文件编辑器）。

#### XML PSI 与 DOM 的比较
如何从IntelliJ平台插件操作XML？通常，首先需要获取`XmlFile`，然后获取其根标签，并通过路径找到所需的子标签。路径由标签名组成，每个标签名都是一个字符串。这样逐字逐句输入可能既繁琐又容易出错。例如，假设有如下XML结构：

```xml
<root>
  <foo>
    <bar>42</bar>
    <bar>239</bar>
  </foo>
</root>
```

如果需要读取第二个`bar`元素的内容“239”，常见的做法如下：

```java
file.getDocument()
    .getRootTag()
    .findFirstSubTag("foo")
    .findSubTags("bar")[1]
    .getValue()
    .getTrimmedText();
```

然而，这种链式调用可能会导致`null`值的情况。因此，实际代码可能如下：

```java
XmlFile file = ...;
XmlDocument document = file.getDocument();
if (document != null) {
  XmlTag rootTag = document.getRootTag();
  if (rootTag != null) {
    XmlTag foo = rootTag.findFirstSubTag("foo");
    if (foo != null) {
      XmlTag[] bars = foo.findSubTags("bar");
      if (bars.length > 1) {
        String s = bars[1].getValue().getTrimmedText();
        // 执行某些操作
      }
    }
  }
}
```

看起来很冗长，对吧？但实际上可以通过扩展一个特殊接口`DomElement`来实现更好的方式。通过创建多个接口，例如：

```java
interface Root extends com.intellij.util.xml.DomElement {
  Foo getFoo();
}

interface Foo extends com.intellij.util.xml.DomElement {
  List<Bar> getBars();
}

interface Bar extends com.intellij.util.xml.DomElement {
  String getValue();
}
```

然后创建一个`DomFileDescription`类，将根标签名称和根元素接口传递给它的构造函数。在`plugin.xml`中注册它，使用`com.intellij.dom.fileMetaData`扩展点并指定`rootTagName`和`domVersion/stubVersion`属性。对于2019.1及以前的版本，使用`com.intellij.dom.fileDescription`扩展点。

#### 构建模型
##### 标签内容
在XML PSI中，标签内容称为标签值（tag value）。为了读取和更改标签值，需要在接口中添加两个方法（getter和setter），如：

```java
String getValue();
void setValue(String s);
```

这些方法名称（`getValue`和`setValue`）是标准的，用于默认访问标签值。如果想使用自定义方法名称，可以使用`@TagValue`注释这些方法。

#### 自定义值类型
如果需要处理更复杂的类型，可以使用`@Convert`注解，并指定一个继承`Converter<T>`类的类。`T`是要处理的类型，而`Converter<T>`是负责在`String`和`T`之间转换的类。

例如，如果需要处理枚举类型，可以让枚举实现`NamedEnum`接口，并提供`getValue()`方法来返回正确的值与XML内容匹配。

```java
enum CmpVersion implements NamedEnum {
  CmpVersion_1_X ("1.x"),
  CmpVersion_2_X ("2.x");

  private final String value;

  CmpVersion(String value) {
    this.value = value;
  }

  public String getValue() {
    return value;
  }
}
```

#### 属性
处理属性相对简单，可以使用类似`GenericDomValue<T>`的方式创建接口，允许读取和设置属性值，例如：

```java
@Attribute("some-class")
GenericAttributeValue<PsiClass> getMyAttributeValue();
```

可以通过`@NameStrategy`注解指定将访问器名称转换为XML元素名称的策略类。默认策略是`HyphenNameStrategy`，它用连字符分隔单词。

#### 固定数量的子标签
当标签具有最多一个特定名称的子标签时，可以提供getter方法来访问这些子标签。这些getter方法的返回类型应该扩展`DomElement`。例如：

```java
GenericDomValue<String> getEjbName();
GenericDomValue<String> getEjbClass();
CmpField getCmpField();
```

#### 动态定义
可以通过实现`com.intellij.util.xml.reflect.DomExtender<T>`来在运行时扩展现有DOM模型。将其注册到`com.intellij.dom.extender`扩展点的`extenderClass`属性中，`domClass`指定要扩展的DOM类<T>。

#### 命名空间支持
使用`Namespace`注解DOM模型，并通过`DomFileDescription.registerNamespacePolicy()`从`DomFileDescription.initializeFileDescription()`注册命名空间键映射。

#### IDE支持
插件DevKit支持以下DOM相关代码的功能：
- `DomElement`：为继承类中定义的所有DOM相关方法提供隐式用法（抑制“未使用的方法”警告）。
- `DomElementVisitor`：为继承类中定义的所有DOM相关访问者方法提供隐式用法（抑制“未使用的方法”警告）。

#### 访问DOM
可以通过`DomManager.getDomElement()`方法获取DOM元素，使用`ensureTagExists()`创建没有底层XML元素的DOM元素。

#### 树结构
使用`getParent()`方法可以获取元素在树中的父元素，使用`getRoot()`方法可以返回`DomFileElement`，这是每个DOM树的根。

#### 有效性
元素可能因显式删除或外部PSI更改而无效。固定数量的子元素和属性应该尽可能保持有效。

#### DOM反射
DOM还有一种称为“泛型信息”的反射。可通过`DomGenericInfo`接口和`getGenericInfo()`方法访问子标签。

#### 事件
可以添加`DomEventListener`到`DomManager`，监听DOM模型中的更改。

#### 高亮注解
DOM支持基于注解的错误检查和高亮显示。需要实现`DomElementAnnotator`接口，并覆盖`DomFileDescription.createAnnotator()`方法。

#### 自动高亮（BasicDomElementsInspection）
可以通过提供`BasicDomElementsInspection`实例自动高亮显示以下错误：
- 缺少`@Required`元素或文本为空。
- XML值无法通过转换器转换。

#### 必须的子标签
使用`@Required`注解子标签getter方法，DOM会自动为你检查是否缺少所需的子标签或属性。

#### 解析
通过`GenericDomValue<T>`接口和其子接口`GenericAttributeValue<T>`，可以将任何类作为`T`。例如，可以将`GenericDomValue<PsiClass>`解释为类的引用。

#### Mock和稳定元素
可以使用`DomManager.createMockElement()`创建虚拟元素。`DomElement.copyFrom()`允许你从一个DOM元素复制信息到另一个。

#### 访问者
DOM模型还有一个访问者模式，称为`DomElementVisitor`。`DomElement`接口有`accept()`和`acceptChildren()`方法，接收访问者作为参数。

#### 实现
如果想扩展你的模型功能，可以在接口中添加方法，并创建一个抽象类实现接口。然后注册这个实现类，使DOM在创建模型元素时使用它。

#### 跨文件的模型
许多框架需要一组XML配置文件作为一个模型工作。可以扩展`DomModelFactory`（或`BaseDomModelFactory`），并提供你的`DomModel`实现。

#### DOM Stubs
DOM元素可以被存根化，以避免频繁访问XML/PSI。性能相关的元素、标签或属性getter可以使用`@com.intellij.util.xml.Stubbed`注解。

以上是详细的XML DOM API翻译和解释，希望对你的插件开发有所帮助。如果有其他问题，请随时提问。



# 自定义语言

### 注册文件类型
编辑页面最后修改时间：2024年7月31日
产品帮助：文件类型关联

在开发自定义语言插件时，第一步是注册与该语言关联的文件类型。

通常，IDE通过查看文件的文件名或扩展名来确定文件的类型。

自定义语言文件类型是从`LanguageFileType`类派生的类，它将一个`Language`子类传递给其基类构造函数。

#### 注册
##### 2019.2+版本
使用`com.intellij.fileType`扩展点来注册`LanguageFileType`的实现和实例，使用`implementationClass`和`fieldName`属性。此外，还必须声明与`FileType.getName()`匹配的名称和与`LanguageFileType.getLanguage()`返回的语言ID相匹配的语言。

要在IDE中关联文件类型，请指定以下表格中列出的一个或多个关联。

| 关联类型                | 属性                                   | 属性值                                     |
| ----------------------- | -------------------------------------- | ------------------------------------------ |
| 文件扩展名              | `extensions`                           | 用分号分隔的扩展名列表，无需.前缀          |
| 硬编码文件名            | `fileNames`/`fileNamesCaseInsensitive` | 用分号分隔的精确（不区分大小写）文件名列表 |
| 文件名模式              | `patterns`                             | 用分号分隔的模式列表（支持*和?通配符）     |
| Hashbang（2020.2+版本） | `hashBangs`                            | 用分号分隔的hash bang模式列表              |

#### 示例
自定义语言支持教程：语言和文件类型

Properties语言插件中的`LanguageFileType`子类

要验证文件类型是否正确注册，可以实现`LanguageFileType.getIcon()`方法，并验证与您的文件类型关联的文件是否显示了正确的图标（参见“使用图标”）。

#### 其他功能
如果希望IDE显示提示，提示用户您的插件支持特定文件类型，请参见插件推荐。

要控制操作系统中IDE与文件类型的关联，请实现`OSFileIdeAssociation`（2020.3）。

### 实现词法分析器
编辑页面最后修改时间：2024年7月31日

词法分析器（Lexer）定义了如何将文件内容拆分为标记（Tokens）。词法分析器是自定义语言插件几乎所有功能的基础，从基本的语法高亮到高级代码分析功能。

词法分析器的API由`Lexer`接口定义。

#### 词法分析器的主要使用场景
IDE在以下三种主要情况下调用词法分析器，插件可以根据需要提供不同的词法分析器实现：

1. **语法高亮：** 词法分析器从实现`SyntaxHighlighterFactory`接口的类中返回，该类注册在`com.intellij.lang.syntaxHighlighterFactory`扩展点。

2. **构建文件的语法树：** 词法分析器应从注册在`com.intellij.lang.parserDefinition`扩展点的`ParserDefinition.createLexer()`实现中返回。

3. **构建文件中包含的单词索引：** 如果使用基于词法分析器的单词扫描器实现，则将词法分析器传递给`DefaultWordsScanner`构造函数。参见"查找用法"。

用于语法高亮的词法分析器可以增量调用，以处理文件中更改的部分。相比之下，用于其他场景的词法分析器始终处理整个文件或嵌入在不同语言文件中的完整语言结构。

#### 词法分析器状态
可增量使用的词法分析器可能需要返回其状态，这意味着文件中每个位置对应的上下文。例如，Java词法分析器可能有顶级上下文、注释上下文和字符串字面量上下文的不同状态。

语法高亮词法分析器的一个基本要求是其状态必须由一个整数表示，该整数由`Lexer.getState()`方法返回。该状态将在词法分析器从文件中间恢复词法分析时，与要处理的片段的起始偏移量一起传递给`Lexer.start()`方法。在其他场景中使用的词法分析器可以始终从`getState()`返回0。

#### 词法分析器的实现
为自定义语言插件创建词法分析器的最简单方法是使用JFlex。

`FlexLexer`和`FlexAdapter`类将JFlex词法分析器适配到IntelliJ平台的词法分析器API。可以使用带有IntelliJ IDEA社区版源码中的`idea-flex.skeleton`词法分析器框架文件的补丁版本JFlex来创建兼容FlexAdapter的词法分析器。补丁版本的JFlex提供了一个新的命令行选项`--charat`，该选项将JFlex生成的代码更改为与IntelliJ平台框架一起工作。启用`--charat`选项将词法分析的源数据作为`java.lang.CharSequence`传递，而不是字符数组。

对于使用JFlex开发词法分析器，Grammar-Kit插件非常有用。它为编辑JFlex文件（*.flex）提供语法高亮和其他有用的功能。

词法分析器（尤其是基于JFlex的词法分析器）需要创建，以确保始终匹配文件的全部内容，不得在标记之间存在任何间隙，并为在其位置无效的字符生成特殊标记。词法分析器不应因无效字符而过早终止。

#### 示例
- Properties语言插件的JFlex定义文件
- 自定义语言支持教程：词法分析器

#### 标记类型
词法分析器的标记类型由`IElementType`实例定义。许多所有语言通用的标记类型在`TokenType`接口中定义。自定义语言插件应在适用的地方重用这些标记类型。对于所有其他标记类型，插件需要创建新的`IElementType`实例，并将其与使用该标记类型的语言关联。每次词法分析器遇到特定的标记类型时，应该返回相同的`IElementType`实例。

示例：Properties语言插件的标记类型

相关类型（例如关键字）可以使用`TokenSet`定义。所有语言的`TokenSet`应在一个专用的`$Language$TokenSets`类中进行分组，以便重用。

#### 嵌入语言
在词法分析器层面可以实现的一个重要功能是文件内混合语言，例如在某些模板语言中嵌入Java代码片段。如果一种语言支持将其片段嵌入到另一种语言中，则需要为可以嵌入的不同类型的片段定义变色龙标记类型，这些标记类型需要实现`ILazyParseableElementType`接口。包含语言的词法分析器需要将嵌入语言的整个片段作为一个单一的变色龙标记返回，该标记类型由嵌入语言定义。为了解析变色龙标记的内容，IDE将通过调用`ILazyParseableElementType.parseContents()`来调用嵌入语言的解析器。



### 实现解析器和PSI
编辑页面最后修改时间：2024年7月31日

在IntelliJ平台中解析文件是一个两步过程。

首先，构建抽象语法树（AST），定义程序的结构。AST节点由IDE内部创建，表示为`ASTNode`类的实例。每个AST节点都有一个关联的元素类型`IElementType`实例，这些元素类型由语言插件定义。文件的AST树的顶级节点需要具有扩展`IFileElementType`类的特殊元素类型。

AST节点与底层文档中的文本范围直接对应。AST的最底层节点与词法分析器返回的单个标记相匹配，而更高层次的节点则匹配多个标记片段。对AST树节点执行的操作，如插入、删除、重新排序节点等，会立即反映为对底层文档文本的更改。

第二步，在AST之上构建程序结构接口（PSI）树，添加语义和用于操作特定语言构造的方法。PSI树的节点由实现`PsiElement`接口的类表示，并由语言插件在`ParserDefinition.createElement()`方法中创建。文件的PSI树的顶级节点需要实现`PsiFile`接口，并在`ParserDefinition.createFile()`方法中创建。

### 示例：Properties语言插件的ParserDefinition

#### 在ParserDefinition中使用TokenSets
为了避免在初始化ParserDefinition扩展点实现时不必要的类加载，所有TokenSet返回值应使用专用`$Language$TokenSets`类中的常量。

PSI的生命周期有更详细的描述在基本原理部分。

IntelliJ平台提供了PSI实现的基本类，包括`PsiFileBase`（`PsiFile`的基本实现）和`ASTWrapperPsiElement`（`PsiElement`的基本实现）。

### 解析器的实现
虽然手动编码解析器是可能的，但我们强烈建议使用Grammar-Kit插件从BNF语法生成解析器和相应的PSI类。除了代码生成，它还提供各种编辑语法文件的功能：语法高亮、快速导航、重构等，以及与Gradle的集成。Grammar-Kit插件使用其自己的引擎构建，其源代码和文档可以在GitHub上找到。

对于重用现有的ANTLRv4语法，可以参考第三方库antlr4-intellij-adaptor。

语言插件通过`PsiParser`接口的实现提供解析器，该接口由`ParserDefinition.createParser()`返回。解析器接收`PsiBuilder`类的实例，用于从词法分析器获取标记流并保存正在构建的AST的中间状态。

解析器必须处理词法分析器返回的所有标记，直到流的结束（直到`PsiBuilder.getTokenType()`返回null）——即使这些标记根据语言语法无效。

解析器通过在从词法分析器接收到的标记流中设置标记对（`PsiBuilder.Marker`实例）来工作。每对标记定义AST树中单个节点的词法分析器标记范围。如果标记对嵌套在另一对中（起始后开始，结束前结束），它成为外部对的子节点。

当设置结束标记时，标记对和由其创建的AST节点的元素类型由调用`PsiBuilder.Marker.done()`方法指定。还可以在设置结束标记之前删除开始标记。`drop()`方法仅删除单个开始标记，不影响其后的任何标记，`rollbackTo()`方法删除开始标记及其后的所有标记，并将词法分析器位置回滚到开始标记。这些方法可用于实现解析时的前瞻。

`PsiBuilder.Marker.precede()`方法在进行从右到左的解析时非常有用，当您在读取更多输入之前不知道在特定位置需要多少标记时。例如，二元表达式`a+b+c`需要解析为`((a+b)+c)`。因此，在标记`a`的位置需要两个开始标记，但在读取标记`c`之前无法知道。当解析器到达`b`后的`+`标记时，它可以调用`precede()`在`a`位置复制开始标记，然后在`c`之后放置其匹配的结束标记。

### 示例：
- 自定义语言支持教程：语法和解析器（使用Grammar-Kit）
- Properties语言插件的简单`PropertiesParser`实现
- RegEx语言的复杂`RegExpParser`

### 空白和注释
`PsiBuilder`的一个重要特性是其处理空白和注释的方式。作为空白或注释处理的标记类型由`ParserDefinition`中的`getWhitespaceTokens()`和`getCommentTokens()`定义。`PsiBuilder`自动从其传递给`PsiParser`的标记流中省略空白和注释标记，并调整AST节点的标记范围，使得前导和尾随空白标记不包含在节点中。

大多数语言不需要覆盖`getWhitespaceTokens()`，该方法默认返回与语言无关的`TokenSet.WHITE_SPACE`。`ParserDefinition.getCommentTokens()`返回的标记集也用于查找TODO项。

### PSI的实现
通常，没有单一正确的方法来实现自定义语言的PSI，插件作者可以选择最方便用于使用PSI（错误分析、重构等）的PSI结构和方法集。

然而，为了支持重命名重构和查找用法等功能，自定义语言PSI实现需要使用一个基础接口。每个可以重命名或引用的元素（类定义、方法定义等）都需要实现`PsiNamedElement`接口，具有`getName()`和`setName()`方法。

可以在`com.intellij.psi.util`包中找到若干用于实现和使用PSI的函数，特别是在`PsiUtilCore`和`PsiTreeUtil`类中。

使用内置工具和PsiViewer插件来探索和检查PSI。

请参阅“索引和PSI存根”了解高级主题。

### 语法和错误高亮
编辑页面最后修改时间：2024年7月31日

语法和错误高亮功能在多个层面上执行：词法分析器（Lexer）、解析器（Parser）和注解器/外部注解器（Annotator/External Annotator）。

#### 文本属性键（Text Attributes Key）
文本的特定范围应如何高亮是通过`TextAttributesKey`定义的。此类的实例为每种不同类型的项（如关键字、数字、字符串字面量等）创建，定义这些项的默认属性（例如，关键字加粗，数字为蓝色，字符串字面量为绿色并加粗）。多个`TextAttributesKey`项的高亮效果可以叠加，例如，一个键可以定义项的粗细度，另一个键可以定义其颜色。

要查看现有的`TextAttributesKey`，可以使用“跳转到颜色和字体”操作在编辑器中检查光标所在元素应用的`TextAttributesKey`。

#### 颜色设置
`TextAttributesKey`到编辑器中使用的特定属性的映射由`EditorColorsScheme`类定义。用户可以通过`Settings | Editor | Color Scheme`配置。通过提供在`com.intellij.colorSettingsPage`扩展点中注册的`ColorSettingPage`实现，可以自定义这些设置。要查找IDE中设置的外部名称，可以使用UI Inspector。

`File | Export | Files or Selection to HTML`功能使用与编辑器相同的语法高亮机制，因此它将自动适用于提供语法高亮的自定义语言。

### 示例：
- Properties语言插件的`ColorSettingsPage`
- 自定义语言支持教程：颜色设置页面

#### 词法分析器
第一级语法高亮基于词法分析器的输出，通过`SyntaxHighlighter`接口提供。语法高亮器为每种需要特殊高亮的标记类型返回`TextAttributesKey`实例。对于高亮词法分析错误，应使用`HighlighterColors.BAD_CHARACTER`。

### 示例：
- Properties语言插件的`SyntaxHighlighter`实现
- 自定义语言支持教程：语法高亮器

创建高亮代码示例可以使用`HtmlSyntaxInfoUtil`，例如用于文档中的示例代码。

#### 语义高亮
语义高亮提供额外的颜色层次，以改善视觉区分多个相关项目（例如，方法参数、本地变量）。在`com.intellij.highlightVisitor`扩展点中注册`RainbowVisitor`。颜色设置页面必须实现`RainbowColorSettingsPage`。

#### 解析器
第二级错误高亮在解析过程中发生。如果根据语言的语法特定的标记序列无效，可以使用`PsiBuilder.error()`方法高亮无效标记，并显示解释它们为何无效的错误消息。

#### 注解器
第三级高亮通过`Annotator`接口进行。插件可以在`com.intellij.annotator`扩展点中注册一个或多个注解器，这些注解器在后台高亮过程中被调用，以处理自定义语言PSI树中的元素。如果高亮数据需要调用外部工具，请使用`External Annotator`。

注解器不仅可以分析语法，还可以使用PSI分析语义，因此可以提供更复杂的语法和错误高亮逻辑。注解器还可以为其检测到的问题提供快速修复。当文件发生更改时，注解器会增量地处理PSI树中更改的元素。

不需要从索引中获取信息的注解器可以标记为`dumb aware`，以便在索引期间工作（例如，用于附加语法高亮）。

#### 错误/警告
要突出显示文本区域为警告或错误：

**2020.1及以后版本**
```java
holder.newAnnotation(HighlightSeverity.WARNING, "Invalid code") // 或 HighlightSeverity.ERROR
    .withFix(new MyFix(psiElement))
    .create();
```

**语法**
要应用附加语法高亮：

**2020.1及以后版本**
```java
holder.newSilentAnnotation(HighlightSeverity.INFORMATION)
    .range(rangeToHighlight)
    .textAttributes(MyHighlighter.EXTRA_HIGHLIGHT_ATTRIBUTE)
    .create();
```

### 示例：
- Properties语言插件的`Annotator`
- 自定义语言支持教程：注解器

#### 外部注解器
如果自定义语言使用外部工具来验证语言中的文件（例如，使用Xerces库进行XML模式验证），可以提供`ExternalAnnotator`接口的实现，并在`com.intellij.externalAnnotator`扩展点中注册它（必须指定语言属性）。

外部注解器的高亮优先级最低，仅在所有其他后台处理完成后调用。它使用相同的`AnnotationHolder`接口将外部工具的输出转换为编辑器高亮。

要跳过给定文件的特定外部注解器，可以在`com.intellij.daemon.externalAnnotatorsFilter`扩展点中注册`ExternalAnnotatorsFilter`。

要在dumb模式下运行外部注解器，可以将其标记为`dumb aware`（2023.3）。

### 控制高亮
可以在某些上下文中以编程方式抑制现有高亮，请参阅“控制高亮”。

要强制重新高亮所有打开的或特定的文件（例如，在更改插件特定设置后），使用`DaemonCodeAnalyzer.restart()`。

### 2024.1+版本 高亮运行顺序
检查和注解器不再顺序地在每个PsiElement上运行。相反，它们在所有相关的PSI上独立并行运行，带来以下后果：

- **独立的注解器**：注解器独立运行：如果一个注解器发现错误，它不再停止注解PsiElement的父级。实际上，现在有“更多”的高亮。
- **高亮范围**：尽可能接近相关PsiElement地生成高亮。例如，代替：
  ```java
  annotate(PsiFile) {
    <<highlight all relevant identifiers>>
  }
  ```
  使用这种方法：
  ```java
  annotate(PsiIdentifier) {
    <<highlight this identifier if it's relevant>>
  }
  ```
  后者的优点是：
- 执行更快的高亮，不必等到所有其他标识符都被访问
- 更快地删除过时的高亮——在访问标识符并且注解器不再生成高亮后立即生效

### 引用和解析
编辑页面最后修改时间：2024年7月31日

在实现自定义语言的PSI时，解析引用是一个非常重要且复杂的部分。解析引用使用户能够从PSI元素的使用（如访问变量、调用方法等）导航到该元素的声明（变量定义、方法声明等）。

该功能支持通过按下 `Ctrl/Cmd+B` 或在按住 `Ctrl/Cmd` 键的同时点击鼠标按钮调用的导航至声明或用法操作，也是实现查找用法、重命名重构和代码补全的前提。

视图 | 快速定义功能基于相同的机制，因此它自动适用于所有可以由语言插件解析的引用。要自定义弹出窗口中显示的具体文档范围（例如，包含“周围的”代码或注释），请提供在`com.intellij.lang.implementationTextSelectioner`扩展点中注册的`ImplementationTextSelectioner`。

#### PSI 引用
所有作为引用的PSI元素（适用于导航至声明或用法操作）需要实现`PsiElement.getReference()`方法，并从该方法返回`PsiReference`实现。`PsiReference`可以由与`PsiElement`相同的类实现，也可以由不同的类实现。一个元素也可以包含多个引用（例如，一个字符串字面量可以包含多个有效的全限定类名的子字符串），在这种情况下，它可以实现`PsiElement.getReferences()`并返回引用的数组。为了优化`PsiElement.getReferences()`的性能，可以考虑实现`HintedReferenceHost`以提供附加提示。

`PsiReference`接口的主要方法是`resolve()`，它返回引用指向的元素，如果无法解析到有效元素（例如，它指向一个未定义的类），则返回`null`。解析的元素应实现`PsiNamedElement`接口。为了实现更高级的功能，尽可能实现`PsiNameIdentifierOwner`而不是`PsiNamedElement`。

虽然引用元素和被引用元素都可能有一个名称，但只有引入名称的元素（例如，定义`int x = 42`）需要实现`PsiNamedElement`。使用点的引用元素（例如表达式`x + 1`中的`x`）不应实现`PsiNamedElement`，因为它没有名称。

`resolve()`方法的对应方法是`isReferenceTo()`，它检查引用是否解析到指定的元素。后者可以通过调用`resolve()`并将结果与传递的PSI元素进行比较来实现，但也可以进行额外的优化（例如，只有在元素文本与引用的文本相等时才执行树遍历）。

#### 实现解析逻辑
实现解析支持的基础接口包括`PsiScopeProcessor`接口和`PsiElement.processDeclarations()`方法。这些接口包含了一些额外的复杂性，这对于大多数自定义语言是不必要的（例如支持替换Java泛型类型）。但是，如果自定义语言可以引用Java代码，则需要它们的支持。如果不需要Java互操作性，插件可以不使用标准接口并提供自己的不同的解析实现。

标准辅助类的解析实现包含以下组件：

1. 一个实现`PsiScopeProcessor`接口的类，该类收集引用的可能声明并在成功完成解析时停止解析过程。需要实现的主要方法是`execute()`，它在解析过程中遇到每个声明时调用，并在需要继续解析时返回`true`，如果找到声明则返回`false`。`getHint()`和`handleEvent()`方法用于内部优化，通常在自定义语言的`PsiScopeProcessor`实现中可以留空。

2. 从引用位置向上遍历PSI树的函数，直到解析成功完成或达到解析范围的终点。如果引用的目标位于不同的文件中，可以使用`FilenameIndex.getFilesByName()`（如果已知文件名）或通过遍历项目中的所有自定义语言文件（通过从`ProjectRootManager.getFileIndex()`获取的`ProjectFileIndex`接口中的`iterateContent()`）找到文件。

3. 在PSI树遍历过程中调用`processDeclarations()`方法的各个PSI元素。如果PSI元素是一个声明，它会将自身传递给传递给它的`PsiScopeProcessor`的`execute()`方法。此外，根据语言的作用域规则，PSI元素还可以将`PsiScopeProcessor`传递给其子元素。

#### 解析到多个目标
允许引用解析到多个目标的`PsiReference`接口扩展是`PsiPolyVariantReference`接口。引用解析到的目标由`multiResolve()`方法返回。对于此类引用的导航至声明或用法操作允许用户在弹出窗口中选择导航目标。`multiResolve()`的实现也可以基于`PsiScopeProcessor`，并收集引用的所有有效目标，而不是在找到第一个有效目标时停止。

### 2022.2+ 额外高亮
实现`HighlightedReference`以为非显而易见的位置（例如字符串字面量内部）添加额外的高亮。这些引用将自动使用`Settings | Editor | Color Scheme | Language Defaults`中的字符串 | 高亮引用文本属性进行高亮。

### 导航 (Navigation)
编辑页面最后修改时间：2024年7月31日

此API目前处于开发阶段，因此处于实验状态。

`Navigate | Declaration or Usages` 操作由多个步骤组成。

#### 直接导航
直接导航是从一个`PsiElement`导航到另一个`PsiElement`，例如从Java中的`break`关键字导航到循环的结束处，而无需显示任何弹出窗口。

要提供用于直接导航的`PsiElement`，请在`com.intellij.lang.directNavigationProvider`扩展点中实现并注册`DirectNavigationProvider`。

#### 符号导航
如果光标下没有可用的直接导航，那么IntelliJ平台会继续进行符号导航。在此步骤中，IntelliJ平台根据通过解析引用获得的目标符号计算导航目标。如果符号有多个目标或多个导航目标，则IDE会显示导航弹出窗口以请求用户选择导航的位置。

`NavigationTarget`本质上是`Navigatable`和`TargetPresentation`实例的一个对（导航目标和弹出窗口中显示的内容）。

要通过符号提供导航目标，可以：

- 在`com.intellij.symbolNavigation`扩展点中实现并注册`SymbolNavigationProvider`；
- 或在符号中实现`NavigatableSymbol`。

#### 显示用法
如果没有可用的导航目标，那么IntelliJ平台会开始查找通过解析引用或从声明中获得的目标符号的用法。



### 查找用法 (Find Usages)
编辑页面最后修改时间：2024年7月31日

#### 概述
`Find Usages` 操作是一个多步骤的过程，每个步骤都需要自定义语言插件的参与。自定义语言插件通过在`com.intellij.lang.findUsagesProvider`扩展点中注册`FindUsagesProvider`的实现以及通过使用`PsiNamedElement`和`PsiReference`接口的PSI实现参与`Find Usages`过程。

对于函数参数和局部变量等情况，考虑重写`PsiElement.getUseScope()`方法以返回一个更狭窄的范围。例如，返回最近函数定义的范围可以显著减少需要解析的文件数量和需要解析的引用数量。

#### 查找用法的步骤
1. **构建词索引**：
   在`Find Usages`操作可以被调用之前，IDE会为每个自定义语言文件中的单词构建一个索引。使用`FindUsagesProvider.getWordsScanner()`返回的`WordsScanner`实现，加载每个文件的内容并传递给词扫描器和单词消费者。词扫描器将文本拆分为单词，为每个单词定义上下文（代码、注释或字面量），并将单词传递给消费者。实现词扫描器的最简单方法是使用`DefaultWordsScanner`实现，向其传递作为标识符、字面量和注释处理的词法分析器标记类型集合。默认词扫描器将使用词法分析器将文本拆分为标记，并处理将注释和字面量标记的文本拆分为单个单词。

2. **定位PSI元素**：
   当用户调用`Find Usages`操作时，IDE会定位要查找其引用的PSI元素。光标处的PSI元素（光标位置的标记的直接树父级）必须是`PsiNamedElement`或解析为`PsiNamedElement`的`PsiReference`。词缓存将用于搜索`PsiNamedElement.getName()`方法返回的文本。此外，如果`PsiNamedElement`的文本范围包括`getName()`返回的标识符之外的其他文本（例如，如果`PsiNamedElement`表示JavaScript函数并且其文本范围包括"function"关键字以及函数名称），则必须重写`getTextOffset()`方法，并返回元素文本范围内名称标识符的起始偏移量。

3. **检查可用性**：
   一旦定位到元素，IDE会调用`FindUsagesProvider.canFindUsagesFor()`来询问插件`Find Usages`操作是否适用于特定元素。

4. **显示对话框**：
   当向用户显示`Find Usages`对话框时，调用`FindUsagesProvider.getType()`和`FindUsagesProvider.getDescriptiveName()`来确定应如何向用户展示元素。

5. **递归解析PSI树**：
   对于包含搜索词的每个文件，IDE会构建PSI树并递归下降。每个元素的文本被分解成单词然后被扫描。如果元素被索引为标识符，则每个单词都会检查是否是解析为搜索元素的`PsiReference`。如果元素被索引为注释或字面量，并且启用了在注释或字面量中搜索的功能，则检查单词是否等于搜索元素的名称。

6. **显示结果**：
   收集到的用法后，结果显示在用法面板中。对于每个找到的元素，显示的文本取自`FindUsagesProvider.getNodeText()`方法。要按类型分组结果，可以实现`UsageTypeProvider`并在`com.intellij.usageTypeProvider`扩展点中注册，以提供自定义或预定义的`UsageType`。

#### 示例
- Properties语言插件中`FindUsagesProvider`的实现
- 自定义语言支持教程：查找用法

#### 结果分组
为了在`Find Usages`工具窗口的标题中正确显示找到的元素的标题，需要提供`ElementDescriptionProvider`接口的实现。在这种情况下传递给提供程序的`ElementDescriptionLocation`将是`UsageViewLongNameLocation`的一个实例。

**示例**: Properties语言插件的`ElementDescriptionProvider`。



### 重命名重构 (Rename Refactoring)
编辑页面最后修改时间：2024年7月31日

重命名重构操作与查找用法非常相似。它使用相同的规则来定位要重命名的元素，以及相同的单词索引来查找可能引用该元素的文件。

在执行重命名重构时，会调用被重命名元素的`PsiNamedElement.setName()`方法，并调用所有引用该元素的`PsiReference.handleElementRename()`方法。这些方法基本上执行相同的操作：将PSI元素的底层AST节点替换为包含用户输入的新文本的节点。从头开始创建完全正确的AST节点相当棘手。因此，最简单的替换节点方法是创建一个包含必要节点的自定义语言虚拟文件，构建解析树并从中提取所需的节点。

### 示例：
- Properties语言插件中的`setName()`实现
- 自定义语言支持教程：引用贡献者

如果重命名的引用扩展了`PsiReferenceBase`，则通过调用`ElementManipulator.handleContentChange()`执行重命名，该方法负责处理内容更改并计算元素内部引用的文本范围。

要禁用特定元素的重命名，可以为类型为`T`的`PsiElement`实现`com.intellij.openapi.util.Condition<T>`并在`com.intellij.vetoRenameCondition`扩展点中注册。

### 名称验证
`NamesValidator`允许插件根据自定义语言规则检查用户在重命名对话框中输入的名称是否为有效标识符（而不是关键字）。如果插件未提供此接口的实现，则使用Java规则验证标识符。`NamesValidator`的实现注册在`com.intellij.lang.namesValidator`扩展点。

**示例：** Properties语言插件的`PropertiesNamesValidator`

另一种检查方式是`RenameInputValidator`，与`NamesValidator`不同，它允许更灵活地根据`isInputValid()`方法中定义的规则检查输入名称的正确性。

要确定此验证器适用于哪些元素，覆盖`getPattern()`方法，返回要验证的元素的模式。

**示例：** 验证YAML语言锚点名称的`YAMLAnchorRenameInputValidator`

`RenameInputValidator`可以扩展为`RenameInputValidatorEx`以覆盖默认错误消息。`getErrorMessage()`方法应返回无效名称时的自定义错误消息，否则返回null。

请注意，`getErrorMessage()`仅在所有`RenameInputValidator`都接受新名称的`isInputValid()`中有效，并且该名称是元素语言的有效标识符时才有效。

**示例：** 验证YAML语言键的`YamlKeyValueRenameInputValidator`

`RenameInputValidator`或`RenameInputValidatorEx`的实现注册在`com.intellij.renameInputValidator`扩展点。

### 自定义重命名UI和工作流程
可以在多个层次上进一步自定义重命名重构处理。提供`RenameHandler`接口的自定义实现允许完全替换重命名重构的UI和工作流程，并支持重命名根本不是`PsiElement`的内容。

**示例：** 在Properties语言插件中重命名资源包的`RenameHandler`

如果您对标准UI没有问题，但需要扩展默认的重命名逻辑，可以提供`RenamePsiElementProcessor`接口的实现。这允许您：

- 重命名与调用操作的元素不同的元素（例如超级方法）
- 一次重命名多个元素（如果它们的名称根据您的语言逻辑链接）
- 检查名称冲突（现有名称等）
- 自定义如何执行代码引用或文本引用的搜索

**示例：** Properties插件语言中重命名属性的`RenamePsiElementProcessor`

### 结构视图 (Structure View)
编辑页面最后修改时间：2024年7月31日

结构视图是用于展示文件内容结构的视图。特定文件类型的结构视图实现可以在多个层面上进行定制。如果自定义语言插件提供了`StructureView`接口的实现，则可以完全替换标准结构视图实现，提供自定义的用户界面组件。然而，对于大多数语言，复用IntelliJ平台提供的标准结构视图实现是足够的。

要修改现有的结构视图（例如，添加/过滤内置语言支持的节点），可以使用在`com.intellij.lang.structureViewExtension`扩展点中注册的`StructureViewExtension`。

#### 起点
结构视图的起点是`PsiStructureViewFactory`接口，它在`com.intellij.lang.psiStructureViewFactory`扩展点中注册。

### 示例：
- Properties语言插件的`PsiStructureViewFactory`
- 自定义语言支持教程：结构视图

要复用IntelliJ平台的结构视图实现，插件应从其`PsiStructureViewFactory.getStructureViewBuilder()`方法返回一个`TreeBasedStructureViewBuilder`。作为构建器模型，插件可以指定`TextEditorBasedStructureViewModel`的子类，并通过重写该子类的方法来自定义特定语言的结构视图。

### 示例：
- Properties语言插件的`StructureViewModel`

主要需要重写的方法是`getRoot()`，该方法返回实现`StructureViewTreeElement`接口的类的实例。此接口没有标准实现，因此插件需要完全实现它。

结构视图树通常是PSI树的部分镜像。在`StructureViewTreeElement.getChildren()`的实现中，插件可以指定特定PSI树节点的哪些子元素需要在结构视图中表示。另一个重要的方法是`getPresentation()`，它可用于自定义在结构视图中表示元素时使用的文本、属性和图标。

`StructureViewTreeElement.getChildren()`的实现需要与`TextEditorBasedStructureViewModel.getSuitableClasses()`方法相匹配。后者方法返回一个由`PsiElement`派生的类数组，这些类可以作为结构视图元素显示。它用于在结构视图首次打开或启用“自动从源滚动”选项时选择与光标位置匹配的结构视图项目。

### 示例：
- Properties语言插件的`StructureViewTreeElement`

### 导航栏 (Navigation Bar)
编辑页面最后修改时间：2024年7月31日

导航栏实现用于自定义和扩展导航栏的结构。

导航栏扩展的起点是`NavBarModelExtension`接口，该接口在`com.intellij.navbar`扩展点中注册。

要复用IntelliJ平台的实现，可以扩展以下两个类之一：

- `DefaultNavBarExtension`
- `StructureAwareNavBarModelExtension`

#### 默认导航栏
`DefaultNavBarExtension`是任何文件的导航栏的基本实现。如果您想为您的语言创建一个简单的导航栏，可以更改文件夹或文件的显示，请继承此类。

在这种情况下，您可能只需要重写以下两个方法：

- `getPresentableText()` – 返回传递给它的导航栏部分元素的字符串表示。
- `getIcon()` – 返回传递给它的导航栏部分的图标。

#### 结构感知导航栏
`StructureAwareNavBarModelExtension`是一个高级实现，提供在导航栏中显示特定文件元素（例如类名、函数名等）的能力。例如，可以在导航栏中显示当前光标位置的类名。如果您想为您的语言添加导航栏支持，并支持显示特定的文件元素，请继承此类。

不要忘记实现结构视图，这是构建文件结构模型的必要条件，导航栏基于该模型显示特定元素。

在这种情况下，除了前面描述的两个方法之外，您还需要重写`getLanguage()`方法，该方法返回此扩展将适用的语言实例。

`adjustElement()`方法允许您修改导航栏元素。例如，当您希望在光标位于附加到类的注释中时在导航栏中显示类时，可以使用该方法。

除非您想编写整个`NavBarModelExtension`接口的自定义实现，否则您可能不需要重写其他方法。

请注意，实现`com.intellij.ide.structureView.TextEditorBasedStructureViewModel`的结构视图模型类上的`getSuitableClasses()`方法必须返回您希望在导航栏中显示的所有元素类型。

**示例：** 自定义语言支持教程：结构感知导航栏

### 转到类和转到符号 (Go to Class and Go to Symbol)
编辑页面最后修改时间：2024年7月31日

自定义语言插件可以提供其项目，以包含在用户选择“导航 | 类”或“导航 | 符号”操作时显示的列表中。

要实现这些功能，需要分别提供`ChooseByNameContributorEx`接口的实现，并将其注册到`com.intellij.gotoClassContributor`和`com.intellij.gotoSymbolContributor`扩展点中。

每个`ChooseByNameContributorEx`实现必须提供以下方法：

#### `processNames(@NotNull Processor<? super String> processor, @NotNull GlobalSearchScope scope, @Nullable IdFilter filter)`
此方法向处理器提供在指定范围内可用的完整名称列表，IDE将根据用户在对话框中输入的文本进行过滤。建议使用索引和PSI存根获取匹配的候选者，以提高性能。

#### `processElementsWithName(String name, Processor<? super NavigationItem> processor, FindSymbolParameters parameters)`
此方法向处理器提供与给定名称和参数匹配的`NavigationItem`实例（通常是`PsiElement`）列表。处理的`NavigationItem`指定了当从列表中选择特定项目时跳转的目的地。

### 示例：
- 自定义语言支持教程：转到符号贡献者

这些实现可以极大地提升自定义语言插件在IntelliJ平台中的导航体验，使用户能够轻松查找和跳转到相关的类和符号。

### 参数信息 (Parameter Info)
编辑页面最后修改时间：2024年7月31日

自定义语言插件可以使用在`com.intellij.codeInsight.parameterInfo`扩展点中注册的`ParameterInfoHandler`来显示方法和函数调用中的参数信息。这是一种方便的方式，可以在编辑器中直接显示类型签名弹出窗口，而无需查阅文档。如果可用，IDE可以在短暂延迟后自动显示此弹出窗口，或者可以通过`View | Parameter Info`显式调用。

参数信息是动态的，可以在光标移动或输入其他代码时更新显示的信息。这允许高亮条目或标记光标位置的当前参数。因此，`ParameterInfoHandler`扩展点的接口包括用于初始收集显示光标位置参数信息所需信息的方法，以及在编辑期间更新显示内容的方法。

### 实现
语言作者实现`ParameterInfoHandler`接口，该接口有两个类型参数：`ParameterOwner`和`ParameterType`。在接下来的解释中，我们假设`ParameterOwner`是表示语言中的函数调用的PSI元素，`ParameterType`代表（可能是多个）函数定义。

此外，`ParameterInfoHandler`使用了几个上下文类型，这些类型是可变的，并用于调整显示的内容和方式。这些上下文包括`CreateParameterInfoContext`、`UpdateParameterInfoContext`和`ParameterInfoUIContext`，它们都继承自`ParameterInfoContext`。

#### 初始阶段
当当前没有显示参数信息，并且该信息被自动或用户手动调用时，初始阶段开始。

1. 调用`findElementForParameterInfo()`方法。重写此方法时，语言作者使用提供的`CreateParameterInfoContext`访问当前编辑器的文件和偏移量。目标是识别`ParameterOwner`，即当前偏移量处的函数调用（如果存在）。建议将实际的函数调用搜索提取到一个单独的方法中，以便后续重用。`findElementForParameterInfo()`实现应找到所有匹配的函数定义，并使用上下文参数的`setItemsToShow()`方法存储它们。

2. 如果返回的函数调用元素有效，则调用`showParameterInfo()`方法。此方法的实现通常只调用`CreateParameterInfoContext`的`showHint()`，提供弹出窗口应出现的偏移量。

3. 对于步骤1中的每个要显示的项目，调用`updateUI()`方法。由于此方法在EDT（事件分派线程）上运行，因此不允许进行繁重的工作，它应仅使用提供的`ParameterInfoUIContext`的`setUIComponentEnabled()`或`setupUIComponentPresentation()`更新UI表示。

接下来调用以下方法，这些方法将在下一阶段解释：`findElementForUpdatingParameterInfo()`、`updateParameterInfo()`、`updateUI()`。

#### 更新阶段
当显示参数信息弹出窗口并且用户输入内容或移动光标时，显示的信息会更新。这允许，例如，高亮显示使用不同参数的函数调用，或简单地将参数信息框移动到光标附近。因此，当用户移动光标或输入内容时，会发生以下情况：

1. 调用`syncUpdateOnCaretMove()`方法。

2. 调用`findElementForUpdatingParameterInfo()`方法，它应找到更改光标位置的正确函数调用（`ParameterOwner`）。如果找不到合适的元素，或者它与提供的`UpdateParameterInfoContext`的`getParameterOwner()`不同，返回`null`。如果返回`null`，则调用`dispose()`方法。

3. 调用`processFoundElementForUpdatingParameterInfo()`方法，该方法允许对`UpdateParameterInfoContext`进行额外调整。默认情况下，此方法不执行任何操作，通常不需要实现它。

4. 调用`updateParameterInfo()`方法。许多实现仅在此处调用`UpdateParameterInfoContext`的`setParameterOwner()`。

5. 调用上下文中`getItemsToShow()`数组中的每个项目的`updateUI()`方法，这些项目在初始阶段收集。

#### 进一步提示
语言作者可以实现`ParameterInfoHandlerWithTabActionSupport`，通过按Tab键在参数位置之间跳转，扩展参数信息功能。对于如在函数调用中查找当前参数索引等重复任务，`ParameterInfoUtils`提供了一些有用的函数。

另外，检查继承自`ParameterInfoContext`的所有上下文接口也很有帮助，它们可以在`com.intellij.lang.parameterInfo`包中找到，这些接口提供了有关不同阶段中可以访问和更改的参数信息的数据。

具有默认实现的`ParameterInfoHandler`方法通常可以忽略。`syncUpdateOnCaretMove()`和`supportsOverloadSwitching()`由IntelliJ平台内部使用，不需要插件实现。`dispose()`方法在当前显示的参数信息无效并销毁时调用。只有当空白在语言中重要时，才应该实现`isWhitespaceSensitive()`，该方法在`ParameterInfoControllerBase`的`getCurrentOffset()`方法中使用。

请注意，当实现标记为`dumb aware`时，参数信息在索引期间工作（使用不完整的数据）。建议适应dumb模式的测试，因为结果可能令人惊讶，可能需要对处理程序进行更多更改以获得更好的结果。

最后，语言作者应注意全局`CodeInsightSettings#SHOW_FULL_SIGNATURES_IN_PARAMETER_INFO`设置，该设置可用于呈现与默认IDE行为一致的结果。例如，对于Java，如果启用此设置，IDE将在参数信息中显示方法/函数的完整签名。

#### 示例
IntelliJ平台中现有的适度复杂的`ParameterInfoHandler`实现可以作为参考：

- `XPathParameterInfoHandler`
- `XmlParameterInfoHandler`

可以使用IntelliJ平台探索工具发现第三方插件的实现。两个示例是：

- Rust插件的`RsParameterInfoHandler`
- TeXiFy插件的`LatexParameterInfoHandler`

### 内嵌提示 (Inlay Hints)
编辑页面最后修改时间：2024年7月31日

内嵌提示是在编辑器中直接渲染的小块信息，为开发人员提供额外的代码洞察，而不会打断工作流程。一个众所周知的例子是参数提示，通常显示函数参数的名称，如其声明中所示。内嵌提示与显示所有可能重载函数的参数类型的参数信息密切相关，但后者以覆盖代码的弹出窗口形式出现。

内嵌提示非常灵活，广泛应用于IntelliJ平台。例如，以下是使用内嵌提示的常见场景：

- Java 使用内嵌提示在Java链式方法调用中显示类型注释。
- Kotlin 在范围表达式中使用内嵌提示显示，例如小于号或小于等于号，以告知开发人员区间是包含的还是排除的。
- 在版本控制项目中，代码的作者通过内嵌提示显示。

### 实现
内嵌提示的主要特征是它在编辑器中的显示方式：

- **inline** - 内嵌提示显示在代码标记之间
- **block** - 内嵌提示显示在代码块上方

根据要求和目标IntelliJ平台版本，有几个扩展点可供选择，以实现内嵌提示。

可以使用UI Inspector检查IDE中现有的内嵌提示。相应的条目也可在`Settings | Editor | Inlay Hints`中查看。

#### 内嵌参数提示提供者
内嵌参数提示是简单的字符串内嵌提示，放置在方法和函数调用中的参数名称之前。无法提供高级的内嵌参数提示的展示和行为。

要提供内嵌参数提示，请实现`InlayParameterHintsProvider`并在`com.intellij.codeInsight.parameterNameHints`扩展点注册。`InlayParameterHintsProvider`的API文档详细解释了所有方法的基本原理。

**示例：**
- `GroovyInlayParameterHintsProvider` - 在Groovy代码中显示参数提示
- `KotlinInlayParameterHintsProvider` - 在Kotlin代码中显示参数提示

要在特定位置抑制内嵌参数提示，请实现`ParameterNameHintsSuppressor`并在`com.intellij.codeInsight.parameterNameHintsSuppressor`扩展点中注册。

#### 声明式内嵌提示提供者 (2023.1+)
声明式内嵌提示是可以容纳可展开的可点击项目列表的内嵌文本提示。请注意，由于其与UI无关的设计，API在展示自定义上有一定限制，这使得它可以被不同的前端技术（不仅限于Swing）利用。

要提供声明式内嵌提示，请实现声明式`InlayHintsProvider`并在`com.intellij.codeInsight.declarativeInlayProvider`扩展点注册。详细信息请参阅API文档。

**示例：**
- `JavaImplicitTypeDeclarativeInlayHintsProvider` - 在Java代码中显示用`var`声明的变量的推断类型
- `JavaMethodChainsDeclarativeInlayProvider` - 在Java代码中显示方法调用链中的方法返回类型

#### 代码视觉提供者 (Code Vision Provider) (2022.1+)
此API仍处于实验状态，可能会更改而不保持向后兼容性。

代码视觉提供者允许为类、方法、字段等元素提供块内嵌提示。如果单个元素提供多个提示，则所有提示将显示在同一行以节省垂直空间。

代码视觉提示可以显示在元素上方，或在行尾的右侧。用户可以在`Settings | Editor | Inlay Hints | Code vision`中通过选择“默认位置”复选框或在特定提供者条目中选择“位置”来配置。

有三个扩展点可用于实现代码视觉提供者：

- 在`com.intellij.codeInsight.daemonBoundCodeVisionProvider`扩展点中注册的`DaemonBoundCodeVisionProvider`
- 在`com.intellij.codeInsight.codeVisionProvider`扩展点中注册的`CodeVisionProvider`
- 在`com.intellij.config.codeVisionGroupSettingProvider`扩展点中注册的`CodeVisionGroupSettingProvider`

当代码视觉条目与PSI相关时，应使用`DaemonBoundCodeVisionProvider` API，以便在PSI更改时计算的值无效并重新计算。

当显示的信息不依赖于PSI时，应使用`CodeVisionProvider` API。

`CodeVisionGroupSettingProvider`对于在设置中显示代码视觉提供者的名称和描述是必要的。`groupId`必须与`CodeVisionProvider`实现中指定的值匹配；如果未指定，则默认为`id`。`groupName`是在代码视觉组中显示的名称，描述将显示在右侧详细信息面板中。

**示例：**
- `JavaInheritorsCodeVisionProvider` - 显示Java类或方法继承者的数量。点击内嵌提示会打开继承者列表。此提供者是`DaemonBoundCodeVisionProvider`。
- `JavaReferencesCodeVisionProvider` - 显示Java类或成员的用法数量。点击内嵌提示会打开用法列表或导航到唯一的用法。此提供者是`DaemonBoundCodeVisionProvider`。
- `VcsCodeVisionProvider` - 显示给定元素（如类或方法）的作者，基于VCS信息。此提供者是`CodeVisionProvider`。

#### 内嵌提示提供者
内嵌提示提供者允许实现带有自定义展示和行为的内嵌提示和块内嵌提示。有关详细信息，请参阅API文档。

### 弃用通知
在2023.1及更高版本中，建议使用声明式内嵌提示提供者实现内嵌提示。

在2022.1及更高版本中，建议使用代码视觉提供者实现块内嵌提示。

要提供内嵌提示，请实现`InlayHintsProvider`并在`com.intellij.codeInsight.inlayProvider`扩展点注册。有关详细信息，请参阅API文档。

**示例：**
- `GroovyLocalVariableTypeHintsInlayProvider` - 在Groovy代码中显示局部变量类型
- `MarkdownTableInlayProvider` - 装饰Markdown文件中的表格

对于更复杂的示例，请参阅`KotlinLambdasHintsProvider`及其父类和所有实现。

#### 进一步提示
转到`Settings | Editor | Inlay Hints`（产品帮助），查看已实现的内嵌提示。

要支持多语言的单一类型内嵌提示，请参阅`declarative InlayHintsProviderFactory`（2023.1+）或`InlayHintsProviderFactory`（2023.1之前）。

有关测试内嵌提示的信息，请参阅`InlayHintsProviderTestCase`和`InlayParameterHintsTest`。



### 附加的次要功能 (Additional Minor Features)
编辑页面最后修改时间：2024年7月31日

以下列出了常用的一些次要功能，这些功能通常通过插件的`plugin.xml`文件中指定的扩展点来实现：

### 扩展点：com.intellij.lang.braceMatcher — 括号匹配
**接口/类**: `PairedBraceMatcher`  
描述：返回一个括号对的数组(`BracePair`)，指定开括号和闭括号的字符及其词法分析器的标记类型。某些类型的括号可以标记为结构性括号，它们比常规括号优先级更高，即使之间有不匹配的不同类型括号，它们也会相互匹配。

### 扩展点：com.intellij.lang.quoteHandler — 引号处理
**接口/类**: `QuoteHandler`  
描述：支持插入配对引号功能。在大多数情况下，`SimpleTokenSetQuoteHandler`基类实现就足够了。

### 扩展点：com.intellij.lang.commenter — 注释代码
**接口/类**: `Commenter`  
描述：返回行注释的前缀，以及块注释的前缀和后缀（如果语言支持）。对于更复杂的逻辑，可以使用`SelfManagingCommenter`。

- 示例：Properties语言插件的`Commenter`

### 扩展点：com.intellij.lang.foldingBuilder — 代码折叠
**接口/类**: `FoldingBuilder`  
描述：返回可折叠文本范围的列表（作为`FoldingDescriptor`对象数组），以及在折叠时显示的替换文本和每个折叠区域的默认状态（折叠或展开）。

### 扩展点：com.intellij.joinLinesHandler — 连接行
**接口/类**: `JoinLinesHandlerDelegate`  
描述：允许扩展支持智能/语义编辑（例如，字符串字面量分成多行）。

### 扩展点：com.intellij.lang.smartEnterProcessor — 智能回车
**接口/类**: `SmartEnterProcessor`  
描述：处理编辑（例如，自动完成缺少的分号/括号）。

### 扩展点：com.intellij.moveLeftRightHandler — 向左/右移动元素
**接口/类**: `MoveElementLeftRightHandler`  
描述：从`MoveElementLeftRightHandler`返回给定元素的子元素，用于“代码 | 向左/右移动元素”，例如方法调用参数。或者，在PSI元素中实现`PsiListLikeElement`。

### 扩展点：com.intellij.nameSuggestionProvider — 命名建议
**接口/类**: `NameSuggestionProvider`  
描述：为给定元素提供名称建议，例如用于重命名重构。

### 扩展点：com.intellij.highlightUsagesHandlerFactory — 语义高亮使用
**接口/类**: `HighlightUsagesHandlerFactory`  
描述：允许高亮显示，例如退出点或异常。

### 扩展点：com.intellij.indexPatternSearch — 额外的搜索位置
**接口/类**: `IndexPatternSearch`  
描述：通过`com.intellij.indexPatternSearch`扩展点注册，提供额外的搜索位置。

### 扩展点：com.intellij.declarationRangeHandler — 上下文信息
**接口/类**: `DeclarationRangeHandler`  
描述：为自定义语言提供视图上下文信息，基于`TreeBasedStructureViewBuilder`构建的结构视图实现。

### 扩展点：com.intellij.referenceInjector — 引用注入
**接口/类**: `ReferenceInjector`  
描述：允许用户将预定义的引用（例如“编码”、“文件引用”）注入到`PsiLanguageInjectionHost`元素中（需要IntelliLang插件）。

### 扩展点：com.intellij.colorProvider — 颜色预览/选择器
**接口/类**: `ElementColorProvider`  
描述：为包含颜色信息的元素渲染装订线图标。

### 扩展点：com.intellij.include.provider — 文件包含
**接口/类**: `FileIncludeProvider`  
描述：提供包含语句解析为文件的信息（例如XML中的`<xi:include>`）。然后可以通过`FileIncludeManager`获取包含/被包含的文件。

### 扩展点：com.intellij.breadcrumbsInfoProvider — 面包屑
**接口/类**: `BreadcrumbsProvider`  
描述：允许语言特定的面包屑。粘性行功能也使用此数据。

### 扩展点：com.intelllij.completion.plainTextSymbol — 纯文本补全
**接口/类**: `PlainTextSymbolCompletionContributor`  
描述：提供一种简单的方法从文件中提取查找元素，以便用户在例如VCS提交消息等纯文本编辑器中拥有补全功能。

### 扩展点：com.intellij.listSplitJoinContext — 分割和连接列表构造
**接口/类**: `ListSplitJoinContext`  
描述：需要实现以定义在语言中分割和连接列表构造的确切行为。UI将在适当位置显示此扩展点的实现作为意图操作。开发者可以使用`DefaultListSplitJoinContext`中的抽象类进行实现。

- 示例：`XmlAttributesSplitJoinContext`

### 扩展点：com.intellij.suggestedRefactoringSupport — 建议的重命名和更改签名重构
**接口/类**: `SuggestedRefactoringSupport`  
描述：提供建议的重命名和更改签名重构功能，用于自定义语言。

- 示例：Kotlin的`KotlinSuggestedRefactoringSupport`

### 扩展点：com.intellij.readerModeMatcher — 阅读器模式
**接口/类**: `ReaderModeMatcher`  
描述：提供一种方式来确定文件是否以正确的模式显示：阅读器模式还是普通编辑模式。

### 扩展点：com.intellij.editorTabColorProvider — 编辑器和项目视图的背景色
**接口/类**: `EditorTabColorProvider`  
描述：允许修改特定文件的背景色。如果不需要访问索引，可以标记为`dumb aware`。

### 扩展点：com.intellij.editorTabTitleProvider — 编辑器选项卡的自定义名称和工具提示
**接口/类**: `EditorTabTitleProvider`  
描述：允许为编辑器选项卡标题指定自定义名称和工具提示。如果不需要访问索引，可以标记为`dumb aware`。

- 示例：`GradleEditorTabTitleProvider`，展示如何为Gradle文件的编辑器选项卡添加项目名称。

### 扩展点：com.intellij.problemHighlightFilter — 防止文件的错误高亮
**接口/类**: `ProblemHighlightFilter` 和 `com.intellij.problemFileHighlightFilter`  
描述：过滤出不应进行错误高亮的文件，因为它们可能位于当前项目范围之外。这些过滤器应当是允许的，只对已知不在范围内的文件进行过滤。

- 示例：`JavaProblemHighlightFilter`，`PyProblemFileHighlightFilter`

### 扩展点：com.intellij.ide.actions.QualifiedNameProvider — 提供元素的完全限定名称 (FQN)
**接口/类**: `QualifiedNameProvider`  
描述：提供例如复制和粘贴FQN引用的功能。实现需要提供从和到FQN的转换逻辑。

- 示例：`PyQualifiedNameProvider`

### 扩展点：com.intellij.openapi.roots.TestSourcesFilter — 将文件标记为测试文件
**接口/类**: `TestSourcesFilter`  
描述：即使文件不位于标记为测试根目录的目录中，也可以告知IDE该文件是测试文件。如果测试文件位于源文件旁边，可以使用此扩展点将它们标记为IDE中的测试文件。

### 扩展点：com.intellij.statementUpDownMover — 在编辑器中上下移动语句
**接口/类**: `StatementUpDownMover`  
描述：允许自定义上下移动语句的行为。这可以用于在编辑器中移动代码时保持代码的语法正确性，例如移动变量声明。

- 示例：`DeclarationMover`

如果您感兴趣的主题不在上述部分中，请通过下面的反馈表或其他渠道告知我们。请具体说明要添加的主题及其原因，并留下您的电子邮件以便我们需要更多细节时联系。感谢您的反馈！



### 自定义语言支持教程 (Custom Language Support Tutorial)

编辑页面最后修改时间：2024年7月31日

在本教程中，我们将添加对 `.properties` 语言及其在 Java 代码中的使用支持。

#### 导航本教程

IntelliJ 平台对自定义语言的支持在自定义语言支持部分有更深入的讨论。相应部分在本教程每页顶部的参考部分链接。

在页面上添加或更改的所有相关代码都在代码部分链接。

随附的测试自定义语言插件教程涵盖了功能测试；相应部分链接在测试部分。

#### 访问代码

本教程中使用的完整且完全可用的示例插件是 `simple_language_plugin` 代码示例。

请参阅代码示例了解如何构建和运行它。

这是一个循序渐进的教程，需要按顺序完成每个步骤：

1. **先决条件**
2. **语言和文件类型**
3. **语法和解析器**
4. **词法分析器和解析器定义**
5. **语法高亮和颜色设置页面**
6. **PSI 助手和实用程序**
7. **注解器**
8. **行标记提供者**
9. **补全贡献者**
10. **引用贡献者**
11. **查找用法提供者**
12. **折叠构建器**
13. **转到符号贡献者**
14. **结构视图工厂**
15. **结构感知导航栏**
16. **格式化器**
17. **代码样式设置**
18. **注释器**
19. **快速修复**
20. **文档**
21. **拼写检查**

在本教程中，我们将一步步指导您通过这些步骤，从最基础的自定义语言支持开始，到实现一些高级功能，如代码补全、语法高亮、代码格式化等。每一步都包括了详细的代码示例和说明，以帮助您理解和实现这些功能。

### 定义语言和文件类型 (Language and File Type)
编辑页面最后修改时间：2024年7月31日

在本教程中，我们将定义和注册一个名为“Simple”的自定义语言，并为其设置文件类型和图标。

#### 定义语言
在本教程中实现的语言名为“Simple”。以下是定义 `SimpleLanguage` 类的代码，它位于 `simple_language_plugin` 代码示例的 `org.intellij.sdk.language` 包中：

```java
public class SimpleLanguage extends Language {

  public static final SimpleLanguage INSTANCE = new SimpleLanguage();

  private SimpleLanguage() {
    super("Simple");
  }

}
```

#### 定义图标
Simple 语言的图标由 `SimpleIcons` 类定义。请参阅工作与图标的相关内容，以了解如何定义和使用图标。

```java
public class SimpleIcons {

  public static final Icon FILE = IconLoader.getIcon("/icons/jar-gray.png", SimpleIcons.class);

}
```

#### 定义文件类型
`SimpleFileType` 是通过继承 `LanguageFileType` 来定义的：

```java
public final class SimpleFileType extends LanguageFileType {

  public static final SimpleFileType INSTANCE = new SimpleFileType();

  private SimpleFileType() {
    super(SimpleLanguage.INSTANCE);
  }

  @NotNull
  @Override
  public String getName() {
    return "Simple File";
  }

  @NotNull
  @Override
  public String getDescription() {
    return "Simple language file";
  }

  @NotNull
  @Override
  public String getDefaultExtension() {
    return "simple";
  }

  @Override
  public Icon getIcon() {
    return SimpleIcons.FILE;
  }

}
```

#### 注册文件类型
在 `plugin.xml` 中通过 `com.intellij.fileType` 扩展点注册 Simple 语言文件类型，并关联 `*.simple` 扩展名：

```xml
<extensions defaultExtensionNs="com.intellij">
  <fileType
      name="Simple File"
      implementationClass="org.intellij.sdk.language.SimpleFileType"
      fieldName="INSTANCE"
      language="Simple"
      extensions="simple"/>
</extensions>
```

#### 运行项目
使用 Gradle 的 `runIde` 任务运行插件。

创建一个扩展名为 `.simple` 的空文件，IntelliJ IDEA 会自动将其与我们定义的语言相关联。请注意，在项目工具窗口中测试文件 `test.simple` 的 Simple 语言文件图标，以及编辑器选项卡中的图标。

通过这些步骤，您已经成功地为一个自定义语言定义了基础的文件类型支持，包括语言定义、文件类型定义和图标设置。接下来，您可以扩展和丰富该语言的功能，例如语法高亮、代码补全等。

### 语法和解析器 (Grammar and Parser)
编辑页面最后修改时间：2024年7月31日

要使 IntelliJ 平台解析 Simple 语言文件，需要基于 `IElementType` 定义标记和元素，并定义 Simple 语言的语法以生成解析器。

#### 定义标记类型
在 `org.intellij.sdk.language.psi` 包中通过继承 `IElementType` 创建 `SimpleTokenType` 类。

```java
public class SimpleTokenType extends IElementType {

  public SimpleTokenType(@NotNull @NonNls String debugName) {
    super(debugName, SimpleLanguage.INSTANCE);
  }

  @Override
  public String toString() {
    return "SimpleTokenType." + super.toString();
  }

}
```

#### 定义元素类型
在 `org.intellij.sdk.language.psi` 包中通过继承 `IElementType` 创建 `SimpleElementType` 类。

```java
public class SimpleElementType extends IElementType {

  public SimpleElementType(@NotNull @NonNls String debugName) {
    super(debugName, SimpleLanguage.INSTANCE);
  }

}
```

#### 定义语法
在 `org/intellij/sdk/language/Simple.bnf` 文件中定义 Simple 语言的语法。

```bnf
{
  parserClass="org.intellij.sdk.language.parser.SimpleParser"
  extends="com.intellij.extapi.psi.ASTWrapperPsiElement"
  psiClassPrefix="Simple"
  psiImplClassSuffix="Impl"
  psiPackage="org.intellij.sdk.language.psi"
  psiImplPackage="org.intellij.sdk.language.psi.impl"
  elementTypeHolderClass="org.intellij.sdk.language.psi.SimpleTypes"
  elementTypeClass="org.intellij.sdk.language.psi.SimpleElementType"
  tokenTypeClass="org.intellij.sdk.language.psi.SimpleTokenType"
}

simpleFile ::= item_*

private item_ ::= (property|COMMENT|CRLF)

property ::= (KEY? SEPARATOR VALUE?) | KEY
```

此语法定义了对语言的支持的灵活性。例如，上述语法允许属性可以有或没有键和值，这种灵活性使 IntelliJ 平台能够识别不正确定义的属性，并提供相应的代码分析和快速修复。

请注意，`elementTypeHolderClass` 属性中的 `SimpleTypes` 类指定的是通过“生成解析器代码”操作从语法生成的类的名称（参见生成解析器）；此时它还不存在。

#### 生成解析器
现在语法已定义，可以通过在 `Simple.bnf` 文件的上下文菜单中选择“生成解析器代码”生成带有 PSI 类的解析器。这一步会在项目的 `/src/main/gen` 文件夹中生成解析器和 PSI 元素。

也可以使用 Gradle Grammar-Kit 插件替代。

#### 添加生成的源文件根目录
要包括生成的源文件到 `/src/main/gen` 中，需要在项目的 Gradle 构建脚本中插入以下行来扩展项目的 `sourceSets`：

```groovy
sourceSets["main"].java.srcDirs("src/main/gen")
```

请参阅 `build.gradle.kts` 了解详细信息。

重新加载 Gradle 项目以使更改生效并构建项目。

通过这些步骤，我们已经定义了 Simple 语言的基础语法和解析器，这些元素将用于进一步开发语言的其他功能，例如语法高亮、代码补全和分析工具。

### 词法分析器和解析器定义 (Lexer and Parser Definition)
编辑页面最后修改时间：2024年7月31日

词法分析器定义了如何将文件的内容分解为标记，这是支持自定义语言功能的基础。最简单的方法是使用 JFlex 创建词法分析器。

#### 定义词法分析器 (Define a Lexer)
在 `org.intellij.sdk.language` 包中定义 Simple 语言词法分析器的规则，并创建 `Simple.flex` 文件。

```java
// Simple.flex
package org.intellij.sdk.language;

import com.intellij.lexer.FlexLexer;
import com.intellij.psi.tree.IElementType;
import org.intellij.sdk.language.psi.SimpleTypes;
import com.intellij.psi.TokenType;

%%

%class SimpleLexer
%implements FlexLexer
%unicode
%function advance
%type IElementType
%eof{  return;
%eof}

CRLF=\R
WHITE_SPACE=[\ \n\t\f]
FIRST_VALUE_CHARACTER=[^ \n\f\\] | "\\"{CRLF} | "\\".
VALUE_CHARACTER=[^\n\f\\] | "\\"{CRLF} | "\\".
END_OF_LINE_COMMENT=("#"|"!")[^\r\n]*
SEPARATOR=[:=]
KEY_CHARACTER=[^:=\ \n\t\f\\] | "\\ "

%state WAITING_VALUE

%%

<YYINITIAL> {END_OF_LINE_COMMENT}                           { yybegin(YYINITIAL); return SimpleTypes.COMMENT; }

<YYINITIAL> {KEY_CHARACTER}+                                { yybegin(YYINITIAL); return SimpleTypes.KEY; }

<YYINITIAL> {SEPARATOR}                                     { yybegin(WAITING_VALUE); return SimpleTypes.SEPARATOR; }

<WAITING_VALUE> {CRLF}({CRLF}|{WHITE_SPACE})+               { yybegin(YYINITIAL); return TokenType.WHITE_SPACE; }

<WAITING_VALUE> {WHITE_SPACE}+                              { yybegin(WAITING_VALUE); return TokenType.WHITE_SPACE; }

<WAITING_VALUE> {FIRST_VALUE_CHARACTER}{VALUE_CHARACTER}*   { yybegin(YYINITIAL); return SimpleTypes.VALUE; }

({CRLF}|{WHITE_SPACE})+                                     { yybegin(YYINITIAL); return TokenType.WHITE_SPACE; }

[^]                                                         { return TokenType.BAD_CHARACTER; }
```

#### 生成词法分析器类 (Generate a Lexer Class)
通过在 Simple.flex 文件的上下文菜单中选择“Run JFlex Generator”生成词法分析器类。

使用 Grammar-Kit 插件生成 JFlex 词法分析器。当第一次运行时，JFlex 会提示选择一个目标文件夹来下载 JFlex 库和骨架文件。选择项目根目录，例如 `code_samples/simple_language_plugin`。

生成的词法分析器将位于 `gen` 目录下，例如 `simple_language_plugin/src/main/gen/org/intellij/sdk/language/SimpleLexer`。

#### 定义词法分析器适配器 (Define a Lexer Adapter)
JFlex 词法分析器需要适配到 IntelliJ 平台的词法分析器 API。通过继承 `FlexAdapter` 实现 `SimpleLexerAdapter`。

```java
public class SimpleLexerAdapter extends FlexAdapter {

  public SimpleLexerAdapter() {
    super(new SimpleLexer(null));
  }

}
```

#### 定义根文件 (Define a Root File)
`SimpleFile` 实现是 Simple 语言文件的 PSI 元素树的顶级节点。

```java
public class SimpleFile extends PsiFileBase {

  public SimpleFile(@NotNull FileViewProvider viewProvider) {
    super(viewProvider, SimpleLanguage.INSTANCE);
  }

  @NotNull
  @Override
  public FileType getFileType() {
    return SimpleFileType.INSTANCE;
  }

  @Override
  public String toString() {
    return "Simple File";
  }

}
```

#### 定义标记集 (Define Token Sets)
在 `SimpleTokenSets` 中定义所有来自 `SimpleTypes` 的相关标记类型集。

```java
public interface SimpleTokenSets {

  TokenSet IDENTIFIERS = TokenSet.create(SimpleTypes.KEY);

  TokenSet COMMENTS = TokenSet.create(SimpleTypes.COMMENT);

}
```

#### 定义解析器 (Define a Parser)
通过继承 `ParserDefinition` 定义 Simple 语言解析器 `SimpleParserDefinition`。为了避免在初始化扩展点实现时不必要的类加载，所有 `TokenSet` 返回值应使用专用的 `$Language$TokenSets` 类中的常量。

```java
final class SimpleParserDefinition implements ParserDefinition {

  public static final IFileElementType FILE = new IFileElementType(SimpleLanguage.INSTANCE);

  @NotNull
  @Override
  public Lexer createLexer(Project project) {
    return new SimpleLexerAdapter();
  }

  @NotNull
  @Override
  public TokenSet getCommentTokens() {
    return SimpleTokenSets.COMMENTS;
  }

  @NotNull
  @Override
  public TokenSet getStringLiteralElements() {
    return TokenSet.EMPTY;
  }

  @NotNull
  @Override
  public PsiParser createParser(final Project project) {
    return new SimpleParser();
  }

  @NotNull
  @Override
  public IFileElementType getFileNodeType() {
    return FILE;
  }

  @NotNull
  @Override
  public PsiFile createFile(@NotNull FileViewProvider viewProvider) {
    return new SimpleFile(viewProvider);
  }

  @NotNull
  @Override
  public PsiElement createElement(ASTNode node) {
    return SimpleTypes.Factory.createElement(node);
  }

}
```

#### 注册解析器定义 (Register the Parser Definition)
在 `plugin.xml` 文件中注册解析器定义，使其可用：

```xml
<extensions defaultExtensionNs="com.intellij">
  <lang.parserDefinition
      language="Simple"
      implementationClass="org.intellij.sdk.language.SimpleParserDefinition"/>
</extensions>
```

#### 运行项目 (Run the Project)
使用 Gradle 的 `runIde` 任务运行插件。

创建一个名为 `test.simple` 的文件，并添加以下内容：

```
# You are reading the ".properties" entry.
! The exclamation mark can also mark text as comments.
website = https://en.wikipedia.org/
language = English
# The backslash below tells the application to continue reading
# the value onto the next line.
message = Welcome to \
          Wikipedia!
# Add spaces to the key
key\ with\ spaces = This is the value that could be looked up with the key "key with spaces".
# Unicode
tab : \u0009
```

打开 PsiViewer 工具窗口，查看词法分析器如何将文件内容分解为标记，以及解析器如何将标记转换为 PSI 元素。

### 语法高亮和颜色设置页面 (Syntax Highlighter and Color Settings Page)
编辑页面最后修改时间：2024年7月31日

语法高亮的第一级基于词法分析器的输出，由 `SyntaxHighlighter` 提供。插件还可以基于 `ColorSettingPage` 定义颜色设置，以便用户配置高亮颜色。

#### 定义语法高亮器 (Define a Syntax Highlighter)
`SimpleSyntaxHighlighter` 类扩展了 `SyntaxHighlighterBase`。如颜色方案管理中所推荐，Simple 语言的高亮文本属性指定为标准 IntelliJ 平台键的依赖项。对于 Simple 语言，只定义一个方案。

```java
public class SimpleSyntaxHighlighter extends SyntaxHighlighterBase {

  public static final TextAttributesKey SEPARATOR =
      createTextAttributesKey("SIMPLE_SEPARATOR", DefaultLanguageHighlighterColors.OPERATION_SIGN);
  public static final TextAttributesKey KEY =
      createTextAttributesKey("SIMPLE_KEY", DefaultLanguageHighlighterColors.KEYWORD);
  public static final TextAttributesKey VALUE =
      createTextAttributesKey("SIMPLE_VALUE", DefaultLanguageHighlighterColors.STRING);
  public static final TextAttributesKey COMMENT =
      createTextAttributesKey("SIMPLE_COMMENT", DefaultLanguageHighlighterColors.LINE_COMMENT);
  public static final TextAttributesKey BAD_CHARACTER =
      createTextAttributesKey("SIMPLE_BAD_CHARACTER", HighlighterColors.BAD_CHARACTER);

  private static final TextAttributesKey[] BAD_CHAR_KEYS = new TextAttributesKey[]{BAD_CHARACTER};
  private static final TextAttributesKey[] SEPARATOR_KEYS = new TextAttributesKey[]{SEPARATOR};
  private static final TextAttributesKey[] KEY_KEYS = new TextAttributesKey[]{KEY};
  private static final TextAttributesKey[] VALUE_KEYS = new TextAttributesKey[]{VALUE};
  private static final TextAttributesKey[] COMMENT_KEYS = new TextAttributesKey[]{COMMENT};
  private static final TextAttributesKey[] EMPTY_KEYS = new TextAttributesKey[0];

  @NotNull
  @Override
  public Lexer getHighlightingLexer() {
    return new SimpleLexerAdapter();
  }

  @Override
  public TextAttributesKey @NotNull [] getTokenHighlights(IElementType tokenType) {
    if (tokenType.equals(SimpleTypes.SEPARATOR)) {
      return SEPARATOR_KEYS;
    }
    if (tokenType.equals(SimpleTypes.KEY)) {
      return KEY_KEYS;
    }
    if (tokenType.equals(SimpleTypes.VALUE)) {
      return VALUE_KEYS;
    }
    if (tokenType.equals(SimpleTypes.COMMENT)) {
      return COMMENT_KEYS;
    }
    if (tokenType.equals(TokenType.BAD_CHARACTER)) {
      return BAD_CHAR_KEYS;
    }
    return EMPTY_KEYS;
  }

}
```

#### 定义语法高亮器工厂 (Define a Syntax Highlighter Factory)
工厂为 IntelliJ 平台实例化 Simple 语言文件的语法高亮器提供了标准方法。在这里，`SimpleSyntaxHighlighterFactory` 继承了 `SyntaxHighlighterFactory`。

```java
final class SimpleSyntaxHighlighterFactory extends SyntaxHighlighterFactory {

  @NotNull
  @Override
  public SyntaxHighlighter getSyntaxHighlighter(Project project, VirtualFile virtualFile) {
    return new SimpleSyntaxHighlighter();
  }

}
```

#### 注册语法高亮器工厂 (Register the Syntax Highlighter Factory)
在 `plugin.xml` 文件中使用 `com.intellij.lang.syntaxHighlighterFactory` 扩展点注册工厂，使其在 IntelliJ 平台中可用。

```xml
<extensions defaultExtensionNs="com.intellij">
  <lang.syntaxHighlighterFactory
      language="Simple"
      implementationClass="org.intellij.sdk.language.SimpleSyntaxHighlighterFactory"/>
</extensions>
```

#### 使用默认颜色运行项目 (Run the Project with Default Colors)
在 IDE 开发实例中打开示例 Simple 语言属性文件 (`test.simple`)。Simple 语言关键字、分隔符和值的高亮颜色默认继承自 IDE 语言默认设置中的关键字、括号、操作符和字符串。

#### 定义颜色设置页面 (Define a Color Settings Page)
颜色设置页面使用户能够自定义 Simple 语言文件中的高亮颜色设置。`SimpleColorSettingsPage` 实现了 `ColorSettingsPage` 接口。

```java
final class SimpleColorSettingsPage implements ColorSettingsPage {

  private static final AttributesDescriptor[] DESCRIPTORS = new AttributesDescriptor[]{
      new AttributesDescriptor("Key", SimpleSyntaxHighlighter.KEY),
      new AttributesDescriptor("Separator", SimpleSyntaxHighlighter.SEPARATOR),
      new AttributesDescriptor("Value", SimpleSyntaxHighlighter.VALUE),
      new AttributesDescriptor("Bad value", SimpleSyntaxHighlighter.BAD_CHARACTER)
  };

  @Override
  public Icon getIcon() {
    return SimpleIcons.FILE;
  }

  @NotNull
  @Override
  public SyntaxHighlighter getHighlighter() {
    return new SimpleSyntaxHighlighter();
  }

  @NotNull
  @Override
  public String getDemoText() {
    return """
        # You are reading the ".properties" entry.
        ! The exclamation mark can also mark text as comments.
        website = https://en.wikipedia.org/
        language = English
        # The backslash below tells the application to continue reading
        # the value onto the next line.
        message = Welcome to \\
                  Wikipedia!
        # Add spaces to the key
        key\\ with\\ spaces = This is the value that could be looked up with the key "key with spaces".
        # Unicode
        tab : \\u0009""";
  }

  @Nullable
  @Override
  public Map<String, TextAttributesKey> getAdditionalHighlightingTagToDescriptorMap() {
    return null;
  }

  @Override
  public AttributesDescriptor @NotNull [] getAttributeDescriptors() {
    return DESCRIPTORS;
  }

  @Override
  public ColorDescriptor @NotNull [] getColorDescriptors() {
    return ColorDescriptor.EMPTY_ARRAY;
  }

  @NotNull
  @Override
  public String getDisplayName() {
    return "Simple";
  }

}
```

可以通过使用 `//` 分隔节点的方式对相关属性（例如操作符或括号）进行分组，如下所示：

```java
AttributesDescriptor[] DESCRIPTORS = new AttributesDescriptor[] {
    new AttributesDescriptor("Operators//Plus", MySyntaxHighlighter.PLUS),
    new AttributesDescriptor("Operators//Minus", MySyntaxHighlighter.MINUS),
    new AttributesDescriptor("Operators//Advanced//Sigma", MySyntaxHighlighter.SIGMA),
    new AttributesDescriptor("Operators//Advanced//Pi", MySyntaxHighlighter.PI),
    //...
};
```

#### 注册颜色设置页面 (Register the Color Settings Page)
在 `plugin.xml` 文件中使用 `com.intellij.colorSettingsPage` 扩展点注册 Simple 语言颜色设置页面。

```xml
<extensions defaultExtensionNs="com.intellij">
  <colorSettingsPage
      implementation="org.intellij.sdk.language.SimpleColorSettingsPage"/>
</extensions>
```

#### 运行项目 (Run the Project)
使用 Gradle 的 `runIde` 任务运行项目。

在 IDE 开发实例中打开 Simple 语言高亮设置页面：`Settings | Editor | Color Scheme | Simple`。每种颜色最初继承自语言默认设置的值。

## PSI 辅助类和工具 (PSI Helpers and Utilities)
编辑页面最后修改时间：2024年7月31日

生成的 PSI 元素中的辅助类和工具可以通过 Grammar-Kit 嵌入代码中。

需要注意的是，由于缺乏对两次生成的支持，无法通过 utils 方法扩展 Gradle Grammar-Kit 插件中的 PSI 元素。

### 为生成的 PSI 元素定义辅助方法 (Define Helper Methods for Generated PSI Elements)
自定义方法在 PSI 类中单独定义，Grammar-Kit 会将它们嵌入生成的代码中。定义包含这些辅助方法的实用类：

```java
package org.intellij.sdk.language.psi.impl;

import com.intellij.lang.ASTNode;
import org.intellij.sdk.language.psi.SimpleProperty;
import org.intellij.sdk.language.psi.SimpleTypes;

public class SimplePsiImplUtil {

  public static String getKey(SimpleProperty element) {
    ASTNode keyNode = element.getNode().findChildByType(SimpleTypes.KEY);
    if (keyNode != null) {
      // 重要：将嵌入的转义空格转换为普通空格
      return keyNode.getText().replaceAll("\\\\ ", " ");
    } else {
      return null;
    }
  }

  public static String getValue(SimpleProperty element) {
    ASTNode valueNode = element.getNode().findChildByType(SimpleTypes.VALUE);
    if (valueNode != null) {
      return valueNode.getText();
    } else {
      return null;
    }
  }

}
```
生成的 SimpleProperty 接口在上述代码中被引用。

### 更新语法并重新生成解析器 (Update Grammar and Regenerate the Parser)
现在，通过 `psiImplUtilClass` 全局属性将实用类添加到语法文件中。为特定规则添加 `methods` 属性，以指定应在 PSI 类中使用的方法。

比较以下 `property` 规则和以前的定义。

```bnf
{
  parserClass="org.intellij.sdk.language.parser.SimpleParser"
  
  extends="com.intellij.extapi.psi.ASTWrapperPsiElement"
  
  psiClassPrefix="Simple"
  psiImplClassSuffix="Impl"
  psiPackage="org.intellij.sdk.language.psi"
  psiImplPackage="org.intellij.sdk.language.psi.impl"
  
  elementTypeHolderClass="org.intellij.sdk.language.psi.SimpleTypes"
  elementTypeClass="org.intellij.sdk.language.psi.SimpleElementType"
  tokenTypeClass="org.intellij.sdk.language.psi.SimpleTokenType"
  
  psiImplUtilClass="org.intellij.sdk.language.psi.impl.SimplePsiImplUtil"
}

simpleFile ::= item_*

private item_ ::= (property|COMMENT|CRLF)

property ::= (KEY? SEPARATOR VALUE?) | KEY
{
  methods=[getKey getValue]
}
```
在语法中包含上述更改后，重新生成解析器和 PSI 类。

### 定义用于搜索属性的实用工具 (Define a Utility to Search Properties)
创建 `SimpleUtil` 实用类，在整个项目中搜索已定义的 PSI 元素属性。它将在实现代码补全时使用。

```java
public class SimpleUtil {

  /**
   * 在整个项目中搜索具有给定键的 Simple 语言文件中的 Simple 属性实例。
   *
   * @param project 当前项目
   * @param key     要检查的键
   * @return 匹配的属性
   */
  public static List<SimpleProperty> findProperties(Project project, String key) {
    List<SimpleProperty> result = new ArrayList<>();
    Collection<VirtualFile> virtualFiles =
        FileTypeIndex.getFiles(SimpleFileType.INSTANCE, GlobalSearchScope.allScope(project));
    for (VirtualFile virtualFile : virtualFiles) {
      SimpleFile simpleFile = (SimpleFile) PsiManager.getInstance(project).findFile(virtualFile);
      if (simpleFile != null) {
        SimpleProperty[] properties = PsiTreeUtil.getChildrenOfType(simpleFile, SimpleProperty.class);
        if (properties != null) {
          for (SimpleProperty property : properties) {
            if (key.equals(property.getKey())) {
              result.add(property);
            }
          }
        }
      }
    }
    return result;
  }

  public static List<SimpleProperty> findProperties(Project project) {
    List<SimpleProperty> result = new ArrayList<>();
    Collection<VirtualFile> virtualFiles =
        FileTypeIndex.getFiles(SimpleFileType.INSTANCE, GlobalSearchScope.allScope(project));
    for (VirtualFile virtualFile : virtualFiles) {
      SimpleFile simpleFile = (SimpleFile) PsiManager.getInstance(project).findFile(virtualFile);
      if (simpleFile != null) {
        SimpleProperty[] properties = PsiTreeUtil.getChildrenOfType(simpleFile, SimpleProperty.class);
        if (properties != null) {
          Collections.addAll(result, properties);
        }
      }
    }
    return result;
  }

  /**
   * 尝试收集 Simple 键/值对上方的任何注释元素。
   */
  public static @NotNull String findDocumentationComment(SimpleProperty property) {
    List<String> result = new LinkedList<>();
    PsiElement element = property.getPrevSibling();
    while (element instanceof PsiComment || element instanceof PsiWhiteSpace) {
      if (element instanceof PsiComment) {
        String commentText = element.getText().replaceFirst("[!# ]+", "");
        result.add(commentText);
      }
      element = element.getPrevSibling();
    }
    return StringUtil.join(Lists.reverse(result), "\n ");
  }

}
```

### 下一步 (Next Steps)
这些实用类和方法为构建更多高级功能提供了基础，例如实现代码补全、参考解析和更多的语义分析。在接下来的步骤中，我们将继续扩展 Simple 语言的功能，并展示如何使用这些工具和实用类来增强语言支持。



## 注解器 (Annotator)
编辑页面最后修改时间：2024年7月31日

参考文献：注解器 (Annotator)

代码：SimpleAnnotator

测试：4. 注解器测试

此页面是多步自定义语言支持教程的一部分。

注解器有助于根据特定规则突出显示和注释任何代码。本节添加注释功能以支持 Java 代码中的 Simple 语言。

### 所需的项目配置更改 (Required Project Configuration Changes)
本教程步骤中定义的类在运行时依赖于 `com.intellij.psi.PsiLiteralExpression`（Java 代码中的字符串字面量的 PSI 表示）。使用 `PsiLiteralExpression` 引入了对 `com.intellij.java` 的依赖。

从 2019.2 版本开始，必须显式声明对 Java 插件的依赖性。首先，在 Gradle 构建脚本中添加对 Java 插件的依赖性：

```kotlin
intellij {
  plugins.set(listOf("com.intellij.java"))
}
```

然后，在 `plugin.xml` 中声明依赖性（使用代码洞察）

```xml
<depends>com.intellij.java</depends>
```

### 定义注解器 (Define an Annotator)
`SimpleAnnotator` 类继承了 `Annotator`。假设以 "simple:" 作为前缀的字面字符串是 Simple 语言键的前缀。它不是 Simple 语言的一部分，但这是检测嵌入其他语言（如 Java）中的 Simple 语言键的有用约定。注释 `simple:key` 字面表达式，并区分格式良好的和未解析的属性。

使用 2020.2 开始的新 `AnnotationHolder` 语法，该语法使用生成器格式。

```java
final class SimpleAnnotator implements Annotator {

  // 定义 Simple 语言前缀的字符串 - 用于注解、行标记等
  public static final String SIMPLE_PREFIX_STR = "simple";
  public static final String SIMPLE_SEPARATOR_STR = ":";

  @Override
  public void annotate(@NotNull final PsiElement element, @NotNull AnnotationHolder holder) {
    // 确保 PSI 元素是一个表达式
    if (!(element instanceof PsiLiteralExpression literalExpression)) {
      return;
    }

    // 确保 PSI 元素包含以前缀和分隔符开头的字符串
    String value = literalExpression.getValue() instanceof String ? (String) literalExpression.getValue() : null;
    if (value == null || !value.startsWith(SIMPLE_PREFIX_STR + SIMPLE_SEPARATOR_STR)) {
      return;
    }

    // 定义文本范围（开始是包含的，结束是排除的）
    // "simple:key"
    //  01234567890
    TextRange prefixRange = TextRange.from(element.getTextRange().getStartOffset(), SIMPLE_PREFIX_STR.length() + 1);
    TextRange separatorRange = TextRange.from(prefixRange.getEndOffset(), SIMPLE_SEPARATOR_STR.length());
    TextRange keyRange = new TextRange(separatorRange.getEndOffset(), element.getTextRange().getEndOffset() - 1);

    // 高亮显示 "simple" 前缀和 ":" 分隔符
    holder.newSilentAnnotation(HighlightSeverity.INFORMATION)
        .range(prefixRange).textAttributes(DefaultLanguageHighlighterColors.KEYWORD).create();
    holder.newSilentAnnotation(HighlightSeverity.INFORMATION)
        .range(separatorRange).textAttributes(SimpleSyntaxHighlighter.SEPARATOR).create();

    // 获取给定键的属性列表
    String key = value.substring(SIMPLE_PREFIX_STR.length() + SIMPLE_SEPARATOR_STR.length());
    List<SimpleProperty> properties = SimpleUtil.findProperties(element.getProject(), key);
    if (properties.isEmpty()) {
      holder.newAnnotation(HighlightSeverity.ERROR, "Unresolved property")
          .range(keyRange)
          .highlightType(ProblemHighlightType.LIKE_UNKNOWN_SYMBOL)
          // ** 教程第 19 步。 - 为包含可能属性的字符串添加快速修复
          .withFix(new SimpleCreatePropertyQuickFix(key))
          .create();
    } else {
      // 找到至少一个属性，将文本属性强制为 Simple 语法值字符
      holder.newSilentAnnotation(HighlightSeverity.INFORMATION)
          .range(keyRange).textAttributes(SimpleSyntaxHighlighter.VALUE).create();
    }
  }

}
```

如果在教程的这一阶段复制了上述代码，请删除注释 "** 教程第 19 步。 …" 下方的行。该行中的快速修复类直到教程的后期才定义。

### 注册注解器 (Register the Annotator)
使用 `com.intellij.annotator` 扩展点在插件配置文件中为 JAVA 语言注册 Simple 语言注解器类：

```xml
<extensions defaultExtensionNs="com.intellij">
  <annotator
      language="JAVA"
      implementationClass="org.intellij.sdk.language.SimpleAnnotator"/>
</extensions>
```

### 运行项目 (Run the Project)
通过使用 Gradle 的 `runIde` 任务运行插件。

作为测试，定义包含 Simple 语言前缀：值对的以下 Java 文件：

```java
public class Test {
  public static void main(String[] args) {
    System.out.println("simple:website");
  }
}
```

在运行 `simple_language_plugin` 的 IDE 开发实例中打开此 Java 文件，以检查 IDE 是否解析属性：

- 如果属性是未定义的名称，注解器会用错误标记代码。

尝试更改 Simple 语言的颜色设置，以将注释与默认语言颜色设置区分开来。



## 行标记提供程序 (Line Marker Provider)
编辑页面最后修改时间：2024年7月31日

代码：SimpleLineMarkerProvider

此页面是多步自定义语言支持教程的一部分。

行标记帮助在边距上注释代码。这些标记可以提供导航到相关代码的目标。

### 定义行标记提供程序 (Define a Line Marker Provider)
行标记提供程序注释 Simple 语言属性在 Java 代码中的用法，并提供导航到这些属性的定义。可视标记是编辑器窗口边距中的 Simple 语言图标。

`SimpleLineMarkerProvider` 类继承自 `RelatedItemLineMarkerProvider`。在此示例中，重写 `collectNavigationMarkers()` 方法以收集 Simple 语言键和分隔符的用法：

```java
final class SimpleLineMarkerProvider extends RelatedItemLineMarkerProvider {

  @Override
  protected void collectNavigationMarkers(@NotNull PsiElement element,
                                          @NotNull Collection<? super RelatedItemLineMarkerInfo<?>> result) {
    // 该元素必须是一个字面表达式的父元素
    if (!(element instanceof PsiJavaTokenImpl) || !(element.getParent() instanceof PsiLiteralExpression literalExpression)) {
      return;
    }

    // 字面表达式必须以 Simple 语言的字面表达式开头
    String value = literalExpression.getValue() instanceof String ? (String) literalExpression.getValue() : null;
    if ((value == null) ||
        !value.startsWith(SimpleAnnotator.SIMPLE_PREFIX_STR + SimpleAnnotator.SIMPLE_SEPARATOR_STR)) {
      return;
    }

    // 获取 Simple 语言属性的用法
    Project project = element.getProject();
    String possibleProperties = value.substring(
        SimpleAnnotator.SIMPLE_PREFIX_STR.length() + SimpleAnnotator.SIMPLE_SEPARATOR_STR.length()
    );
    final List<SimpleProperty> properties = SimpleUtil.findProperties(project, possibleProperties);
    if (!properties.isEmpty()) {
      // 将属性添加到行标记信息集合中
      NavigationGutterIconBuilder<PsiElement> builder =
          NavigationGutterIconBuilder.create(SimpleIcons.FILE)
              .setTargets(properties)
              .setTooltipText("Navigate to Simple language property");
      result.add(builder.createLineMarkerInfo(element));
    }
  }

}
```

从 `GutterIconDescriptor` 扩展允许通过 `Settings | Editor | General | Gutter Icons` 配置要显示的边距图标。

### 实现行标记提供程序的最佳实践 (Best Practices for Implementing Line Marker Providers)
本节介绍实现标记提供程序的关键细节。

`collectNavigationMarkers()` 方法应该：

1. 仅返回与传入方法的元素一致的行标记信息。例如，不要在 `getLineMarkerInfo()` 被调用时返回类标记。

2. 返回适当范围的 PSI 树元素的行标记信息。仅返回叶子元素，即最小的元素。例如，不要为 `PsiMethod` 返回方法标记，而应为包含方法名称的 `PsiIdentifier` 返回标记。

### 行标记位置 (Line Marker Location)
如果 `LineMarkerProvider` 为 PSI 树中的较高节点返回标记信息会发生什么？例如，如果 `MyWrongLineMarkerProvider()` 错误地返回 `PsiMethod` 而不是 `PsiIdentifier` 元素：

```java
final class MyWrongLineMarkerProvider implements LineMarkerProvider {
  public LineMarkerInfo getLineMarkerInfo(@NotNull PsiElement element) {
    if (element instanceof PsiMethod) {
      return new LineMarkerInfo(element, ...);
    }
    return null;
  }
}
```

`MyWrongLineMarkerProvider()` 实现的结果与 IntelliJ 平台执行检查的方式有关。出于性能原因，检查（特别是 `LineMarkersPass`）在两个阶段中查询所有 `LineMarkerProviders`：

1. 第一阶段是为编辑器窗口中可见的所有元素，

2. 第二阶段是为文件中的其余元素。

如果提供程序在任一区域中不返回任何内容，则行标记将被清除。但是，如果方法 `actionPerformed()` 在编辑器窗口中没有完全可见，并且 `MyWrongLineMarkerProvider()` 为 `PsiMethod` 而不是 `PsiIdentifier` 返回标记信息，则：

1. 第一阶段会删除行标记信息，因为整个 `PsiMethod` 没有显示。

2. 第二阶段尝试添加行标记，因为 `MyWrongLineMarkerProvider()` 被调用。

因此，行标记图标可能会闪烁。为了解决这个问题，应将 `MyWrongLineMarkerProvider` 重写为返回 `PsiIdentifier` 而不是 `PsiMethod`，如下所示：

```java
final class MyCorrectLineMarkerProvider implements LineMarkerProvider {
  public LineMarkerInfo getLineMarkerInfo(@NotNull PsiElement element) {
    if (element instanceof PsiIdentifier &&
            element.getParent() instanceof PsiMethod) {
      return new LineMarkerInfo(element, ...);
    }
    return null;
  }
}
```

### 注册行标记提供程序 (Register the Line Marker Provider)
`SimpleLineMarkerProvider` 实现使用 `com.intellij.codeInsight.lineMarkerProvider` 扩展点在插件配置文件中注册到 IntelliJ 平台。

```xml
<extensions defaultExtensionNs="com.intellij">
  <codeInsight.lineMarkerProvider
      language="JAVA"
      implementationClass="org.intellij.sdk.language.SimpleLineMarkerProvider"/>
</extensions>
```

### 运行项目 (Run the Project)
通过使用 Gradle 的 `runIde` 任务运行插件。

打开 Java 测试文件。现在图标出现在第 3 行的边距中。用户可以点击图标导航到属性定义。



## 完成贡献者 (Completion Contributor)
编辑页面最后修改时间：2024年7月31日

参考：代码补全 (Code Completion)

代码：SimpleCompletionContributor

这是多步自定义语言支持教程的一部分。

自定义语言可以使用两种方法之一提供代码补全：贡献者 (Contributor) 和基于引用 (Reference-based) 的补全（请参阅10. 引用贡献者）。

### 定义完成贡献者 (Define a Completion Contributor)
在本教程中，`simple_language_plugin` 提供对 Simple 语言属性文件中的值的自定义补全。通过继承 `CompletionContributor` 创建 `SimpleCompletionContributor`。此基础的补全贡献者总是将 "Hello" 添加到补全变体的结果集中，而不管上下文如何：

```java
final class SimpleCompletionContributor extends CompletionContributor {

  SimpleCompletionContributor() {
    extend(CompletionType.BASIC, PlatformPatterns.psiElement(SimpleTypes.VALUE),
        new CompletionProvider<>() {
          public void addCompletions(@NotNull CompletionParameters parameters,
                                     @NotNull ProcessingContext context,
                                     @NotNull CompletionResultSet resultSet) {
            resultSet.addElement(LookupElementBuilder.create("Hello"));
          }
        }
    );
  }

}
```

### 注册完成贡献者 (Register the Completion Contributor)
`SimpleCompletionContributor` 实现通过 `com.intellij.completion.contributor` 扩展点在插件配置文件中注册，并指定 `language="Simple"`。

```xml
<extensions defaultExtensionNs="com.intellij">
  <completion.contributor
      language="Simple"
      implementationClass="org.intellij.sdk.language.SimpleCompletionContributor"/>
</extensions>
```

### 运行项目 (Run the Project)
通过使用 Gradle 的 `runIde` 任务运行插件。

打开 `test.simple` 文件。删除属性 "English" 并调用基本代码补全。显示选项 "Hello"。



## 引用贡献者 (Reference Contributor)
编辑页面最后修改时间：2024年7月31日

参考：引用与解析 (References and Resolve)、PSI 引用 (PSI References)

代码：SimpleNamedElement, SimpleNamedElementImpl, SimpleReference.java, SimpleReferenceContributor, SimpleRefactoringSupportProvider

测试：3. 补全测试，6. 重命名测试，10. 引用测试

这是多步自定义语言支持教程的一部分。

引用功能是自定义语言支持实现中最重要的部分之一。解析引用意味着能够从元素的使用跳转到其声明、补全、重命名重构、查找用法等。

每个可以重命名或引用的 PSI 元素都需要实现 `PsiNamedElement` 接口。

### 定义命名元素类 (Define a Named Element Class)
以下类展示了 Simple 语言如何实现 `PsiNamedElement`。

`SimpleNamedElement` 接口是 `PsiNameIdentifierOwner` 的子类。

```java
public interface SimpleNamedElement extends PsiNameIdentifierOwner {

}
```

`SimpleNamedElementImpl` 类实现 `SimpleNamedElement` 接口并继承 `ASTWrapperPsiElement`。

```java
public abstract class SimpleNamedElementImpl extends ASTWrapperPsiElement implements SimpleNamedElement {

  public SimpleNamedElementImpl(@NotNull ASTNode node) {
    super(node);
  }

}
```

### 为生成的 PSI 元素定义帮助方法 (Define Helper Methods for Generated PSI Elements)
修改 `SimplePsiImplUtil` 以支持为 Simple 语言的 PSI 类添加的新方法。请注意，在下一步之前 `SimpleElementFactory` 尚未定义，因此现在它显示为错误。

```java
public class SimplePsiImplUtil {

  // ...

  public static String getName(SimpleProperty element) {
    return getKey(element);
  }

  public static PsiElement setName(SimpleProperty element, String newName) {
    ASTNode keyNode = element.getNode().findChildByType(SimpleTypes.KEY);
    if (keyNode != null) {
      SimpleProperty property =
          SimpleElementFactory.createProperty(element.getProject(), newName);
      ASTNode newKeyNode = property.getFirstChild().getNode();
      element.getNode().replaceChild(keyNode, newKeyNode);
    }
    return element;
  }

  public static PsiElement getNameIdentifier(SimpleProperty element) {
    ASTNode keyNode = element.getNode().findChildByType(SimpleTypes.KEY);
    return keyNode != null ? keyNode.getPsi() : null;
  }

  // ...

}
```

### 定义元素工厂 (Define an Element Factory)
`SimpleElementFactory` 提供创建 `SimpleFile` 的方法。

```java
package org.intellij.sdk.language.psi;

import com.intellij.openapi.project.Project;
import com.intellij.psi.*;
import org.intellij.sdk.language.SimpleFileType;

public class SimpleElementFactory {

  public static SimpleProperty createProperty(Project project, String name) {
    SimpleFile file = createFile(project, name);
    return (SimpleProperty) file.getFirstChild();
  }

  public static SimpleFile createFile(Project project, String text) {
    String name = "dummy.simple";
    return (SimpleFile) PsiFileFactory.getInstance(project).
        createFileFromText(name, SimpleFileType.INSTANCE, text);
  }
}
```

### 更新语法并重新生成解析器 (Update Grammar and Regenerate the Parser)
现在，通过以下几行替换 `Simple.bnf` 语法文件中的 `property` 定义来进行相应更改。更新文件后，请不要忘记重新生成解析器！右键单击 `Simple.bnf` 文件并选择 "Generate Parser Code"。

```bnf
property ::= (KEY? SEPARATOR VALUE?) | KEY {
  mixin="org.intellij.sdk.language.psi.impl.SimpleNamedElementImpl"
  implements="org.intellij.sdk.language.psi.SimpleNamedElement"
  methods=[getKey getValue getName setName getNameIdentifier]
}
```

### 定义引用 (Define a Reference)
现在定义一个引用类 `SimpleReference.java` 来从其使用位置解析属性。这需要扩展 `PsiReferenceBase` 并实现 `PsiPolyVariantReference`。后者使引用能够解析为多个元素或解析结果的超集。

```java
final class SimpleReference extends PsiReferenceBase<PsiElement> implements PsiPolyVariantReference {

  private final String key;

  SimpleReference(@NotNull PsiElement element, TextRange textRange) {
    super(element, textRange);
    key = element.getText().substring(textRange.getStartOffset(), textRange.getEndOffset());
  }

  @Override
  public ResolveResult @NotNull [] multiResolve(boolean incompleteCode) {
    Project project = myElement.getProject();
    final List<SimpleProperty> properties = SimpleUtil.findProperties(project, key);
    List<ResolveResult> results = new ArrayList<>();
    for (SimpleProperty property : properties) {
      results.add(new PsiElementResolveResult(property));
    }
    return results.toArray(new ResolveResult[0]);
  }

  @Nullable
  @Override
  public PsiElement resolve() {
    ResolveResult[] resolveResults = multiResolve(false);
    return resolveResults.length == 1 ? resolveResults[0].getElement() : null;
  }

  @Override
  public Object @NotNull [] getVariants() {
    Project project = myElement.getProject();
    List<SimpleProperty> properties = SimpleUtil.findProperties(project);
    List<LookupElement> variants = new ArrayList<>();
    for (final SimpleProperty property : properties) {
      if (property.getKey() != null && !property.getKey().isEmpty()) {
        variants.add(LookupElementBuilder
            .create(property).withIcon(SimpleIcons.FILE)
            .withTypeText(property.getContainingFile().getName())
        );
      }
    }
    return variants.toArray();
  }

}
```

### 定义引用贡献者 (Define a Reference Contributor)
引用贡献者允许 `simple_language_plugin` 从其他语言（如 Java）提供对 Simple 语言的引用。通过继承 `PsiReferenceContributor` 创建 `SimpleReferenceContributor`。为每个属性的使用提供引用：

```java
final class SimpleReferenceContributor extends PsiReferenceContributor {

  @Override
  public void registerReferenceProviders(@NotNull PsiReferenceRegistrar registrar) {
    registrar.registerReferenceProvider(PlatformPatterns.psiElement(PsiLiteralExpression.class),
        new PsiReferenceProvider() {
          @Override
          public PsiReference @NotNull [] getReferencesByElement(@NotNull PsiElement element,
                                                                 @NotNull ProcessingContext context) {
            PsiLiteralExpression literalExpression = (PsiLiteralExpression) element;
            String value = literalExpression.getValue() instanceof String ?
                (String) literalExpression.getValue() : null;
            if ((value != null && value.startsWith(SIMPLE_PREFIX_STR + SIMPLE_SEPARATOR_STR))) {
              TextRange property = new TextRange(SIMPLE_PREFIX_STR.length() + SIMPLE_SEPARATOR_STR.length() + 1,
                  value.length() + 1);
              return new PsiReference[]{new SimpleReference(element, property)};
            }
            return PsiReference.EMPTY_ARRAY;
          }
        });
  }

}
```

### 注册引用贡献者 (Register the Reference Contributor)
`SimpleReferenceContributor` 实现使用 `com.intellij.psi.referenceContributor` 扩展点在插件配置文件中注册，并指定 `language="JAVA"`。

```xml
<extensions defaultExtensionNs="com.intellij">
  <psi.referenceContributor language="JAVA"
                            implementation="org.intellij.sdk.language.SimpleReferenceContributor"/>
</extensions>
```

### 使用引用贡献者运行项目 (Run the Project with the Reference Contributor)
通过使用 Gradle 的 `runIde` 任务运行项目。

IDE 现在解析属性并提供补全建议：

![Reference Contributor](https://resources.jetbrains.com/help/img/idea/2022.1/tutorial_reference_contributor.png)

重命名重构功能现在可用于定义和用法。

![Rename](https://resources.jetbrains.com/help/img/idea/2022.1/tutorial_reference_contributor2.png)

### 定义重构支持提供者 (Define a Refactoring Support Provider)
对位重构的支持在重构支持提供者中显式指定。通过继承 `RefactoringSupportProvider` 创建 `SimpleRefactoringSupportProvider`。只要元素是 `SimpleProperty`，就允许对其进行重构：

```java
final class SimpleRefactoringSupportProvider extends RefactoringSupportProvider {

  @Override
  public boolean isMemberInplaceRenameAvailable(@NotNull PsiElement elementToRename, @Nullable PsiElement context) {
    return (elementToRename instanceof SimpleProperty);
  }

}
```

### 注册重构支持提供者 (Register the Refactoring Support Provider)
`SimpleRefactoringSupportProvider` 实现使用 `com.intellij.lang.refactoringSupport` 扩展点在插件配置文件中注册。

```xml
<extensions defaultExtensionNs="com.intellij">
  <lang.refactoringSupport
      language="Simple"
      implementationClass="org.intellij.sdk.language.SimpleRefactoringSupportProvider"/>
</extensions>
```

这一步完成后，插件将能够提供引用解析、代码补全、重命名重构等功能。



## 查找用法提供者 (Find Usages Provider)
编辑页面最后修改时间：2024年7月31日

参考：查找用法 (Find Usages)

代码：SimpleFindUsagesProvider

测试：8. 查找用法测试 (Find Usages Test)

这是自定义语言支持教程的多个步骤之一。

`FindUsagesProvider` 通过单词扫描器（word scanner）构建每个文件的单词索引。扫描器负责将文本分解为单词并为每个单词确定上下文。

### 定义查找用法提供者 (Define a Find Usages Provider)
`SimpleFindUsagesProvider` 实现了 `FindUsagesProvider` 接口。使用 `DefaultWordsScanner` 可以确保扫描器实现是线程安全的。有关详细信息，请参见 `FindUsagesProvider` 接口的文档注释。

```java
final class SimpleFindUsagesProvider implements FindUsagesProvider {

  @Override
  public WordsScanner getWordsScanner() {
    return new DefaultWordsScanner(new SimpleLexerAdapter(),
        SimpleTokenSets.IDENTIFIERS,
        SimpleTokenSets.COMMENTS,
        TokenSet.EMPTY);
  }

  @Override
  public boolean canFindUsagesFor(@NotNull PsiElement psiElement) {
    return psiElement instanceof PsiNamedElement;
  }

  @Nullable
  @Override
  public String getHelpId(@NotNull PsiElement psiElement) {
    return null;
  }

  @NotNull
  @Override
  public String getType(@NotNull PsiElement element) {
    if (element instanceof SimpleProperty) {
      return "simple property";
    }
    return "";
  }

  @NotNull
  @Override
  public String getDescriptiveName(@NotNull PsiElement element) {
    if (element instanceof SimpleProperty) {
      return ((SimpleProperty) element).getKey();
    }
    return "";
  }

  @NotNull
  @Override
  public String getNodeText(@NotNull PsiElement element, boolean useFullName) {
    if (element instanceof SimpleProperty) {
      return ((SimpleProperty) element).getKey() +
          SimpleAnnotator.SIMPLE_SEPARATOR_STR +
          ((SimpleProperty) element).getValue();
    }
    return "";
  }

}
```

### 注册查找用法提供者 (Register the Find Usages Provider)
在插件配置文件中，通过 `com.intellij.lang.findUsagesProvider` 扩展点注册 `SimpleFindUsagesProvider` 实现。

```xml
<extensions defaultExtensionNs="com.intellij">
  <lang.findUsagesProvider
      language="Simple"
      implementationClass="org.intellij.sdk.language.SimpleFindUsagesProvider"/>
</extensions>
```

### 运行项目 (Run the Project)
使用 Gradle 的 `runIde` 任务运行插件。

现在，IDE 支持为任意具有引用的属性执行“查找用法”操作：

![Find Usages](https://resources.jetbrains.com/help/img/idea/2022.1/tutorial_find_usages.png)

这使得用户能够通过在代码中定位属性的使用位置来更好地理解和维护代码。这种功能在大型代码库中尤其有用，有助于快速定位相关定义和使用，提高代码可维护性。

## 折叠构建器 (Folding Builder)
编辑页面最后修改时间：2024年7月31日

代码：`SimpleFoldingBuilder`

测试：7. 折叠测试 (Folding Test)

这是自定义语言支持教程的多个步骤之一。

折叠构建器用于识别代码中的折叠区域。在本教程中，折叠构建器用于标识折叠区域并用特定文本替换这些区域。不同于通常使用折叠构建器将类、方法或注释折叠成更少的行数，本折叠构建器将简单语言 (Simple Language) 的键替换为相应的值。

### 定义折叠构建器
`SimpleFoldingBuilder` 默认情况下将属性的使用替换为其值。首先，通过继承 `FoldingBuilderEx` 来创建折叠构建器。

注意，`SimpleFoldingBuilder` 被标记为 `dumb aware`，这意味着该类允许在 `dumb mode`（索引正在后台更新时）下运行。

折叠构建器必须实现 `DumbAware` 接口，以便在本教程中正常工作并通过测试。

`buildFoldRegions()` 方法从根节点向下搜索 PSI 树，寻找所有包含简单前缀 `simple:` 的字面量表达式。这样的字符串的其余部分预计包含一个简单语言的键，因此文本范围被存储为 `FoldingDescriptor`。

`getPlaceholderText()` 方法检索与提供的 `(ASTNode)` 相关联的键对应的简单语言值。IntelliJ 平台在代码折叠时使用该值替换键。

```java
final class SimpleFoldingBuilder extends FoldingBuilderEx implements DumbAware {

  @Override
  public FoldingDescriptor @NotNull [] buildFoldRegions(@NotNull PsiElement root,
                                                        @NotNull Document document,
                                                        boolean quick) {
    // 初始化将一起展开/折叠的折叠区域组
    FoldingGroup group = FoldingGroup.newGroup(SimpleAnnotator.SIMPLE_PREFIX_STR);
    // 初始化折叠区域列表
    List<FoldingDescriptor> descriptors = new ArrayList<>();

    root.accept(new JavaRecursiveElementWalkingVisitor() {

      @Override
      public void visitLiteralExpression(@NotNull PsiLiteralExpression literalExpression) {
        super.visitLiteralExpression(literalExpression);

        String value = PsiLiteralUtil.getStringLiteralContent(literalExpression);
        if (value != null &&
            value.startsWith(SimpleAnnotator.SIMPLE_PREFIX_STR + SimpleAnnotator.SIMPLE_SEPARATOR_STR)) {
          Project project = literalExpression.getProject();
          String key = value.substring(
              SimpleAnnotator.SIMPLE_PREFIX_STR.length() + SimpleAnnotator.SIMPLE_SEPARATOR_STR.length()
          );
          // 在项目中查找给定键的 SimpleProperty
          SimpleProperty simpleProperty = ContainerUtil.getOnlyItem(SimpleUtil.findProperties(project, key));
          if (simpleProperty != null) {
            // 为此节点上的字面量表达式添加折叠描述符
            descriptors.add(new FoldingDescriptor(literalExpression.getNode(),
                new TextRange(literalExpression.getTextRange().getStartOffset() + 1,
                    literalExpression.getTextRange().getEndOffset() - 1),
                group, Collections.singleton(simpleProperty)));
          }
        }
      }
    });

    return descriptors.toArray(FoldingDescriptor.EMPTY_ARRAY);
  }

  @Nullable
  @Override
  public String getPlaceholderText(@NotNull ASTNode node) {
    if (node.getPsi() instanceof PsiLiteralExpression psiLiteralExpression) {
      String text = PsiLiteralUtil.getStringLiteralContent(psiLiteralExpression);
      if (text == null) {
        return null;
      }

      String key = text.substring(SimpleAnnotator.SIMPLE_PREFIX_STR.length() +
          SimpleAnnotator.SIMPLE_SEPARATOR_STR.length());

      SimpleProperty simpleProperty = ContainerUtil.getOnlyItem(
          SimpleUtil.findProperties(psiLiteralExpression.getProject(), key)
      );
      if (simpleProperty == null) {
        return StringUtil.THREE_DOTS;
      }

      String propertyValue = simpleProperty.getValue();
      if (propertyValue == null) {
        return StringUtil.THREE_DOTS;
      }

      return propertyValue
          .replaceAll("\n", "\\n")
          .replaceAll("\"", "\\\\\"");
    }

    return null;
  }

  @Override
  public boolean isCollapsedByDefault(@NotNull ASTNode node) {
    return true;
  }

}
```

### 注册折叠构建器
使用 `com.intellij.lang.foldingBuilder` 扩展点在插件配置文件中注册 `SimpleFoldingBuilder` 实现。

```xml
<extensions defaultExtensionNs="com.intellij">
  <lang.foldingBuilder
      language="JAVA"
      implementationClass="org.intellij.sdk.language.SimpleFoldingBuilder"/>
</extensions>
```

### 运行项目
使用 Gradle 的 `runIde` 任务运行插件。

现在，当在编辑器中打开 Java 文件时，它显示属性的值而不是键。这是因为 `SimpleFoldingBuilder.isCollapsedByDefault()` 总是返回 true。尝试使用 `Code | Folding | Expand All` 来显示键而不是值。



## 符号跳转贡献者 (Go To Symbol Contributor)
编辑页面最后修改时间：2024年7月31日

参考文档：跳转到类和跳转到符号 (Go to Class and Go to Symbol)

代码：`SimpleChooseByNameContributor`、`SimplePsiImplUtil`

这是自定义语言支持教程的多个步骤之一。

符号跳转贡献者帮助用户通过名称导航到任何 PSI 元素。

### 为生成的 PSI 元素定义辅助方法
为了在 "导航 | 符号" 弹出窗口、结构工具窗口或其他组件中指定 PSI 元素的外观，应该实现 `getPresentation()` 方法。此方法在 `SimplePsiImplUtil` 工具类中定义，并且必须重新生成解析器和 PSI 类。将以下方法添加到 `SimplePsiImplUtil` 中：

```java
public static ItemPresentation getPresentation(final SimpleProperty element) {
  return new ItemPresentation() {
    @Nullable
    @Override
    public String getPresentableText() {
      return element.getKey();
    }

    @Nullable
    @Override
    public String getLocationString() {
      PsiFile containingFile = element.getContainingFile();
      return containingFile == null ? null : containingFile.getName();
    }

    @Override
    public Icon getIcon(boolean unused) {
      return element.getIcon(0);
    }
  };
}
```

另外，为了为显示的项目提供图标，可以扩展 `IconProvider` 并在 `com.intellij.iconProvider` 扩展点中注册它。参见 `SimplePropertyIconProvider`：

```java
final class SimplePropertyIconProvider extends IconProvider {

  @Override
  public @Nullable Icon getIcon(@NotNull PsiElement element, int flags) {
    return element instanceof SimpleProperty ? SimpleIcons.FILE : null;
  }

}
```

### 更新语法并重新生成解析器
现在，将 `SimplePsiImplUtil.getPresentation()` 添加到 Simple.bnf 语法文件中的属性方法定义中，通过以下行替换属性定义。更新文件后，别忘了重新生成解析器！右键单击 Simple.bnf 文件并选择 "Generate Parser Code"。

```java
property ::= (KEY? SEPARATOR VALUE?) | KEY {
  mixin="org.intellij.sdk.language.psi.impl.SimpleNamedElementImpl"
  implements="org.intellij.sdk.language.psi.SimpleNamedElement"
  methods=[getKey getValue getName setName getNameIdentifier getPresentation]
}
```

### 定义符号跳转贡献者
为了将项目贡献给 "导航 | 符号" 结果，继承 `ChooseByNameContributorEx` 创建 `SimpleChooseByNameContributor`：

```java
final class SimpleChooseByNameContributor implements ChooseByNameContributorEx {

  @Override
  public void processNames(@NotNull Processor<? super String> processor,
                           @NotNull GlobalSearchScope scope,
                           @Nullable IdFilter filter) {
    Project project = Objects.requireNonNull(scope.getProject());
    List<String> propertyKeys = ContainerUtil.map(
        SimpleUtil.findProperties(project), SimpleProperty::getKey);
    ContainerUtil.process(propertyKeys, processor);
  }

  @Override
  public void processElementsWithName(@NotNull String name,
                                      @NotNull Processor<? super NavigationItem> processor,
                                      @NotNull FindSymbolParameters parameters) {
    List<NavigationItem> properties = ContainerUtil.map(
        SimpleUtil.findProperties(parameters.getProject(), name),
        property -> (NavigationItem) property);
    ContainerUtil.process(properties, processor);
  }

}
```

### 注册符号跳转贡献者
使用 `com.intellij.gotoSymbolContributor` 扩展点在插件配置文件中注册 `SimpleChooseByNameContributor` 实现。

```xml
<extensions defaultExtensionNs="com.intellij">
  <gotoSymbolContributor
      implementation="org.intellij.sdk.language.SimpleChooseByNameContributor"/>
</extensions>
```

### 运行项目
使用 Gradle 的 `runIde` 任务运行插件。

现在，IDE 支持通过 "导航 | 符号" 操作按名称模式导航到属性定义。



## 结构视图工厂 (Structure View Factory)
编辑页面最后修改时间：2024年7月31日

参考文档：结构视图 (Structure View)

代码：`SimpleStructureViewFactory`、`SimpleStructureViewModel`、`SimpleStructureViewElement`

这是自定义语言支持教程的多个步骤之一。

结构视图可以为特定的文件类型自定义。创建结构视图工厂允许在结构工具窗口或通过调用“导航 | 文件结构”显示文件的结构，以便在当前编辑器中轻松导航。

### 定义结构视图工厂
`SimpleStructureViewFactory` 实现了 `PsiStructureViewFactory` 接口。`getStructureViewBuilder()` 方法复用了 IntelliJ 平台的 `TreeBasedStructureViewBuilder` 类。在实现 `SimpleStructureViewModel` 之前，项目将无法编译。

```java
final class SimpleStructureViewFactory implements PsiStructureViewFactory {

  @Override
  public StructureViewBuilder getStructureViewBuilder(@NotNull final PsiFile psiFile) {
    return new TreeBasedStructureViewBuilder() {
      @NotNull
      @Override
      public StructureViewModel createStructureViewModel(@Nullable Editor editor) {
        return new SimpleStructureViewModel(editor, psiFile);
      }
    };
  }

}
```

### 定义结构视图模型
`SimpleStructureViewModel` 通过实现 `StructureViewModel` 来创建，它定义了在标准结构视图中显示的数据模型。它还扩展了 `StructureViewModelBase`，这是一个将模型与文本编辑器连接起来的实现。

```java
public class SimpleStructureViewModel extends StructureViewModelBase implements
    StructureViewModel.ElementInfoProvider {

  public SimpleStructureViewModel(@Nullable Editor editor, PsiFile psiFile) {
    super(psiFile, editor, new SimpleStructureViewElement(psiFile));
  }

  @NotNull
  public Sorter @NotNull [] getSorters() {
    return new Sorter[]{Sorter.ALPHA_SORTER};
  }

  @Override
  public boolean isAlwaysShowsPlus(StructureViewTreeElement element) {
    return false;
  }

  @Override
  public boolean isAlwaysLeaf(StructureViewTreeElement element) {
    return element.getValue() instanceof SimpleProperty;
  }

  @Override
  protected Class<?> @NotNull [] getSuitableClasses() {
    return new Class[]{SimpleProperty.class};
  }

}
```

### 定义结构视图元素
`SimpleStructureViewElement` 实现了 `StructureViewTreeElement` 和 `SortableTreeElement` 接口。`StructureViewTreeElement` 表示结构视图树模型中的一个元素。`SortableTreeElement` 表示智能树中的一个项目，允许使用不同于可显示文本的文本作为字母排序的键。

```java
public class SimpleStructureViewElement implements StructureViewTreeElement, SortableTreeElement {

  private final NavigatablePsiElement myElement;

  public SimpleStructureViewElement(NavigatablePsiElement element) {
    this.myElement = element;
  }

  @Override
  public Object getValue() {
    return myElement;
  }

  @Override
  public void navigate(boolean requestFocus) {
    myElement.navigate(requestFocus);
  }

  @Override
  public boolean canNavigate() {
    return myElement.canNavigate();
  }

  @Override
  public boolean canNavigateToSource() {
    return myElement.canNavigateToSource();
  }

  @NotNull
  @Override
  public String getAlphaSortKey() {
    String name = myElement.getName();
    return name != null ? name : "";
  }

  @NotNull
  @Override
  public ItemPresentation getPresentation() {
    ItemPresentation presentation = myElement.getPresentation();
    return presentation != null ? presentation : new PresentationData();
  }

  @Override
  public TreeElement @NotNull [] getChildren() {
    if (myElement instanceof SimpleFile) {
      List<SimpleProperty> properties = PsiTreeUtil.getChildrenOfTypeAsList(myElement, SimpleProperty.class);
      List<TreeElement> treeElements = new ArrayList<>(properties.size());
      for (SimpleProperty property : properties) {
        treeElements.add(new SimpleStructureViewElement((SimplePropertyImpl) property));
      }
      return treeElements.toArray(new TreeElement[0]);
    }
    return EMPTY_ARRAY;
  }

}
```

### 注册结构视图工厂
使用 `com.intellij.lang.psiStructureViewFactory` 扩展点在插件配置文件中注册 `SimpleStructureViewFactory` 实现。

```xml
<extensions defaultExtensionNs="com.intellij">
  <lang.psiStructureViewFactory
      language="Simple"
      implementationClass="org.intellij.sdk.language.SimpleStructureViewFactory"/>
</extensions>
```



## 结构感知导航栏 (Structure Aware Navigation Bar)
编辑页面最后修改时间：2024年7月31日

参考文档：导航栏 (Navigation Bar)

代码：`SimpleStructureAwareNavbar`

这是自定义语言支持教程的多个步骤之一。

结构感知导航栏允许根据光标位置在导航栏中显示特定的文件元素。例如，在Java中，这用于显示光标当前所在的类和方法。

### 定义结构感知导航栏
`SimpleStructureAwareNavbar` 实现了 `StructureAwareNavBarModelExtension`。

```java
final class SimpleStructureAwareNavbar extends StructureAwareNavBarModelExtension {

  @NotNull
  @Override
  protected Language getLanguage() {
    return SimpleLanguage.INSTANCE;
  }

  @Override
  public @Nullable String getPresentableText(Object object) {
    if (object instanceof SimpleFile) {
      return ((SimpleFile) object).getName();
    }
    if (object instanceof SimpleProperty) {
      return ((SimpleProperty) object).getName();
    }

    return null;
  }

  @Override
  @Nullable
  public Icon getIcon(Object object) {
    if (object instanceof SimpleProperty) {
      return AllIcons.Nodes.Property;
    }

    return null;
  }

}
```

### 注册结构感知导航栏
使用 `com.intellij.navbar` 扩展点在插件配置文件中注册 `SimpleStructureAwareNavbar` 实现。

```xml
<extensions defaultExtensionNs="com.intellij">
  <navbar implementation="org.intellij.sdk.language.SimpleStructureAwareNavbar"/>
</extensions>
```

### 运行项目
通过使用 Gradle 的 `runIde` 任务运行项目。

打开 `test.simple` 文件并将光标定位在任何属性上。导航栏显示该属性的名称和图标。

## 代码格式化器 (Formatter)
编辑页面最后修改时间：2024年7月31日

参考：代码格式化器 (Code Formatter)

代码：`SimpleBlock`, `SimpleFormattingModelBuilder`

这是自定义语言支持教程的多个步骤之一。

IntelliJ 平台包括一个强大的框架，用于为自定义语言实现格式化功能。格式化器可以根据代码样式设置自动重新格式化代码。格式化器控制空格、缩进、换行和对齐。

### 定义一个 Block
格式化模型将文件的格式结构表示为 `Block` 对象的树，具有关联的缩进、换行、对齐和间距设置。目标是用这样的块覆盖每个 PSI 元素。由于每个块构建其子块，它可以生成额外的块或跳过任何 PSI 元素。基于 `AbstractBlock` 定义 `SimpleBlock`。

```java
public class SimpleBlock extends AbstractBlock {

  private final SpacingBuilder spacingBuilder;

  protected SimpleBlock(@NotNull ASTNode node, @Nullable Wrap wrap, @Nullable Alignment alignment,
                        SpacingBuilder spacingBuilder) {
    super(node, wrap, alignment);
    this.spacingBuilder = spacingBuilder;
  }

  @Override
  protected List<Block> buildChildren() {
    List<Block> blocks = new ArrayList<>();
    ASTNode child = myNode.getFirstChildNode();
    while (child != null) {
      if (child.getElementType() != TokenType.WHITE_SPACE) {
        Block block = new SimpleBlock(child, Wrap.createWrap(WrapType.NONE, false), Alignment.createAlignment(),
            spacingBuilder);
        blocks.add(block);
      }
      child = child.getTreeNext();
    }
    return blocks;
  }

  @Override
  public Indent getIndent() {
    return Indent.getNoneIndent();
  }

  @Nullable
  @Override
  public Spacing getSpacing(@Nullable Block child1, @NotNull Block child2) {
    return spacingBuilder.getSpacing(this, child1, child2);
  }

  @Override
  public boolean isLeaf() {
    return myNode.getFirstChildNode() == null;
  }

}
```

### 定义一个格式化模型构建器
定义一个格式化器，该格式化器会移除多余的空格，除了属性分隔符周围的单个空格外：

**格式化前**
```properties
foo  =    bar
```

**格式化后**
```properties
foo = bar
```

通过实现 `FormattingModelBuilder` 创建 `SimpleFormattingModelBuilder`。

```java
final class SimpleFormattingModelBuilder implements FormattingModelBuilder {

  private static SpacingBuilder createSpaceBuilder(CodeStyleSettings settings) {
    return new SpacingBuilder(settings, SimpleLanguage.INSTANCE)
        .around(SimpleTypes.SEPARATOR)
        .spaceIf(settings.getCommonSettings(SimpleLanguage.INSTANCE.getID()).SPACE_AROUND_ASSIGNMENT_OPERATORS)
        .before(SimpleTypes.PROPERTY)
        .none();
  }

  @Override
  public @NotNull FormattingModel createModel(@NotNull FormattingContext formattingContext) {
    final CodeStyleSettings codeStyleSettings = formattingContext.getCodeStyleSettings();
    return FormattingModelProvider
        .createFormattingModelForPsiFile(formattingContext.getContainingFile(),
            new SimpleBlock(formattingContext.getNode(),
                Wrap.createWrap(WrapType.NONE, false),
                Alignment.createAlignment(),
                createSpaceBuilder(codeStyleSettings)),
            codeStyleSettings);
  }

}
```

### 注册格式化器
使用 `com.intellij.lang.formatter` 扩展点在插件配置文件中注册 `SimpleFormattingModelBuilder` 实现。

```xml
<extensions defaultExtensionNs="com.intellij">
  <lang.formatter
      language="Simple"
      implementationClass="org.intellij.sdk.language.SimpleFormattingModelBuilder"/>
</extensions>
```

### 运行项目
通过使用 Gradle 的 `runIde` 任务运行项目。

在 IDE 开发实例中打开示例 Simple Language 属性文件。在 `language` 和 `English` 之间的 `=` 分隔符周围添加一些额外的空格。通过调用 `Code | Reformat File...` 对话框并选择 `Run` 来重新格式化代码。

## 代码风格设置
编辑页面最后修改时间：2024年7月31日

参考：代码风格设置

代码：`SimpleCodeStyleSettings`, `SimpleCodeStyleSettingsProvider`, `SimpleLanguageCodeStyleSettingsProvider`

这是自定义语言支持教程的多个步骤之一。

代码风格设置允许定义格式化选项。代码风格设置提供者创建设置的实例，并在设置中创建一个选项页面。本示例创建了一个设置页面，该页面使用默认的语言代码风格设置，并由语言代码风格设置提供者进行自定义。

### 定义代码风格设置
通过继承 `CustomCodeStyleSettings` 为 Simple 语言定义 `SimpleCodeStyleSettings`。

```java
public class SimpleCodeStyleSettings extends CustomCodeStyleSettings {

  public SimpleCodeStyleSettings(CodeStyleSettings settings) {
    super("SimpleCodeStyleSettings", settings);
  }

}
```

### 定义代码风格设置提供者
代码风格设置提供者为 IntelliJ 平台提供了一个标准方式来实例化 Simple 语言的 `CustomCodeStyleSettings`。

通过继承 `CodeStyleSettingsProvider` 定义 Simple 语言的 `SimpleCodeStyleSettingsProvider`。

```java
final class SimpleCodeStyleSettingsProvider extends CodeStyleSettingsProvider {

  @Override
  public CustomCodeStyleSettings createCustomSettings(@NotNull CodeStyleSettings settings) {
    return new SimpleCodeStyleSettings(settings);
  }

  @Override
  public String getConfigurableDisplayName() {
    return "Simple";
  }

  @NotNull
  public CodeStyleConfigurable createConfigurable(@NotNull CodeStyleSettings settings,
                                                  @NotNull CodeStyleSettings modelSettings) {
    return new CodeStyleAbstractConfigurable(settings, modelSettings, this.getConfigurableDisplayName()) {
      @Override
      protected @NotNull CodeStyleAbstractPanel createPanel(@NotNull CodeStyleSettings settings) {
        return new SimpleCodeStyleMainPanel(getCurrentSettings(), settings);
      }
    };
  }

  private static class SimpleCodeStyleMainPanel extends TabbedLanguageCodeStylePanel {

    public SimpleCodeStyleMainPanel(CodeStyleSettings currentSettings, CodeStyleSettings settings) {
      super(SimpleLanguage.INSTANCE, currentSettings, settings);
    }

  }

}
```

### 注册代码风格设置提供者
使用 `com.intellij.codeStyleSettingsProvider` 扩展点在插件配置文件中注册 `SimpleCodeStyleSettingsProvider` 实现。

```xml
<extensions defaultExtensionNs="com.intellij">
  <codeStyleSettingsProvider
      implementation="org.intellij.sdk.language.SimpleCodeStyleSettingsProvider"/>
</extensions>
```

### 定义语言代码风格设置提供者
通过继承 `LanguageCodeStyleSettingsProvider` 为 Simple 语言定义 `SimpleLanguageCodeStyleSettingsProvider`，它为特定语言提供通用的代码风格设置。

```java
final class SimpleLanguageCodeStyleSettingsProvider extends LanguageCodeStyleSettingsProvider {

  @NotNull
  @Override
  public Language getLanguage() {
    return SimpleLanguage.INSTANCE;
  }

  @Override
  public void customizeSettings(@NotNull CodeStyleSettingsCustomizable consumer, @NotNull SettingsType settingsType) {
    if (settingsType == SettingsType.SPACING_SETTINGS) {
      consumer.showStandardOptions("SPACE_AROUND_ASSIGNMENT_OPERATORS");
      consumer.renameStandardOption("SPACE_AROUND_ASSIGNMENT_OPERATORS", "Separator");
    } else if (settingsType == SettingsType.BLANK_LINES_SETTINGS) {
      consumer.showStandardOptions("KEEP_BLANK_LINES_IN_CODE");
    }
  }

  @Override
  public String getCodeSample(@NotNull SettingsType settingsType) {
    return """
        # You are reading the ".properties" entry.
        ! The exclamation mark can also mark text as comments.
        website = https://en.wikipedia.org/

        language = English
        # The backslash below tells the application to continue reading
        # the value onto the next line.
        message = Welcome to \\
                  Wikipedia!
        # Add spaces to the key
        key\\ with\\ spaces = This is the value that could be looked up with the key "key with spaces".
        # Unicode
        tab : \\u0009""";
  }

}
```

### 注册语言代码风格设置提供者
使用 `com.intellij.langCodeStyleSettingsProvider` 扩展点在插件配置文件中注册 `SimpleLanguageCodeStyleSettingsProvider` 实现。

```xml
<extensions defaultExtensionNs="com.intellij">
  <langCodeStyleSettingsProvider
      implementation="org.intellij.sdk.language.SimpleLanguageCodeStyleSettingsProvider"/>
</extensions>
```

### 运行项目
通过使用 Gradle 的 `runIde` 任务运行项目。

在 IDE 开发实例中，打开 Simple 语言的代码格式化页面：设置 | 编辑器 | 代码风格 | Simple。

## 注释器
编辑页面最后修改时间：2024年7月31日

参考：注释代码

代码：`SimpleCommenter`

测试：9. 注释器测试

这是自定义语言支持教程的多个步骤之一。

注释器允许用户自动注释掉光标处的代码行或选定的代码。注释器定义了对 `Code | Comment with Line Comment` 和 `Code | Comment with Block Comment` 操作的支持。

### 定义注释器
`SimpleCommenter` 为 Simple 语言定义了行注释前缀为 `#`。

```java
final class SimpleCommenter implements Commenter {

  @Override
  public String getLineCommentPrefix() {
    return "#";
  }

  @Override
  public String getBlockCommentPrefix() {
    return "";
  }

  @Nullable
  @Override
  public String getBlockCommentSuffix() {
    return null;
  }

  @Nullable
  @Override
  public String getCommentedBlockCommentPrefix() {
    return null;
  }

  @Nullable
  @Override
  public String getCommentedBlockCommentSuffix() {
    return null;
  }

}
```

### 注册注释器
使用 `com.intellij.lang.commenter` 扩展点在插件配置文件中注册 `SimpleCommenter` 实现。

```xml
<extensions defaultExtensionNs="com.intellij">
  <lang.commenter
      language="Simple"
      implementationClass="org.intellij.sdk.language.SimpleCommenter"/>
</extensions>
```

### 运行项目
通过使用 Gradle 的 `runIde` 任务运行项目。

在 IDE 开发实例中，打开 Simple 语言的示例属性文件。将光标置于 `website` 行。选择 `Code | Comment with Line Comment`。该行将被转换为注释。再次选择 `Code | Comment with Line Comment`，注释将被转换回有效代码。



## 快速修复
编辑页面最后修改时间：2024年7月31日

参考：代码检查与意图

代码：`SimpleElementFactory`，`SimpleCreatePropertyQuickFix`，`SimpleAnnotator`

这是自定义语言支持教程的多个步骤之一。

自定义语言的快速修复支持基于 IntelliJ 平台的 IDE 功能——意图动作（Intention Actions）。对于 Simple 语言，本教程添加了一个快速修复功能，帮助用户在使用未解析的属性时定义该属性。

### 更新元素工厂
`SimpleElementFactory` 现在包含两个新方法，支持用户选择为 Simple 语言的快速修复创建新属性。新的 `createCRLF()` 方法支持在 `test.simple` 文件的末尾添加新行之前添加换行符。新重载的 `createProperty()` 方法创建一个新的键值对。

```java
public class SimpleElementFactory {

  public static SimpleProperty createProperty(Project project, String name) {
    final SimpleFile file = createFile(project, name);
    return (SimpleProperty) file.getFirstChild();
  }

  public static SimpleFile createFile(Project project, String text) {
    String name = "dummy.simple";
    return (SimpleFile) PsiFileFactory.getInstance(project).createFileFromText(name, SimpleFileType.INSTANCE, text);
  }

  public static SimpleProperty createProperty(Project project, String name, String value) {
    final SimpleFile file = createFile(project, name + " = " + value);
    return (SimpleProperty) file.getFirstChild();
  }

  public static PsiElement createCRLF(Project project) {
    final SimpleFile file = createFile(project, "\n");
    return file.getFirstChild();
  }

}
```

### 定义意图动作
`SimpleCreatePropertyQuickFix` 创建了一个由用户选择的文件中的属性——在这种情况下，一个包含 `prefix:key` 的 Java 文件——并在创建后导航到该属性。实际上，`SimpleCreatePropertyQuickFix` 是一个意图动作。有关意图动作的更深入示例，请参阅 `conditional_operator_intention`。

```java
class SimpleCreatePropertyQuickFix extends BaseIntentionAction {

  private final String key;

  SimpleCreatePropertyQuickFix(String key) {
    this.key = key;
  }

  @NotNull
  @Override
  public String getText() {
    return "Create property '" + key + "'";
  }

  @NotNull
  @Override
  public String getFamilyName() {
    return "Create property";
  }

  @Override
  public boolean isAvailable(@NotNull Project project, Editor editor, PsiFile file) {
    return true;
  }

  @Override
  public void invoke(@NotNull final Project project, final Editor editor, PsiFile file) throws
      IncorrectOperationException {
    ApplicationManager.getApplication().invokeLater(() -> {
      Collection<VirtualFile> virtualFiles =
          FileTypeIndex.getFiles(SimpleFileType.INSTANCE, GlobalSearchScope.allScope(project));
      if (virtualFiles.size() == 1) {
        createProperty(project, virtualFiles.iterator().next());
      } else {
        final FileChooserDescriptor descriptor =
            FileChooserDescriptorFactory.createSingleFileDescriptor(SimpleFileType.INSTANCE);
        descriptor.setRoots(ProjectUtil.guessProjectDir(project));
        final VirtualFile file1 = FileChooser.chooseFile(descriptor, project, null);
        if (file1 != null) {
          createProperty(project, file1);
        }
      }
    });
  }

  private void createProperty(final Project project, final VirtualFile file) {
    WriteCommandAction.writeCommandAction(project).run(() -> {
      SimpleFile simpleFile = (SimpleFile) PsiManager.getInstance(project).findFile(file);
      assert simpleFile != null;
      ASTNode lastChildNode = simpleFile.getNode().getLastChildNode();
      // TODO: Add another check for CRLF
      if (lastChildNode != null/* && !lastChildNode.getElementType().equals(SimpleTypes.CRLF)*/) {
        simpleFile.getNode().addChild(SimpleElementFactory.createCRLF(project).getNode());
      }
      // IMPORTANT: change spaces to escaped spaces or the new node will only have the first word for the key
      SimpleProperty property = SimpleElementFactory.createProperty(project, key.replaceAll(" ", "\\\\ "), "");
      simpleFile.getNode().addChild(property.getNode());
      ((Navigatable) property.getLastChild().getNavigationElement()).navigate(true);
      Editor editor = FileEditorManager.getInstance(project).getSelectedTextEditor();
      assert editor != null;
      editor.getCaretModel().moveCaretRelatively(2, 0, false, false, false);
    });
  }

}
```

### 更新注释器
在创建 `badProperty` 注释时，`SimpleAnnotator` 中的 `badProperty.registerFix()` 方法被调用。此方法调用注册 `SimpleCreatePropertyQuickFix` 作为意图动作，供 IntelliJ 平台用于解决问题。

```java
final class SimpleAnnotator implements Annotator {

  // Define strings for the Simple language prefix - used for annotations, line markers, etc.
  public static final String SIMPLE_PREFIX_STR = "simple";
  public static final String SIMPLE_SEPARATOR_STR = ":";

  @Override
  public void annotate(@NotNull final PsiElement element, @NotNull AnnotationHolder holder) {
    // Ensure the PSI Element is an expression
    if (!(element instanceof PsiLiteralExpression literalExpression)) {
      return;
    }

    // Ensure the PSI element contains a string that starts with the prefix and separator
    String value = literalExpression.getValue() instanceof String ? (String) literalExpression.getValue() : null;
    if (value == null || !value.startsWith(SIMPLE_PREFIX_STR + SIMPLE_SEPARATOR_STR)) {
      return;
    }

    // Define the text ranges (start is inclusive, end is exclusive)
    // "simple:key"
    //  01234567890
    TextRange prefixRange = TextRange.from(element.getTextRange().getStartOffset(), SIMPLE_PREFIX_STR.length() + 1);
    TextRange separatorRange = TextRange.from(prefixRange.getEndOffset(), SIMPLE_SEPARATOR_STR.length());
    TextRange keyRange = new TextRange(separatorRange.getEndOffset(), element.getTextRange().getEndOffset() - 1);

    // highlight "simple" prefix and ":" separator
    holder.newSilentAnnotation(HighlightSeverity.INFORMATION)
        .range(prefixRange).textAttributes(DefaultLanguageHighlighterColors.KEYWORD).create();
    holder.newSilentAnnotation(HighlightSeverity.INFORMATION)
        .range(separatorRange).textAttributes(SimpleSyntaxHighlighter.SEPARATOR).create();

    // Get the list of properties for given key
    String key = value.substring(SIMPLE_PREFIX_STR.length() + SIMPLE_SEPARATOR_STR.length());
    List<SimpleProperty> properties = SimpleUtil.findProperties(element.getProject(), key);
    if (properties.isEmpty()) {
      holder.newAnnotation(HighlightSeverity.ERROR, "Unresolved property")
          .range(keyRange)
          .highlightType(ProblemHighlightType.LIKE_UNKNOWN_SYMBOL)
          // ** Tutorial step 19. - Add a quick fix for the string containing possible properties
          .withFix(new SimpleCreatePropertyQuickFix(key))
          .create();
    } else {
      // Found at least one property, force the text attributes to Simple syntax value character
      holder.newSilentAnnotation(HighlightSeverity.INFORMATION)
          .range(keyRange).textAttributes(SimpleSyntaxHighlighter.VALUE).create();
    }
  }

}
```





## 文档提供者
编辑页面最后修改时间：2024年7月31日

参考：文档

代码：`SimpleDocumentationProvider`，`SimpleUtil`

测试：11. 文档测试

这是自定义语言支持教程的多个步骤之一。

对于目标版本为 IntelliJ 平台 2023.1 或更高版本的插件，建议使用文档目标 API，如文档目标 API 中所述。

`DocumentationProvider` 帮助用户在编辑器内显示符号（如方法调用）的文档。对于自定义语言教程，我们为 Simple 语言实现了此扩展点（EP）的一个版本，该版本显示键/值、定义文件以及相关的文档注释。

### 实现文档提供者并注册扩展点
首先，我们创建一个扩展 `AbstractDocumentationProvider` 的空类并在 `plugin.xml` 中注册它。

```java
final class SimpleDocumentationProvider extends AbstractDocumentationProvider { }
```

确保该类在 `plugin.xml` 的 `extensions` 标签之间注册，如下所示：

```xml
<extensions defaultExtensionNs="com.intellij">
  <!-- 其他扩展… -->
  <lang.documentationProvider
      language="Simple"
      implementationClass="org.intellij.sdk.language.SimpleDocumentationProvider"/>
</extensions>
```

### 确保使用正确的 PSI 元素
对于 Simple 语言，我们考虑两种使用情况：

1. Simple 键在 Java 字符串字面量中使用，我们希望直接在 Java 文件中的引用显示该键/值的文档。

2. 光标已经位于 Simple 文件中的键/值定义上，在这种情况下，我们也希望显示其文档。

为确保 IntelliJ 平台在调用 View | Quick Documentation 时选择正确的 `SimpleProperty` 类型的元素，我们创建了一个 `generateDoc()` 的空实现：

```java
@Override
public @Nullable String generateDoc(PsiElement element,
                                    @Nullable PsiElement originalElement) {
  return super.generateDoc(element, originalElement);
}
```

现在，我们在空实现中设置一个断点，调试插件，并在 Java 文件和 Simple 文件中分别为 Simple 属性调用 View | Quick Documentation。我们通过将光标放在键上并调用 View | Quick Documentation 来显示文档。

在这两种情况下，提供的元素都是 `SimplePropertyImpl`，这是我们所希望的。然而，有两个问题：在 Java 字符串中，光标需要直接位于字符串 `"simple:key"` 中的键上才能使 Quick Documentation 工作。由于 Simple 语言只允许每个字符串中包含一个属性，因此如果光标位于字符串中的任何位置，只要字符串包含 Simple 属性，就希望 Quick Documentation 都能工作。在 Simple 文件中，情况类似，调用 View | Quick Documentation 仅在光标位于键上时有效。

请参阅下面的附录，其中解释了如何通过另外覆盖 `getCustomDocumentationElement()` 方法来改进此情况。

### 从键/值定义中提取文档注释
虽然 `SimpleProperty` 元素将为我们提供键和值，但我们无法直接访问可能在键/值定义前的注释。由于我们希望在文档中显示此注释，因此我们需要一个小助手函数来提取注释中的文本。此函数将驻留在 `SimpleUtil` 类中，并将找到例如在以下短例子中位于 `apikey` 前面的注释：

```properties
# An application programming interface key (API key) is a unique identifier
# used to authenticate a user, developer, or calling program to an API.
apikey=ph342m91337h4xX0r5k!11Zz!
```

以下实现将检查是否有任何注释位于 `SimpleProperty` 前，如果有，它将收集所有注释行，直到它到达前一个键/值定义或文件的开头。一个警告是，由于我们是向后收集注释行，因此需要在将它们连接成一个字符串之前反转列表。使用简单的正则表达式去除每行前导的井号和空白字符。

```java
/**
 * 尝试收集 Simple 键/值对上方的任何注释元素。
 */
public static @NotNull String findDocumentationComment(SimpleProperty property) {
  List<String> result = new LinkedList<>();
  PsiElement element = property.getPrevSibling();
  while (element instanceof PsiComment || element instanceof PsiWhiteSpace) {
    if (element instanceof PsiComment) {
      String commentText = element.getText().replaceFirst("[!# ]+", "");
      result.add(commentText);
    }
    element = element.getPrevSibling();
  }
  return StringUtil.join(Lists.reverse(result), "\n ");
}
```

### 渲染文档
通过简便的方法访问键、值、文件和可能的文档注释，我们现在已准备好提供 `generateDoc()` 的有用实现。

```java
/**
 * 提取 Simple 键/值条目的键、值、文件和文档注释，并返回
 * 信息的格式化表示。
 */
@Override
public @Nullable String generateDoc(PsiElement element, @Nullable PsiElement originalElement) {
  if (element instanceof SimpleProperty) {
    final String key = ((SimpleProperty) element).getKey();
    final String value = ((SimpleProperty) element).getValue();
    final String file = SymbolPresentationUtil.getFilePathPresentation(element.getContainingFile());
    final String docComment = SimpleUtil.findDocumentationComment((SimpleProperty) element);

    return renderFullDoc(key, value, file, docComment);
  }
  return null;
}
```

为清晰起见，渲染文档的创建在单独的方法中进行。它使用 `DocumentationMarkup` 对内容进行对齐和格式化。

```java
/**
 * 使用 {@link DocumentationMarkup} 创建格式化的文档。
 */
private String renderFullDoc(String key, String value, String file, String docComment) {
  StringBuilder sb = new StringBuilder();
  sb.append(DocumentationMarkup.DEFINITION_START);
  sb.append("Simple Property");
  sb.append(DocumentationMarkup.DEFINITION_END);
  sb.append(DocumentationMarkup.CONTENT_START);
  sb.append(value);
  sb.append(DocumentationMarkup.CONTENT_END);
  sb.append(DocumentationMarkup.SECTIONS_START);
  addKeyValueSection("Key:", key, sb);
  addKeyValueSection("Value:", value, sb);
  addKeyValueSection("File:", file, sb);
  addKeyValueSection("Comment:", docComment, sb);
  sb.append(DocumentationMarkup.SECTIONS_END);
  return sb.toString();
}
```

`addKeyValueSection()` 方法是一个小助手函数，用于减少重复。

```java
/**
 * 为渲染的文档创建一个键/值行。
 */
private void addKeyValueSection(String key, String value, StringBuilder sb) {
  sb.append(DocumentationMarkup.SECTION_HEADER_START);
  sb.append(key);
  sb.append(DocumentationMarkup.SECTION_SEPARATOR);
  sb.append("<p>");
  sb.append(value);
  sb.append(DocumentationMarkup.SECTION_END);
}
```

实现上述所有步骤后，当使用 View | Quick Documentation 调用时，IDE 将显示 Simple 键的渲染文档。

### 实现额外功能
我们可以为 `DocumentationProvider` 提供的额外功能提供实现。例如，当鼠标悬停在代码上时，它还会在短暂延迟后显示文档。该弹出窗口显示的信息不一定与调用 Quick Documentation 时相同，但为了本教程的目的，我们将这样做。

```java
/**
 * 当鼠标悬停在 Simple 语言元素上时提供文档。
 */
@Override
public @Nullable String generateHoverDoc(@NotNull PsiElement element, @Nullable PsiElement originalElement) {
  return generateDoc(element, originalElement);
}
```

当鼠标悬停在按下 
Ctrl
/
Cmd
 的代码上时，IDE 会显示光标下符号的导航信息，例如其命名空间或包。下面的实现将显示 Simple 键及其定义的文件。

```java
/**
 * 提供 Simple 语言键/值定义所在文件的信息。
 */
@Override
public @Nullable String getQuickNavigateInfo(PsiElement element, PsiElement originalElement) {
  if (element instanceof SimpleProperty) {
    final String key = ((SimpleProperty) element).getKey();
    final String file = SymbolPresentationUtil.getFilePathPresentation(element.getContainingFile());
    return "\"" + key + "\" in " + file;
  }
  return null;
}
```

最后，可以从自动补全弹出窗口中的选定条目调用 View | Quick Documentation。在这种情况下，语言开发人员需要确保为生成文档提供正确的 PSI 元素。在 Simple 语言的情况下，查找元素已经是 `SimpleProperty`，不需要做额外的工作。在其他情况下，您可以覆盖 `getDocumentationElementForLookupItem()` 并返回正确的 PSI 元素。

### 附录：选择更好的目标元素
要能够在 Java 字符串字面量的所有位置为 Simple 属性调用 View | Quick Documentation，需要执行两个步骤：

1. 需要将扩展点从 `lang.documentationProvider` 更改为 `documentationProvider`，因为只有这样 Simple 文档提供者才会为具有不同语言的 PSI 元素调用。

2. 需要实现 `getCustomDocumentationElement()` 方法以找到生成文档的正确目标 PSI 元素。

因此，代码的当前版本可以扩展以检查是否从 Java 字符串内部或 Simple 文件内部调用 View | Quick Documentation。然后，它使用 PSI 和 `PsiReference

` 功能确定正确的目标元素。这允许在 Java 字符串字面量或 Simple 属性定义的任意位置调用文档。

```java
@Override
public @Nullable PsiElement getCustomDocumentationElement(@NotNull Editor editor,
                                                          @NotNull PsiFile file,
                                                          @Nullable PsiElement context,
                                                          int targetOffset) {
  if (context != null) {
    // 在此部分，SimpleProperty 元素从 Java 字符串内部提取
    if (context instanceof PsiJavaToken &&
        ((PsiJavaToken) context).getTokenType().equals(JavaTokenType.STRING_LITERAL)) {
      PsiElement parent = context.getParent();
      PsiReference[] references = parent.getReferences();
      for (PsiReference ref : references) {
        if (ref instanceof SimpleReference) {
          PsiElement property = ref.resolve();
          if (property instanceof SimpleProperty) {
            return property;
          }
        }
      }
    }
    // 在此部分，SimpleProperty 元素在 .simple 文件内时被提取
    else if (context.getLanguage() == SimpleLanguage.INSTANCE) {
      PsiElement property =
        PsiTreeUtil.getParentOfType(context, SimpleProperty.class);
      if (property != null) {
        return property;
      }
    }
  }
  return super.getCustomDocumentationElement(
      editor, file, context, targetOffset);
}
```



## 拼写检查
编辑页面最后修改时间：2024年7月31日

参考：拼写检查

代码：`SimpleSpellcheckingStrategy`

这是自定义语言支持教程的多个步骤之一。

拼写检查允许用户在编辑代码时看到拼写错误。

### 定义 `SimpleSpellcheckingStrategy`
`SimpleSpellcheckingStrategy` 继承自 `SpellcheckingStrategy`，并实现了拼写检查策略。

```java
final class SimpleSpellcheckingStrategy extends SpellcheckingStrategy {

  @Override
  public @NotNull Tokenizer<?> getTokenizer(PsiElement element) {
    if (element instanceof PsiComment) {
      return new SimpleCommentTokenizer();
    }

    if (element instanceof SimpleProperty) {
      return new SimplePropertyTokenizer();
    }

    return EMPTY_TOKENIZER;
  }

  private static class SimpleCommentTokenizer extends Tokenizer<PsiComment> {

    @Override
    public void tokenize(@NotNull PsiComment element, @NotNull TokenConsumer consumer) {
      // 排除拼写检查时注释开头的 # 字符
      int startIndex = 0;
      for (char c : element.textToCharArray()) {
        if (c == '#' || Character.isWhitespace(c)) {
          startIndex++;
        } else {
          break;
        }
      }
      consumer.consumeToken(element, element.getText(), false, 0,
          TextRange.create(startIndex, element.getTextLength()),
          CommentSplitter.getInstance());
    }

  }

  private static class SimplePropertyTokenizer extends Tokenizer<SimpleProperty> {

    public void tokenize(@NotNull SimpleProperty element, @NotNull TokenConsumer consumer) {
      // 对属性的键和值使用不同的分词器进行拼写检查
      final ASTNode key = element.getNode().findChildByType(SimpleTypes.KEY);
      if (key != null && key.getTextLength() > 0) {
        final PsiElement keyPsi = key.getPsi();
        final String text = key.getText();
        // 对键使用标识符分词器
        consumer.consumeToken(keyPsi, text, true, 0,
            TextRange.allOf(text), IdentifierSplitter.getInstance());
      }

      final ASTNode value = element.getNode().findChildByType(SimpleTypes.VALUE);
      if (value != null && value.getTextLength() > 0) {
        final PsiElement valuePsi = value.getPsi();
        final String text = valuePsi.getText();
        // 对值使用纯文本分词器
        consumer.consumeToken(valuePsi, text, false, 0,
            TextRange.allOf(text), PlainTextSplitter.getInstance());
      }
    }

  }

}
```

### 注册 `SimpleSpellcheckingStrategy`
在插件配置文件 `plugin.xml` 中使用 `com.intellij.spellchecker.support` 扩展点注册该实现。

```xml
<extensions defaultExtensionNs="com.intellij">
  <spellchecker.support language="Simple" implementationClass="org.intellij.sdk.language.SimpleSpellcheckingStrategy"/>
</extensions>
```

通过定义和注册 `SimpleSpellcheckingStrategy`，插件现在支持对 Simple 语言文件中的注释和属性进行拼写检查。这允许开发人员更容易地发现和修复拼写错误。

























## 插件设计

### 一般建议

优秀的 IntelliJ 平台插件，如同其他产品一样，应该为用户带来显著的价值。在规划工作时，应与用户交流并尝试了解他们最需要什么，并优先考虑关键功能。

如果对已实现的解决方案不确定，可以考虑与一个有限的用户群体（如同事或活跃的社区成员）分享正在进行的工作，并收集反馈以帮助改进最终结果。有关如何自动共享预发布插件版本的信息，请参阅“自定义发布渠道”部分。

收集现有功能的反馈可以帮助识别哪些工作做得好，哪些需要改进，这也有助于提高插件的整体质量。

### 易用性

插件应该易于使用。理想情况下，所有功能在安装后应开箱即用，无需用户进行特殊操作，例如手动启用关键插件功能。默认设置应反映典型项目中的插件使用情况。

所有的设置和操作都应易于查找，并且放置在适当的设置或操作组中，例如：

- 框架插件的设置应放在“设置 | 语言与框架”下
- 标记目录为插件特定根类型的操作应添加到“将目录标记为...”组中

难以配置、功能难以找到的插件可能会因用户的挫败感而被迅速放弃。

### 稳定性

经常在消息面板中显示大量错误并导致错误行为（如生成不正确的代码）的插件被认为是不稳定且不可靠的。

为提高整体稳定性并最大限度地减少引入回归问题的风险，实施维护成本低的功能测试非常关键。这是一种很好的安全网，从长远来看，它将加快开发速度，并帮助您在发布新版本时不必担心破坏现有插件功能。

开发没有错误的软件几乎是不可能的，因此建议设置一个问题跟踪系统，用户可以在其中报告错误。此外，值得实现错误报告功能，允许用户直接从 IDE 中发送报告，并附上所有必要的信息。为了在生产环境中重现和理解问题，请始终一致地使用日志记录。

### 性能

即使插件功能正常且外观令人愉悦，性能差仍会影响用户的满意度。缓慢的高亮显示、代码补全、代码生成等功能可能会打断用户的工作流程，并导致挫败感，进而导致插件被卸载。始终尝试遵循相关主题页面上描述的性能提示，例如 PSI 性能、避免 UI 冻结、提高索引性能。

在哑模式（dumb mode）期间尽可能多地使功能正常工作，也可以提高用户感知的性能。

### 分发大小

除了插件执行性能外，建议优化从 JetBrains Marketplace 下载的插件分发包的大小。网络连接不佳的用户可能会取消下载，因为他们意识到等待插件下载时间太长，而他们并不确定该插件是否符合他们的期望。

考虑以下优化插件分发大小的技术：

- 减少依赖项的数量。检查目标平台是否包含您需要的实用程序（如用户界面常见问题中提到的那些）或库，并重用它们。
- 确保插件分发包中没有不必要的或多个版本的相同依赖项。
- 优化图标、图片、视频等资产。
- 如果大型资源（如 SDK、参考文档）仅在特定设置中需要，请考虑在插件分发包中按需下载它们，而不是捆绑在插件分发包中。
- 混淆也可能有助于减少分发文件的大小。
