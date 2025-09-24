---
title: 如何使用API调用comfyUI工作流
published: 2025-09-20
tags:
  - 教程
  - AI工具
  - 推荐
lang: zh
abbrlink:  comfyui-api-invoke
---
### 1. 前置条件

> 1. 本地安装了ComfyUI软件，并启动。
> 2. 确认端口信息，如下面代码块寻找启动入口，保证连通。
> 3. API 接口参数信息，可使用`图1.1`导出API 或 web socket 调用

:::fold[ 启动入口 ]
```py
def start_comfyui(asyncio_loop=None):
    """
    Starts the ComfyUI server using the provided asyncio event loop or creates a new one.
    Returns the event loop, server instance, and a function to start the server asynchronously.
    """
    if args.temp_directory:
        temp_dir = os.path.join(os.path.abspath(args.temp_directory), "temp")
        logging.info(f"Setting temp directory to: {temp_dir}")
        folder_paths.set_temp_directory(temp_dir)
    cleanup_temp()

    if args.windows_standalone_build:
        try:
            import new_updater
            new_updater.update_windows_updater()
        except:
            pass

    if not asyncio_loop:
        asyncio_loop = asyncio.new_event_loop()
        asyncio.set_event_loop(asyncio_loop)
    prompt_server = server.PromptServer(asyncio_loop)

    hook_breaker_ac10a0.save_functions()
    asyncio_loop.run_until_complete(nodes.init_extra_nodes(
        init_custom_nodes=(not args.disable_all_custom_nodes) or len(args.whitelist_custom_nodes) > 0,
        init_api_nodes=not args.disable_api_nodes
    ))
    hook_breaker_ac10a0.restore_functions()

    cuda_malloc_warning()
    setup_database()

    prompt_server.add_routes()
    hijack_progress(prompt_server)

    threading.Thread(target=prompt_worker, daemon=True, args=(prompt_server.prompt_queue, prompt_server,)).start()

    if args.quick_test_for_ci:
        exit(0)

    os.makedirs(folder_paths.get_temp_directory(), exist_ok=True)
    call_on_start = None
    if args.auto_launch:
        def startup_server(scheme, address, port):
            import webbrowser
            if os.name == 'nt' and address == '0.0.0.0':
                address = '127.0.0.1'
            if ':' in address:
                address = "[{}]".format(address)
            webbrowser.open(f"{scheme}://{address}:{port}")
        call_on_start = startup_server

    async def start_all():
        await prompt_server.setup()
        await run(prompt_server, address=args.listen, port=args.port, verbose=not args.dont_print_server, call_on_start=call_on_start)

    # Returning these so that other code can integrate with the ComfyUI loop and server
    return asyncio_loop, prompt_server, start_all

# RUN , 默认端口号，另查看日志具体端口号，curl 测试连通性
async def run(server_instance, address='', port=8188, verbose=True, call_on_start=None):
    addresses = []
    for addr in address.split(","):
        addresses.append((addr, port))
    await asyncio.gather(
        server_instance.start_multi_address(addresses, call_on_start, verbose), server_instance.publish_loop()
    )

```
:::

![图 1.1 ](https://img2024.cnblogs.com/blog/3426265/202509/3426265-20250924144753830-2038147523.png)

:::fold[ 导出 API 接口参数信息 ]
```json
// 导出的 API 接口参数信息
{
  "1": {
    "inputs": {
      "seed": 117705328545051,
      "steps": 30,
      "cfg": 7,
      "sampler_name": "euler_ancestral",
      "scheduler": "normal",
      "denoise": 1,
      "model": [
        "52",
        0
      ],
      "positive": [
        "3",
        0
      ],
      "negative": [
        "4",
        0
      ],
      "latent_image": [
        "5",
        0
      ]
    },
    "class_type": "KSampler",
    "_meta": {
      "title": "K采样器"
    }
  },
  "2": {
    "inputs": {
      "ckpt_name": "SD1.5\\relastic\\dreamshaper_8.safetensors"
    },
    "class_type": "CheckpointLoaderSimple",
    "_meta": {
      "title": "Checkpoint加载器（简易）"
    }
  },
  "3": {
    "inputs": {
      "text": "beautiful japanese woman in a business red dress, slender, standing in a japanese bar",
      "clip": [
        "50",
        0
      ]
    },
    "class_type": "CLIPTextEncode",
    "_meta": {
      "title": "CLIP文本编码"
    }
  },
  "4": {
    "inputs": {
      "text": "ugly, deformed, embedding:bad-artist, embedding:bad-hands-5, ",
      "clip": [
        "50",
        0
      ]
    },
    "class_type": "CLIPTextEncode",
    "_meta": {
      "title": "CLIP文本编码"
    }
  }
  ......
}
```
:::

### 2. 参考实现
[官网 Git Hub 参考脚本](https://github.com/comfyanonymous/ComfyUI/blob/master/script_examples/basic_api_example.py)

:::fold[ 参考 Python 代码 ]
```python
def run_comfyui_workflow(workflow_path, params):
    # 加载并修改工作流
    with open(workflow_path, "r", encoding="utf-8") as f:
        workflow = json.load(f)

    # 动态替换参数（示例）
    workflow["3"]["inputs"]["text"] = params["prompt_text"]

    # 提交工作流
    response = requests.post("http://localhost:8000/prompt", json={"prompt": workflow})
    prompt_id = response.json()["prompt_id"]

    # 等待执行完成
    while True:
        history_response = requests.get("http://localhost:8000/history")
        history = history_response.json()
        # 正确访问层级结构
        current_task = history.get(prompt_id, {})
        status_info = current_task.get("status", {})
        if status_info.get("completed") == True:
            break
        time.sleep(1)
    print("Workflow execution completed.")
    # 提取输出结果
    # outputs = result["outputs"]
    # images = []
    # for node_id in outputs:
    #     if "images" in outputs[node_id]:
    #         for img in outputs[node_id]["images"]:
    #             images.append(img["filename"])
    #
    # return images
```
:::

### 3. 更多接口调用
> `comfyui server.py` 提供了丰富的接口,如图`1.2` 展示了部分接口。

![图 1.2 ](https://img2024.cnblogs.com/blog/3426265/202509/3426265-20250924160115802-1424074.png)

[API 接口文档](https://opendeep.wiki/comfyanonymous/ComfyUI/rest-api)