# 在数据库配置表依赖关系Demo

###  需求背景

airflow的task的依赖关系需要在python脚本中配置,一旦任务比较多,且多人开发的话容易造成混乱,所以现在希望能在通过在mysql中建依赖表,通过配置mysql中的依赖表来设置表依赖关系.

### 方案

因为现在的任务类型都是shell脚本形式,每个task需要改到的地方也就是 kettle Script/HQL Script/Spark Script 的名称.所以可以在mysql建任务表将任务写入表中,同时通过字段INDXE来标示任务.

`任务表`

> `id`, `INDEX`, `TASK`, `AUTHOR`, `CREATE_TIME`, `UPDATE_TIME`

| INDEX | TASK |
|--------|--------|
|     ods_asset_pay_detail   |   BashOperator(task_id='ods_asset_pay_detail',bash_command='sh /home/app/airflow/bash/test.sh public ods_asset_pay_detail.ktr ',dag=dag)     |
| 		ods_asset_repay_plan  | BashOperator(task_id='ods_asset_repay_plan',bash_command='sh /home/app/airflow/bash/test.sh public ods_asset_repay_plan.ktr ',dag=dag)|
|ods_asset_repay_serial|BashOperator(task_id='ods_asset_repay_serial',bash_command='sh /home/app/airflow/bash/test.sh public ods_asset_repay_serial.ktr ',dag=dag)|
|ods_asset_repay_plan2|BashOperator(task_id='ods_asset_repay_plan2',bash_command='sh /home/app/airflow/bash/test.sh public ods_asset_repay_plan2.ktr ',dag=dag)|
|repay_plan|BashOperator(task_id='repay_plan',bash_command='sh /home/app/airflow/bash/test.sh public repay_plan.ktr ',dag=dag)|
|dim_product|BashOperator(task_id='dim_product',bash_command='sh /home/app/airflow/bash/test.sh public dim_product.ktr ',dag=dag)|


`依赖表`

| PARENT | CHILD |
|--------|--------|
|   ods\_asset\_repay\_plan     |   ods\_asset\_repay\_plan2     |
|   ods\_asset\_repay\_serial   |ods\_asset\_repay\_plan2		   |
|ods\_asset\_pay\_detail		 |ods\_asset\_repay\_plan2		   |
|dim\_product				 |ods\_asset\_repay\_plan2        |
|ods\_asset\_repay\_plan2		 |repay\_plan                   |


`生成DAG脚本`

```python

#!/usr/bin/env python
# -*- coding: utf-8 -*-

from  airflow import  DAG
from  airflow.operators.bash_operator import BashOperator
from  datetime import  datetime ,timedelta
from airflow.utils.dates import  days_ago
from  airflow.utils.dates import datetime
from  airflow.settings import Session
from  airflow.models import Variable
import  pandas as pd
from  sqlalchemy import  create_engine
import pytz



path = [Variable.get('shell_path'),Variable.get('sql_path')]

tz = pytz.timezone('Asia/Shanghai')

dt = datetime(2018,12, 10, 8, 6, tzinfo=tz)

utc_dt = dt.astimezone(pytz.utc).replace(tzinfo=None)

default_args = {
    'owner': 'airflow',
    'depends_on_past': False,
    'email_on_failure': False,
    'email_on_retry': False,
    'retries': 2,
	'email': ['a1@airflow.com','a2@airflow.com'], # 报警时发送到哪个邮箱，可以填多个
    'retry_delay': timedelta(minutes=5),
    'start_date':utc_dt,
	 # 'queue': 'bash_queue', 队列，默认是default，决定实际执行任务会发送到哪个worker
 	# 'pool': 'backfill', pool是一个分类限制并发量的设计，目前来说可以忽略，默认所有的Task都在一个pool里。
	 # 'priority_weight': 10, 优先级权重，在任务需要排队时而你需要优先执行某些任务时会有用
}

dag = DAG(
    'task_from_mysql',
    default_args=default_args,
    description='my DAG',
    schedule_interval='20 05 * * *',
    template_searchpath=path
)

engine  = create_engine('mysql+pymysql://username:password@WSX@192.192.0.xxx/WKL')

# 任务表
task_table = pd.read_sql_table(table_name='TASK_TABLE',con=engine)

#依赖表
parent_table = pd.read_sql_table(table_name='PARENT_TABLE',con=engine)

# 通过local() 函数动态生成对象
for index,task in zip(task_table['INDEX'],task_table['TASK']):
    locals()[index] = eval(task)

dp_data = parent_table.to_dict('split')['data']

# 设置依赖
for a,b in dp_data:
    locals()[a] >> locals()[b]


```
### 调试

在编写完成之后就进入调试阶段，这一阶段主要的目的就是DeBug，步骤如下： 
1. 语法检查：用Python命令执行该脚本，确保没有语法错误。 
2. 上传到服务器：将脚本上传至服务器，使用airflow list_dags | grep your_dag_name（你的dag的名字，比如上面的例子里，就是tutorial)来检查Airflow是否识别出了你的dag。一般来说，只要第一个Python运行没问题，就能识别出来。 
3. DAG测试：在服务器上输入命令 airflow backfill your_dag_name -s '2018-03-06' -e '2018-03-07' -l。backfill命令本身是用于回填的，-s表示开始时间，-e表示结束时间(到该时间为止，比如在这条语句中，如果DAG的执行间隔为1天，则只会生成03-06的DAG Run，不会生成03-07的），使用该命令可以回填多个DAG Run，按当前语句设置则只会生成一个，适合用于测试整个DAG的运行状况，最后有个-l参数，表示些任务在本地机器上执行（在部署好的集群上做测试时，这样做可以仅仅更新用于测试服务器上的DAG文件，直到测试完成之后再更新worker服务器上的DAG文件）。 
4. Task Debug:如果在3中的测试出了问题，系统会提示是哪个Task出了问题，这时候首先修改代码，然后重复1,2的操作，再执行airflow test your_dag_name bug_task_id 2018-03-06，单独测试这个Task的执行情况。依此法修复所有有问题的task。然后再执行命令airflow clear your_dag_id，清理干净所有的Task，最后重复3的操作，直到能成功运行完所有的task。整个调试阶段就算完成了。

### 部署

当完成了调试之后，就可以把代码部署到线上了。 
Airflow的生产环境一般使用Celery Worker来执行任务，可以搞出分布式的worker，方便在任务量增加之后通过增加机器的方式来保证计算的效率。相应的，这就要求所有机器上的DAG脚本都是一致的。因此在调试完成之后，就需要把所有Worker上的DAG脚本都更新好。最后在Scheduler所在的服务器，执行airflow unpause your_dag_id，激活这个DAG。然后在Airflow Web UI上检查这个DAG是否已经在激活列表里，是否已经在回填DAG Run（在激活之后会自动回填，比如设定DAG 的start_date为2018-03-05，在2018-03-07上线激活的话，会自动回填2018-03-05和2018-03-06两天的DAG Run）

