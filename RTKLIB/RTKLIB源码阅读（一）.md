[TOC]

# <h1 style="text-align: center;">源码阅读（一）</h1>
> ***程序介绍、卫星位置获取***
> [参考文献：李同学的github](https://github.com/LiZhengXiao99/Navigation-Learning)


## 一 . RTKLIB下载
- 有两个版本，一个是[ "界面版" ](https://www.rtklib.com/)，一个是[ "命令行版" ](https://www.rtklib.com/)。


## 二 . 工程文件介绍
![RTKLIB](https://pic-bed-1316053657.cos.ap-nanjing.myqcloud.com/img/RTKLIB.png)

> [来源：安徽理工的李同学] 


## 三 . 程序运行流程

![image-20231016123911526](https://pic-bed-1316053657.cos.ap-nanjing.myqcloud.com/img/image-20231016123911526.png)

> ##后处理解算流程（ rtk2rtkp.c ）
> ##实时处理（ rtkrcv.c ）


## 四 . 求卫星位置

> satpos()函数是卫星位置计算的 “路由器”，通过抽象不同星历算法的差异，提供了统一的调用接口。

### **1 . satpos()**

```C
int satpos(gtime_t time, gtime_t teph, int sat, int ephopt,
                  const nav_t *nav, double *rs, double *dts, double *var,
                  int *svh)
{
    trace(4,"satpos : time=%s sat=%2d ephopt=%d\n",time_str(time,3),sat,ephopt);
    // 调试信息输出函数，用于在调试级别为4时打印函数调用信息（时间、卫星号、星历选项）
    *svh=0;             // 初始化卫星健康状态(默认卫星状态正常)
    switch (ephopt)     // 选择不同的星历计算方式
    {
        case EPHOPT_BRDC:       // 广播星历
            return ephpos(time,teph,sat,nav,-1,rs,dts,var,svh);
        case EPHOPT_SBAS:       // SBAS星历
            return satpos_sbas(time,teph,sat,nav,rs,dts,var,svh);
        case EPHOPT_SSRAPC:     // SSR增强（不含天线偏移改正）
            return satpos_ssr(time,teph,sat,nav, 0,rs,dts,var,svh);
        case EPHOPT_SSRCOM:     // SSR增强（含天线偏移改正）
            return satpos_ssr (time,teph,sat,nav, 1,rs,dts,var,svh);
        case EPHOPT_PREC:       // 精密星历
            if (!peph2pos(time,sat,nav,1,rs,dts,var)) 
                break; 
            else 
                return 1;
    }
    *svh=-1;        // 卫星状态异常
    return 0;       // 计算失败
}
```
#### 1 . 函数参数说明
- gtime_t time：观测时间（接收端接收信号的时间）
- gtime_t teph：星历参考时间
- int sat：卫星编号（如 GPS 卫星编号）
- int ephopt：星历选项（决定使用哪种类型的星历计算）
- const nav_t *nav：导航数据结构体指针（存储星历、历书等信息）
- double *rs：输出参数，卫星位置（ECEF 坐标系，单位：米）
- double *dts：输出参数，卫星钟差（包括相对论效应，单位：秒）
- double *var：输出参数，位置计算的方差（用于精度评估）
- int *svh：输出参数，卫星健康状态（0 表示健康，-1 表示异常）

---

### **2 . ephpos()**
> - ephpos()函数根据卫星所属系统，选择对应的星历数据和解析算法，计算卫星在指定时刻的位置、速度、钟差及相关参数。
```C
/* 通过广播星历获取的卫星位置和时钟（信息）-----------------------*/
static int ephpos(gtime_t time, gtime_t teph, int sat, const nav_t *nav,int iode, double *rs, double *dts, double *var, int *svh)       
{
    eph_t  *eph;
    geph_t *geph;
    seph_t *seph;
    double rst[3],dtst[1],tt=1E-3;
    int i,sys;
    
    trace(4,"ephpos  : time=%s sat=%2d iode=%d\n",time_str(time,3),sat,iode);
    // 调试日志：打印计算时刻、卫星号、星历标识（IOD）

    sys=satsys(sat,NULL);   // 确定卫星所属系统（如GPS、GLONASS等）
    
    *svh=-1;                // 初始化卫星健康状态为“异常”（后续成功计算后更新
    
    if (sys==SYS_GPS||sys==SYS_GAL||sys==SYS_QZS||sys==SYS_CMP||sys==SYS_IRN)   
    // GPS、 伽利略、 日本QZSS、 "北斗"、 印度IRNSS 
    {
        if(!(eph=seleph(teph,sat,iode,nav))) return 0;
        eph2pos(time,eph,rs,dts,var);
        time=timeadd(time,tt);
        eph2pos(time,eph,rst,dtst,var);
        *svh=eph->svh;
    }
    else if (sys==SYS_GLO) 
    {
        if (!(geph=selgeph(teph,sat,iode,nav))) return 0;
        geph2pos(time,geph,rs,dts,var);
        time=timeadd(time,tt);
        geph2pos(time,geph,rst,dtst,var);
        *svh=geph->svh;
    }
    else if (sys==SYS_SBS) 
    {
        if (!(seph=selseph(teph,sat,nav))) return 0;
        seph2pos(time,seph,rs,dts,var);
        time=timeadd(time,tt);
        seph2pos(time,seph,rst,dtst,var);
        *svh=seph->svh;
    }
    else return 0;
    /* 通过微分近似计算卫星速度和时钟漂移 */
    for(i=0;i<3;i++)            //计算卫星在三个轴上的速度分量,存入rs[3,4,5],rs[1,2,3]是前一时刻的卫星位置（x,y,x）,rst是第二时刻的
    {
        rs[i+3]=(rst[i]-rs[i])/tt;
    }
    dts[1]=(dtst[0]-dts[0])/tt;
    return 1;
}
```
### **3 . seleph()**
> static eph_t *seleph(gtime_t time, int sat, int iode, const nav_t *nav)
- 根据卫星编号去星历数据里找到这个卫星的星历！！！
- 找到之后返回一个指向这个星历数据的指针。
---

### **4 . eph2pos()**
> 1.函数定义：void eph2pos(gtime_t time, const eph_t *eph, double *rs, double *dts,double *var)
> 
> 2.函数功能：将广播星历数据转换为卫星在指定时刻的位置、时钟偏差及误差方差（GPS、北斗、伽利略、QZSS、IRNSS）

- 参数说明：
    - time：输入参数，观测时间（GPS 时）
    - eph：输入参数，指向广播星历数据（eph_t结构体）的指针
    - rs：输出参数，卫星在 ECEF 坐标系（地心地固坐标系）中的位置（x,y,z，单位米）
    - dts：输出参数，卫星时钟偏差（单位秒）
    - var：输出参数，卫星位置和时钟的方差（单位平方米，用于精度评估）
  ---
  
### **5 . geph2pos()**
> 专用于格洛纳斯卫星的星历转位置函数。

### **6 . seph2pos()**
> 专用于SBAS卫星的星历转位置函数。

### **7 . satposs()**
> 批量处理多个卫星的位置、速度和时钟参数

- teph:	筛选星历的参考时间（GPS时）
- obs	: 观测数据结构体数组（存储每个卫星的观测值，如伪距）
- n	  : 观测数据的数量（即需要处理的卫星数量）
- nav : 导航数据结构体（存储所有卫星的星历）
- ephopt: 星历选择选项（如优先用广播星历还是精密星历）
- rs  :	卫星位置+速度数组（ECEF坐标系），按卫星分组：每组6个值，前3个是x/y/z位置，后3个是vx/vy/vz速度
- dts	: 卫星时钟数组，按卫星分组：每组2个值，分别是时钟偏差（s）和时钟漂移率（s/s）
- var	: 每个卫星的位置+时钟误差方差（评估精度）
- svh	: 每个卫星的健康状态标志（0=正常，其他值=异常，-1=无数据）



