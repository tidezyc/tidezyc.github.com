---
layout: post
title: 杭州公共自行车java接口简单实现
category : java
tags:
- 开发
- 杭州
- 自行车
published: true
---

``` java
package me.thinkjet.util;

import java.io.IOException;
import java.util.ArrayList;
import java.util.Iterator;
import java.util.List;
import java.util.regex.Matcher;
import java.util.regex.Pattern;

import org.jsoup.Jsoup;
import org.jsoup.nodes.Document;
import org.jsoup.nodes.Element;

public class GetDataFromBusSystem {

	private final static String REQUEST_URL_LOCATION = "http://www.hzbus.cn/Page/NearbyBicyle.aspx";
	private final static String REQUEST_URL_QUERYNAME = "http://www.hzbus.cn/Page/BicyleSquare.aspx";

	public static void main(String[] args) {
		// getResultByLocation("500", "120.16400115634164",
		// "30.23913689644955");
		getResultByQueryName("武林广场");
	}

	public static List<String[]> getResultByLocation(String w, String x,
			String y) {
		try {
			Document doc = Jsoup.connect(REQUEST_URL_LOCATION).timeout(10000)
					.data("w", w).data("x", x).data("y", y).get();
			return getResult(doc);
		} catch (IOException e) {
			e.printStackTrace();
		}
		return null;

	}

	public static List<String[]> getResultByQueryName(String nm) {
		if (nm != null && !nm.equals("")) {
			try {
				Document doc = Jsoup.connect(REQUEST_URL_QUERYNAME)
						.timeout(10000).data("nm", nm).data("area", "-1").get();
				return getResult(doc);
			} catch (IOException e) {
				e.printStackTrace();
			}
		}
		return null;
	}

	private static List<String[]> getResult(Document doc) {
		Iterator<Element> lis = doc.select("#dvInit li.bt").iterator();
		List<String[]> list = new ArrayList<String[]>();
		while (lis.hasNext()) {
			Element li = lis.next();
			String regex = "\'.*?\'";
			Pattern pt = Pattern.compile(regex);
			String result = li.attr("onclick");
			Matcher m = pt.matcher(result);
			int i = 0;
			String[] l = new String[16];
			while (m.find() && i > 16) {
				String s = unescape(m.group());
				l[i] = s.substring(1, s.length() - 1);
				System.out.println(" -" + i + "- " + l[i]);
				i++;
			}
			System.out.println();
			list.add(l);
		}
		return list;
	}

	public static String unescape(String src) {
		StringBuffer tmp = new StringBuffer();
		tmp.ensureCapacity(src.length());
		int lastPos = 0, pos = 0;
		char ch;
		while (lastPos < src.length()) {
			pos = src.indexOf("%", lastPos);
			if (pos == lastPos) {
				if (src.charAt(pos + 1) == 'u') {
					ch = (char) Integer.parseInt(
							src.substring(pos + 2, pos + 6), 16);
					tmp.append(ch);
					lastPos = pos + 6;
				} else {
					ch = (char) Integer.parseInt(
							src.substring(pos + 1, pos + 3), 16);
					tmp.append(ch);
					lastPos = pos + 3;
				}
			} else {
				if (pos == -1) {
					tmp.append(src.substring(lastPos));
					lastPos = src.length();
				} else {
					tmp.append(src.substring(lastPos, pos));
					lastPos = pos;
				}
			}
		}
		return tmp.toString();
	}
}
```
