---
title: Lucene初体验
categories:
  - 大数据
tags:
  - lucene
keywords: lucene, lucene7.4
date: 2018-08-18 16:23:02
---

### 前言
本文的简要内容：

1. Lucene简介
2. 体验Lucene Demo
3. Lucene 核心类介绍
4. Lucene 索引文件格式


### Lucene简介
Lucene是目前最流行的Java开源搜索引擎类库,最新版本为7.4.0。Lucene通常用于全文检索,Lucene具有简单高效跨平台等特点,因此有不少搜索引擎都是基于Lucene构建的,例如:Elasticsearch,Solr等等。

现代搜索引擎的两大核心就是索引和搜索，建立索引的过程就是对源数据进行处理，例如过滤掉一些特殊字符或词语，单词大小写转换，分词，建立倒排索引等支持后续高效准确的搜索。而搜索则是直接提供给用户的功能，尽管面向的用户不同，诸如百度，谷歌等互联网公司以及各种企业都提供了各自的搜索引擎。搜索过程需要对搜索关键词进行分词等处理，然后再引擎内部构建查询，还要根据相关度对搜索结果进行排序，最终把命中结果展示给用户。

Lucene只是一个提供索引和查询的**类库**，并不是一个应用，程序员需要根据自己的应用场景进行如数据获取、数据预处理、用户界面提供等工作。

搜索程序的典型组件如下所示：

![搜索程序的典型组件](http://image.laijianfeng.org/20180818_132723.jpg)


下图为Lucene与应用程序的关系:

![Lucene与应用程序的关系](http://image.laijianfeng.org/fig001.jpg)

### 体验Lucene Demo

接下来先来看一个简单的demo

> note:
> 代码在 [start Lucene](https://github.com/whirlys/elastic-example/tree/master/startlucene/src/main/java/startlucene)

#### 引入 Maven 依赖

```
	<properties>
		<project.build.sourceEncoding>UTF-8</project.build.sourceEncoding>
		<maven.compiler.source>1.8</maven.compiler.source>
		<maven.compiler.target>1.8</maven.compiler.target>
		<lucene.version>7.4.0</lucene.version>
	</properties>

	<dependencies>
		<dependency>
			<groupId>junit</groupId>
			<artifactId>junit</artifactId>
			<version>4.11</version>
			<scope>test</scope>
		</dependency>

		<dependency>
			<groupId>org.apache.lucene</groupId>
			<artifactId>lucene-core</artifactId>
			<version>${lucene.version}</version>
		</dependency>

		<dependency>
			<groupId>org.apache.lucene</groupId>
			<artifactId>lucene-queryparser</artifactId>
			<version>${lucene.version}</version>
		</dependency>

	</dependencies>
```

#### 索引类 IndexFiles.java

```
import org.apache.lucene.analysis.*;
import org.apache.lucene.analysis.standard.*;
import org.apache.lucene.document.*;
import org.apache.lucene.index.*;
import org.apache.lucene.store.*;

import java.io.*;
import java.nio.charset.*;
import java.nio.file.*;
import java.nio.file.attribute.*;

public class IndexFiles {
	public static void main(String[] args) {
		String indexPath = "D:/lucene_test/index"; // 建立索引文件的目录
		String docsPath = "D:/lucene_test/docs"; // 读取文本文件的目录

		Path docDir = Paths.get(docsPath);

		IndexWriter writer = null;
		try {
			// 存储索引数据的目录
			Directory dir = FSDirectory.open(Paths.get(indexPath));
			// 创建分析器
			Analyzer analyzer = new StandardAnalyzer();
			IndexWriterConfig iwc = new IndexWriterConfig(analyzer);
			iwc.setOpenMode(IndexWriterConfig.OpenMode.CREATE);

			writer = new IndexWriter(dir, iwc);
			indexDocs(writer, docDir);

			writer.close();
		} catch (IOException e) {
			e.printStackTrace();
		}
	}

	private static void indexDocs(final IndexWriter writer, Path path) throws IOException {
		if (Files.isDirectory(path)) {
			Files.walkFileTree(path, new SimpleFileVisitor<Path>() {
				@Override
				public FileVisitResult visitFile(Path file, BasicFileAttributes attrs) {
					try {
						indexDoc(writer, file);
					} catch (IOException ignore) {
						// 不索引那些不能读取的文件,忽略该异常
					}
					return FileVisitResult.CONTINUE;
				}
			});
		} else {
			indexDoc(writer, path);
		}
	}

	private static void indexDoc(IndexWriter writer, Path file) throws IOException {
		try (InputStream stream = Files.newInputStream(file)) {
			// 创建一个新的空文档
			Document doc = new Document();
			// 添加字段
			Field pathField = new StringField("path", file.toString(), Field.Store.YES);
			doc.add(pathField);
			Field contentsField = new TextField("contents",
					new BufferedReader(new InputStreamReader(stream, StandardCharsets.UTF_8)));
			doc.add(contentsField);
			System.out.println("adding " + file);
			// 写文档
			writer.addDocument(doc);
		}
	}
}
```

#### 查询类 SearchFiles.java

```
import org.apache.lucene.analysis.*;
import org.apache.lucene.analysis.standard.*;
import org.apache.lucene.document.*;
import org.apache.lucene.index.*;
import org.apache.lucene.queryparser.classic.*;
import org.apache.lucene.search.*;
import org.apache.lucene.store.*;

import java.io.*;
import java.nio.charset.*;
import java.nio.file.*;

public class SearchFiles {
	public static void main(String[] args) throws Exception {
		String indexPath = "D:/lucene_test/index"; // 建立索引文件的目录
		String field = "contents";
		IndexReader reader = DirectoryReader.open(FSDirectory.open(Paths.get(indexPath)));
		IndexSearcher searcher = new IndexSearcher(reader);
		Analyzer analyzer = new StandardAnalyzer();

		BufferedReader in = null;
		in = new BufferedReader(new InputStreamReader(System.in, StandardCharsets.UTF_8));
		QueryParser parser = new QueryParser(field, analyzer);
		System.out.println("Enter query:");
		// 从Console读取要查询的语句
		String line = in.readLine();
		if (line == null || line.length() == -1) {
			return;
		}
		line = line.trim();
		if (line.length() == 0) {
			return;
		}

		Query query = parser.parse(line);
		System.out.println("Searching for:" + query.toString(field));
		doPagingSearch(searcher, query);
		in.close();
		reader.close();
	}

	private static void doPagingSearch(IndexSearcher searcher, Query query) throws IOException {
		// TopDocs保存搜索结果
		TopDocs results = searcher.search(query, 10);
		ScoreDoc[] hits = results.scoreDocs;
		int numTotalHits = Math.toIntExact(results.totalHits);
		System.out.println(numTotalHits + " total matching documents");
		for (ScoreDoc hit : hits) {
			Document document = searcher.doc(hit.doc);
			System.out.println("文档:" + document.get("path"));
			System.out.println("相关度:" + hit.score);
			System.out.println("================================");
		}

	}
}

```

#### 测试

首先创建文件夹 `D:\lucene_test`，在 `lucene_test` 下再创建 `docs` 文件夹，用来存储要索引的测试文件

在 `docs` 下创建3个文件 test1.txt, test2.txt, test3.txt，分别写入 hello world、 hello lucene、 hello elasticsearch

运行索引类 IndexFiles.java，可看到Console输出

```
adding D:\lucene_test\docs\test1.txt
adding D:\lucene_test\docs\test2.txt
adding D:\lucene_test\docs\test3.txt
```

![Lucene的索引文件](http://image.laijianfeng.org/20180818_135449.png)

运行查询类 SearchFiles.java，搜索 hello ，三个文件相关度一样

```
Enter query:
hello
Searching for:hello
3 total matching documents
文档:D:\lucene_test\docs\test1.txt
相关度:0.13353139
================================
文档:D:\lucene_test\docs\test2.txt
相关度:0.13353139
================================
文档:D:\lucene_test\docs\test3.txt
相关度:0.13353139
================================
```

搜索 hello lucene，test2.txt的相关度比其他两个高

```
Enter query:
hello lucene
Searching for:hello lucene
3 total matching documents
文档:D:\lucene_test\docs\test2.txt
相关度:1.1143606
================================
文档:D:\lucene_test\docs\test1.txt
相关度:0.13353139
================================
文档:D:\lucene_test\docs\test3.txt
相关度:0.13353139
================================
```


### Lucene 核心类介绍

#### 核心索引类

IndexWriter

```
进行索引写操作的一个中心组件
不能进行读取和搜索
```

Directory

```
Directory代表Lucene索引的存放位置
常用的实现：
    FSDerectory:表示一个存储在文件系统中的索引的位置
    RAMDirectory:表示一个存储在内存当中的索引的位置
作用：
    IndexWriter通过获取Directory的一个具体实现，在Directory指向的位置中操作索引
```

Analyzer

```
Analyzer，分析器，相当于筛子，对内容进行过滤，分词，转换等
作用：把过滤之后的数据交给indexWriter进行索引
```

Document

```
用来存放文档（数据），该文档为非结构化数据中抓取的相关数据
通过Field(域)组成Document，类似于mysql中的一个个字段组成的一条记录
```

Field

```
Document中的一个字段
```

#### 核心搜索类

IndexSearcher

```
IndexSearcher在建立好的索引上进行搜索
它只能以 只读 的方式打开一个索引，所以可以有多个IndexSearcher的实例在一个索引上进行操作
```

Term

```
Term是搜索的基本单元，一个Term由 key:value 组成（类似于mysql中的  字段名称=查询的内容）
例子： Query query = new TermQuery(new Term("filename", "lucene"));
```

Query

```
Query是一个抽象类，用来将用户输入的查询字符串封装成Lucene能够识别的Query
```

TermQuery
```
Query子类，Lucene支持的最基本的一个查询类
例子：TermQuery termQuery = new TermQuery(new Term("filename", "lucene"));
```


BooleanQuery

```
BooleanQUery，布尔查询,是一个组合Query（多个查询条件的组合）
BooleanQuery是可以嵌套的

栗子：
BooleanQuery query = new BooleanQuery();
BooleanQuery query2 = new BooleanQuery();
TermQuery termQuery1 = new TermQuery(new Term("fileName", "lucene"));
TermQuery termQuery2 = new TermQuery(new Term("fileName", "name"));
query2.add(termQuery1, Occur.SHOULD);
query.add(termQuery2, Occur.SHOULD);
query.add(query2, Occur.SHOULD);;		//BooleanQuery是可以嵌套的

Occur枚举：
	MUST
	SHOULD
	FILTER
	MUST_NOT
```

NumericRangeQuery

```
数字区间查询
栗子：
Query newLongRange = NumericRangeQuery.newLongRange("fileSize",0l, 100l, true, true);
```

PrefixQuery

```
前缀查询，查询分词中含有指定字符开头的内容
栗子：
PrefixQuery query = new PrefixQuery(new Term("fileName","hell"));
```

PhraseQuery

```
短语查询
栗子1：
	PhraseQuery query = new PhraseQuery();
	query.add(new Term("fileName","lucene"));
```

FuzzyQuery

```
模糊查询
栗子：
FuzzyQuery query = new FuzzyQuery(new Term("fileName","lucene"));
```

WildcardQuery

```
通配符查询：
* ：任意字符（0或多个）
? : 一个字符

栗子：
WildcardQuery query = new WildcardQuery(new Term("fileName","*"));
```

RegexQuery

```
正则表达式查询
栗子：搜索含有最少1个字符，最多6个字符的
RegexQuery query = new RegexQuery(new Term("fileName","[a-z]{1,6}"));
```

MultiFieldQueryParser

```
查询多个field
栗子：
String[] fields = {"fileName","fileContent"};
MultiFieldQueryParser queryParser = new MultiFieldQueryParser(fields, new StandardAnalyzer());
Query query = queryParser.parse("fileName:lucene AND filePath:a");
```

TopDocs

```
TopDocs类是一个简单的指针容器,指针一般指向前N个排名的搜索结果,搜索结果即匹配条件的文档
TopDocs会记录前N个结果中每个结果的int docID和浮点数型分数(反映相关度)

栗子：
    TermQuery searchingBooks = new TermQuery(new Term("subject","search")); 
    Directory dir = TestUtil.getBookIndexDirectory();
    IndexSearcher searcher = new IndexSearcher(dir);
    TopDocs matches = searcher.search(searchingBooks, 10);
```


### Lucene 6.0 索引文件格式
#### 倒排索引
谈到倒排索引，那么首先看看正排是什么样子的呢？假设文档1包含【中文、英文、日文】，文档2包含【英文、日文、韩文】，文档3包含【韩文，中文】，那么根据文档去查找内容的话

```
文档1->【中文、英文、日文】
文档2->【英文、日文、韩文】
文档3->【韩文，中文】
```

反过来，根据内容去查找文档

```
中文->【文档1、文档3】
英文->【文档1、文档2】
日文->【文档1、文档2】
韩文->【文档2、文档3】
```

这就是倒排索引，而Lucene擅长的也正在于此

#### 段（Segments）

Lucene的索引可能是由多个子索引或Segments组成。每个Segment是一个完全独立的索引，可以单独用于搜索，索引涉及

1. 为新添加的documents创建新的segments
2. 合并已经存在的segments

搜索可能涉及多个segments或多个索引，每个索引可能由一组segments组成

#### 文档编号
Lucene通过一个整型的文档编号指向每个文档，第一个被加入索引的文档编号为0，后续加入的文档编号依次递增。
注意文档编号是可能发生变化的，所以在Lucene外部存储这些值时需要格外小心。

#### 索引结构概述

每个segment索引包括信息

- Segment info：包含有关segment的元数据，例如文档编号，使用的文件
- Field names：包含索引中使用的字段名称集合
- Stored Field values：对于每个document，它包含属性-值对的列表，其中属性是字段名称。这些用于存储有关文档的辅助信息，例如其标题、url或访问数据库的标识符
- Term dictionary：包含所有文档的所有索引字段中使用的所有terms的字典。字典还包括包含term的文档编号，以及指向term的频率和接近度的指针
- Term Frequency data：对于字典中的每个term，包含该term的所有文档的数量以及该term在该文档中的频率，除非省略频率（IndexOptions.DOCS）
- Term Proximity data：对于字典中的每个term，term在每个文档中出现的位置。注意，如果所有文档中的所有字段都省略位置数据，则不会存在
- Normalization factors：对于每个文档中的每个字段，存储一个值，该值将乘以该字段上的匹配的分数
- Term Vectors：对于每个文档中的每个字段，可以存储term vector，term vector由term文本和term频率组成
- Per-document values：与存储的值类似，这些也以文档编号作为key，但通常旨在被加载到主存储器中以用于快速访问。存储的值通常用于汇总来自搜索的结果，而每个文档值对于诸如评分因子是有用的
- Live documents：一个可选文件，指示哪些文档是活动的
- Point values：可选的文件对，记录索引字段尺寸，以实现快速数字范围过滤和大数值（例如BigInteger、BigDecimal（1D）、地理形状交集（2D，3D））

#### 文件命名
属于一个段的所有文件具有相同的名称和不同的扩展名。当使用复合索引文件，这些文件（除了段信息文件、锁文件和已删除的文档文件）将压缩成单个.cfs文件。当任何索引文件被保存到目录时，它被赋予一个从未被使用过的文件名字

![复合索引文件](http://image.laijianfeng.org/20180818_161352.png)

#### 文件扩展名摘要

| 名称                    | 文件扩展名      | 简短描述                                     |
| --------------------- | ---------- | ---------------------------------------- |
| Segments File         | segments_N | 保存了一个提交点（a commit point）的信息              |
| Lock File             | write.lock | 防止多个IndexWriter同时写到一份索引文件中               |
| Segment Info          | .si        | 保存了索引段的元数据信息                             |
| Compound File         | .cfs，.cfe  | 一个可选的虚拟文件，把所有索引信息都存储到复合索引文件中             |
| Fields                | .fnm       | 保存fields的相关信息                            |
| Field Index           | .fdx       | 保存指向field data的指针                        |
| Field Data            | .fdt       | 文档存储的字段的值                                |
| Term Dictionary       | .tim       | term词典，存储term信息                          |
| Term Index            | .tip       | 到Term Dictionary的索引                      |
| Frequencies           | .doc       | 由包含每个term以及频率的docs列表组成                   |
| Positions             | .pos       | 存储出现在索引中的term的位置信息                       |
| Payloads              | .pay       | 存储额外的per-position元数据信息，例如字符偏移和用户payloads |
| Norms                 | .nvd，.nvm  | .nvm文件保存索引字段加权因子的元数据，.nvd文件保存索引字段加权数据    |
| Per-Document Values   | .dvd，.dvm  | .dvm文件保存索引文档评分因子的元数据，.dvd文件保存索引文档评分数据    |
| Term Vector Index     | .tvx       | 将偏移存储到文档数据文件中                            |
| Term Vector Documents | .tvd       | 包含有term vectors的每个文档信息                   |
| Term Vector Fields    | .tvf       | 字段级别有关term vectors的信息                    |
| Live Documents        | .liv       | 哪些是有效文件的信息                               |
| Point values          | .dii，.dim  | 保留索引点，如果有的话                              |



#### 锁文件
默认情况下，存储在索引目录中的锁文件名为 `write.lock`。如果锁目录与索引目录不同，则锁文件将命名为“XXXX-write.lock”，其中XXXX是从索引目录的完整路径导出的唯一前缀。此锁文件确保每次只有一个写入程序在修改索引。

> 参考：
>
> 1. [Lucene初识及核心类介绍](http://sndragon.com/2018/05/03/Lucene%E5%88%9D%E8%AF%86%E5%8F%8A%E6%A0%B8%E5%BF%83%E7%B1%BB%E4%BB%8B%E7%BB%8D/)
> 2. [Lucene核心类](http://qguofeng.oschina.io/2018/03/17/Lucene/2018-03-17-Lucene%E6%A0%B8%E5%BF%83%E7%B1%BB/)
> 3. [Lucene 6.0 索引文件格式](http://codepub.cn/2016/12/05/Lucene-6-0-index-file-format/)
> 4. Lucene实战.pdf