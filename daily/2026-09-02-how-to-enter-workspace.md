1.打开终端，进入wsl终端

2.进入课程代码目录，以便随时运行我们的课程代码

bash
cd ~/ai-engineering-from-scratch

3.激活虚拟的python环境
source ~/.venv/bin/activate

执行后，终端提示符显示（.venv）说明成功进入虚拟环境

4.（不一定，但推荐）验证我们的环境
bash
python -c"import torch;print(torch.cuda.is.available())"

如果输出true,说明GPU正常，一切就绪
