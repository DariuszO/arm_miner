# om-miner01-sm-miner-master main入口及初始化

## 编译

```
//安装gcc-arm-linux-gnueabihf/g++-arm-linux-gnueabihf
apt-get install -y gcc-arm-linux-gnueabihf
apt-get install -y g++-arm-linux-gnueabihf

//编译sm-miner-master
unzip om-miner.zip
cd om-miner/
cd sm-miner-master/
make -j8

//sm-miner-master运行
./sm-miner-master -psu_v 48 -osc 50 -log_chip_stat
//需在ARM平台下运行

//安装arm-none-eabi-g++
add-apt-repository ppa:team-gcc-arm-embedded/ppa
apt-get update
apt-get install -y gcc-arm-embedded
arm-none-eabi-gcc -v

//编译sm-miner-slave
cd ../sm-miner-slave/
make -j8

//sm-miner-slave运行
./exe/sm-miner-slave.elf
//需在ARM平台下运行
```

## sm-miner-master main入口

```c++
//解析命令行
Application::parseCommandLine(argc, argv);
if (Application::config()->showUsage)
{
	//打印帮助信息
	CommandLineParser::showUsage();
	throw ExitException();
}
//暂时什么也没做
Application::configPostProcessing();
//程序初始化
Application::init();
//程序运行
Application::run();
//代码位置sm-miner-master/src/main.cpp
```

## 程序初始化

```c++
//判断是否已完成初始化
if (m_initialized)
	throw ApplicationException("Application already initialized.");
//获取配置信息
const Config config = *this->config();
//配置日志
//config.traceAll即命令行中-trace、config.traceStratum即命令行中-trace_stratum
configureLogging(config);
LOG_TRACE(logger) << "Initializing application...\n";
//文件加锁保证程序唯一运行
SingleInstanceApp::lock("/var/run/sm-miner.pid");
//MasterStat初始化
g_masterStat.init();
//CpuInfo初始化
LOG_INFO(logger) << "Init gpio\n";
g_cpuInfo.init();
//GpioManager初始化
g_gpioManager.init();
//EnvManager初始化（未做任何处理）
LOG_INFO(logger) << "Init env\n";
g_envManager.init();
//SlaveGate初始化（未做任何处理）
LOG_INFO(logger) << "Init slave MCU\n";
g_slaveGate.init();
//AppRegistry初始化
if (m_appRegistry.selectComponents())
{
	while (AppComponent* appComponent = m_appRegistry.nextComponent())
		appComponent->init(config);
}
m_initialized = true;
LOG_TRACE(logger) << "Application initialized.\n";
//程序初始化完成
//代码位置sm-miner-master/src/app/ApplicationImpl.cpp
```

## MasterStat初始化

```c++
//MAX_SLAVE_COUNT为4
for (int slaveId = 0; slaveId < MAX_SLAVE_COUNT; slaveId++)
	//SlaveStat初始化
	slaves[slaveId].init(slaveId);
//dummySlave初始化
dummySlave.init(0);
//代码位置sm-miner-master/src/stats/MasterStat.cpp
```

SlaveStat初始化

```c++
slaveId = _slaveId;
hasData = false;
//msSpiTx、msSpiRxOk、msSpiRxError类型均为StatCounter32
//msSpiTx清零
msSpiTx.clear();
//msSpiRxOk清零
msSpiRxOk.clear();
//msSpiRxError清零
msSpiRxError.clear();
//noncesContainer清零，类型为typedef NonceContainerT<40> NonceContainer;
noncesContainer.clear();
psuNum = 0;
memset(psuInfo, 0, sizeof(psuInfo));
memset(psuSpec, 0, sizeof(psuSpec));
//设置挖矿指示灯正常，每个slave6块控制板，均设置指示灯正常
setLedsNormalMinig();
//MAX_BOARD_PER_SLAVE为6
for (int boardId = 0; boardId < MAX_BOARD_PER_SLAVE; boardId++)
{
	//取控制板id
	int boardNum = slaveId * MAX_BOARD_PER_SLAVE + boardId;
	//BoardStat初始化
	boards[boardId].init(boardId, boardNum);
}
//dummyBoard初始化
dummyBoard.init(0, 0);
//代码位置sm-miner-master/src/stats/SlaveStat.cpp
```

BoardStat初始化

```c++
this->boardId  = boardId;
this->boardNum = boardNum;
memset(&info, 0, sizeof(info));
memset(&spec, 0, sizeof(spec));
boardCurrent    = 0;
boardPower      = 0;
boardTotal.clear();
//MAX_SPI_PER_BOARD为1
//每块控制板有1个SPI总线
for (int spiId = 0; spiId < MAX_SPI_PER_BOARD; spiId++)
{
	spiTotal[spiId].clear();
}
for (int spiId = 0; spiId < MAX_SPI_PER_BOARD; spiId++)
{
	//MAX_PWC_PER_SPI为12
	//每个SPI总线有12块芯片
	for (int chipId = 0; chipId < MAX_PWC_PER_SPI; chipId++)
	{
		//PwcStat初始化
		pwcChips[spiId][chipId].init(this, spiId, chipId);
	}
}
dummyPwcChip.init(&SlaveStat::dummyBoard, 0, 0);
//代码位置sm-miner-master/src/stats/BoardStat.cpp
```

PwcStat初始化

```c++
parentBoard   = _parentBoard;
spiId         = _spiId;
spiSeq        = _spiSeq;
//MAX_BTC16_PER_PWC为11
//每个PWC有11个BTC16
for (int pwcSeq = 0; pwcSeq < MAX_BTC16_PER_PWC; pwcSeq++)
{
	//ChipStat初始化
	chips[pwcSeq].init(parentBoard, spiId, spiSeq, pwcSeq);
}
dummyChip.init(&SlaveStat::dummyBoard, 0, 0, 0);
quickTest.clear();
//代码位置sm-miner-master/src/stats/PwcStat.cpp
```

ChipStat初始化

```c++
parentBoard   = _parentBoard;
spiId         = _spiId;
spiSeq        = _spiSeq;
pwcSeq        = _pwcSeq;
hasData = false;
osc             = 0;
reads           = 0;
lastJobTime     = 0;
//如下类型均为StatCounter64，即清零操作
solutions.clear();
errors.clear();
jobsDone.clear();
restarts.clear();
//代码位置sm-miner-master/src/stats/ChipStat.cpp
```

## CpuInfo初始化

```c++
//读取/proc/cpuinfo
const char* fileName = "/proc/cpuinfo";
FILE* fp = fopen(fileName, "r");
if (!fp) {
	throw SystemException("CpuInfo: unable to open %s", fileName);
}

char buffer[4*1024];
size_t bytes_read = fread(buffer, 1, sizeof(buffer), fp);
fclose (fp);

if (bytes_read == 0 || bytes_read == sizeof(buffer)) {
	throw SystemException("CpuInfo: %s: file read error %s", fileName);
}
buffer[bytes_read] = '\0';

//获取cpuinfo中Hardware
char* match = strstr(buffer, "Hardware");
if (!match) {
	throw SystemException("CpuInfo: can not find 'Hardware' item");
}

char hwid[1024];
sscanf(match, "Hardware : %s", &hwid);
printf("CpuInfo: hardware id: %s\n", hwid);

//从Hardware中判断CPU类型
//目前支持BCM sun8i Altera
//BCM：博通，https://www.broadcom.cn/，CPU_RPI
//sun8i：珠海全志，http://www.allwinnertech.com/，CPU_OPI
//Altera：英特尔，https://www.altera.com.cn，CPU_SOC

if (strstr(hwid, "BCM")) {
	cpuType = CPU_RPI;
	printf("CpuInfo: CPU_RPI\n");
}
else if (strstr(hwid, "sun8i")) {
	cpuType = CPU_OPI;
	printf("CpuInfo: CPU_OPI\n");
}
else if (strstr(hwid, "Altera")) {
	cpuType = CPU_SOC;
	printf("CpuInfo: CPU_SOC\n");
}
else {
	throw SystemException("CpuInfo: unexpected hardware id");
}
//代码位置sm-miner-master/src/hw/CpuInfo.cpp
```

## GpioManager初始化

```c++
//树莓派和普通电脑不一样的地方在于它还带了17个可编程的GPIO
//（General Purpose Input/Output），可以用来驱动各种外设
//GPIO，即General Purpose Input Output，通用输入输出，也称总线扩展器
//SPI，即Serial Peripheral interface，串行外围设备接口
//-slave_spi_drv <dm>       Master SPI: drv mode 0-hw, 1-sw, 2-auto
switch (Application::configRW().slaveSpiDrv)
{
case SPI_DRV_SW:
	spiSwMode = true;
	break;
case SPI_DRV_AUTO:
	break;
}

if (spiSwMode) {
	printf("WARNING: GpioManager: running SPI in SW mode\n");
}

//判断CPU类型
//此处CPU类型仅支持CPU_RPI和CPU_OPI，即BCM和sun8i
switch (g_cpuInfo.cpuType)
{
//CPU_RPI，即BCM博通，将引入hw/bcm2835/bcm2835.h，调取bcm2835_init()
case CpuInfo::CPU_RPI: GpioPinPi::initMapping(); break;
//CPU_OPI，即sun8i珠海全志，
case CpuInfo::CPU_OPI: GpioPinOPi::initMapping(); break;
default:
	throw ApplicationException("GpioManager: unexpected cpu type");
}

//根据pin头编号而非gpio，创建pin管理器，可以为（RaspberryPi, OrangePi等）映射pin到gpio
//红、绿指示灯？
//树莓派 40Pin 引脚对照表：http://shumeipai.nxez.com/raspberry-pi-pins-version-40
//树莓派GPIO的编号规范：http://shumeipai.nxez.com/2014/10/18/raspberry-pi-gpio-port-naming.html
pinLedR         = createPin(16);
pinLedG         = createPin(18);

//其中true表示是否为输入
pinKey0         = createPin(22, true);
pinKey1         = createPin(13, true);

if (spiSwMode) {
	//MOSI Master Output Slave Input 主机输出从机输入
	pinSpiMosi      = createPin(19);
	//MISO Master Input Slave Output 主机输入从机输出
	pinSpiMiso      = createPin(21, true);
	pinSpiClk       = createPin(23);
}
else {
	//我们不能在硬件模式下创建 pin, 因为我们将从替代函数中切换针脚, 我们将失去使用硬件驱动程序的可能性, 因此我们正在创建虚拟针脚
	pinSpiMosi      = new GpioPinDummy();
	pinSpiMiso      = new GpioPinDummy();
	pinSpiClk       = new GpioPinDummy();

	//对于OrangePi Zero pull-up (or down) 输入MISO pin, 以避免随机数据, 当slave没有准备好时
	if (g_cpuInfo.cpuType == CpuInfo::CPU_OPI)
	{
		GpioPinOPi::setupPull(GpioPinOPi::pinToGpio(21), GpioPinOPi::PULL_UP);
	}
}

pinMux0    = createPin(12);
pinMux1    = createPin(15);
pinHwVer   = createPin(26, true);

pinNRst    = createPin(7);
pinBoot    = createPin(11);
pinNss     = createPin(24);
readHwVer();

if (Application::configRW().noMcuReset)
{
	printf("GpioManager: no MCU reset mode\n");
}
else {
	uint32_t slaveMask = Application::configRW().slaveMask;
	printf("GpioManager: reset slaves, mask 0x%02x\n", slaveMask);
	slaveResetByMask(slaveMask);
}
//代码位置sm-miner-master/src/hw/GpioManager.cpp
```

createPin

```c++
switch (g_cpuInfo.cpuType)
{
//RaspberryPi
case CpuInfo::CPU_RPI: return new GpioPinPi(GpioPinPi::pinToGpio(pinNumber), isInput);
//OrangePi
case CpuInfo::CPU_OPI: return new GpioPinOPi(GpioPinOPi::pinToGpio(pinNumber), isInput);
default:
	throw ApplicationException("GpioManager: unexpected cpu type");
}
//代码位置sm-miner-master/src/hw/GpioManager.cpp
```

## AppRegistry初始化（即AppComponent初始化）

```c++
//是否已完成初始化
if (isInitialized())
	throw ApplicationException("Application component has been already initialized.");
//此处doInit即：GpioPolling::doInit()、EventManager::doInit()和StratumPool::doInit()
//源自ApplicationImpl::ApplicationImpl()构造函数中：m_eventManager(m_appRegistry)、m_gpioPolling(m_appRegistry)和m_stratumPool(m_appRegistry)
doInit(config);
m_initialized = true;
//代码位置sm-miner-master/src/app/AppComponent.h
```

```c++
void GpioPolling::doInit(const Config& /*config*/)
{
    return;
    LOG_TRACE(logger) << "Initializing GpioPolling manager...\n";
    ledG.init(g_gpioManager.pinLedG);
    ledR.init(g_gpioManager.pinLedR);
    key0.init(g_gpioManager.pinKey0);
    key1.init(g_gpioManager.pinKey1);
}
//代码位置sm-miner-master/src/env/GpioPolling.cpp

void EventManager::doInit(const Config& config)
{
    LOG_TRACE(logger) << "Initializing Event Manager...\n";
    m_eventFilePath = config.eventFilePath;
}
//代码位置sm-miner-master/src/events/EventManager.cpp

void StratumPool::doInit(const Config& config)
{
    LOG_TRACE(logger) << "Initializing Pool...\n";
    m_poolConfig = config.poolConfig;
    m_totalPoolTimer.start();
}
//代码位置sm-miner-master/src/pool/StratumPool.cpp
```

## 参考文档

* [树莓派 40Pin 引脚对照表](http://shumeipai.nxez.com/raspberry-pi-pins-version-40)
* [树莓派GPIO的编号规范](http://shumeipai.nxez.com/2014/10/18/raspberry-pi-gpio-port-naming.html)