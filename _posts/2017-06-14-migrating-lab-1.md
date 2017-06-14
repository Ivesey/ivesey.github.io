---
layout: post
title:  Bash scripts for Lab 1 of Migrating to AWS
subtitle: A more bash-y approach
date:   2017-06-14
category: training
tags: [labguides, migrating, ads]
---

### Copy-and-pastable bash scripts
#### No warranties express or implied...

1.2.3:  

``
AGENTS=$(aws discovery describe-agents --query agentsInfo[?health==\`HEALTHY\`].agentId --output text)
``

1.2.4:

``
aws discovery start-data-collection-by-agent-ids --agent-ids $AGENTS
``

1.2.5: Status of agents

``
aws discovery describe-agents --query agentsInfo[?health!=\`UNKNOWN\`].[agentId,health] --output text
``

(Easier to read the output)

1.3.4:

``
aws discovery list-configurations --configuration-type PROCESS --filters name="process.name",values="nginx",condition="CONTAINS"
``

``
NGINX=$(aws discovery list-configurations --configuration-type PROCESS --filters name="process.name",values="nginx",condition="CONTAINS" --query configurations[].\"server.agentId\" --output text) | sed "s/ /,/g"
``

1.3.5:

``
aws discovery list-configurations --configuration-type PROCESS --filters name="server.agentId",values=$NGINX,condition="CONTAINS"
``

1.3.6:

``
MYSQL=$(aws discovery list-configurations --configuration-type PROCESS --filters name="process.name",values="mysqld",condition="CONTAINS" --query configurations[].\"server.configurationId\" --output text)
``

1.3.7:

But they told us to get AgentIds, not config ids!

``
NGINXIDS=$(aws discovery list-configurations --configuration-type PROCESS --filters name="process.name",values="nginx",condition="CONTAINS" --query configurations[].\"server.configurationId\" --output text)
``

``
aws discovery create-tags --tags key=webserver,value=nginx --configuration-ids $NGINXIDS
``
