# AUTONOMOUS-DRIVING-2D-OBJECT-DETECTION

## Overview
* 2D Detection
Given a set of camera images, produce a set of 2D boxes for the objects in the scene.
* 將訓練一個模型，讓他可以對waymo Challenges所要求的三種物件（vehicle,pedestrian,cyclist）進行辨識,並且用方框選取。
* 在這個專案中，嘗試使用**Yolov4**對waymo資料集進行預測。
## DataSet
* 使用的資料集是"Waymo Open Dataset"。
* 該資料集已經經過處理，分成訓練集、驗證集、測試集。訓練集總容量將近2TB，如此大量的資料對一般人來說太大了，並且也沒有足夠的設備去訓練。因此我下載25GB訓練集、3GB驗證集、3GB測試集(8:1:1)。
* waymo資料集的格式是tfrecord，為了方便使用將資料轉換成coco資料格式：
    1. 完整轉換：訓練集:23805 items, 驗證集:3965 items, 測試集:3965 items
    2. 精簡轉換：訓練集:2400 items, 驗證集:400 items, 測試集:400 items
* 訓練集預覽
![](https://i.imgur.com/LIcosQE.png)
* bounding box座標位置
![](https://i.imgur.com/EkIQ0u2.png)
* 訓練集、驗證集，資料分布
![](https://i.imgur.com/5KRUPZy.jpg)
* 訓練集類別標籤數量
    | vehicle | pedestrian | cyclist |
    | -------- | -------- | -------- |
    | 17063     | 4965     |  54     |
* 驗證集類別標籤數量
    | vehicle | pedestrian | cyclist |
    | -------- | -------- | -------- |
    | 4511     | 2037     |  79     |
## Yolov4
* Yolov4的架構如下：
![](https://i.imgur.com/BKB83Od.png)

## Yolov4 訓練
1. 使用[ Darknet ](https://github.com/alexeyab/darknet)來訓練yolo
2. 建立專案資料夾並建立參數資料夾（cfg）、權重資料夾（weight） 
&rArr; 從darknet資料夾複製face.data, face.names放到cgf資料夾下
&rArr; 更改.data檔, .names檔內容
```
❗ .names檔案寫入要辨識的類別label, .data檔案寫入class類別數量, 
    train 的 bndBox txt 檔、valid 的 bndBox txt 檔、
    names 參數檔、backup (weights 檔) 的位置路徑
```
* 修改後的names檔（Waymo資料集三種類別vehicle, pedestrian, cyclist） 
    ![](https://i.imgur.com/ypKr31y.png)
* 修改後的data檔
    ![](https://i.imgur.com/YIdY5GT.png)
3. 修改 yolov4-tiny.cfg檔案
&rArr; 將 darknet/cfg/yolov4-tiny-custom.cfg 複製到參數資料夾 cfg 中，並改名為 yolov4-tiny-obj.cfg
&rArr; 更改yolov4-tiny-obj.cfg裡的 filters 跟 classes參數
```
❗ 在更改 yolov4-tiny-obj.cfg 之前，先算一下 filters 跟 classes 要更改為多少
yolov4 偵測的濾鏡(filter) 大小為 (C+5)*B
- B 是每個Feature Map可以偵測的bndBox數量，這裡設定為3
- 5 是bndBox輸出的5個預測值: x,y,w,h 以及 Confidence
- C 是類別數量
filters=(classes + 5)*3  # 因為是一個類別，所以filters更改為 24
classes=3  
``` 
``` 
# 查看參數
sed -n -e 212p -e 220p -e 263p -e 269p cfg/yolov4-tiny-obj.cfg
# 呈現如下 
filters=24
classes=1
filters=24
classes=1
``` 
4. 修改預設 anchors 值，使用以下指令 
``` 
cd ../darknet
./darknet detector calc_anchors ../Face_detection/cfg/face.data -num_of_clusters 6 -width 416 -height 416 -showpause
``` 
&rArr; Darknet 官方提供可以自動算出 anchors 值
&rArr; 將 yolov4-tiny-obj.cfg 裡第 219, 268 行的 anchors 更改為輸出的值。
因為 num_of_clusters 設為6，因此會有6組值 。
![](https://i.imgur.com/NMHCZc4.png)


5. 下載 yolov4-tiny 預訓練權重
&rArr; Darknet 官方事先訓練好的 weight (yolov4-tiny.conv.29) 
&rArr; 放入../cfg 中，就可以開始做訓練啦!
6. 訓練模型
``` 
./darknet detector train ./WM_data/cfg/WM.data ./WM_data/cfg/yolov4-tiny-obj.cfg ./WM_data/cfg/yolov4-tiny.conv.29 -dont_show
``` 
* loss圖
![](https://i.imgur.com/FHGkorJ.png)
* mAP
``` 
class_id = 0, name = vehicle, ap = 24.44%   	 (TP = 1250, FP = 1473) 
class_id = 1, name = pedestrian, ap = 15.55%   	 (TP = 331, FP = 337) 
class_id = 2, name = cyclist, ap = 10.21%   	 (TP = 1, FP = 0) 

 for conf_thresh = 0.25, precision = 0.47, recall = 0.24, F1-score = 0.32 
 for conf_thresh = 0.25, TP = 1582, FP = 1810, FN = 5045, average IoU = 34.21 % 

 IoU threshold = 50 %, used Area-Under-Curve for each unique Recall 
 mean average precision (mAP@0.50) = 0.167300, or 16.73 % 
```
* recall
```
Number Correct Total   Rps/Img        IOU                Recall 
    0    14    44	RPs/Img: 70.00	IOU: 34.61%	Recall:31.82%
    1    31   102	RPs/Img: 66.50	IOU: 33.29%	Recall:30.39%
    2    31   102	RPs/Img: 47.00	IOU: 33.29%	Recall:30.39%
    3    41   149	RPs/Img: 57.50	IOU: 31.83%	Recall:27.52%
    4    46   158	RPs/Img: 52.80	IOU: 32.63%	Recall:29.11%
    5    51   166	RPs/Img: 53.67	IOU: 33.68%	Recall:30.72%
    6    55   170	RPs/Img: 49.29	IOU: 34.73%	Recall:32.35%
    7    69   206	RPs/Img: 48.62	IOU: 35.16%	Recall:33.50%
    8    75   224	RPs/Img: 48.33	IOU: 35.63%	Recall:33.48%
    9    81   235	RPs/Img: 47.10	IOU: 36.46%	Recall:34.47%
   10    89   258	RPs/Img: 46.36	IOU: 36.56%	Recall:34.50%
```
7. 預測結果
* 圖片比較
![](https://i.imgur.com/ZGFlpKR.jpg =50%x)![](https://i.imgur.com/jjMzrgx.jpg =50%x)
![](https://i.imgur.com/NM0gtrH.jpg =50%x)![](https://i.imgur.com/rkRKhVs.jpg =50%x)
![](https://i.imgur.com/acZM2VS.jpg =50%x)![](https://i.imgur.com/dpPS2d2.jpg =50%x)
![](https://i.imgur.com/mYLJxkK.jpg =50%x)![](https://i.imgur.com/CRkNyrI.jpg =50%x)
* 影片
    * 台北
    {%youtube ReQoUIeOWW4 %}
    * 倫敦
    {%youtube EoRyj6FtVtk %}

* 可以發現在London影片中，物件辨識的成果還不錯，這是由於自行車騎士（cyclist）並沒有出現
* 在Taipel影片中，包含有很多摩托車騎士，但在識別預測後標示成車輛（Vehicle），我認為摩托車騎士應該被歸類為cyclist類別裡。這可能是因為在訓練資料中，cyclist類別的數量是極度的少，造成訓練資料分佈不均的狀況。若要提高準確度，我認為可以先做資料擴增，增加cyclist類別的數量。


## Reference
[Darknet](https://github.com/alexeyab/darknet)
[YOLOv4 訓練教學](https://medium.com/ching-i/yolo-c49f70241aa7)


<img src="./images/image (2).jpg" />
<img src="./images/image (3).jpg" />
<img src="./images/image (4).jpg" />
<img src="./images/image (5).jpg" />
<img src="./images/image (6).jpg" />
<img src="./images/image (7).jpg" />
<img src="./images/image (8).jpg" />
<img src="./images/image (9).jpg" />
<img src="./images/image (10).jpg" />
<img src="./images/image (11).jpg" />
<img src="./images/image (12).jpg" />
<img src="./images/image (13).jpg" />
<img src="./images/image (14).jpg" />
<img src="./images/image (15).jpg" />
<img src="./images/image (16).jpg" />
<img src="./images/image (17).jpg" />

* 影片
    * [台北](
    https://www.youtube.com/watch?v=59zzxkmw65Q&ab_channel=Bloss0m)
    * [倫敦](
        https://www.youtube.com/watch?v=LLzuFMYALpk&ab_channel=Bloss0m)

<img src="./images/image (18).jpg" />