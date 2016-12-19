# Quick Spark Lab to process Wikipedia datasets:
## This lab is part of what was taught in Spark Summit, I reapplied it in Apache Zeppelin as a self exercise and I'm sharing my experience with the community.

## 1- The Dataset:
Part of the data is uploaded here in the repo, the rest (and full set) can be found here:
- [Page counts](https://dumps.wikimedia.org/other/pagecounts-raw/)
- [Page views](https://datahub.io/en/dataset/english-wikipedia-pageviews-by-second)
- [Click stream](https://datahub.io/dataset/wikipedia-clickstream)
- [English Wikipedia](https://meta.wikimedia.org/wiki/Data_dump_torrents)
- [Live edits stream for other Languages](http://wikistream.wmflabs.org/)

## 2- Requirements
I used the following, which you can also use, or feel free to use whatever you like:
- IntelliJ
- Hortonworks Data Platform HDP (2.5 or 2.4)
- Apache Spark 1.6.2 (as part of HDP)
- Zeppelin notebook (as part of HDP)
- Download the data and push it to HDFS on your cluster/docker/VM

### 2.1- How did I setup my IntelliJ?
If you have installed the HDInsight & Azure plugins for IntelliJ then create a HDInsight scala project, otherwise just simply create a regular Scala project.

Right click your project, choose **Add Framework Support..**

Add Maven support (or SBT if you want!), then all you need to do is to add the Spark dependencies that you need, for example, I used the following in my maven pom.xml:
```xml
    <dependencies>
        <dependency>
            <groupId>org.apache.spark</groupId>
            <artifactId>spark-core_2.10</artifactId>
            <version>1.6.2</version>
        </dependency>
        <dependency>
            <groupId>org.apache.spark</groupId>
            <artifactId>spark-sql_2.10</artifactId>
            <version>1.6.2</version>
        </dependency>
        <dependency>
            <groupId>org.apache.spark</groupId>
            <artifactId>spark-streaming_2.10</artifactId>
            <version>1.6.2</version>
        </dependency>
        <dependency>
            <groupId>org.apache.spark</groupId>
            <artifactId>spark-mllib_2.10</artifactId>
            <version>1.6.2</version>
        </dependency>
        <dependency>
            <groupId>com.databricks</groupId>
            <artifactId>spark-csv_2.10</artifactId>
            <version>1.5.0</version>
        </dependency>
    </dependencies>
```

## 3- Running the Labs
### Using Zeppelin
  Import the notebook and run it step by step.
### Using IntelliJ
  The code is below has explainatory commments if you want to run it in IntelliJ:
```scala
import org.apache.spark.sql.{DataFrame, SQLContext}
import org.apache.spark.{SparkConf, SparkContext}
import org.apache.spark.sql.functions._
import org.apache.spark.sql.types._

/**
  * Created by ahmedmos on 18-12-2016.
  */
object WikiExplorer {
	setLoggingOff
	
	val PARQUET_DATA = "C:\\Users\\ahmedmos\\Downloads\\wikipedia_pagecounts_single\\pagecounts-20160801-000000.parquet"
	val PAGEVIEWS_PER_SECOND = "C:\\Users\\ahmedmos\\Downloads\\wikipedia_pagecounts_single\\pageviews-by-second-tsv.gz"
	val CLICKSTREAMS_DATA = "C:\\Users\\ahmedmos\\Downloads\\wikipedia_pagecounts_single\\clickstreams\\"
	val GRAPH_DIR = "C:\\Users\\ahmedmos\\Downloads\\wikipedia_pagecounts_single\\clickstreams\\graph\\"
	
	val sparkContext = new SparkContext("local[*]", "WikiExplorer")
	
	val sqlContext = new SQLContext(sparkContext)
	
	import sqlContext.implicits._
	
	def setLoggingOff: Unit = {
		import org.apache.log4j.{Level, Logger}
		
		Logger.getLogger(this.getClass).getLevel
		Logger.getLogger("org").setLevel(Level.OFF)
		Logger.getLogger("akka").setLevel(Level.OFF)
	}
	
	def loadParquetData(): DataFrame = {
		val data = sqlContext.read.parquet(PARQUET_DATA)
		data
	}
	
	def loadParquetData(path: String): DataFrame = {
		val data = sqlContext.read.parquet(path)
		data
	}
	
	def main(args: Array[String]): Unit = {
		wikipediaETL()
		
		wikipediaSQL()
	}
	
	def wikipediaETL() = {
		val parquetData = loadParquetData()
		println("parquet data subset:")
		parquetData.show(10, truncate = false)
		
		println("parequet data schema:")
		parquetData.printSchema()
		
		println(s"parquetData count: ${parquetData.count()}")
		
		println("parquetData ordered desc by requests:")
		parquetData.orderBy($"requests".desc).show(20, truncate = false)
		println("note that we imported sqlContext.implicits to make it work!")
		
		println(s"available wikipedia projects:")
		parquetData.select($"project").distinct().show()
		
		println("How much traffic did each project get?")
		parquetData.select($"project", $"requests").groupBy($"project").sum().orderBy($"sum(requests)".desc).show()
		
		println("selecting only the English Wikipedia")
		val pagecountsEnWikipediaDF = parquetData.filter($"project" === "en")
		
		println("What are 25 most popular articles on English wikipedia?")
		pagecountsEnWikipediaDF.orderBy($"requests".desc).show(25, truncate = false)
		
		println("Removing Wikipedia Namespaces")
		val pagecountsEnWikipediaArticlesOnlyDF = pagecountsEnWikipediaDF
		                                          .filter($"item".rlike("""^((?!Special:)+)"""))
		                                          .filter($"item".rlike("""^((?!File:)+)"""))
		                                          .filter($"item".rlike("""^((?!Category:)+)"""))
		                                          .filter($"item".rlike("""^((?!User:)+)"""))
		                                          .filter($"item".rlike("""^((?!Talk:)+)"""))
		                                          .filter($"item".rlike("""^((?!Template:)+)"""))
		                                          .filter($"item".rlike("""^((?!Help:)+)"""))
		                                          .filter($"item".rlike("""^((?!Wikipedia:)+)"""))
		                                          .filter($"item".rlike("""^((?!MediaWiki:)+)"""))
		                                          .filter($"item".rlike("""^((?!Portal:)+)"""))
		                                          .filter($"item".rlike("""^((?!Book:)+)"""))
		                                          .filter($"item".rlike("""^((?!Draft:)+)"""))
		                                          .filter($"item".rlike("""^((?!Education_Program:)+)"""))
		                                          .filter($"item".rlike("""^((?!TimedText:)+)"""))
		                                          .filter($"item".rlike("""^((?!Module:)+)"""))
		                                          .filter($"item".rlike("""^((?!Topic:)+)"""))
		                                          .filter($"item".rlike("""^((?!Images/)+)"""))
		                                          .filter($"item".rlike("""^((?!%22//upload.wikimedia.org)+)"""))
		                                          .filter($"item".rlike("""^((?!%22//en.wikipedia.org)+)"""))
		pagecountsEnWikipediaArticlesOnlyDF.orderBy($"requests".desc).show(25, truncate = false)
	}
	
	def wikipediaSQL() = {
		val schema = StructType(
			                       List(
				                           StructField("timestamp", StringType, true),
				                           StructField("site", StringType, true),
				                           StructField("requests", IntegerType, true)
			                           )
		                       )
		
		val df = sqlContext.read
		         .format("com.databricks.spark.csv")
		         .option("delimiter", "\t")
		         .option("header", "true")
		         .schema(schema)
		         .load(PAGEVIEWS_PER_SECOND)
		         .select($"site", $"requests",
		                 (
			                 unix_timestamp($"timestamp", "yyyy-MM-dd'T'HH:mm:ss")
			                 ).cast("timestamp").as("timestamp"))
		df.show(truncate = false)
		
		println("reorder the data:")
		val pageviewsDF = df.orderBy($"timestamp", $"site".desc)
		pageviewsDF.show(truncate = false)
		
		println("register data as a temp table")
		pageviewsDF.registerTempTable("pageviews_by_second_ordered")
		sqlContext.cacheTable("pageviews_by_second_ordered")
		
		println("materializing the data using 'count' action..")
		pageviewsDF.count // materialize the cache
		
		println("how many total requests to mobile vs desktop?")
		println("Mobile:")
		pageviewsDF.filter($"site" === "mobile").select(sum($"requests")).show
		println("Desktop:")
		pageviewsDF.filter($"site" === "desktop").select(sum($"requests")).show
		
		println("change timestamp datatype from String to Timestamp")
		val pageviewsTimestampDF = pageviewsDF.select($"timestamp".cast("timestamp"), $"site", $"requests")
		pageviewsTimestampDF.printSchema()
		pageviewsTimestampDF.show(10, truncate = false)
		
		println("Using SQL API you can use timestamp-specific functions")
		pageviewsTimestampDF.select(year($"timestamp")).distinct().show()
		pageviewsTimestampDF.select(weekofyear($"timestamp")).distinct().show()
		
		println("What is the avg/min/max number of requests received for Mobile vs Desktop views?")
		println("Mobile:")
		pageviewsTimestampDF.filter("site = 'mobile'").select(avg($"requests"), min($"requests"), max($"requests")).show()
		println("Desktop:")
		pageviewsTimestampDF.filter("site = 'desktop'").select(avg($"requests"), min($"requests"), max($"requests")).show()
		
		println("Which day of the week does Wikipedia gets the most traffic?")
		val pageviewsByDayOfWeekDF = pageviewsTimestampDF.groupBy(date_format(($"timestamp"), "E").alias("Day of week")).sum()
		pageviewsByDayOfWeekDF.registerTempTable("pageviews_by_DOW")
		sqlContext.cacheTable("pageviews_by_DOW")
		pageviewsByDayOfWeekDF.show()
		
		println("User Defined Functions:")
		def matchDayOfWeek(day: String): String = {
			day match {
				case "Mon" => "1-Mon"
				case "Tue" => "2-Tue"
				case "Wed" => "3-Wed"
				case "Thu" => "4-Thu"
				case "Fri" => "5-Fri"
				case "Sat" => "6-Sat"
				case "Sun" => "7-Sun"
				case "ma" => "1-Mon"
				case "di" => "2-Tue"
				case "wo" => "3-Wed"
				case "do" => "4-Thu"
				case "vr" => "5-Fri"
				case "za" => "6-Sat"
				case "zo" => "7-Sun"
				case _ => "UNKNOWN"
			}
		}
		
		matchDayOfWeek("Tue")
		
		val prependNumberUDF = sqlContext.udf.register("prependNumber", (s: String) => matchDayOfWeek(s))
		pageviewsByDayOfWeekDF.select(prependNumberUDF($"Day of week")).show(7)
		
		pageviewsByDayOfWeekDF.withColumnRenamed("sum(requests)", "total requests")
		.select(prependNumberUDF($"Day of week"), $"total requests")
		.orderBy("UDF(Day of week)").show()
		
		//		val mobileViewsByDayOfWeekDF = pageviewsTimestampDF.filter("site = 'mobile'").
		//		                               groupBy(date_format(($"timestamp"), "E").alias("Day of week")).sum().
		//		                               withColumnRenamed("sum(requests)", "total requests").
		//		                               select(prependNumberUDF($"Day of week"), $"total requests").
		//		                               orderBy("UDF(Day of week)").
		//		                               toDF("DOW", "mobile_requests")
		
		val mobileDF = pageviewsTimestampDF.filter("site = 'mobile'")
		val groupedData = mobileDF.groupBy(date_format(($"timestamp"), "E").alias("Day of week"))
		val sumDF = groupedData.sum().withColumnRenamed("sum(requests)", "total requests")
		val selectDF = sumDF.select(prependNumberUDF($"Day of week"), $"total requests")
		val orderedDF = selectDF.orderBy("UDF(Day of week)")
		val mobileViewsByDayOfWeekDF = orderedDF.toDF("DOW", "mobile_requests")
		
		// Cache this DataFrame
		mobileViewsByDayOfWeekDF.cache()
		mobileViewsByDayOfWeekDF.show(truncate = false)
		
		val desktopViewsByDayOfWeekDF = pageviewsTimestampDF
		                                .filter("site = 'desktop'")
		                                .groupBy(date_format(($"timestamp"), "E").alias("Day of week")).sum().withColumnRenamed("sum(requests)", "total requests")
		                                .select(prependNumberUDF($"Day of week"), $"total requests")
		                                .orderBy("UDF(Day of week)")
		                                .toDF("DOW", "desktop_requests")
		
		// Cache this DataFrame
		desktopViewsByDayOfWeekDF.cache()
		desktopViewsByDayOfWeekDF.show(truncate = false)
		
		val allViewsByDayOfWeekDF = mobileViewsByDayOfWeekDF
		                            .join(desktopViewsByDayOfWeekDF,
		                                  mobileViewsByDayOfWeekDF("DOW") === desktopViewsByDayOfWeekDF("DOW"))
		
		allViewsByDayOfWeekDF.show(truncate = false)
		
		
		println("check catalyst optimizer explanation")
		allViewsByDayOfWeekDF.explain()
	}
}
```
