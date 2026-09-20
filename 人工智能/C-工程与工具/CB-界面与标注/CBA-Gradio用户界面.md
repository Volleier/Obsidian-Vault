# 概念
[Gradio](https://www.gradio.app/) 是一个开源的 Python 库，用于快速构建机器学习和数据科学演示应用。它使得开发者可以在几行代码中创建一个简单、可调整的用户界面，用于展示机器学习模型或数据科学工作流程。Gradio 支持多种输入输出组件，如文本、图片、视频、音频等，并且可以轻松地分享应用，包括在互联网上分享和在局域网内分享。
# 使用
在gradio中，我们可以把每个组件创建在 gr.Blocks() 包裹的块当中，你可以把它当作一个展示台，我们可以在展示台上放满不同的组件（比如这里的 gr.Button, gr.Textbox 等等），你可以自由组合组件来实现想要的目标。此外，在gradio中同样也有各类可注册事件，这有助于我们在和组件互动时、互动后来指定我们想要发生的事情（比如执行一段函数），比如我们希望按钮按下后执行一个函数，然后根据某些组件的输入输出做反应，我们就可以像下面的 .click 以及 .submit 一样注册相应的事件，根据绑定的函数创作出更多样的可能。
最后，我们只需要 launch 我们的”展示台“，就可以看到gradio前端展示画面。
``` python
# 使用Gradio创建web界面
with gr.Blocks() as demo:
    gr.Markdown("# 聊天机器人")
    chatbot = gr.Chatbot()  # 聊天界面组件
    msg = gr.Textbox()  # 聊天界面的输入
    clear = gr.Button("清除")  # 清除按钮
    stop = gr.Button("停止")  # 停止生成按钮
  
    # 设置用户输入提交后的处理流程
    msg.submit(user, [msg, chatbot], [msg, chatbot], queue=False).then(
        bot, chatbot, chatbot
    )
    clear.click(lambda: None, None, chatbot, queue=False)  # 清除按钮功能
    stop.click(stop_generation, queue=False)  # 停止按钮功能
  
if __name__ == "__main__":
    print("启动Gradio服务")
    demo.queue()  # 启动队列处理请求
    demo.launch()  # 启动服务
```