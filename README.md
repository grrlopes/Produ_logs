# Produ_logs(título tempo)

O objetivo dessa documentação é descrever todas as tecnologias que foram utilizadas para criar o ambiente de coleta, provisionamento dos logs.

<p align="center">
  <img src="https://blog.netapsys.fr/wp-content/uploads/2016/05/ELASTIC_LOGSTASH_KIBANA.jpg">
</p>
--

* [Objetivo do projeto](#objetivo-do-projeto)
* [Tecnologia usada](#tecnologia-usada)
* [Download Instalação](#download-instalação)
* [Configuração do Elasticsearch](#configuração-do-elasticsearch)
* [Configuração do Kibana](#configuração-do-Kibana)
* [Configuração do Mongodb](#Configuração-do-mongodb)
* [Configuração do logstash](#Configuração-do-logstash)
* [Modelo conector](#modelo-conector)
* [Topologia](#Topologia)

## Objetivo do projeto

Desenvolver um ambiente que capture, filtre e que persista as informação de logs gerada pelo lançador rundeck / ansible.

## tecnologia usada

* **Elasticsearch** --> Baseado no Apache Lucene, um servidor de busca e indexação textual, o objetivo do Elasticsearch 
    é fornecer um método de se catalogar e efetuar buscas em grandes massas de informação por meio de interfaces REST 
    que recebem/provêm informações em formato JSON.

* **Logstash** --> Criado pela Elastic, o conceito do Logstash é fornecer pipelines de dados, através do qual podemos 
    suprir as informações contidas nos arquivos de logs das nossas aplicações – além de outras fontes – para diversos 
    destinos, como uma instância de Elasticsearch, um bucket S3 na Amazon, um banco de dados MongoDB, entre outros

* **Mongodb** --> O MongoDB é um document database(banco de dados de documentos), mas não são os documentos da 
    família Microsoft, mas documentos com informações no formato JSON. A ideia é o documento representar toda a 
    informação necessária, sem a restrição dos bancos relacionais

* **Grafana** --> Ferramentas web para criação e exibição de gráficos.

* **Kibana** --> O Kibana foi desenvolvido pela Elastic com o intuito de fornecer uma interface rica que permita 
    consultas analíticas e/ou a construção de dashboards, com base nas informações contidas dentro de um Elasticsearch.

* **

## Download / Instalação

* **Elasticsearch** 
    
    1. https://artifacts.elastic.co/downloads/elasticsearch/elasticsearch-5.5.1.tar.gz
    2. tar -zxvf elasticsearch-5.5.1.tar.gz

* **Logstash**

    1. https://artifacts.elastic.co/downloads/logstash/logstash-5.5.1.tar.gz
    2. tar -zxvf logstash-5.5.1.tar.gz

* **Mongodb**

    1. https://fastdl.mongodb.org/linux/mongodb-linux-x86_64-amazon-3.4.6.tar.gz
    2. tar -zxvf mongodb-linux-x86_64-amazon-3.4.6.tar.gz

* **Grafana**

    1. https://s3-us-west-2.amazonaws.com/grafana-releases/release/grafana-4.4.2.linux-x64.tar.gz 
    2. tar -zxvf grafana-4.4.2.linux-x64.tar.gz 

* **Kibana**

    1. https://artifacts.elastic.co/downloads/kibana/kibana-5.5.1-linux-x86_64.tar.gz
    2. tar -zxvf kibana-5.5.1-linux-x86_64.tar.gz

* **

## Configuração do mongodb

* **Usuário mongod**

  1. useradd -d /usr/local/mongodb -g mongodb -r mongodb
  2. chown mongodb.mongodb mongodb/ -R

* **Execução do mongod**

  1. bin/mongod -dbpath data/db/ --directoryperdb > /tmp/mongodb.log 2<&1 &
  
## Configuração do logstash

* **Execução do logstash**
  
  1. bin/logstash -f "etc/pipeline-rundeck.conf"

* **Pipeline de configuração**

```
input {
	tcp {
		port => 6511
		type => 'rundeck'
	}
}

filter {
	json {
		source => message
	}
	ruby {
	code => "
		hash = event.to_hash
		hash.each do |k,v|
		if v == nil
			event.remove(k)
		end
		end
	"
	}
	mutate {
		convert => { "message" => "string" }
		remove_field => [
			"line", "event.stepctx"
		]
	}
	if[message] =~ /nil/ {
		mutate {
			remove_field => ["execution.id","execution.serverUrl", 
			"execution.group", "@timestamp", "port", "execution.executionType", 
			"execution.username", "execution.serverUUID", "@version", "host", 
			"execution.url","totallines", "execution.retryAttempt", "execution.wasRetry", 
			"execution.loglevel","execution.name", "line", "datetime", "event.step", "event.node", 
			"event.user", "execution.user.name", "eventType", "message", "execution.project", 
			"event.stepctx", "execution.execid", "loglevel"]
		}
	}
}

output {
	if[message] != "PLAY RECAP **************************
  *******************************************" and 
	[message] != "TASK [apache : Configuring Apache files] **********************
  *****************" and 
	[message] != "TASK [Gathering Facts] ***************************
  ******************************" and 
	[message] != "PLAY [all] ********************************
  *************************************" and 
	[message] != "" {
		stdout  { codec => rubydebug }
		mongodb {
			uri => "mongodb://localhost"
			database => "rundecklog"
			collection => "ansible"
		}
		elasticsearch {
			manage_template => true
			hosts => ["localhost:9200"]
			index => "rundecklog-%{+YYYY.MM.dd}"
		}
	}
}
```

## Modelo conector

* **mongodb doc sucesso**

```
{
        "_id" : ObjectId("59848657b959ec530700001b"),
        "execution.id" : "7ac894bd-3629-49e1-94cb-4dd449f7f050",
        "execution.name" : "Ansible Playbook",
        "type" : "rundeck",
        "datetime" : NumberLong("1501857383983"),
        "execution.serverUrl" : "http://172.20.150.95/",
        "execution.group" : null,
        "@version" : "1",
        "host" : "172.20.150.95",
        "event.step" : "1",
        "execution.wasRetry" : "false",
        "event.node" : "ansible-core",
        "event.user" : "root",
        "execution.user.name" : "admin",
        "eventType" : "log",
        "message" : "node-03: ok=2  changed=0  unreachable=0  failed=0",
        "execution.project" : "ansible-lab",
        "execution.execid" : "6049",
        "@timestamp" : "\"2017-08-04T14:36:07.680Z\"",
        "port" : 43880,
        "execution.executionType" : "scheduled",
        "execution.username" : "admin",
        "loglevel" : "NORMAL",
        "execution.serverUUID" : null,
        "execution.url" : "http://172.20.150.95/project/ansible-lab/execution/follow/6049",
        "execution.retryAttempt" : "0",
        "execution.loglevel" : "INFO"
}

        message: [node1, node2, node3]
```

* **mongodb doc falha**

```
{
        "_id" : ObjectId("5984f9ccb959ecd3a60000ae"),
        "execution.id" : "7ac894bd-3629-49e1-94cb-4dd449f7f050",
        "execution.name" : "Ansible Playbook",
        "execution.user.name" : "admin",
        "eventType" : "log",
        "message" : "Execution failed: 9005 in project ansible-lab: [Workflow result: , 
            step failures: {1=Dispatch failed on 1 nodes: [ansible-core: NonZeroResultCode: Remote command failed with 
            exit status 1]}, Node failures: {ansible-core=[NonZeroResultCode: Remote command failed with exit status 1]}, 
            status: failed]",
        "type" : "rundeck",
        "execution.project" : "ansible-lab",
        "execution.execid" : "9005",
        "datetime" : NumberLong("1501886940930"),
        "@timestamp" : "\"2017-08-04T22:48:44.905Z\"",
        "execution.serverUrl" : "http://172.20.150.95/",
        "port" : 50460,
        "execution.executionType" : "scheduled",
        "execution.username" : "admin",
        "loglevel" : "ERROR",
        "@version" : "1",
        "host" : "172.20.150.95",
        "execution.url" : "http://172.20.150.95/project/ansible-lab/execution/follow/9005",
        "execution.retryAttempt" : "0",
        "execution.wasRetry" : "false",
        "execution.loglevel" : "INFO"
}
```

## Topologia

![Produb](/home/gabriel/workspace_produban/documentacao_foguete/logo.png)
