运行online_demo时遇到的问题：
1、unsupported export of dynamic inputs, dynamic slice is a deprecated experimental op, please statically allocate or change to a higher version
	a) 修改torch.onnx.export函数：
	torch.onnx.export(torch_module, torch_inputs, buffer, input_names=input_names, output_names=["o" + str(i) for i in range(len(torch_inputs))], opset_version=10)
	
	b) onnx-simplifier
	pip3 install onnx-simplifier		 # 中途遇到无法安装runtime的问题，更新pip3后，再安装runtime就成功了。
	在代码中使用：
  from onnxsim import simplify		 # 似乎需要将import onnx放在import tvm前面
  onnx_model, _ = simplify(onnx_model)
  relay_module, params = tvm.relay.frontend.from_onnx(onnx_model, shape=input_shapes)


2、通过上面的修改后，成功调出视频窗口，但是处理的速度仅有0.7帧/s。
第一次运行main.py函数时，需要将pytorch模型转变为onnx，再输入tvm，之后将参数保存下来，得到.tar、.parmas、.json文件（json文件作者提供了），第二次调用时可直接读取这些文件。
我遇到的问题是：
	第一次运行main.py时，就遇到问题了，但是同样可以生成.tar、.parmas、.json文件，此时会报错误，但是视频窗口可以正常调出，只不过是0.7帧/s，而第二次调用时则直接调出了视频窗口，
同样是0.7帧/s，但是第一次调用时出现的错误信息却不再显示，这里很有迷惑性，一度让我认为是自己设备的问题。后期通过time.time()函数发现.asnumpy()花了1.42s，就在想办法替换该函数，
走向了一个完全错误的方向，因为实际上我的错误时是onnx → TVM这个部分根本没有成功，也是在因为偶然的一次操作发现第一次调用main.py会爆出错误信息，这时才开始产生怀疑的。
  通过在Jetson nano上用VScode调试后发现，进不去下面的代码：
		with tvm.relay.build_config(opt_level=3):
        graph, tvm_module, params = tvm.relay.build(relay_module, target, params=params)
    return graph, tvm_module, params
  这时才认为是TVM没有成功，产生的错误信息有很多，前面的都无关紧要，后面选择了：
	WARNING:root:Failed to download tophub package for cuda: <urlopen error [Errno 111] Connection refused>
  到网上查了一下，发现是下载一个与TVM相关的文件没有成功，藏在报错信息中间很不起眼，但却是问题的关键。因为该文件需要访问外网，而我的Jetson nano是通过网卡连接家里的路由器，因此注定
下载失败，解决的教程如下：
  a) 首先进入~/.tvm文件夹，会发现目录下有一个tophub文件夹，该文件夹为空。
	b) 通过git clone https://github.com/uwsampl/tvm-distro克隆文件
	c) mv 克隆后的tophub文件夹到~/.tvm/，替代原有的tophub文件夹。
	d) 到online_demo目录下执行python3 main.py，发现视频处理速度变为27~31帧/s
	
