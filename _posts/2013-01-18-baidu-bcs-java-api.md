---
layout: post
title: 百度云存储的java接口实现
category : bae
tags:
- bcs
- J2EE
- 云存储
- 百度云
published: true
---

``` java
package me.thinkjet.util;

import java.io.File;
import java.io.FileInputStream;
import java.io.FileNotFoundException;
import java.io.InputStream;

import com.baidu.inf.iis.bcs.BaiduBCS;
import com.baidu.inf.iis.bcs.auth.BCSCredentials;
import com.baidu.inf.iis.bcs.auth.BCSSignCondition;
import com.baidu.inf.iis.bcs.http.HttpMethodName;
import com.baidu.inf.iis.bcs.model.DownloadObject;
import com.baidu.inf.iis.bcs.model.Empty;
import com.baidu.inf.iis.bcs.model.ObjectListing;
import com.baidu.inf.iis.bcs.model.ObjectMetadata;
import com.baidu.inf.iis.bcs.request.GenerateUrlRequest;
import com.baidu.inf.iis.bcs.request.GetObjectRequest;
import com.baidu.inf.iis.bcs.request.ListObjectRequest;
import com.baidu.inf.iis.bcs.request.PutObjectRequest;
import com.baidu.inf.iis.bcs.response.BaiduBCSResponse;

@SuppressWarnings("unused")
public class BaiduCloudStorage {

	static String host = "bcs.duapp.com";
	static String accessKey = "9f246d1b92ad869e3d5ac3d9a31febba";
	static String secretKey = "9cf6e0ba7036c915d07caabd7125a595";
	static String bucket = "thinkjet";

	private static BaiduBCS getBCS() {
		BCSCredentials credentials = new BCSCredentials(accessKey, secretKey);
		BaiduBCS baiduBCS = new BaiduBCS(credentials, host);
		baiduBCS.setDefaultEncoding("UTF-8"); // Default UTF-8
		return baiduBCS;
	}

	public static String generateUrl(String object) {
		BaiduBCS baiduBCS = getBCS();
		GenerateUrlRequest generateUrlRequest = new GenerateUrlRequest(
				HttpMethodName.GET, bucket, object);
		generateUrlRequest.setBcsSignCondition(new BCSSignCondition());
		generateUrlRequest.getBcsSignCondition().setIp("1.1.1.1");
		generateUrlRequest.getBcsSignCondition().setTime(123455L);
		generateUrlRequest.getBcsSignCondition().setSize(123455L);
		return baiduBCS.generateUrl(generateUrlRequest);
	}

	public static Empty deleteObject(String object) {
		BaiduBCS baiduBCS = getBCS();
		return baiduBCS.deleteObject(bucket, object).getResult();
	}

	public static void getObjectMetadata(String object) {
		BaiduBCS baiduBCS = getBCS();
		ObjectMetadata objectMetadata = baiduBCS.getObjectMetadata(bucket,
				object).getResult();
	}

	private static BaiduBCSResponse&lt;DownloadObject&gt; getObjectWithDestFile(
			String object, File destFile) {
		BaiduBCS baiduBCS = getBCS();
		GetObjectRequest getObjectRequest = new GetObjectRequest(bucket, object);
		return baiduBCS.getObject(getObjectRequest, destFile);
	}

	private static BaiduBCSResponse&lt;ObjectListing&gt; listObject() {
		BaiduBCS baiduBCS = getBCS();
		ListObjectRequest listObjectRequest = new ListObjectRequest(bucket);
		listObjectRequest.setStart(0);
		listObjectRequest.setLimit(20);
		return baiduBCS.listObject(listObjectRequest);
	}

	public static ObjectMetadata putObjectByFile(String object, File file) {
		BaiduBCS baiduBCS = getBCS();
		PutObjectRequest request = new PutObjectRequest(bucket, object, file);
		ObjectMetadata metadata = new ObjectMetadata();
		request.setMetadata(metadata);
		BaiduBCSResponse&lt;ObjectMetadata&gt; response = baiduBCS.putObject(request);
		return response.getResult();
	}

	public static ObjectMetadata putObjectByInputStream(String object, File file)
			throws FileNotFoundException {
		BaiduBCS baiduBCS = getBCS();
		InputStream fileContent = new FileInputStream(file);
		ObjectMetadata objectMetadata = new ObjectMetadata();
		objectMetadata.setContentLength(file.length());
		PutObjectRequest request = new PutObjectRequest(bucket, object,
				fileContent, objectMetadata);
		return baiduBCS.putObject(request).getResult();
	}

	public static void setObjectMetadata(String object, String contentType) {
		BaiduBCS baiduBCS = getBCS();
		ObjectMetadata objectMetadata = new ObjectMetadata();
		objectMetadata.setContentType(contentType);
		baiduBCS.setObjectMetadata(bucket, object, objectMetadata);
	}

}
```
