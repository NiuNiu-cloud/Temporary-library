# 模块（一）：卫星位置计算

## 1. 简介：广播星历计算卫星位置——satellite position calculation from broadcast satellite ephemeris

## 2. 函数描述：
    1. 批量卫星计算：satposs();
    2. 单个卫星计算：satpos();   //总入口

    3. 流程控制：ephpos();       //广播
                satpos_sbas();  //广播+SBAS
                satpos_ssr();   //广播+SSR
                peph2pos();     //精密星历

    4. 选择星历：*seleph();     //GPS、北斗、伽利略、QZSS、印度IRNSS
                *selgeph();    //格洛纳斯
                *selseph();    //SBAS

    5. 核心实现：eph2pos();     //GPS、北斗、伽利略、QZSS、印度IRNSS
                geph2pos();    //格洛纳斯
                seph2pos();    //SBAS
                
    6. 卫星钟差的调度函数： ephclk();               //基于广播星历
    7. 计算卫星钟差的底层核心函数：eph2clk();

---
# ***%% 广播星历求卫星位置 %%***

##   一. 函数说明

### 1. ==satposs()==
> - [x] 批量处理星历数据总入口 
```C
void satposs(gtime_t teph, const obsd_t *obs, int n, const nav_t *nav, int ephopt, double *rs, double *dts, double *var, int *svh)
{
    gtime_t time[2*MAXOBS]={{0}};   // 存储每颗卫星的“信号发射时刻”
    double dt,pr;                   // dt=卫星钟差，pr=伪距
    int i,j;                               
    trace(3,"satposs : teph=%s n=%d ephopt=%d\n",time_str(teph,3),n,ephopt);
    
    for (i=0;i<n&&i<2*MAXOBS;i++)           //初始化每颗卫星的输出参数
    {
        for (j=0;j<6;j++) rs [j+i*6]=0.0;   // 位置+速度初始化为0
        for (j=0;j<2;j++) dts[j+i*2]=0.0;   // 时钟偏差和漂移初始化为0
        var[i]=0.0; svh[i]=0;               // 方差和健康状态初始化为0        
        //循环遍历各个频率，找到第一个非零伪距值就退出循环
        for (j=0,pr=0.0;j<NFREQ;j++) if ((pr=obs[i].P[j])!=0.0) break;
        /*NFREQ = 3，代表GPS有三个频段（L1; L2; L5）。
        *obs[i]是一个结构体，包含了第i个卫星的全部观测数据；obs[i].P[j]表示第i颗卫星在第j个频率上的伪距观测值。      */
        
        if (j>=NFREQ)       //如果没找到可用的伪距
        {
            trace(3,"no pseudorange %s sat=%2d\n",time_str(obs[i].time,3),obs[i].sat);
            continue;
        }

        // 0. 未修正的发射时刻
        time[i]=timeadd(obs[i].time,-pr/CLIGHT);    
        //obs[i].time:接收机接收到数据的时刻 || CLIGHT：光速。

        // 1. 计算卫星钟差dt
        if (!ephclk(time[i],teph,obs[i].sat,nav,&dt)) 
        {
            trace(3,"no broadcast clock %s sat=%2d\n",time_str(time[i],3),obs[i].sat);
            continue;
        }
        // 2. 得到修正后的发射时刻
        time[i]=timeadd(time[i],-dt);  

        // 3. 计算该时刻的卫星位置、速度、钟差及健康状态:satpos();
        if (!satpos(time[i],teph,obs[i].sat,ephopt,nav,rs+i*6,dts+i*2,var+i,svh+i)) 
        {
            trace(3,"no ephemeris %s sat=%2d\n",time_str(time[i],3),obs[i].sat);
            continue;
        }
        /* if no precise clock available, use broadcast clock instead */
        /* 若选择精密星历，但无精密星历钟差数据，则使用广播星历钟差 */
        if (dts[i*2]==0.0)      // 精密钟差无效
        {   // 用广播星历重新计算钟差
            if (!ephclk(time[i],teph,obs[i].sat,nav,dts+i*2)) continue;
            dts[1+i*2]=0.0;             // 广播星历通常不提供钟漂，置0
            *var=SQR(STD_BRDCCLK);      // 广播钟差的误差方差
        }
    }
    for (i=0;i<n&&i<2*MAXOBS;i++)       //循环打印每颗卫星的计算结果
    {
        trace(4,"%s sat=%2d rs=%13.3f %13.3f %13.3f dts=%12.3f var=%7.3f svh=%02X\n",
              time_str(time[i],6),obs[i].sat,rs[i*6],rs[1+i*6],rs[2+i*6],
              dts[i*2]*1E9,var[i],svh[i]);
    }
}
```

### 2. ==satpos()==
> - [x] 根据星历类型，调度底层模块。
```C
int satpos(gtime_t time, gtime_t teph, int sat, int ephopt,const nav_t *nav, double *rs, double *dts, double *var,int *svh)
{
    trace(4,"satpos  : time=%s sat=%2d ephopt=%d\n",time_str(time,3),sat,ephopt);
    
    *svh=0;         // 初始将卫星健康状态设为 0（默认正常）
    
    switch (ephopt) 
    {
        case EPHOPT_BRDC  :         //广播星历
            return ephpos(time,teph,sat,nav,-1,rs,dts,var,svh);         
        case EPHOPT_SBAS  :         //SBAS增强
            return satpos_sbas(time,teph,sat,nav,   rs,dts,var,svh);    
        case EPHOPT_SSRAPC:         //SSR校正
            return satpos_ssr (time,teph,sat,nav, 0,rs,dts,var,svh);    
        case EPHOPT_SSRCOM:         //SSR +（天线偏移改正）
            return satpos_ssr (time,teph,sat,nav, 1,rs,dts,var,svh);    
        case EPHOPT_PREC  :         //精密星历
            if (!peph2pos(time,sat,nav,1,rs,dts,var)) break; 
            else return 1;             
    }
    *svh=-1;        //所有模式均失败，设为-1。
    return 0;
}
```
### 3. ==ephpos()==

> - [x] 广播星历求位置
```C
/* satellite position and clock by broadcast ephemeris -----------------------*/
static int ephpos(gtime_t time, gtime_t teph, int sat, const nav_t *nav,int iode, double *rs, double *dts, double *var, int *svh)
{
    eph_t  *eph;        //广播星历数据结构体（GPS、北斗、、、）
    geph_t *geph;   
    seph_t *seph;
    double rst[3],dtst[1],tt=1E-3;
    int i,sys;
    
    trace(4,"ephpos  : time=%s sat=%2d iode=%d\n",time_str(time,3),sat,iode);
    
    sys=satsys(sat,NULL);   //确定卫星系统
    *svh=-1;                //初始化卫星状态-1（未知）
    
    if (sys==SYS_GPS||sys==SYS_GAL||sys==SYS_QZS||sys==SYS_CMP||sys==SYS_IRN) 
    {
        if (!(eph=seleph(teph,sat,iode,nav))) return 0;     // 1. 调用seleph从导航数据中筛选“有效期包含teph时刻”且“卫星编号匹配”的星历
        eph2pos(time,eph,rs,dts,var);                       // 2. 调用eph2pos计算目标时间time的卫星位置（rs）和钟差（dts）
        time=timeadd(time,tt);                              // 3. 构造“目标时间+1毫秒”的新时刻
        eph2pos(time,eph,rst,dtst,var);                     // 4. 计算新时刻"time"的卫星位置（rst）和钟差(dtst)
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
    
    /* satellite velocity and clock drift by differential approx */
    for (i=0;i<3;i++) rs[i+3]=(rst[i]-rs[i])/tt;        //计算卫星x,y,z轴的速度：(1毫秒后的位置 - 原位置) / 时间间隔tt
    dts[1]=(dtst[0]-dts[0])/tt;                         //计算钟速：(1毫秒后的钟差 - 原钟差) / 时间间隔tt
    
    return 1;
}
```

### 4. ==eph2pos()==

> - [x] 最底层函数，核心代码，针对GPS、Galileo、QZSS、BD、IRNSS卫星系统。
```C
void eph2pos(gtime_t time, const eph_t *eph, double *rs, double *dts,double *var)
{
    double tk,M,E,Ek,sinE,cosE,u,r,i,O,sin2u,cos2u,x,y,sinO,cosO,cosi,mu,omge;
    double xg,yg,zg,sino,coso;
    int n,sys,prn;
    
    trace(4,"eph2pos : time=%s sat=%2d\n",time_str(time,3),eph->sat);
    
    if (eph->A<=0.0) 
    {
        rs[0]=rs[1]=rs[2]=*dts=*var=0.0;
        return;
    }
    tk=timediff(time,eph->toe);
    
    switch ((sys=satsys(eph->sat,&prn))) 
    {
        case SYS_GAL: mu=MU_GAL; omge=OMGE_GAL; break;
        case SYS_CMP: mu=MU_CMP; omge=OMGE_CMP; break;
        default:      mu=MU_GPS; omge=OMGE;     break;
    }
    M=eph->M0+(sqrt(mu/(eph->A*eph->A*eph->A))+eph->deln)*tk;
    
    for (n=0,E=M,Ek=0.0;fabs(E-Ek)>RTOL_KEPLER&&n<MAX_ITER_KEPLER;n++) {
        Ek=E; E-=(E-eph->e*sin(E)-M)/(1.0-eph->e*cos(E));
    }
    if (n>=MAX_ITER_KEPLER) {
        trace(2,"eph2pos: kepler iteration overflow sat=%2d\n",eph->sat);
        return;
    }
    sinE=sin(E); cosE=cos(E);
    
    trace(4,"kepler: sat=%2d e=%8.5f n=%2d del=%10.3e\n",eph->sat,eph->e,n,E-Ek);
    
    u=atan2(sqrt(1.0-eph->e*eph->e)*sinE,cosE-eph->e)+eph->omg;
    r=eph->A*(1.0-eph->e*cosE);
    i=eph->i0+eph->idot*tk;
    sin2u=sin(2.0*u); cos2u=cos(2.0*u);
    u+=eph->cus*sin2u+eph->cuc*cos2u;
    r+=eph->crs*sin2u+eph->crc*cos2u;
    i+=eph->cis*sin2u+eph->cic*cos2u;
    x=r*cos(u); y=r*sin(u); cosi=cos(i);
    
    /* beidou geo satellite */
    if (sys==SYS_CMP&&(prn<=5||prn>=59)) { /* ref [9] table 4-1 */
        O=eph->OMG0+eph->OMGd*tk-omge*eph->toes;
        sinO=sin(O); cosO=cos(O);
        xg=x*cosO-y*cosi*sinO;
        yg=x*sinO+y*cosi*cosO;
        zg=y*sin(i);
        sino=sin(omge*tk); coso=cos(omge*tk);
        rs[0]= xg*coso+yg*sino*COS_5+zg*sino*SIN_5;
        rs[1]=-xg*sino+yg*coso*COS_5+zg*coso*SIN_5;
        rs[2]=-yg*SIN_5+zg*COS_5;
    }
    else {
        O=eph->OMG0+(eph->OMGd-omge)*tk-omge*eph->toes;
        sinO=sin(O); cosO=cos(O);
        rs[0]=x*cosO-y*cosi*sinO;
        rs[1]=x*sinO+y*cosi*cosO;
        rs[2]=y*sin(i);
    }
    tk=timediff(time,eph->toc);
    *dts=eph->f0+eph->f1*tk+eph->f2*tk*tk;
    
    /* relativity correction */
    *dts-=2.0*sqrt(mu*eph->A)*eph->e*sinE/SQR(CLIGHT);
    
    /* position and clock error variance */
    *var=var_uraeph(sys,eph->sva);
}
```

### 5. ==geph2pos()==
> - [x] 格洛纳斯

### 6. ==seph2pos()==
> - [x] SBAS