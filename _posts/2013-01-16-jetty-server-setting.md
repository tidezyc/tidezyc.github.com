---
layout: post
title: JettyServer服务器配置java实现
category : server
tags:
- jetty
- server
- web server
published: true
---

``` java
package me.thinkjet.hzbike;

import java.io.File;
import java.io.IOException;
import java.lang.management.ManagementFactory;
import java.net.DatagramSocket;
import java.net.ServerSocket;
import java.util.ArrayList;
import java.util.Collections;
import java.util.List;

import javax.management.MBeanServer;

import org.mortbay.jetty.Server;
import org.mortbay.jetty.nio.SelectChannelConnector;
import org.mortbay.jetty.webapp.WebAppContext;
import org.mortbay.management.MBeanContainer;
import org.mortbay.util.Scanner;

public class JettyServer {

	private final static boolean enablescanner = true;
	private static WebAppContext web;

	public static void start(String webAppDir, Integer port, String context,
			int scanIntervalSeconds) {
		Server server = new Server();

		if (port != null) {
			if (!available(port)) {
				throw new IllegalStateException("port: " + port
						+ " already in use!");
			}
			SelectChannelConnector connector = new SelectChannelConnector();
			connector.setPort(port);

			server.addConnector(connector);
		}

		web = new WebAppContext();

		// 警告: 设置成 true 无法支持热加载
		// web.setParentLoaderPriority(false);
		web.setContextPath(context);
		web.setWar(webAppDir);
		web.setInitParams(Collections.singletonMap(
				"org.mortbay.jetty.servlet.Default.useFileMappedBuffer",
				"false"));
		server.addHandler(web);

		MBeanServer mBeanServer = ManagementFactory.getPlatformMBeanServer();
		MBeanContainer mBeanContainer = new MBeanContainer(mBeanServer);
		server.getContainer().addEventListener(mBeanContainer);
		mBeanContainer.start();

		// configureScanner
		if (enablescanner) {
			final ArrayList&lt;File&gt; scanList = new ArrayList&lt;File&gt;();
			scanList.add(new File(getRootClassPath()));
			Scanner scanner = new Scanner();
			scanner.setReportExistingFilesOnStartup(false);
			scanner.setScanInterval(scanIntervalSeconds);
			scanner.setScanDirs(scanList);
			scanner.addListener(new Scanner.BulkListener() {
				public void filesChanged(
						@SuppressWarnings("rawtypes") List changes) {
					try {
						System.err.println("Loading changes ......");
						web.stop();
						web.start();
						System.err.println("Loading complete.\n");

					} catch (Exception e) {
						System.err
								.println("Error reconfiguring/restarting webapp after change in watched files");
						e.printStackTrace();
					}
				}
			});
			System.err.println("Starting scanner at interval of "
					+ scanIntervalSeconds + " seconds.");
			scanner.start();
		}

		try {
			server.start();
			server.join();
		} catch (Exception e) {
			e.printStackTrace();
			System.exit(100);
		}
		return;
	}

	private static boolean available(int port) {
		if (port &lt;= 0) {
			throw new IllegalArgumentException("Invalid start port: " + port);
		}

		ServerSocket ss = null;
		DatagramSocket ds = null;
		try {
			ss = new ServerSocket(port);
			ss.setReuseAddress(true);
			ds = new DatagramSocket(port);
			ds.setReuseAddress(true);
			return true;
		} catch (IOException e) {
		} finally {
			if (ds != null) {
				ds.close();
			}

			if (ss != null) {
				try {
					ss.close();
				} catch (IOException e) {
				}
			}
		}
		return false;
	}

	public static String getRootClassPath() {
		String path = JettyServer.class.getClassLoader().getResource("").getPath();
		return new File(path).getAbsolutePath();
	}
}

```

调用：
``` java
	public static void main(String[] args) {
	JettyServer.start("WebRoot", 80, "/", 5);
}
```
