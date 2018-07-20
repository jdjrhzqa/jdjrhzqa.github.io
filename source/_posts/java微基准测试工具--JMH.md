---
title: java微基准性能测试工具 -- JMH
categories: 工具学习
author: 郑琳
tags: [microbenchmark,performance,java]
date: 2018-07-20
---

#### JMH简介
发现JMH这个工具源于阅读jdk8教程时看到了一篇名为[Java8 Lambda表达式和流操作如何让你的代码变慢5倍](https://wizardforcel.gitbooks.io/java8-tutorials/content/Java%208%20Lambda%20%E8%A1%A8%E8%BE%BE%E5%BC%8F%E5%92%8C%E6%B5%81%E6%93%8D%E4%BD%9C%E5%A6%82%E4%BD%95%E8%AE%A9%E4%BD%A0%E7%9A%84%E4%BB%A3%E7%A0%81%E5%8F%98%E6%85%A25%E5%80%8D.html)的文章，其在做性能测试时用到了JDK官方的microbenchmark 工具JMH，后来在阅读oracle的[lambda表达式的性能研究报告](http://www.oracle.com/technetwork/java/jvmls2013kuksen-2014088.pdf)时发现其测试代码也用了JMH工具，以此认为，对于测试同学来说，JMH这个工具是很有必要了解的。

JMH 全称为java mircobenchmark harness。microbenchmark代表其测试粒度可以足够小，可能比测试计时代码本身还要小，此时观察者会影响到被观察者，测试统计结果会出现偏差，所以需要专门的工具。JMH的开发者和JIT开发者是同一个群人，其很大可能会与官方JRE的更新同步，也因为作者更了解JVM，其测试结果会更可靠。 

[官方](http://openjdk.java.net/projects/code-tools/jmh/)对于JMH的介绍惜墨如金，简单总结下来就是：
1. 可用于构建、运行及分析不限于java 等基于jvm语言的毫微、微小及大的基准测试。
2. 方便使用maven快速搭建基准测试工程

JMH目前典型使用的场景有：
1. 在定位到热点方法后，为热点方法进行进一步优化时，可以使用JMH进行定量调优分析。
2. 想定量了解某个方法的执行时长，及参数对执行时长的影响
3. 当方法有两种不同实现时，用JMH确定哪种实现性能更优。

从使用场景上来看，JMH对于我们当前性能测试场景直接使用的意义可能不大，但是对性能测试结果分析和调优会有一些帮助。本篇引入这个工具主要有两个目的，其一是了解官方工具，拓宽一下眼界，二则为性能调优做一些知识积累。

#### JMH使用
##### 项目搭建、构建及运行
官方推荐的项目搭建方式是，使用mvn快速搭建一个独立的项目用于基准测试，将应用的jar文件做为依赖引入。这种方法可以更好地保证结果的可靠性。在已有项目或者IDE中运行基准测试也是可行的，但是因启动更加复杂，结果可靠性会降低。

使用JMH的关键在于使用注解或字节码生成器来生成合成的基准代码。maven可以帮助更好的实现这一目的，具体操作步骤如下：

1. 使用maven命令行快速搭建起基准测试工程
```
mvn archetype:generate \
          -DinteractiveMode=false \
          -DarchetypeGroupId=org.openjdk.jmh \
          -DarchetypeArtifactId=jmh-java-benchmark-archetype \
          -DgroupId=${yourself-groupId} \
          -DartifactId=test \
          -Dversion=${whatever-you-want}
```
此处，如果使用了其他jvm 语言，可以将archetypeArtifactId修改成对应语言的，具体参见[list](http://central.maven.org/maven2/org/openjdk/jmh/)

该步骤可以快速搭建使用JMH所需要的所有依赖，并配置好项目的编译打包参数。
对于已有项目，可以参照mvn 原型中的pom文件增加JMH相关依赖和maven-shade-plugin插件来配置JMH工程所需。
```
 <dependency>
            <groupId>org.openjdk.jmh</groupId>
            <artifactId>jmh-core</artifactId>
            <version>${jmh.version}</version>
        </dependency>
        <dependency>
            <groupId>org.openjdk.jmh</groupId>
            <artifactId>jmh-generator-annprocess</artifactId>
            <version>${jmh.version}</version>
            <scope>provided</scope>
        </dependency>
        
    <build>
        <plugins>
         <plugin>
                <groupId>org.apache.maven.plugins</groupId>
                <artifactId>maven-shade-plugin</artifactId>
                <version>2.2</version>
                <executions>
                    <execution>
                        <phase>package</phase>
                        <goals>
                            <goal>shade</goal>
                        </goals>
                        <configuration>
                            <finalName>${uberjar.name}</finalName>
                            <transformers>
                                <transformer implementation="org.apache.maven.plugins.shade.resource.ManifestResourceTransformer">
                                    <mainClass>org.openjdk.jmh.Main</mainClass>
                                </transformer>
                            </transformers>
                            <filters>
                                <filter>
                                    <!--
                                        Shading signed JARs will fail without this.
                                        http://stackoverflow.com/questions/999489/invalid-signature-file-when-attempting-to-run-a-jar
                                    -->
                                    <artifact>*:*</artifact>
                                    <excludes>
                                        <exclude>META-INF/*.SF</exclude>
                                        <exclude>META-INF/*.DSA</exclude>
                                        <exclude>META-INF/*.RSA</exclude>
                                    </excludes>
                                </filter>
                            </filters>
                        </configuration>
                    </execution>
                </executions>
            </plugin>
        </plugins>
    </build>
```


2. 执行基准构建
```
cd /test
mvn clean install
```
该步骤执行mvn install，完成打包并将jar放在本地仓库中。按照pom文件中打包配置，打好的包为可执行jar包。

3. 补充完成基准测试代码，注意代码路径应位于/src/main/java 中, 需要为测试代码添加注解@Benchmark
4. 运行基准测试。命令为
```
java -jar target/benchmarks.jar
```

以上为项目快速搭建、构建及执行所需要的命令。

##### 使用示例和说明
官方提供了很多示例用于说明工具的使用及注解，但是文档方面的内容几乎没有。下面将结合[官网的demo](http://hg.openjdk.java.net/code-tools/jmh/file/3769055ad883/jmh-samples/src/main/java/org/openjdk/jmh/samples/JMHSample_38_PerInvokeSetup.java)和[Java Performance Tuning Guide的文章](http://java-performance.info/jmh/)及其[译文](http://www.importnew.com/12548.html)对工具如何使用和注解做详细说明。
```
/*
 * Copyright (c) 2015, Oracle America, Inc.
 * All rights reserved.
 *
 * Redistribution and use in source and binary forms, with or without
 * modification, are permitted provided that the following conditions are met:
 *
 *  * Redistributions of source code must retain the above copyright notice,
 *    this list of conditions and the following disclaimer.
 *
 *  * Redistributions in binary form must reproduce the above copyright
 *    notice, this list of conditions and the following disclaimer in the
 *    documentation and/or other materials provided with the distribution.
 *
 *  * Neither the name of Oracle nor the names of its contributors may be used
 *    to endorse or promote products derived from this software without
 *    specific prior written permission.
 *
 * THIS SOFTWARE IS PROVIDED BY THE COPYRIGHT HOLDERS AND CONTRIBUTORS "AS IS"
 * AND ANY EXPRESS OR IMPLIED WARRANTIES, INCLUDING, BUT NOT LIMITED TO, THE
 * IMPLIED WARRANTIES OF MERCHANTABILITY AND FITNESS FOR A PARTICULAR PURPOSE
 * ARE DISCLAIMED. IN NO EVENT SHALL THE COPYRIGHT HOLDER OR CONTRIBUTORS BE
 * LIABLE FOR ANY DIRECT, INDIRECT, INCIDENTAL, SPECIAL, EXEMPLARY, OR
 * CONSEQUENTIAL DAMAGES (INCLUDING, BUT NOT LIMITED TO, PROCUREMENT OF
 * SUBSTITUTE GOODS OR SERVICES; LOSS OF USE, DATA, OR PROFITS; OR BUSINESS
 * INTERRUPTION) HOWEVER CAUSED AND ON ANY THEORY OF LIABILITY, WHETHER IN
 * CONTRACT, STRICT LIABILITY, OR TORT (INCLUDING NEGLIGENCE OR OTHERWISE)
 * ARISING IN ANY WAY OUT OF THE USE OF THIS SOFTWARE, EVEN IF ADVISED OF
 * THE POSSIBILITY OF SUCH DAMAGE.
 */
package org.openjdk.jmh.samples;

import org.openjdk.jmh.annotations.*;
import org.openjdk.jmh.runner.Runner;
import org.openjdk.jmh.runner.RunnerException;
import org.openjdk.jmh.runner.options.Options;
import org.openjdk.jmh.runner.options.OptionsBuilder;

import java.util.Arrays;
import java.util.Random;
import java.util.concurrent.TimeUnit;

@BenchmarkMode(Mode.AverageTime)
@OutputTimeUnit(TimeUnit.NANOSECONDS)
@Warmup(iterations = 5, time = 1, timeUnit = TimeUnit.SECONDS)
@Measurement(iterations = 5, time = 1, timeUnit = TimeUnit.SECONDS)
@Fork(5)
public class JMHSample_38_PerInvokeSetup {

    /*
     * This example highlights the usual mistake in non-steady-state benchmarks.
     *
     * Suppose we want to test how long it takes to bubble sort an array. Naively,
     * we could make the test that populates an array with random (unsorted) values,
     * and calls sort on it over and over again:
     */

    private void bubbleSort(byte[] b) {
        boolean changed = true;
        while (changed) {
            changed = false;
            for (int c = 0; c < b.length - 1; c++) {
                if (b[c] > b[c + 1]) {
                    byte t = b[c];
                    b[c] = b[c + 1];
                    b[c + 1] = t;
                    changed = true;
                }
            }
        }
    }

    // Could be an implicit State instead, but we are going to use it
    // as the dependency in one of the tests below
    @State(Scope.Benchmark)
    public static class Data {

        @Param({"1", "16", "256"})
        int count;

        byte[] arr;

        @Setup
        public void setup() {
            arr = new byte[count];
            Random random = new Random(1234);
            random.nextBytes(arr);
        }
    }

    @Benchmark
    public byte[] measureWrong(Data d) {
        bubbleSort(d.arr);
        return d.arr;
    }

    /*
     * The method above is subtly wrong: it sorts the random array on the first invocation
     * only. Every subsequent call will "sort" the already sorted array. With bubble sort,
     * that operation would be significantly faster!
     *
     * This is how we might *try* to measure it right by making a copy in Level.Invocation
     * setup. However, this is susceptible to the problems described in Level.Invocation
     * Javadocs, READ AND UNDERSTAND THOSE DOCS BEFORE USING THIS APPROACH.
     */

    @State(Scope.Thread)
    public static class DataCopy {
        byte[] copy;

        @Setup(Level.Invocation)
        public void setup2(Data d) {
           copy = Arrays.copyOf(d.arr, d.arr.length);
        }
    }

    @Benchmark
    public byte[] measureNeutral(DataCopy d) {
        bubbleSort(d.copy);
        return d.copy;
    }

    /*
     * In an overwhelming majority of cases, the only sensible thing to do is to suck up
     * the per-invocation setup costs into a benchmark itself. This work well in practice,
     * especially when the payload costs dominate the setup costs.
     */

    @Benchmark
    public byte[] measureRight(Data d) {
        byte[] c = Arrays.copyOf(d.arr, d.arr.length);
        bubbleSort(c);
        return c;
    }

    /*
        Benchmark                                   (count)  Mode  Cnt      Score     Error  Units

        JMHSample_38_PerInvokeSetup.measureWrong          1  avgt   25      2.408 Â±   0.011  ns/op
        JMHSample_38_PerInvokeSetup.measureWrong         16  avgt   25      8.286 Â±   0.023  ns/op
        JMHSample_38_PerInvokeSetup.measureWrong        256  avgt   25     73.405 Â±   0.018  ns/op

        JMHSample_38_PerInvokeSetup.measureNeutral        1  avgt   25     15.835 Â±   0.470  ns/op
        JMHSample_38_PerInvokeSetup.measureNeutral       16  avgt   25    112.552 Â±   0.787  ns/op
        JMHSample_38_PerInvokeSetup.measureNeutral      256  avgt   25  58343.848 Â± 991.202  ns/op

        JMHSample_38_PerInvokeSetup.measureRight          1  avgt   25      6.075 Â±   0.018  ns/op
        JMHSample_38_PerInvokeSetup.measureRight         16  avgt   25    102.390 Â±   0.676  ns/op
        JMHSample_38_PerInvokeSetup.measureRight        256  avgt   25  58812.411 Â± 997.951  ns/op

        We can clearly see that "measureWrong" provides a very weird result: it "sorts" way too fast.
        "measureNeutral" is neither good or bad: while it prepares the data for each invocation correctly,
        the timing overheads are clearly visible. These overheads can be overwhelming, depending on
        the thread count and/or OS flavor.
     */

    /*
     * ============================== HOW TO RUN THIS TEST: ====================================
     *
     * You can run this test:
     *
     * a) Via the command line:
     *    $ mvn clean install
     *    $ java -jar target/benchmarks.jar JMHSample_38
     *
     * b) Via the Java API:
     *    (see the JMH homepage for possible caveats when running from IDE:
     *      http://openjdk.java.net/projects/code-tools/jmh/)
     */
    public static void main(String[] args) throws RunnerException {
        Options opt = new OptionsBuilder()
                .include(".*" + JMHSample_38_PerInvokeSetup.class.getSimpleName() + ".*")
                .build();

        new Runner(opt).run();
    }

}

```
上文示例中给出了对于冒泡排序的jmh示例，同时给出了基准测试方法的正确和错误的示范。先通过示例看官方提供注解的含义，后续再解析如何写正确的基准测试。

###### 注解说明

**测试控制注解**

|  名称    |  描述  |
|  ------  | ------ |
|  @Fork   |  需要运行的试验数量，每个实验运行在独立的jvm进程中。可以指定额外的jvm参数 |
|  @Measurement   |  提供真正的测试阶段参数。指定迭代的次数，每次迭代的运行时间和每次迭代测试调用的数量(通常使用@BenchmarkMode(Mode.SingleShotTime)测试一组操作的开销——而不使用循环) |
|  @Warmup     |  与@Measurement相同，但是用于预热阶段 |
|  @Threads  |  该测试使用的线程数。默认是Runtime.getRuntime().availableProcessors() |
|@BenchmarkMode|测试模式，可以理解为测试结果统计维度|
|@OutputTimeUnit|指定测试时间单位|


测试控制注解可以用于class和method级别，method优先级高于class优先级。

**测试模式**

测试方法上@BenchmarkMode注解表示使用特定的测试模式，测试模式此处指测试结果统计的维度。

|  名称    |  描述  |
|  ------  | ------ |
|  Mode.Throughput  |  计算一个时间单位内操作数量 |
|  Mode.AverageTime   |计算平均运行时间   |
|  Mode.SampleTime    |  计算一个方法的运行时间(包括百分位) |
| Mode.SingleShotTime  |  方法仅运行一次(用于冷测试模式)。或者特定批量大小的迭代多次运行（具体查看“`@Measurement`”注解）—这种情况下JMH将计算批处理运行时间(一次批处理所有调用的总时间) |
|任意组合|  该测试使用的线程数。默认是Runtime.getRuntime().availableProcessors() |
| Mode.All |  所有模式 |

**时间单位**

使用@OutputTimeUnit指定时间单位，它需要一个标准Java类型java.util.concurrent.TimeUnit作为参数。可是如果在一个测试中指定了多种测试模式，给定的时间单位将用于所有的测试(比如，测试SampleTime适宜使用纳秒，但是throughput使用更长的时间单位测量更合适)。


**测试参数状态**

测试方法可以接受参数，需要提供单个参数类，这个类必须遵循以下4条原则:
- 有无参构造函数(默认构造函数)
- 是公共类
- 内部类应该是静态的
- 该类必须使用@State注解
@State注解定义了给定类实例的可用范围。JMH可以在多线程同时运行的环境测试，因此需要选择正确的状态。

|  名称    |  描述  |
|  ------  | ------ |
|  Scope.Thread  | 默认状态。实例将分配给运行给定测试的每个线程。 |
|  Scope.Benchmark | 运行相同测试的所有线程将共享实例。可以用来测试状态对象的多线程性能 |
|  Scope.Group  | 实例分配给每个线程组 |
除了单独的类标记@State，也可以将自己的benchmark类标记为@state标记。
@Param 注解用于标记 @State对象中基本数据类型参数，@Param注解使用String数组做为参数，这些字符串在任何@SetUp方法被调用前转换为字段类型。


很多情况下，测试代码包含多个参数集合，要测试不同参数集合时，JMH不需要写多个测试方法，准确来说，测试参数是基本类型，基本包装类型或者String时，JMH提供了解决方法。
程序中需要完成：
1. 定义@State对象
2. 在其中定义所有的参数字段
3. 每个字段都使用@Param注解

JMH使用所有@Param字段的输出结果，如果第一个字段有2个参数，第二个字段有5个参数，测试将运行2 * 5 * Forks次。


**状态设置与清理**

与JUnit测试类似，使用@Setup和@TearDown注解标记状态类的方法(这些方法在JMH文档中称为fixtures)。setup/teardown方法的数量是任意的。这些方法不会影响测试时间(但是Level.Invocation可能影响测量精度)。

@Setup/@TearDown注解使用Level参数来指定何时调用fixture：

|  名称    |  描述  |
|  ------  | ------ |
|  Level.Trial	| 默认level。全部benchmark运行(一组迭代)之前/之后 |
|  Level.Iteration |	一次迭代之前/之后(一组调用) |
| Level.Invocation | 每个方法调用之前/之后(不推荐使用，除非你清楚这样做的目的) |


###### 示例说明
如上示例中，class中声明了5个线程，每个线程迭代运行5次，每次运行一秒钟，取性能测试平均响应时间作为统计维度，在测试执行前预热五次，确保热点方法可以被加载到本地内存中，尽可能接近实际使用场景，输出结果的时间单位为纳秒。

示例中给出了三种基准测试方法，measureWrong使用参数Data，Data参数的scope是整个基准测试阶段，即运行测试的所有线程都使用同一个Data实例，Data中count字段对应三组变量，在setUp时，会将string类型转化为int类型，同时生成对应参数指定长度的随机数组，setUp方法默认为在一组迭代中只初始化一次。在measureWrong方法中会对Data d 做冒泡排序，当第一次冒泡完成后，Data d已经有序的，后续再次执行冒泡排序不会进行位置互换操作，执行速度得到较大提高。这种方法来做基准测试并不能真正反映方法的执行效率。
measureNeutral方法使用参数DataCopy对象，DataCopy对象的范围是线程级别，即每个线程会使用一个DataCopy实例。而其setup级别代表方法在每次方法调用前会执行数组copy工作，即能保证每次调用时数组都与第一次执行的数组排序相同。但是官方文档中不并建议使用Invocation级别，除非很清楚如何使用，虽然setup调用时间不会被计算在方法调用时间中，但方法调用 会有一些开销，可能会影响到基准测试的精度。

measureRight方法依然使用Data对象做为传入参数，只是在每次方法调用中使用传入参数的copy，这样保证方法调用不会影响到传入参数。
官方给出的建议是，在绝大多数情况下，特别是在setUp方法的开销远小于测试方法时，将setUp中的方法中的每次方法调用放在benchmark中是明智的。

从最终测试结果上看，measureWrong方法的结果比较奇怪，排序速度太快。measureNeutral属于中规中矩的情况，虽然在每次方法调用前都能够保证数据正确性，但是开销是很大的，取决于线程数量和操作系统的情况。与上文分析相同。

##### 使用注意事项
对于JMH的使用，除上述内容外，还有以下建议：
1. 不要在测试中使用循环。JIT非常聪明，在循环中经常出现不可预料的处理。要测试真实的计算，让JMH处理剩余的部分。

    在非统一开销操作情况下(比如测试处理列表的时间，这个列表在每个测试后有所增加)，你可能使用@BenchmarkMode(Mode.SingleShotTime) 和@Measurement(batchSize = N)。但是不允许你自己实现测试的循环

2. 默认JMH为每个试验(迭代集合)fork一个新的java进程。这样可以防止前面收集的“资料”——其他被加载类以及它们执行的信息对当前测试的影响。比如，实现了相同接口的两个类，测试它们的性能，那么第一个实现(目标测试类)可能比第二个快，因为JIT发现第二个实现类后就把第一个实现的直接方法调用替换为接口方法调用。因此，不要把forks设为0，除非你清楚这样做的目的。极少数情况下需要指定JVM分支数量时，使用@Fork对方法注解，就可以设置分支数量，预热(warmup)迭代数量和JVM分支的其他参数。

    可能通过JMH API调用来指定JVM分支参数也有优势——可以使用一些JVM -XX:参数，通过JMH API访问不到它。这样就可以根据你的代码自动选择最佳的JVM设置(new Runner(opt).run()以简便的形式返回了所有的测试结果)。

3. 测试前需要预热，尽量贴近实际使用场景。

##### 其他

- IntelliJ 有 [JMH 的插件](https://github.com/artyushov/idea-jmh-plugin)，让jmh的使用和junit一样方便。具体使用方法参看github上相关文档。

- 在本篇资料收集过程中，发现了国外一些不错的站点，对于java知识学习和性能方面知识积累会有一些帮助，如下：

  [jaxenter](https://jaxenter.com/) java相关资讯和技术

  [java performance](http://java-performance.info) 关注java性能

  [jenkov的教程](http://tutorials.jenkov.com/)   java教程

#### 总结
本篇对JMH基准测试工具做了简单介绍，主要关注使用JMH工具的项目搭建、运行和工具关键注解的使用。未尽事项，请参考[官方示例](http://hg.openjdk.java.net/code-tools/jmh/file/tip/jmh-samples/src/main/java/org/openjdk/jmh/samples/)。
