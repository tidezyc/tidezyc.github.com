---
layout: post
title: SHA1加密实现
category : java
tags:
- Java
- sha
- sha1
- 加密
published: true
---
SHA1加密算法的java实现

``` java
package me.thinkjet.util;
public class SHA1Encryption {
	//hex output format. false - lowercase; true - uppercase
	private static final boolean hexcase = false;
	//base-64 pad character. "=" for strict RFC compliance
	private static final String b64pad = "=";
	//bits per input character. 8 - ASCII; 16 - Unicode
	private static final int chrsz = 8;

	/**
	 * This is one of the functions you'll usually want to call It take a string
	 * arguments and returns either hex or base-64 encoded strings
	 */
	public static String b64_hmac_sha1(String key, String data) {
		return binb2b64(core_hmac_sha1(key, data));
	}

	/**
	 * This is one of the functions you'll usually want to call It take a string
	 * argument and returns either hex or base-64 encoded strings
	 */
	public static String b64_sha1(String s) {
		s = (s == null) ? "" : s;
		return binb2b64(core_sha1(str2binb(s), s.length() * chrsz));
	}

	/**
	 * Convert an array of big-endian words to a base-64 string
	 */
	private static String binb2b64(int[] binarray) {
		String tab = "ABCDEFGHIJKLMNOPQRSTUVWXYZabcdefghijklmnopqrstuvwxyz0123456789+/";
		String str = "";
		binarray = strechBinArray(binarray, binarray.length * 4);
		for (int i = 0; i &lt; binarray.length * 4; i += 3) {
			int triplet = (((binarray[i &gt;&gt; 2] &gt;&gt; 8 * (3 - i % 4)) &amp; 0xFF) &lt;&lt; 16)
					¦ (((binarray[i + 1 &gt;&gt; 2] &gt;&gt; 8 * (3 - (i + 1) % 4)) &amp; 0xFF) &lt;&lt; 8)
					¦ ((binarray[i + 2 &gt;&gt; 2] &gt;&gt; 8 * (3 - (i + 2) % 4)) &amp; 0xFF);
			for (int j = 0; j &lt; 4; j++) {
				if (i * 8 + j * 6 &gt; binarray.length * 32)
					str += b64pad;
				else
					str += tab.charAt((triplet &gt;&gt; 6 * (3 - j)) &amp; 0x3F);
			}
		}
		return cleanB64Str(str);
	}

	/**
	 * Convert an array of big-endian words to a hex string.
	 */
	private static String binb2hex(int[] binarray) {
		String hex_tab = hexcase ? "0123456789ABCDEF" : "0123456789abcdef";
		String str = "";
		for (int i = 0; i &lt; binarray.length * 4; i++) {
			char a = (char) hex_tab
					.charAt((binarray[i &gt;&gt; 2] &gt;&gt; ((3 - i % 4) * 8 + 4)) &amp; 0xF);
			char b = (char) hex_tab
					.charAt((binarray[i &gt;&gt; 2] &gt;&gt; ((3 - i % 4) * 8)) &amp; 0xF);
			str += (new Character(a).toString() + new Character(b).toString());
		}
		return str;
	}

	/**
	 * Convert an array of big-endian words to a string
	 */
	private static String binb2str(int[] bin) {
		String str = "";
		int mask = (1 &lt;&lt; chrsz) - 1;
		for (int i = 0; i &lt; bin.length * 32; i += chrsz)
			str += (char) ((bin[i &gt;&gt; 5] &gt;&gt;&gt; (24 - i % 32)) &amp; mask);
		return str;
	}

	/**
	 * Cleans a base64 String from all the trailing 'A' or other characters put
	 * there by binb2b64 that made the bin array 4 times larger than it
	 * originally was.
	 * 
	 */
	private static String cleanB64Str(String str) {
		str = (str == null) ? "" : str;
		int len = str.length();
		if (len &lt;= 1)
			return str;
		char trailChar = str.charAt(len - 1);
		String trailStr = "";
		for (int i = len - 1; i &gt;= 0 &amp;&amp; str.charAt(i) == trailChar; i--)
			trailStr += str.charAt(i);
		return str.substring(0, str.indexOf(trailStr));
	}

	/**
	 * Makes an int array of a length less than 16 an array of length 16 with
	 * all previous cells at their previous indexes.
	 */
	private static int[] complete216(int[] oldbin) {
		if (oldbin.length &gt;= 16)
			return oldbin;
		int[] newbin = new int[16 - oldbin.length];
		for (int i = 0; i &lt; newbin.length; newbin[i] = 0, i++)
			;
		return concat(oldbin, newbin);
	}

	/**
	 * Joins two int arrays and return one that contains all the previous
	 * values. This corresponds to the concat method of the JavaScript Array
	 * object.
	 */
	private static int[] concat(int[] oldbin, int[] newbin) {
		int[] retval = new int[oldbin.length + newbin.length];
		for (int i = 0; i &lt; (oldbin.length + newbin.length); i++) {
			if (i &lt; oldbin.length)
				retval[i] = oldbin[i];
			else
				retval[i] = newbin[i - oldbin.length];
		}
		return retval;
	}

	/**
	 * Calculate the HMAC-SHA1 of a key and some data
	 */
	private static int[] core_hmac_sha1(String key, String data) {
		key = (key == null) ? "" : key;
		data = (data == null) ? "" : data;
		int[] bkey = complete216(str2binb(key));
		if (bkey.length &gt; 16)
			bkey = core_sha1(bkey, key.length() * chrsz);
		int[] ipad = new int[16];
		int[] opad = new int[16];
		for (int i = 0; i &lt; 16; ipad[i] = 0, opad[i] = 0, i++)
			;
		for (int i = 0; i &lt; 16; i++) {
			ipad[i] = bkey[i] ^ 0x36363636;
			opad[i] = bkey[i] ^ 0x5C5C5C5C;
		}
		int[] hash = core_sha1(concat(ipad, str2binb(data)),
				512 + data.length() * chrsz);
		return core_sha1(concat(opad, hash), 512 + 160);
	}

	/**
	 * Calculate the SHA-1 of an array of big-endian words, and a bit length
	 */
	private static int[] core_sha1(int[] x, int len) {
		int size = (len &gt;&gt; 5);
		x = strechBinArray(x, size);
		x[len &gt;&gt; 5] ¦= 0x80 &lt;&lt; (24 - len % 32);
		size = ((len + 64 &gt;&gt; 9) &lt;&lt; 4) + 15;
		x = strechBinArray(x, size);
		x[((len + 64 &gt;&gt; 9) &lt;&lt; 4) + 15] = len;
		int[] w = new int[80];
		int a = 1732584193;
		int b = -271733879;
		int c = -1732584194;
		int d = 271733878;
		int e = -1009589776;
		for (int i = 0; i &lt; x.length; i += 16) {
			int olda = a;
			int oldb = b;
			int oldc = c;
			int oldd = d;
			int olde = e;
			for (int j = 0; j &lt; 80; j++) {
				if (j &lt; 16)
					w[j] = x[i + j];
				else
					w[j] = rol(w[j - 3] ^ w[j - 8] ^ w[j - 14] ^ w[j - 16], 1);
				int t = safe_add(safe_add(rol(a, 5), sha1_ft(j, b, c, d)),
						safe_add(safe_add(e, w[j]), sha1_kt(j)));
				e = d;
				d = c;
				c = rol(b, 30);
				b = a;
				a = t;
			}
			a = safe_add(a, olda);
			b = safe_add(b, oldb);
			c = safe_add(c, oldc);
			d = safe_add(d, oldd);
			e = safe_add(e, olde);
		}
		int[] retval = new int[5];
		retval[0] = a;
		retval[1] = b;
		retval[2] = c;
		retval[3] = d;
		retval[4] = e;
		return retval;
	}

	/**
	 * This is one of the functions you'll usually want to call It take a string
	 * arguments and returns either hex or base-64 encoded strings
	 */
	public static String hex_hmac_sha1(String key, String data) {
		return binb2hex(core_hmac_sha1(key, data));
	}

	/**
	 * This is one of the functions you'll usually want to call It take a string
	 * argument and returns either hex or base-64 encoded strings
	 */
	public static String hex_sha1(String s) {
		s = (s == null) ? "" : s;
		return binb2hex(core_sha1(str2binb(s), s.length() * chrsz));
	}

	/**
	 * Bitwise rotate a 32-bit number to the left.
	 */
	private static int rol(int num, int cnt) {
		return (num &lt;&lt; cnt) ¦ (num &gt;&gt;&gt; (32 - cnt));
	}

	/**
	 * Add ints, wrapping at 2^32. This uses 16-bit operations internally to
	 * work around bugs in some JS interpreters. The original function is part
	 * of the sha1.js library. It's here for compatibility.
	 */
	private static int safe_add(int x, int y) {
		int lsw = (int) (x &amp; 0xFFFF) + (int) (y &amp; 0xFFFF);
		int msw = (x &gt;&gt; 16) + (y &gt;&gt; 16) + (lsw &gt;&gt; 16);
		return (msw &lt;&lt; 16) ¦ (lsw &amp; 0xFFFF);
	}

	/**
	 * Perform the appropriate triplet combination function for the current
	 */
	private static int sha1_ft(int t, int b, int c, int d) {
		if (t &lt; 20)
			return (b &amp; c) ¦ ((~b) &amp; d);
		if (t &lt; 40)
			return b ^ c ^ d;
		if (t &lt; 60)
			return (b &amp; c) ¦ (b &amp; d) ¦ (c &amp; d);
		return b ^ c ^ d;
	}

	/**
	 * Determine the appropriate additive constant for the current iteration
	 */
	private static int sha1_kt(int t) {
		return (t &lt; 20) ? 1518500249 : (t &lt; 40) ? 1859775393
				: (t &lt; 60) ? -1894007588 : -899497514;
	}

	/**
	 * This is one of the functions you'll usually want to call It take a string
	 * arguments and returns either hex or base-64 encoded strings
	 */
	public static String str_hmac_sha1(String key, String data) {
		return binb2str(core_hmac_sha1(key, data));
	}

	/**
	 * This is one of the functions you'll usually want to call It take a string
	 * argument and returns either hex or base-64 encoded strings
	 */
	public static String str_sha1(String s) {
		s = (s == null) ? "" : s;
		return binb2str(core_sha1(str2binb(s), s.length() * chrsz));
	}

	/**
	 * Convert an 8-bit or 16-bit string to an array of big-endian words In
	 * 8-bit function, characters &gt;255 have their hi-byte silently ignored.
	 */
	private static int[] str2binb(String str) {
		str = (str == null) ? "" : str;
		int[] tmp = new int[str.length() * chrsz];
		int mask = (1 &lt;&lt; chrsz) - 1;
		for (int i = 0; i &lt; str.length() * chrsz; i += chrsz)
			tmp[i &gt;&gt; 5] ¦= ((int) (str.charAt(i / chrsz)) &amp; mask) &lt;&lt; (24 - i % 32);

		int len = 0;
		for (int i = 0; i &lt; tmp.length &amp;&amp; tmp[i] != 0; i++, len++)
			;
		int[] bin = new int[len];
		for (int i = 0; i &lt; len; i++)
			bin[i] = tmp[i];
		return bin;
	}

	/**
	 * increase an int array to a desired sized + 1 while keeping the old
	 * values.
	 */
	private static int[] strechBinArray(int[] oldbin, int size) {
		int currlen = oldbin.length;
		if (currlen &gt;= size + 1)
			return oldbin;
		int[] newbin = new int[size + 1];
		for (int i = 0; i &lt; size; newbin[i] = 0, i++)
			;
		for (int i = 0; i &lt; currlen; i++)
			newbin[i] = oldbin[i];
		return newbin;
	}
}

```
