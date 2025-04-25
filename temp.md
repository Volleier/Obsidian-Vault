以下是根据您的需求设计的详细PUML架构图，包含三大设计模式的实现细节和模块交互流程：

```plantuml
@startuml AI_Image_System_Architecture

skinparam monochrome true
skinparam shadowing false
skinparam defaultFontName Helvetica
skinparam classFontSize 14
skinparam arrowFontSize 12

' 定义颜色
!define MAIN_COLOR #333333
!define GENERATE_COLOR #555555
!define SEGMENT_COLOR #777777
!define FACTORY_COLOR #888888
!define OBSERVER_COLOR #999999
!define FACADE_COLOR #AAAAAA

' 包定义
package "Flask Web Service" as web {
    class "Flask App" as flask {
        + /generate (POST)
        + /segment (POST)
    }
    
    class "Client Frontend" as frontend {
        + submitPrompt()
        + displayResults()
    }
}

package "AI Processing System" as ai {
    package "Factory Pattern" as factory {
        class "AITaskFactory" as task_factory {
            + createTask(type): AITask
            {static} getInstance(): AITaskFactory
        }
        
        interface "AITask" as ai_task {
            + execute(params): Result
        }
    }
    
    package "Generation Module" as gen {
        class "ImageGenerationTask" as gen_task {
            + execute(params): GeneratedImage
            - comfyUI: ComfyUIAdapter
        }
        
        class "ComfyUIAdapter" as comfyui {
            + generateImage(prompt): Image
            - validatePrompt()
            - callComfyAPI()
        }
    }
    
    package "Segmentation Module" as seg {
        class "SegmentationTask" as seg_task {
            + execute(params): SegmentedImage
            - sam: SAMAdapter
        }
        
        class "SAMAdapter" as sam {
            + segmentImage(image): Mask
            - loadModel()
            - preprocess()
        }
    }
    
    package "Observer Pattern" as observer {
        class "TaskNotifier" as notifier {
            + register(observer)
            + notifyAll()
            - observers: List
        }
        
        interface "TaskObserver" as task_observer {
            + update(task_result)
        }
        
        class "SegmentationTrigger" as seg_trigger {
            + update(gen_result)
        }
    }
    
    package "Facade Pattern" as facade {
        class "AIServiceFacade" as facade {
            + generateAndSegment(prompt): FinalResult
            - taskFactory: AITaskFactory
            - notifier: TaskNotifier
        }
    }
}

' 关系定义
frontend -> flask : HTTP请求\n(prompt/text)
flask -> facade : 调用服务\n(generateAndSegment)
facade -> task_factory : 1.创建任务
task_factory -> gen_task : 生成图像任务
gen_task -> comfyui : 2.调用ComfyUI
comfyui -> gen_task : 返回生成的图像
gen_task -> notifier : 3.通知观察者
notifier -> seg_trigger : 触发更新
seg_trigger -> task_factory : 4.创建分割任务
task_factory -> seg_task : 分割图像任务
seg_task -> sam : 5.调用SAM模型
sam -> seg_task : 返回分割结果
seg_task -> facade : 返回最终结果
facade -> flask : 处理后的结果
flask -> frontend : JSON响应\n(图像+分割结果)

' 实现关系
ai_task <|-- gen_task
ai_task <|-- seg_task
task_observer <|-- seg_trigger

' 单例标记
task_factory : Singleton Instance

' 观察者关联
notifier o-- task_observer

' 外观模式关联
facade --> task_factory
facade --> notifier
facade --> gen_task
facade --> seg_task

' 颜色设置
flask    #MAIN_COLOR
frontend #MAIN_COLOR
facade   #FACADE_COLOR

task_factory #FACTORY_COLOR
ai_task      #FACTORY_COLOR

gen_task     #GENERATE_COLOR
comfyui      #GENERATE_COLOR

seg_task     #SEGMENT_COLOR
sam          #SEGMENT_COLOR

notifier     #OBSERVER_COLOR
seg_trigger  #OBSERVER_COLOR

note right of facade
 **外观模式** 封装了:
 1. 任务创建流程
 2. 观察者注册
 3. 多步骤协调
end note

note left of task_factory
 **工厂模式** 根据输入类型返回:
 - ImageGenerationTask
 - SegmentationTask
end note

note bottom of notifier
 **观察者模式**:
 1. 生成任务完成时自动触发分割
 2. 可扩展其他后续处理
end note

@enduml
```

这个架构图详细展示了：

1. **工厂模式**实现：
- AITaskFactory 根据输入类型创建不同的具体任务（生成/分割）
- 通过统一的AITask接口保证执行一致性

2. **观察者模式**实现：
- TaskNotifier 维护观察者列表
- SegmentationTrigger 在图像生成完成后自动触发分割任务
- 松耦合的事件驱动架构

3. **外观模式**实现：
- AIServiceFacade 封装了复杂的处理流程
- 为前端提供简单的generateAndSegment接口

4. 完整处理流程：
① 前端提交提示词 → ② Flask路由 → ③ 外观层协调 → ④ 工厂创建生成任务 → ⑤ 调用ComfyUI → ⑥ 通知观察者 → ⑦ 创建分割任务 → ⑧ 调用SAM模型 → ⑨ 返回最终结果

5. 模块职责：
- 生成模块：处理ComfyUI API交互和提示词验证
- 分割模块：管理SAM模型加载和图像预处理
- 工厂模块：集中化任务创建逻辑
- 观察者模块：处理跨模块事件通知