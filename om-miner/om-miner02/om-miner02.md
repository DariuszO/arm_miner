# om-miner(02) sm-miner-master程序运行

## 程序运行

```c++
void ApplicationImpl::run()
{
    LOG_TRACE(logger) << "Running the application...\n";
	//检查是否已初始化
    ensureInitialized();
	//是否测试模式
    if (config()->testConfig.testMode != TEST_MODE_NONE)
    {
        g_serverTest.run();
    }
    else
    {
        //程序组件启动
        start();
        appMain();
        stop();
    }
}
//代码位置sm-miner-master/src/app/ApplicationImpl.cpp
```

## 程序组件启动

```c++
void ApplicationImpl::start()
{
    LOG_TRACE(logger) << "Starting application components...\n";
	//是否已启动
    if (m_started)
        throw ApplicationException("Application is already started.");
	//是否已初始化
    ensureInitialized();
    //启动所有程序组件
	//
    if (m_appRegistry.selectComponents())
    {
        while (AppComponent* appComponent = m_appRegistry.nextComponent())
			//即调取doStart()
			//即GpioPolling::doStart()和StratumPool::doStart()
            appComponent->start();
    }
    m_started = true;
}
//代码位置sm-miner-master/src/app/ApplicationImpl.cpp
```

## StratumPool启动

![](StratumPool.png)

## appMain主循环

```c++
void ApplicationImpl::appMain()
{
	LOG_INFO(logger) << "Enter main cycle.\n";
	PollTimer statTimer;
	PollTimer minerAliveEventTimer;
	m_eventManager.reportEvent(EventType::MINER_START, "Enjoy your mining!");
	while (!m_needExit)
	{
		Config cfg = *config().getConfig();
		//SlaveGate轮询迭代，即SlaveGate::runTxRx()
		g_slaveGate.runPollingIteration();
		g_envManager.runPollingIteration();
		if (statTimer.isElapsedSec(cfg.logDelaySec))
		{
			statTimer.start();
			// Collect the iteration statistics and aggregate.
			g_masterStat.aggregate();
			StreamWriter wr(stdout);
			g_masterStat.printStat(wr);
            //JsonStat::exportStat();  //chenbo delete 20171126

			//chenbo add begin 20171127
			g_webgate.PushMinerStatus();
			//g_webgate.PushASICStatus();
			g_webgate.GetMinerConfig(cfg);
			Config& webminerconf = g_webgate.minerconfig;
			onConfigChange(webminerconf);
			//chenbo add end

			g_masterStat.saveStat();
		}

		if (cfg.aliveEventIntervalSec > 0 &&
			minerAliveEventTimer.isElapsedSec(cfg.aliveEventIntervalSec))
		{
			m_eventManager.reportEvent(EventType::MINER_ALIVE, "Enjoy your mining!");
			minerAliveEventTimer.start();
		}
        ::usleep(1000 * cfg.pollingDelayMs);
	}
	LOG_TRACE(logger) << "Leaving main cycle...\n";
}
```

## SlaveGate轮询迭代

待补充

## 参考文档



