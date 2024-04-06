---
layout: single
title:  "Flutter 渲染流程"
date:   2024-04-06 16:20:00 +0800
categories: mobile_devs
---

基于 Flutter 3.13 对 Flutter 的渲染流程进行分析研究，本文按照自顶向下的方式进行梳理。

## Dart Framework 层

### `setState()` 函数触发 Widget 渲染流程

`setState()` 函数在去除断言后，核心代码只有一行：

```dart
@protected
void setState(VoidCallback fn) {
	_element!.markNeedsBuild();
}
```

_element 是 `StatefulElement` 的实例，代码中将组件标记为需要重新构建的状态。

`markNeedsBuild()` 函数中的代码（去除断言）如下：

```dart
void markNeedsBuild() {
  if (_lifecycleState != _ElementLifecycle.active) { // element 不在前台活跃，无需 rebuild
    return;
  }
  if (dirty) { // element 已经标记为 dirty 了
    return;
  }
  _dirty = true;
  owner!.scheduleBuildFor(this);
}
```

该函数进行了渲染步骤的判断，判断后执行 `owner!.scheduleBuildFor(this);`，其中 owner 定义如下：

```dart
/// The object that manages the lifecycle of this element.
@override
BuildOwner? get owner => _owner;
BuildOwner? _owner;
```

它的类型为 `BuildOwner`，`BuildOwner` 的主要作用是追踪和更新 dirty element。

接下来观察一下 `scheduleBuildFor` 函数：

```dart
void scheduleBuildFor(Element element) {
  if (element._inDirtyList) { // element 已经在 dirty list 中
    _dirtyElementsNeedsResorting = true; // 标记 dirty list 应当重新排序
    return;
  }
  if (!_scheduledFlushDirtyElements && onBuildScheduled != null) {
    _scheduledFlushDirtyElements = true;
    onBuildScheduled!(); // 调用 onBuildScheduled 回调函数
  }
  _dirtyElements.add(element); // 将 element 添加到 dirty list 中
  element._inDirtyList = true;
}
```

其中 `onBuildScheduled` 函数定义如下：

```dart
/// Called on each build pass when the first buildable element is marked dirty.
VoidCallback? onBuildScheduled;
```

### `onBuildScheduled`回调触发`scheduleFrame`

上文中的 `onBuildScheduled` 在 `WidgetsBinding` 初始化的时候被赋值：

```dart
/// The glue between the widgets layer and the Flutter engine.
mixin WidgetsBinding on BindingBase, ServicesBinding, SchedulerBinding, GestureBinding, RendererBinding, SemanticsBinding {
  @override
  void initInstances() {
    super.initInstances();
    _instance = this;

    // Initialization of [_buildOwner] has to be done after
    // [super.initInstances] is called, as it requires [ServicesBinding] to
    // properly setup the [defaultBinaryMessenger] instance.
    _buildOwner = BuildOwner();
    buildOwner!.onBuildScheduled = _handleBuildScheduled;
    platformDispatcher.onLocaleChanged = handleLocaleChanged;
    SystemChannels.navigation.setMethodCallHandler(_handleNavigationInvocation);
    platformMenuDelegate = DefaultPlatformMenuDelegate();
  }
  
  // ...
}
```

接下来依次调用 `_handleBuildScheduled` --> `ensureVisualUpdate`:

```dart
void _handleBuildScheduled() {
  ensureVisualUpdate();
}

void ensureVisualUpdate() {
  switch (schedulerPhase) {
    case SchedulerPhase.idle:
    case SchedulerPhase.postFrameCallbacks:
      scheduleFrame();
      return;
    case SchedulerPhase.transientCallbacks:
    case SchedulerPhase.midFrameMicrotasks:
    case SchedulerPhase.persistentCallbacks:
      return;
  }
}
```

`SchedulerPhase` 具有五种状态：

- idle: 没有正在处理的帧
- postFrameCallbacks: 一般为清除和准备下一帧工作
- transientCallbacks: scheduleFrameCallback正在执行，一般为更新动画状态
- midFrameMicrotasks: transient处理阶段执行的微任务
- persistentCallbacks: addPersistentFrameCallback正在执行，一般为build/layout/paint流水线工作

当SchedulerPhase处于 `transientCallbacks` 或 `midFrameMicrotasks` （帧正处于准备状态）， `persistentCallbacks`（帧正处于渲染状态）时，渲染步骤不会执行。只有当SchedulerPhase处于 `idle` （帧与帧之间）或 `postFrameCallbacks` （一帧结束）时渲染才会执行，此时执行 `scheduleFrame`。

### `scheduleFrame` 调用 engine 层函数

`scheduleFrame` 方法最终会调用引擎层对应的 `scheduleFrame` 方法：

```dart
void scheduleFrame() {
  if (_hasScheduledFrame || !framesEnabled) {
    return;
  }
  ensureFrameCallbacksRegistered();
  platformDispatcher.scheduleFrame();
  _hasScheduledFrame = true;
}
```

` platformDispatcher.scheduleFrame()` 对应到：

```dart
void scheduleFrame() => _scheduleFrame();

@Native<Void Function()>(symbol: 'PlatformConfigurationNativeApi::ScheduleFrame')
external static void _scheduleFrame();
```

## Engine 层

### Platform 发起绘制请求

/engine/lib/ui/window/platform_configuration.cc

```cpp
void PlatformConfigurationNativeApi::ScheduleFrame() {
  UIDartState::ThrowIfUIOperationsProhibited();
  UIDartState::Current()->platform_configuration()->client()->ScheduleFrame();
}
```

/engine/shell/common/platform_view.cc

```cpp
void PlatformView::ScheduleFrame() {
  delegate_.OnPlatformViewScheduleFrame();
}
```

/engine/shell/common/shell.cc

```cpp
// |PlatformView::Delegate|
void Shell::OnPlatformViewScheduleFrame() {
  task_runners_.GetUITaskRunner()->PostTask([engine = engine_->GetWeakPtr()]() {
    if (engine) {
      engine->ScheduleFrame();
    }
  });
}
```

经过 Plateform 层一系列代理和调用，最终调用engine的`ScheduleFrame`方法。

/engine/shell/common/engine.h

```cpp
/// Schedule a frame with the default parameter of regenerating the layer tree.
void ScheduleFrame() { ScheduleFrame(true); }
```

/engine/shell/common/engine.cc

```cpp
void Engine::ScheduleFrame(bool regenerate_layer_trees) {
  animator_->RequestFrame(regenerate_layer_trees);
}
```

### vsync 信号注册

由上文可知 Engine::ScheduleFrame 函数中调用了 Animator 的 RequestFrame 方法。`RequestFrame`方法在对 vsync 信号进行判断后将真正需要发送的 vsync 信号通过 `PostTask` 方法将 `AwaitVsync` 任务放入 UI Task Runner 中执行。

/engine/shell/common/animator.cc

```cpp
void Animator::RequestFrame(bool regenerate_layer_trees = true) {
  if (regenerate_layer_trees) {
    // This event will be closed by BeginFrame. BeginFrame will only be called
    // if regenerating the layer trees. If a frame has been requested to update
    // an external texture, this will be false and no BeginFrame call will
    // happen.
    regenerate_layer_trees_ = true;
  }

  if (!pending_frame_semaphore_.TryWait()) {
		// 将一帧内的多个 vsync 请求转为同一个 vsync 请求
    // Multiple calls to Animator::RequestFrame will still result in a
    // single request to the VsyncWaiter.
    return;
  }

  // The AwaitVSync is going to call us back at the next VSync. However, we want
  // to be reasonably certain that the UI thread is not in the middle of a
  // particularly expensive callout. We post the AwaitVSync to run right after
  // an idle. This does NOT provide a guarantee that the UI thread has not
  // started an expensive operation right after posting this message however.
  // To support that, we need edge triggered wakes on VSync.

  task_runners_.GetUITaskRunner()->PostTask(
      [self = weak_factory_.GetWeakPtr()]() {
        if (!self) {
          return;
        }
        self->AwaitVSync();
      });
  frame_scheduled_ = true;
}
```

`AwaitVsync` 函数中判断能否复用 LayerTree，若能则绘制上一个 LayerTree（调用 `DrawLastLayerTree`），若不能则开始新的 LayerTree 的绘制（调用 `BeginFrame`）

/engine/shell/common/animator.cc

```cpp
void Animator::AwaitVSync() {
  waiter_->AsyncWaitForVsync(
      [self = weak_factory_.GetWeakPtr()](
          std::unique_ptr<FrameTimingsRecorder> frame_timings_recorder) {
        if (self) {
          if (self->CanReuseLastLayerTrees()) {
            self->DrawLastLayerTrees(std::move(frame_timings_recorder));
          } else {
            self->BeginFrame(std::move(frame_timings_recorder));
          }
        }
      });
  if (has_rendered_) {
    delegate_.OnAnimatorNotifyIdle(dart_frame_deadline_);
  }
}
```

### `BeginFrame` 执行绘制流程

engine 从 `BeginFrame` 开始执行绘制流程，

/engine/shell/common/animator.cc

```dart
void Animator::BeginFrame(
    std::unique_ptr<FrameTimingsRecorder> frame_timings_recorder) {
  frame_request_number_++; // 仅用于记录 frame request number

  // 计时
  frame_timings_recorder_ = std::move(frame_timings_recorder);
  frame_timings_recorder_->RecordBuildStart(fml::TimePoint::Now());

  size_t flow_id_count = trace_flow_ids_.size();
  std::unique_ptr<uint64_t[]> flow_ids =
      std::make_unique<uint64_t[]>(flow_id_count);
  for (size_t i = 0; i < flow_id_count; ++i) {
    flow_ids.get()[i] = trace_flow_ids_.at(i);
  }

  while (!trace_flow_ids_.empty()) {
    uint64_t trace_flow_id = trace_flow_ids_.front();
    trace_flow_ids_.pop_front();
  }

  frame_scheduled_ = false;
  regenerate_layer_trees_ = false;
  pending_frame_semaphore_.Signal();

  // 如果此时没有可用的流水线
  if (!producer_continuation_) {
    // 重新获取一遍流水线
    // We may already have a valid pipeline continuation in case a previous
    // begin frame did not result in an Animator::Render. Simply reuse that
    // instead of asking the pipeline for a fresh continuation.
    producer_continuation_ = layer_tree_pipeline_->Produce();

    // 流水线繁忙，无法开始绘制操作
    if (!producer_continuation_) {
      // If we still don't have valid continuation, the pipeline is currently
      // full because the consumer is being too slow. Try again at the next
      // frame interval.
			// 在下个帧周期重试
      RequestFrame();
      return;
    }
  }

  // We have acquired a valid continuation from the pipeline and are ready
  // to service potential frame.
  FML_DCHECK(producer_continuation_);
  // 获取该帧预期的显示时间(Timestamp of when the frame was targeted to be presented, this is typically the next vsync signal timestamp.)
  const fml::TimePoint frame_target_time =
      frame_timings_recorder_->GetVsyncTargetTime();
  dart_frame_deadline_ = frame_target_time.ToEpochDelta();
  uint64_t frame_number = frame_timings_recorder_->GetFrameNumber();
  // 回调执行 shell 相应方法
  delegate_.OnAnimatorBeginFrame(frame_target_time, frame_number);

  if (!frame_scheduled_ && has_rendered_) {
    // Wait a tad more than 3 60hz frames before reporting a big idle period.
    // This is a heuristic that is meant to avoid giving false positives to the
    // VM when we are about to schedule a frame in the next vsync, the idea
    // being that if there have been three vsyncs with no frames it's a good
    // time to start doing GC work.
    task_runners_.GetUITaskRunner()->PostDelayedTask(
        [self = weak_factory_.GetWeakPtr()]() {
          if (!self) {
            return;
          }
          // If there's a frame scheduled, bail.
          // If there's no frame scheduled, but we're not yet past the last
          // vsync deadline, bail.
          if (!self->frame_scheduled_) {
            auto now =
                fml::TimeDelta::FromMicroseconds(Dart_TimelineGetMicros());
            if (now > self->dart_frame_deadline_) {
              TRACE_EVENT0("flutter", "BeginFrame idle callback");
              self->delegate_.OnAnimatorNotifyIdle(
                  now + fml::TimeDelta::FromMilliseconds(100));
            }
          }
        },
        kNotifyIdleTaskWaitTime);
  }
}
```

Animator 的内部类 Delegate 定义了一系列回调：

```dart
class Delegate {
public:
  virtual void OnAnimatorBeginFrame(fml::TimePoint frame_target_time,
                                      uint64_t frame_number) = 0;

  virtual void OnAnimatorNotifyIdle(fml::TimeDelta deadline) = 0;

  virtual void OnAnimatorUpdateLatestFrameTargetTime(
        fml::TimePoint frame_target_time) = 0;

  virtual void OnAnimatorDraw(std::shared_ptr<FramePipeline> pipeline) = 0;

  virtual void OnAnimatorDrawLastLayerTrees(
        std::unique_ptr<FrameTimingsRecorder> frame_timings_recorder) = 0;
};
```

最终 `delegate_.OnAnimatorBeginFrame` 会映射到

/engine/shell/common/shell.cc

```cpp
// |Animator::Delegate|
void Shell::OnAnimatorBeginFrame(fml::TimePoint frame_target_time,
                                 uint64_t frame_number) {
  FML_DCHECK(is_set_up_);
  FML_DCHECK(task_runners_.GetUITaskRunner()->RunsTasksOnCurrentThread());

  // record the target time for use by rasterizer.
  {
    std::scoped_lock time_recorder_lock(time_recorder_mutex_);
    latest_frame_target_time_.emplace(frame_target_time);
  }
  if (engine_) {
    engine_->BeginFrame(frame_target_time, frame_number);
  }
}
```

其中 engine_ 类型为 `unique_ptr<Engine>`。

/engine/shell/common/engine.cc

```cpp
void Engine::BeginFrame(fml::TimePoint frame_time, uint64_t frame_number) {
  runtime_controller_->BeginFrame(frame_time, frame_number);
}
```

/engine/runtime/runtime_controller.cc

```cpp
bool RuntimeController::BeginFrame(fml::TimePoint frame_time,
                                   uint64_t frame_number) {
  if (auto* platform_configuration = GetPlatformConfigurationIfAvailable()) {
    platform_configuration->BeginFrame(frame_time, frame_number);
    return true;
  }

  return false;
}
```

### 回调 Framework

/lib/ui/window/platform_configuration.cc

```cpp
void PlatformConfiguration::BeginFrame(fml::TimePoint frameTime,
                                       uint64_t frame_number) {
  std::shared_ptr<tonic::DartState> dart_state =
      begin_frame_.dart_state().lock();
  if (!dart_state) {
    return;
  }
  tonic::DartState::Scope scope(dart_state);

  int64_t microseconds = (frameTime - fml::TimePoint()).ToMicroseconds();

  // 开始帧 begin_frame_ 对应调用 Dart 层的 handleBeginFrame
  tonic::CheckAndHandleError(
      tonic::DartInvoke(begin_frame_.Get(), {
                                                Dart_NewInteger(microseconds),
                                                Dart_NewInteger(frame_number),
                                            }));

  // 处理微任务
  UIDartState::Current()->FlushMicrotasksNow();
	// 绘制帧 draw_frame_ 对应调用 Dart 层的 handleDrawFrame
  tonic::CheckAndHandleError(tonic::DartInvokeVoid(draw_frame_.Get()));
}
```

## 回调 Framework 层

### Engine 回调 `handleBeginFrame`

`handleBeginFrame`方法调用全部已注册的`transient frame callbacks`回调函数，当其返回时，全部微任务处于正在运行的状态，并开始执行`handleDrawFrame`方法继续渲染过程。

```dart
/// Called by the engine to prepare the framework to produce a new frame.
///
/// This function calls all the transient frame callbacks registered by
/// [scheduleFrameCallback]. It then returns, any scheduled microtasks are run
/// (e.g. handlers for any [Future]s resolved by transient frame callbacks),
/// and [handleDrawFrame] is called to continue the frame.
///
/// If the given time stamp is null, the time stamp from the last frame is
/// reused.
///
/// To have a banner shown at the start of every frame in debug mode, set
/// [debugPrintBeginFrameBanner] to true. The banner will be printed to the
/// console using [debugPrint] and will contain the frame number (which
/// increments by one for each frame), and the time stamp of the frame. If the
/// given time stamp was null, then the string "warm-up frame" is shown
/// instead of the time stamp. This allows frames eagerly pushed by the
/// framework to be distinguished from those requested by the engine in
/// response to the "Vsync" signal from the operating system.
///
/// You can also show a banner at the end of every frame by setting
/// [debugPrintEndFrameBanner] to true. This allows you to distinguish log
/// statements printed during a frame from those printed between frames (e.g.
/// in response to events or timers).
void handleBeginFrame(Duration? rawTimeStamp) {
  _frameTimelineTask?.start('Frame');
  _firstRawTimeStampInEpoch ??= rawTimeStamp;
  _currentFrameTimeStamp = _adjustForEpoch(rawTimeStamp ?? _lastRawTimeStamp);
  if (rawTimeStamp != null) {
    _lastRawTimeStamp = rawTimeStamp;
  }
  _hasScheduledFrame = false;
  try {
    // TRANSIENT FRAME CALLBACKS
    _frameTimelineTask?.start('Animate');
    _schedulerPhase = SchedulerPhase.transientCallbacks;
    final Map<int, _FrameCallbackEntry> callbacks = _transientCallbacks;
    _transientCallbacks = <int, _FrameCallbackEntry>{};
    callbacks.forEach((int id, _FrameCallbackEntry callbackEntry) {
      if (!_removedIds.contains(id)) {
        _invokeFrameCallback(callbackEntry.callback, _currentFrameTimeStamp!, callbackEntry.debugStack);
      }
    });
    _removedIds.clear();
  } finally {
    _schedulerPhase = SchedulerPhase.midFrameMicrotasks;
  }
}
```

### Engine 回调 `handleDrawFrame`

`handleDrawFrame`由引擎层调用用以创建新的一帧。

```dart
/// Called by the engine to produce a new frame.
///
/// This method is called immediately after [handleBeginFrame]. It calls all
/// the callbacks registered by [addPersistentFrameCallback], which typically
/// drive the rendering pipeline, and then calls the callbacks registered by
/// [addPostFrameCallback].
///
/// See [handleBeginFrame] for a discussion about debugging hooks that may be
/// useful when working with frame callbacks.
void handleDrawFrame() {
  assert(_schedulerPhase == SchedulerPhase.midFrameMicrotasks);
  _frameTimelineTask?.finish(); // end the "Animate" phase
  try {
    // PERSISTENT FRAME CALLBACKS
  	_schedulerPhase = SchedulerPhase.persistentCallbacks;
  	for (final FrameCallback callback in List<FrameCallback>.of(_persistentCallbacks)) {
      // 依次调用 PersistFrameCallback
      _invokeFrameCallback(callback, _currentFrameTimeStamp!);
    }
    // POST-FRAME CALLBACKS
    _schedulerPhase = SchedulerPhase.postFrameCallbacks;
    final List<FrameCallback> localPostFrameCallbacks =
        List<FrameCallback>.of(_postFrameCallbacks);
    _postFrameCallbacks.clear();
    for (final FrameCallback callback in localPostFrameCallbacks) {
      _invokeFrameCallback(callback, _currentFrameTimeStamp!);
    }
  } finally {
    _schedulerPhase = SchedulerPhase.idle;
    _frameTimelineTask?.finish(); // end the Frame
    _currentFrameTimeStamp = null;
  }
}
```

该方法在`handleBeginFrame`方法后立即被调用，依次调用渲染流水线的回调函数—— `addPersistentFrameCallback`，接着调用由 `addPostFrameCallback`(常用于注册当前帧渲染结束后的回调) 注册的一系列回调函数。

其中，`_invokeFrameCallback` 定义如下：

```dart
void _invokeFrameCallback(FrameCallback callback, Duration timeStamp, [ StackTrace? callbackStack ]) {
  try {
    callback(timeStamp);
  } catch (exception, exceptionStack) {
    ...
  }
}
```



PersistentFrameCallback 是在 render binding 初始化的时候进行赋值的：

```dart
/// The glue between the render tree and the Flutter engine.
mixin RendererBinding on BindingBase, ServicesBinding, SchedulerBinding, GestureBinding, SemanticsBinding, HitTestable {
  @override
  void initInstances() {
    super.initInstances();
    _instance = this;
    _pipelineOwner = PipelineOwner(
      onSemanticsOwnerCreated: _handleSemanticsOwnerCreated,
      onSemanticsUpdate: _handleSemanticsUpdate,
      onSemanticsOwnerDisposed: _handleSemanticsOwnerDisposed,
    );
    platformDispatcher
      ..onMetricsChanged = handleMetricsChanged
      ..onTextScaleFactorChanged = handleTextScaleFactorChanged
      ..onPlatformBrightnessChanged = handlePlatformBrightnessChanged;
    initRenderView();
    
    // 注册 persistent frame callback
    addPersistentFrameCallback(_handlePersistentFrameCallback);
    initMouseTracker();
    if (kIsWeb) {
      addPostFrameCallback(_handleWebFirstFrame);
    }
    _pipelineOwner.attach(_manifold);
  }
  
  // ...
}
```

```dart
void _handlePersistentFrameCallback(Duration timeStamp) {
  drawFrame();
  _scheduleMouseTrackerUpdate();
}
```

### Render 的 `drawFrame` 方法

```dart
/// Pump the rendering pipeline to generate a frame.
@protected
void drawFrame() {
  pipelineOwner.flushLayout();
  pipelineOwner.flushCompositingBits();
  pipelineOwner.flushPaint();
  if (sendFramesToEngine) {
    renderView.compositeFrame(); // this sends the bits to the GPU
    pipelineOwner.flushSemantics(); // this also sends the semantics to the OS.
    _firstFrameSent = true;
  }
}
```

#### `flushLayout` 

`flushLayout` 更新了所有 dirty layout 的布局信息：

```dart
/// Update the layout information for all dirty render objects.
///
/// This function is one of the core stages of the rendering pipeline. Layout
/// information is cleaned prior to painting so that render objects will
/// appear on screen in their up-to-date locations.
///
/// See [RendererBinding] for an example of how this function is used.
void flushLayout() {
  if (!kReleaseMode) {
    Map<String, String>? debugTimelineArguments;
   	assert(() {
      if (debugEnhanceLayoutTimelineArguments) {
        debugTimelineArguments = <String, String>{
            'dirty count': '${_nodesNeedingLayout.length}',
            'dirty list': '$_nodesNeedingLayout',
        };
      }
      return true;
    }());
    FlutterTimeline.startSync(
      'LAYOUT',
      arguments: debugTimelineArguments,
    );
  }
  try {
    while (_nodesNeedingLayout.isNotEmpty) {
      final List<RenderObject> dirtyNodes = _nodesNeedingLayout;
      _nodesNeedingLayout = <RenderObject>[];
      dirtyNodes.sort((RenderObject a, RenderObject b) => a.depth - b.depth);
      for (int i = 0; i < dirtyNodes.length; i++) {
        if (_shouldMergeDirtyNodes) {
          _shouldMergeDirtyNodes = false;
          if (_nodesNeedingLayout.isNotEmpty) {
            _nodesNeedingLayout.addAll(dirtyNodes.getRange(i, dirtyNodes.length));
            break;
          }
        }
        final RenderObject node = dirtyNodes[i];
        if (node._needsLayout && node.owner == this) {
          node._layoutWithoutResize();
        }
      }
      // No need to merge dirty nodes generated from processing the last
      // relayout boundary back.
      _shouldMergeDirtyNodes = false;
    }
    for (final PipelineOwner child in _children) {
      child.flushLayout();
    }
  } finally {
    _shouldMergeDirtyNodes = false;
    if (!kReleaseMode) {
      FlutterTimeline.finishSync();
    }
  }
}
```

#### `flushCompositingBits` 

flushCompositingBits 更新被标记为 `RenderObject.needsCompositing` 的元素的bits:

```dart
/// Updates the [RenderObject.needsCompositing] bits.
///
/// Called as part of the rendering pipeline after [flushLayout] and before
/// [flushPaint].
void flushCompositingBits() {
  if (!kReleaseMode) {
    FlutterTimeline.startSync('UPDATING COMPOSITING BITS');
  }
  _nodesNeedingCompositingBitsUpdate.sort((RenderObject a, RenderObject b) => a.depth - b.depth);
  for (final RenderObject node in _nodesNeedingCompositingBitsUpdate) {
    if (node._needsCompositingBitsUpdate && node.owner == this) {
      node._updateCompositingBits(); // 更新 bits
    }
  }
  _nodesNeedingCompositingBitsUpdate.clear();
  for (final PipelineOwner child in _children) {
    child.flushCompositingBits();
  }
  assert(_nodesNeedingCompositingBitsUpdate.isEmpty, 'Child PipelineOwners must not dirty nodes in their parent.');
  if (!kReleaseMode) {
    FlutterTimeline.finishSync();
  }
}
```

#### `flushPaint`

 `flushPaint`为全部渲染元素更新展示列表，是渲染流水线的核心步骤，在`layout`之后，`recomposited`（重新合成）之前，保证使用最新的展示列表进行渲染。

```dart
/// Update the display lists for all render objects.
///
/// This function is one of the core stages of the rendering pipeline.
/// Painting occurs after layout and before the scene is recomposited so that
/// scene is composited with up-to-date display lists for every render object.
///
/// See [RendererBinding] for an example of how this function is used.
void flushPaint() {
  if (!kReleaseMode) {
    Map<String, String>? debugTimelineArguments;
    FlutterTimeline.startSync(
      'PAINT',
      arguments: debugTimelineArguments,
    );
  }
  try {
		final List<RenderObject> dirtyNodes = _nodesNeedingPaint;
    _nodesNeedingPaint = <RenderObject>[];

    // Sort the dirty nodes in reverse order (deepest first).
    for (final RenderObject node in dirtyNodes..sort((RenderObject a, RenderObject b)
                                                     => b.depth - a.depth)) {
      if ((node._needsPaint || node._needsCompositedLayerUpdate) && node.owner == this) {
        if (node._layerHandle.layer!.attached) {
          assert(node.isRepaintBoundary);
          if (node._needsPaint) {
            PaintingContext.repaintCompositedChild(node);
          } else {
            PaintingContext.updateLayerProperties(node);
          }
        } else {
          node._skippedPaintingOnLayer();
        }
      }
    }
    for (final PipelineOwner child in _children) {
      child.flushPaint();
    }
  } finally {
    if (!kReleaseMode) {
      FlutterTimeline.finishSync();
    }
  }
}
```

#### `compositeFrame`

compositeFrame 上传合成的layer tree至engine

```dart
/// Uploads the composited layer tree to the engine.
///
/// Actually causes the output of the rendering pipeline to appear on screen.
void compositeFrame() {
  if (!kReleaseMode) {
    FlutterTimeline.startSync('COMPOSITING');
  }
  try {
    final ui.SceneBuilder builder = ui.SceneBuilder();
    final ui.Scene scene = layer!.buildScene(builder);
    if (automaticSystemUiAdjustment) {
			_updateSystemChrome();
    }
    _view.render(scene);
    scene.dispose();
  } finally {
    if (!kReleaseMode) {
      FlutterTimeline.finishSync();
    }
  }
}
```

### Widget 的 `drawFrame` 方法

`WidgetsBinding` 混入了 `RendererBinding`，同时重写了 `drawFrame` 函数：

```dart
@override
void drawFrame() {
  TimingsCallback? firstFrameCallback;
  if (_needToReportFirstFrame) {

    firstFrameCallback = (List<FrameTiming> timings) {
      assert(sendFramesToEngine);
      if (!kReleaseMode) {
        // Change the current user tag back to the default tag. At this point,
        // the user tag should be set to "AppStartUp" (originally set in the
        // engine), so we need to change it back to the default tag to mark
        // the end of app start up for CPU profiles.
        developer.UserTag.defaultTag.makeCurrent();
        developer.Timeline.instantSync('Rasterized first useful frame');
        developer.postEvent('Flutter.FirstFrame', <String, dynamic>{});
      }
      SchedulerBinding.instance.removeTimingsCallback(firstFrameCallback!);
      firstFrameCallback = null;
      _firstFrameCompleter.complete();
    };
    // Callback is only invoked when FlutterView.render is called. When
    // sendFramesToEngine is set to false during the frame, it will not be
    // called and we need to remove the callback (see below).
    SchedulerBinding.instance.addTimingsCallback(firstFrameCallback!);
  }

  try {
    if (rootElement != null) {
      // 建立需要更新的 widget tree,调用注册过的 callback，并以深度遍历顺序 build 所有在 scheduleBuildFor 阶段被标记的脏元素。
      buildOwner!.buildScope(rootElement!);
    }
    super.drawFrame();
    buildOwner!.finalizeTree();
  } finally {
    assert(() {
      debugBuildingDirtyElements = false;
      return true;
    }());
  }
  if (!kReleaseMode) {
    if (_needToReportFirstFrame && sendFramesToEngine) {
      developer.Timeline.instantSync('Widgets built first useful frame');
    }
  }
  _needToReportFirstFrame = false;
  if (firstFrameCallback != null && !sendFramesToEngine) {
    // This frame is deferred and not the first frame sent to the engine that
    // should be reported.
    _needToReportFirstFrame = true;
    SchedulerBinding.instance.removeTimingsCallback(firstFrameCallback!);
  }
}
```

所有脏元素构建结束后，调用`buildOwner!.finalizeTree()`完成构建阶段：

```dart
@pragma('vm:notify-debugger-on-exception')
void finalizeTree() {
  if (!kReleaseMode) {
    Timeline.startSync('FINALIZE TREE');
  }
  try {
    lockState(_inactiveElements._unmountAll); // this unregisters the GlobalKeys
  } catch (e, stack) {
    _debugReportException(ErrorSummary('while finalizing the widget tree'), e, stack);
  } finally {
    if (!kReleaseMode) {
      Timeline.finishSync();
    }
  }
}
```

