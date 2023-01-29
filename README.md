# 如何处理SNPP-VIIRS的灯光数据

1. 从官方数据接口提供的数据下载灯光数据集，一般为tiff或者geotiff格式；并下载最新的中国省市县区级行政边界1：400 shapefile文件(一定要是最新的，不然某些海岸线会发生变化）
2. 在Arcgis Pro中，添加数据 -- 打开图层，得到一个栅格数据文件（灯光数据），一个要素图层文件（城市边界）。然后对SNPP-VIIRS的栅格要素裁剪到城市图层上来，并及时保存裁剪的栅格数据。

    1） 在工具栏中搜索 -- 裁剪栅格，输入SNPP-VIIRS光源地图，依据地级市城市的边界进行裁剪（√）
    
    2） 右击已经创建完毕的图层 -- 导出栅格，选择合适的地理边界对它进行导出(当然也可以不选择这么做)


3. 调整投影坐标系与重采样。在工具栏里输入 -- “投影” 或 “投影栅格”，选中灯光栅格数据，将原先的图层投影到WGS 1984上（√）
   
   1） 如果X轴与y轴的像元不为整数，可以同时调整x与y像元的大小，例如将416改为500，方法选择“最近邻法”便于计算（当然也可以不选择这么做）

   2） Arcgis Pro使用的世界地形底图投影坐标系为WGS 1984 Web Mercator (auxiliary sphere)，EPSG:3587；而Payne Institute提供的无云DNB数据集的坐标轴投影参考为WGS 1984，EPSG:4326；中国的区县边界地图坐标轴投影参考多为WGS 1984 Asia_North_Albers_Equal_Area_Conic，EPSG:102025。在不需要使用底图做其他地理统计分析的情况下，实际上只需要保证地级市边界与灯光数据集的范围一致即可(例如WGS 1984)。
   
   3） 右击图层的“属性”选项卡，查看投影参考坐标与xy轴像素大小（√）
   
   4）如果需要对像元进行四舍五入，或者对像元值进行缩放分级，需要在工具箱 -- 重采样，划定采样的分级与方法（因为DMSP的图像为64级，如需要将SNPP-VIIRS与DMSP进行匹配，则可能需要转换标度）
   
4. 异常值处理。需要注意的是这一步仅能识别一些统计范围内可能存在的信源异常值，例如<0，过大的局部值等；而卫星型号，云层，水域反射，数据跨年可比性等问题不会在这一步得到解决。可选择的办法有：

   1） 直接对数据在区域内以像元为单位进行截断，统计每一个地理区块内部的所有像元值异常分位数，截取1% - 99%分位数内的数据（X）
     
   2） 参考GLND数据库对SNPP-VIIRS数据的处理方法，提取当前时间段"北京","上海"，"广州"范围内的像元最大值为全国的"有效最大值"。利用栅格计算器将全图的像元最大值替换为“有效最大值”（√）
   
   3） 由于Payne Institute提供的处理产品已经消除了杂散光的影响，且我们并未有与DMSP相互校正的需求，这里并未与DMSP做掩膜提取。而是直接参照GLND数据库的方法，将<0的像元值视为异常值全部替换为0。
     

5.  对栅格的pixel value由浮点型转换成整型（pixel value不是整型在栅格的拼接上会出现问题，但与要素的连接不会，这一步可以不做）：

    1） 打开Arcgis Pro的工具箱 -- “栅格计算器”，在命令行输入计算表达式：
      `Con((RoundUp("NPP_VIIRS_202011.tif")- 
      "NPP_VIIRS_202011.tif")>0.5,RoundDown("NPP_VIIRS_202011.tif"),RoundUp("NPP_VIIRS_202011.tif"))`，这是做四舍五入的操作。当然如果pixel value过小，则应当先全部放大乘于整数后再转换成整型，在空间分区统计之后再转换成原先的大小

    2） 在工具栏里输入 -- “转为整型”并搜索，导入经过四舍五入转换过后的整数浮点值，可以直接转换成整型
     
6. 在工具栏里输入 -- “以表格显示分区统计”，将行政边界文件输入“要素区域数据”，将栅格文件输入“赋值栅格”，计算得到一张以地方区县界为统计单元的表，其中会提供一系列的统计信息。`count`代表落在该区域内的像元个数，`area`代表区域面积，`sum`代表区域内的像元值加总，`mean`代表区域的像元值加总除以总面积。

# 示例处理过程 -- SNPP-VIIRS 201903
1）导入由Clorado Payne Institute下载的原始数据文件`SVDNB_npp_20190301-20190331_75N060E_vcmcfg_v10_c201904071900.avg_rade9h.tif`, 其显示东亚半区的所有图像，像元值范围从-0.82到48754.1

2）依据上述方法与中国的省市边界裁剪后得到中国全域灯光图，像元值范围从-0.57到2869.97

3）先做一次"以表格显示分区统计"以获得基本统计数据，2019年3月的北京最大亮度栅格为271.890015，上海最大亮度栅格为284.109985，广州为306.600006，那么将全地图像元值大于306.6的像元值全部替换成306.6--`Con(("SVDNB_npp_2019030120190_Clip1">306.6),306.6,"SVDNB_npp_2019030120190_Clip1")`。同时将小于0的像元值全部替换为0-- `Con(("con_raste"<0),0,"con_raste")`. 我们得到一个像元值在0-306.6的中国灯光地图。

<div align=center><image src= "https://user-images.githubusercontent.com/82168423/215340056-edd644e4-ebef-4dab-acd9-42bcb20e0170.png" /></div>

   同时我们分别展示像元值大于0.5，像元值大于1，像元值大于5的掩膜地图(以0与1表示数值范围，顺序由上到下)

<div align=center><image src="https://user-images.githubusercontent.com/82168423/215341989-34a647c4-c54c-4b9e-8b1b-2bdb9a78be33.png" /></div>
   
<div align=center><image src="https://user-images.githubusercontent.com/82168423/215342040-959e1431-f6e6-4e63-a1a2-c634ed26beb1.png" /></div>

<div align=center><image src="https://user-images.githubusercontent.com/82168423/215342147-57758ab1-4b1a-4409-85b1-5dff0fc916c1.png" /></div>

4）再次运行"以表格显示分区统计"(只计算有像元值地区),得到数据集201903_light.csv, 一般选用其中的`mean`作为指标。




# Robust Check of Generate Way
1. 由于对原始数据集不同的处理方法导致灯光pixel的标度不一，我们同时寻找一些国内外地理测绘使用的经过转换后的SNPP-VIIRS数据集，观察归一化后中国各个主要城市的DNvalue的差异情况，如果各份数据集之间的差异并不大，则说明其处理效果良好。


