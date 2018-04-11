## “应用层参数存取模块”和“系统状态监控模块”在各模块中的布置

#### 1. 说明

根据之前的方案设计，“应用层参数存取模块”仅提供参数存取接口以及参数总体划分规则，真正的参数存取过程由各模块自己执行；同样，“系统状态监控模块”需要向各个模块获取获取状态相关的信息，因此各模块需要实现相关的接口，具体方案详见下文；

#### 2.“应用层参数存取模块”及其在各模块中的布置

##### 2.1 参数划分

![](.\para.png)

如上图所示，参数根据模块号和num号标识，module定义具体实现请看下文，num编号从0-0xFFFFFFFF，由各模块自行决定，参数长度未限制（esp32环境下底层限制为最大为1984Bytes）;

##### 2.2应用层参数存取服务模块

```c
typedef enum
{
    APS_PARA_MODULE_GENERAL=1, // MODULE 0  FOR LOG
    APS_PARA_MODULE_DAEMON,
	APS_PARA_MODULE_LAMP,
	APS_PARA_MODULE_APS_LAMP,
	APS_PARA_MODULE_APS_METER,
	APS_PARA_MODULE_APS_RTC,
	APS_PARA_MODULE_ASP_PTCHUAWEI,
	APS_PARA_MODULE_APS_PTCOPPLE,
}t_aps_para_module;

extern void AppParaInit(void);
extern int AppParaRead(int module,int num,unsigned char* data,unsigned int* lenMax);
extern int AppParaWrite(int module,int num,unsigned char* data,unsigned int len);
extern int AppParaReadU32(int module,int num,unsigned int* data);
extern int AppParaWriteU32(int module,int num,unsigned int data);

```

由上文可知，一个模块需要使用参数存取服务，必须从t_aps_para_module中分配一个module号；

##### 2.3各模块中的布置

一个包含参数存取功能的模块，需要和外界交互，实现参数修改，读出功能，因此需要自身实现以下接口（具体代码可以参照系统状态监控模块）：

```c
int paraSet(para,len); // 根据不同模块不同功能，可能对每个num单独提供接口
int paraGet(para,len);
```

且其内部实现模式一般为：

```c
static U8 para[6];  // 定义参数本地存储结构
const U8 paraDefault[6]; // 默认参数

/*从flash加载参数*/
int loadPara(void)
{
    return AppParaRead(module,num,para,6);
}

/*加载默认参数*/
void loadParaDefault(void)
{
    memcpy(para,paraDefault,6);
}

/*参数设置*/
int paraSet(U8* data)
{
    if(AppParaWrite(module,num,para,6) ==0) // 写入Flash
    {
        memcpy(para,data,6); // 写入Flash成功后，拷贝到ram中
        return 0;
    }else{
        return 1;
    }
}

/*参数获取*/
int paraGet(int* data,int lenMax)
{
    if(lenMax < 6) return 1;
    memcpy(data,para,6); // 直接返回ram中的数据，不直接执行flash操作,节省时间
    return 0;
}

void moduleInit(void)
{
    if(loadPara()!=0){ // 加载flash中的参数
        loadParaDefault(); // 加载失败则加载默认参数
    }
}
```



#### 3.“系统状态监控模块”及其在各模块中的布置

##### 3.1系统状态

![](.\status.png)

如图所示，当输入模块名和状态码后，模块将提供详细的信息，得到该信息后是否上报、报警、生成日志可以配置，模块名由本模块分配，code内容由各模块自行定义；

##### 3.2系统状态监控模块

（1）配置表

模块提供一个配置表，即每个模块的状态码产生的状态信息是否生成上报、报警、日志可配置；（这部分内容其他模块只需要了解即可，不直接交互）

```c
typedef struct
{
    unsigned int  enable;
    t_mc_module   module; // 
    unsigned int  action; // action | t_mc_action
    unsigned int  code;
}t_module_status_code;

typedef enum
{
    MC_ACTION_LOG=0x01,
    MC_ACTION_REPORT=0x02,
    MC_ACTION_ALARM=0x04,
}t_mc_action;

static t_module_status_code code[MODULE_STATUS_ITEM_MAX]; // 配置表

extern int ApsDaemonActionItemParaSet(unsigned int index,t_module_status_code* code);
extern int ApsDaemonActionItemParaGet(unsigned int index,t_module_status_code* code);
```

（2）注册表

本模块需要其他模块提供获取状态码和状态码生成日志的接口：

```c
#define MODULE_INFO_LIST {\
	{.module=MC_MODULE_GENERAL,.name="",.fun=(void*)0,.cfun=(void*)0},\
	{.module=MC_MODULE_NB,     .name="",.fun=(void*)0,.cfun=(void*)0},\
	{.module=MC_MODULE_WIFI,   .name="",.fun=(void*)0,.cfun=(void*)0},\
	{.module=MC_MODULE_PM,     .name="",.fun=(void*)0,.cfun=(void*)0},\
	{.module=MC_MODULE_LAMP,   .name="",.fun=(void*)0,.cfun=(void*)0},\
	{.module=MC_MODULE_PARA,   .name="",.fun=(void*)0,.cfun=(void*)0},\
}; // 注册表

typedef enum
{
    MC_MODULE_GENERAL=0,
    MC_MODULE_NB,
    MC_MODULE_WIFI,
    MC_MODULE_PM, 
    MC_MODULE_LAMP,
    MC_MODULE_PARA,
/********************************/
    MC_MODULE_MAX,
}t_mc_module;

typedef struct
{
	unsigned char status; // 0:alarm disappear,1:alarm appear
	unsigned char level;  // 
	unsigned char time[6]; // Y-M-D H-M-S(Year=Y+2000)
    unsigned char len;
    unsigned int desp[MODULE_STATUS_DESP_LEN_MAX];
}t_module_status_result;

typedef int (*pfGetModuleStatus)(unsigned int code,t_module_status_result*);
typedef void (*pfGetCodeLog)(unsigned code,t_module_status_result*,char* log,int lenMax);

static t_mc_module_info moduleInfo[MC_MODULE_MAX]=\
MODULE_INFO_LIST;
```

各子模块将接口实现后，注册到以上注册表即可；



##### 3.3各模块中的布置

（1）各模块在t_mc_module中注册一个模块名

（2）各模块自定义状态码code内容，并将code对应内容获取接口（参照t_module_status_result）注册在本模块

（3）各模块将code生成显示log的接口注册在本模块，将生成以下spec部分内容

```c
// [Date time][Module,level]code_spec
// [2018-4-8 16:29:38][module,warn/error...]spec
```

（4）t_module_status_result中desp[]内容需要有一定的固定格式，详细格式参照华为协议处理模块中各上报消息接口；

