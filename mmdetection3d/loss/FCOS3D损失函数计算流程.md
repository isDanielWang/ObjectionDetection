FCOS3D损失函数计算流程

- 真值：

  - 训练/验证/测试数据的ann_file分别加载数据集中的
    - nuscenes_infos_train_mono3d.coco.json
    - nuscenes_infos_val_mono3d.coco.json
    - nuscenes_infos_test_mono3d.coco.json(这块mmdetection3d源码中测试集来源仍然是nuscenes_infos_val_mono3d.coco.json)
  - 从以上文件中读取制定训练数据：
    - img（验证与测试时仅需读此key）
    - gt_bboxes
    - gt_labels
    - attr_labels
    - gt_bboxes_3d
    - gt_labels_3d
    - centers2d
    - depth
  - 数据路径在configs/_base_/datasets/nus-mono3d.py中配置

- 预测值：

  - 在对应模型的dense_head中生成

  - fcos_mono3d_head继承anchor_free_mono3d_head

  - fcos3d中有5个输出特征图对应5个输入特征图，每个输出特征图有一个检测头，每个检测头有两个分支：分类分枝和回归分枝，每个分枝包括4个卷积层，且参数不共享。但是5个输出层的检测头的模块权重共享。

  - 初始化分类分枝中的卷积

    - <img src="/Users/shixiangwang/Library/Application Support/typora-user-images/image-20220613164640934.png" alt="image-20220613164640934" style="zoom:50%;" />

  - 初始化回归分枝中的卷积

    - <img src="/Users/shixiangwang/Library/Application Support/typora-user-images/image-20220613164916071.png" alt="image-20220613164916071" style="zoom:50%;" />

  - 初始化分类分枝和回归分枝的预测模块

    - <img src="/Users/shixiangwang/Library/Application Support/typora-user-images/image-20220613165229387.png" alt="image-20220613165229387" style="zoom:50%;" />
    - group_reg_dims的长度代表回归分枝中不同预测功能模块的个数，对应位置的值代表该功能模块需要回归的值的个数，该list的求和值应该要和bbox_coder中的code_size值保持一致
    - nchor_free_mono3d_head中的回归分枝的预测模块有5个，分别是offset, depth, size,  rot, velocity, 总共预测sum(2,1,3,1,2) = 9个值，实际运行中不预测velo上的两个值

  - 分层调用forward_single函数

    - 计算分类分枝特征和分类值
      - <img src="/Users/shixiangwang/Library/Application Support/typora-user-images/image-20220613165849795.png" alt="image-20220613165849795" style="zoom:50%;" />
      - 计算回归分枝特征和对应9个bbox属性的预测值
        - <img src="/Users/shixiangwang/Library/Application Support/typora-user-images/image-20220613170059556.png" alt="image-20220613170059556" style="zoom:50%;" />
      - 在回归分枝中预测方向类别和在分类分枝中预测属性值
        - <img src="/Users/shixiangwang/Library/Application Support/typora-user-images/image-20220613170204854.png" alt="image-20220613170204854" style="zoom:50%;" />

  - fcos_mono3d_head子类中额外实现在回归分枝的特征图上预测中心度

    - <img src="/Users/shixiangwang/Library/Application Support/typora-user-images/image-20220613170427459.png" alt="image-20220613170427459" style="zoom:50%;" />

  - 调用loss函数利用真是值和预测值计算损失

    - 损失函数的配置在backbbone中

      - <img src="/Users/shixiangwang/Library/Application Support/typora-user-images/image-20220613170954942.png" alt="image-20220613170954942" style="zoom:50%;" />

    - 计算分类损失

      - <img src="/Users/shixiangwang/Library/Application Support/typora-user-images/image-20220613171057308.png" alt="image-20220613171057308" style="zoom:50%;" />

    - 分别计算bbox上5个模块的损失，前两个为偏移，第三个为深度，四五六为大小，七位角度的sin值，八九为两个方向的速度当然速度默认是不预测的，当然在这之前会挑选出正样本

      - 挑选正样本，标签在0-9上的都是正样本，否则是负样本

        - <img src="/Users/shixiangwang/Library/Application Support/typora-user-images/image-20220613173328139.png" alt="image-20220613173328139" style="zoom:50%;" />

      - 计算损失

        - <img src="/Users/shixiangwang/Library/Application Support/typora-user-images/image-20220613171522236.png" alt="image-20220613171522236" style="zoom:50%;" />

      - 计算中心度损失

        - <img src="/Users/shixiangwang/Library/Application Support/typora-user-images/image-20220613171551182.png" alt="image-20220613171551182" style="zoom:50%;" />

      - 计算方向分类损失

        - <img src="/Users/shixiangwang/Library/Application Support/typora-user-images/image-20220613171651033.png" alt="image-20220613171651033" style="zoom:50%;" />

      - 计算属性值损失

        - <img src="/Users/shixiangwang/Library/Application Support/typora-user-images/image-20220613171718038.png" alt="image-20220613171718038" style="zoom:50%;" />

      - 返回损失字典

        - <img src="/Users/shixiangwang/Library/Application Support/typora-user-images/image-20220613172525943.png" alt="image-20220613172525943" style="zoom:50%;" />

        