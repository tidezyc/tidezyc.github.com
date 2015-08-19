---
title: Solr的中英文分词实现
category : solr
tags:
- solr
- lucene
published: true
---

对于Solr应该不需要过多介绍了，强大的功能也是都体验过了，但是solr一个较大的问题就是分词问题，特别是中英文的混合分词，处理起来非常棘手。
虽然solr自带了支持中文分词的cjk，但是其效果实在不好，所以solr要解决的一个问题就是中文分词问题，这里推荐的方案是利用ik进行分词。

ik是较早作中文分词的工具，其效果也是得到多数用户认同。但是现在作者似乎更新缓慢，对于最新的solr4.4支持不好，最新的更新也停留在2012年。

虽然不支持4.4版本（这也不是作者的错，solr的lucene的新版本接口都进行了修改，除非修改实现不然就没法向下兼容），但是我们也有办法的，我们可以利用他的分词工具自己封装一个TokenizerFactory，通过实现最新的4.4接口就可以让solr4.4用上ik了。

首先就是就在下载ik的原码，最新版是
然后自己实现一个TokenizerFactory：

``` java
package org.wltea.analyzer.lucene;

import java.io.Reader;
import java.util.Map;

import org.apache.lucene.analysis.Tokenizer;
import org.apache.lucene.analysis.util.TokenizerFactory;
import org.apache.lucene.util.AttributeSource.AttributeFactory;

public class IKAnalyzerTokenizerFactory extends TokenizerFactory{

    private boolean useSmart;

    public boolean useSmart() {
        return useSmart;
    }

    public void setUseSmart(boolean useSmart) {
        this.useSmart = useSmart;
    }

    public IKAnalyzerTokenizerFactory(Map<String, String> args) {
        super(args);
        assureMatchVersion();
        this.setUseSmart(args.get("useSmart").toString().equals("true"));
    }


    @Override
    public Tokenizer create(AttributeFactory factory, Reader input) {
        Tokenizer _IKTokenizer = new IKTokenizer(input , this.useSmart);
        return _IKTokenizer;
    }

}
```

然后重新打包jar放到solr的执行lib里，同时新建一个fieldType

``` xml
<fieldType name="text_ik" class="solr.TextField" >
  <analyzer type="index">
    <tokenizer class="org.wltea.analyzer.lucene.IKAnalyzerTokenizerFactory" useSmart="false"/>
  </analyzer>
  <analyzer type="query">
    <tokenizer class="org.wltea.analyzer.lucene.IKAnalyzerTokenizerFactory" useSmart="true"/>
  </analyzer>
</fieldType>
```
测试一下我们新的分词器：

```
// 输入
移动互联网

// 输出
移动，互联网，互联，联网
```
从结果来看，其效果还是比较不错的。

<!--mores-->

搞定了中文我们需要搞定英文
英文简单的分词是按照空格，标点，stopword等来分词。
比如`I'm coding`一般可以分词为`I'm, coding`或者`I, m, coding`。一般情况下这样也是可以接受的，但是如果用户输入`code`，是否应该搜到结果呢，如果要搜到该结果，那么我们需要处理我们的英文分词。

这里提供一种简单的实现，就是采用NGramFilterFactory，该过滤器简单的按照长度对词进行切分，该过滤器有两个参数`minGramSize`和`maxGramSize`，分别表示最小和最大的切分长度，默认是`1`和`2`。

``` xml
<analyzer>
  <tokenizer class="solr.StandardTokenizerFactory"/>
  <filter class="solr.NGramFilterFactory" minGramSize="1" maxGramSize="4"/>
</analyzer>
```
比如设置(min,max)为(3,5)，我们上面的句子“I'm coding”会得到以下的结果：

```
I'm，cod，codi，codin，coding，odi，odin，oding，din，ding，ing
```

当然这里也会有问题，就是小于3个词长的都会被过滤调，特别是中文和英文采用的是同一词长处理，如果min设为3，那么像`我，我们`这样的都会被过滤，解决办法就是min设为1，这样的结果就是会大大增加索引记录。影响检索速度。好处就是可以实现字母级别的匹配，然后通过设置匹配度阔值提升了搜索质量。

分别处理完了中文和英文，那么就要混合中英文处理了

+ 方案一是使用StandardTokenizerFactory和NGramFilterFactory，加上辅助的StopFilterFactory和LowerCaseFilterFactory等过滤器处理。也就是中文默认是按字逐个分开，当然前提是NGramFilterFactory的`minGramSize`要设置为1。

+ 方案二则是IKAnalyzerTokenizerFactory和NGramFilterFactory，通过ik实现对词的索引，然后在通过ngram进行长度分割。即在方案一的基础上增加对词的索引，提升索引质量。

+ 方案一和方案二如果还不够和谐的，那么我们还有个办法就是自定义的反感三，所谓自定义，自己写个tokenizer或者filter不就可以了，而且这一点也不复杂，这里就不细说了，有机会再专门写一个。

最后来个整合的配置参考一下：

``` xml
<fieldType name="text_ik" class="solr.TextField" positionIncrementGap="100">
  <analyzer type="index">
    <tokenizer class="org.wltea.analyzer.lucene.IKAnalyzerTokenizerFactory"  useSmart="false"/>
    <filter class="solr.StopFilterFactory" ignoreCase="true" words="stopwords.txt" />
    <filter class="solr.LowerCaseFilterFactory"/>
    <filter class="solr.NGramFilterFactory" minGramSize="1" maxGramSize="20"/>
  </analyzer>
  <analyzer type="query">
    <tokenizer class="org.wltea.analyzer.lucene.IKAnalyzerTokenizerFactory"  useSmart="true"/>
    <filter class="solr.StopFilterFactory" ignoreCase="true" words="stopwords.txt" />
    <filter class="solr.SynonymFilterFactory" synonyms="synonyms.txt" ignoreCase="true" expand="true"/>
    <filter class="solr.LowerCaseFilterFactory"/>
    <filter class="solr.NGramFilterFactory" minGramSize="1" maxGramSize="10"/>
  </analyzer>
</fieldType>
```
这里所提出的并不是最优的方案，或者说可能是比较傻瓜化的方案，但是solr的优势就是自由，你可以自己组合各种tokenizer和filter来实现你要的效果，或者干脆自己去实现tokenizer和filter，然后让强大的solr服务于你的项目。

参考：

+ [https://cwiki.apache.org/confluence/display/solr/About+This+Guide](https://cwiki.apache.org/confluence/display/solr/About+This+Guide)
+ [https://code.google.com/p/ik-analyzer/](https://code.google.com/p/ik-analyzer/)
