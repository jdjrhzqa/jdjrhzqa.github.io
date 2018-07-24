---
title: Jenkins Pipeline实践分享
categories: 工具学习
author: Mota
tags: [CI,Jenkins]
date: 2018-07-27
---

#### 前言
本文为实践Jenkins Pipeline后的分享，介绍一些本人感受到的pipeline优势、创建pipeline项目的步骤以及调试运行的方法，适合建立过Jenkins项目但是没有接触过其pipeline功能的人群参考.

#### Jenkins Pipeline介绍
Jenkins Pipeline是一系列支持执行且集成`持续发布pipeline`到Jenkins的插件组成的子系统.
`持续发布pipeline`指从版本控制系统中获得软件到交付给用户中间自动化的流程.每次软件更新在发布前都要经过编译打包以及多个测试和开发的场景.
Jenkins Pipeline提供一组可扩展的工具通过[Pipeline专用语言DSL](https://jenkins.io/doc/book/pipeline/syntax)对简单到复杂的发布Pipeline流程建模实现.
Jenkins Pipeline定义成文本文件`Jenkinsfile`，提交到源码仓库中进行版本维护及审查.
使用pipeline，用户可以获得以下优势：
- 代码：Pipeline以代码的形式实现，通常被检入源代码控制，使团队能够编辑，审查和迭代其传送流程。
- 持久化：Pipeline可以在计划和计划外重新启动Jenkins管理时同时存在。
- 可暂停(交互)：Pipeline可以选择停止并等待人工输入或批准，然后再继续Pipeline运行。
- 多功能：Pipeline支持复杂的现实世界连续交付要求，包括并行分叉/连接，循环和执行工作的能力。
- 可扩展：Pipeline插件支持其DSL的自定义扩展以及与其他插件集成的多个选项。

#### Jenkins自由风格项目
先简要介绍下之前未使用pipeline时建立的Jenkins项目以做对比.
_新建任务->构建一个自由风格的软件项目_ 进入项目配置如图

![自由风格项目配置](/images/Jenkins_Pipeline/freeStyleProjectConf.png)

在项目配置里主要有`构建`和`构建后操作`模块需要编辑，在本项目中，将参数处理/编译/代码检查/单元测试/集成测试/覆盖率生成的执行均写入shell脚本，通过`构建`中添加`Execute shell`步骤执行. 另外在`build.sh`后面的执行参数是在`General`模块中`参数化构建过程`中配置.
`构建后操作`模块中需添加`Publish Cppcheck results`步骤处理`cppcheckreport.xml`生成静态代码检查报告，添加`Publish Junit test result report`步骤处理`testreport.xml`生成用例运行测试报告，添加`Publish HTML reports`步骤处理`lcov-html`目录生成覆盖率报告等.这些文件及目录均为执行shell后生成. 

![自由风格项目构建](/images/Jenkins_Pipeline/freeStyleProjectBuild.png)

除了这两个模块外，还需要在`源码管理`模块中填写版本控制软件类型/仓库地址/用户证书/分支号等.

#### Jenkins Pipeline项目
_新建任务->流水线_，进入项目配置如图

![Pipeline项目配置](/images/Jenkins_Pipeline/pipelineProjectConf.png)

主要模块`流水线`如图

![Pipeline项目模块](/images/Jenkins_Pipeline/pipelineProjectModule.png)

定义中选择`Pipeline script from SCM`，表示Pipeline脚本使用SCM仓库中的Jenkinsfile，如果选择选项`Pipeline script`则会出现`脚本`的文本编辑框，直接在页面上进行编辑运行.

__该项目中编写的Jenkinsfile (Declarative Pipeline)__

	pipeline {
	  agent any//指定在哪台机器上执行任务，配置Node时指定的标签名，any表示任一节点，如果出现在stage内则指定该stage任务在哪台机器执行
	  stages {
		stage('Checkout') { //定义一个下拉代码的任务
		  steps {//执行该任务的各个步骤
			checkout([$class: 'GitSCM', branches: [[name: '$Branch']], doGenerateSubmoduleConfigurations: false, 
            extensions: [], submoduleCfg: [], userRemoteConfigs: [[url: '$git_root']]]) 
		  }
		}

		stage('Pre process') {
		  steps {
			dir(path: 'jenkins/build') {//切换到某目录下执行，执行完steps会回退到原来所在目录
	...        }
			script {
	...          REPORT="jenkins/report"  //定义工具和报告的路径变量
			}
		}

		stage('Quality Analysis') {
		  parallel {// 并行运行任务
			stage('Static Analysis') {
	...  }
			stage('Unit Testing') {
			  when {//条件判断是否执行该stage，return为真时执行
				expression {
				  return params.type == 'unit' || params.type == 'all'
				}
			  }
			  steps {
				  sh "./build/${params.Target} --gtest_filter=${params.uargs} --gtest_output=
                  xml:../../../../${UNIT_REPORT}/testreport.xml" //运行gtest生成测试报告
			}
		  }
		}

		stage('Code Building') {
	...    }

		stage('Integration Testing') {
	...     }

		stage('Coverage HTML Report') {
		 ...
		  parallel {
			stage('Integration Part') {
			  ...             }

			stage ('Unit Part') {
			  ...        }
		  }
		}

		stage('Coverage Report') {
		  ...      }

		stage('Deploy') {
		  steps {
			input "Can be deploy into product environment?"//等待手动输入确认
			echo 'TO DO Deploy'
		  }
		}
	  }

	  post {//后处理模块,根据pipeline任务执行结果进入不同分支处理,可将该模块定义在stage内单独处理其结果
		aborted {//任务中断，Web界面上该任务为灰色的点
		  echo 'Pipeline WAS ABORTED'
		}

		always {//始终运行分支
		  echo 'Pipeline finished'
	...    }

		changed {//任务状态与上一次不一样
		  echo 'Stage HAVE CHANGED'
		}

		failure {//任务失败，Web界面上该任务为红色的点
		  echo 'Stage FAILED'
		}

		success {//任务成功，Web界面上该任务为蓝色的点
		  echo 'Stage was Successful'
		}
	  }

	  options {
		buildDiscarder(logRotator(numToKeepStr: '10'))//可选项，表示保留最多10个构建日志.
	  }
	  parameters {//可配置任务参数，运行时可在Web界面上配置不同参数
		string(name: 'git_root', defaultValue: 'http://gitlab.cbpmgt.com/test.git', description: 'git代码路径')
	  ...    string(name: 'Tool_Path', defaultValue: '/home/jenkins/toolkit', description: 'Toolkit路径')
	  }
	}

#### 项目对比
1. Pipeline模式把自由风格模式项目中除了`构建触发器`外的所有模块都编码到Jenkinsfile文件中，发布配置和代码绑定在一起在SCM进行审查维护，大大增加其可移植性.
2. Pipeline项目执行后可在项目首页形象地看到每个stage执行状态及运行时间.
![Pipeline项目Stages](/images/Jenkins_Pipeline/pipelineStages.png)
3. Pipeline日志系统简洁，当某次任务失败，需要找到其失败位置，在自由风格项目中需要点击到该任务的`控制台输出`，在任务执行的所有日志中查找.而Pipeline项目可直接在项目首页点击任务失败的stage看到原因.
![Pipeline项目日志](/images/Jenkins_Pipeline/pipelineLogs.png)
4. Pipeline通过`input`命令可实现人工介入该过程，更加可靠地判断是否需进行如发布到生产环境的后续步骤.
5. Pipeline通过`parallel`命令可便捷实现并行任务，加快交付验证的速度.

#### Pipeline开发工具
##### 命令行Pipeline Linter
Jenkins可以在实际运行之前从命令行[Jenkins CLI](https://www.w3cschool.cn/jenkins/jenkins-zh3e28mj.html)验证[声明式Pipeline](https://www.w3cschool.cn/jenkins/jenkins-jg9528pb.html)，本项目用SSH接口运行linter.
默认Jenkins的ssh接口是禁用的，需要在Jenkins管理页面上开启: 以管理员身份进入全局安全配置，在`SSH Server`下选择`随机选取`即可开启.
验证ssh接口是否开启，在需要验证Jenkinsfile的机器A上执行命令获得ssh接口的ip和端口

	$ curl -Lv http://$JENKINSURL/login 2>&1 | grep 'SSH'
	< X-SSH-Endpoint: 10.9.0.149:36340

则ssh端口为36340.
下一步需配置认证: 将机器A上公钥内容拷贝到`https://$JENKINSURL/user/$USERNAME/configure`页面上的`SSH Public Keys`框体中，点击`Apply`按钮完成.
验证密钥认证是否配置成功:

	$ ssh -v -l $USERNAME -p 36340 10.9.0.149
	...Authentication succeeded (publickey)..

即认证成功. 执行以下命令验证Jenkinsfile语法是否正确

	$ ssh -l $USERNAME -p 36340  10.9.0.149 declarative-linter < Jenkinsfile
	Jenkinsfile successfully validated.

##### Replay调试运行
当提交的Jenkinsfile执行后出错或者有其他需要更新时，可以直接使用其`Replay`功能基于上一次直接在页面上修改. 
点击上次运行的任务，左侧点击`Replay`，显示如下.

![Pipeline项目Replay](/images/Jenkins_Pipeline/pipelineReplay.png)

该功能方便快速更新当前的Pipeline过程，在未最终完成修改时无需对Jenkinsfile进行新的修改提交，最终完成后复制`Main Script`窗体内容到Jenkinsfile正常提交即可.

##### Blue Ocean插件
在Pipeline实践中了解到其相关的插件`Blue Ocean`，该插件使Pipeline实现及运行的流程更加的可视化.
_系统管理->管理插件->可选插件_ 中选择`Blue Ocean`进行安装
安装后在自己工程中打开`Open Blue Ocean`并进入Blue Ocean界面，某次执行任务后视图

![Blue Ocean视图](/images/Jenkins_Pipeline/blueOceanView.png)

视图中显示的之前编写Jenkinsfile文件运行的Pipeline项目各个步骤的运行路径.
Blue Ocean界面下可直接创建Pipeline项目.详见[BlueOcean 创建Pipeline](https://www.w3cschool.cn/jenkins/jenkins-nqkh28q1.html)

##### Pipeline Syntax
除了上面的工具外，pipeline项目视图的左侧还有个`Pipeline Syntax`模块，里面可以在页面上填写便捷地生成很多步骤的脚本.

#### 总结
本文对比了`自由风格项目`与`Pipeline项目`的区别，重点突出了后者可视化及代码配置可维护的特点，展示pipeline项目的执行效果，最后简要介绍实践中用到的几个开发工具Blue Ocean及Replay等.

#### 参考
[Jenkins Pipeline官网介绍](https://jenkins.io/doc/book/pipeline/)
[W3C school Pipeline 介绍](https://www.w3cschool.cn/jenkins/jenkins-epas28oi.html)
[BlueOcean介绍](https://www.w3cschool.cn/jenkins/jenkins-627f28pt.html)

