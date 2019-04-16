To try and reproduce the issue in https://github.com/gocd/gocd/issues/6041

Get H2 jar from: https://repo1.maven.org/maven2/com/h2database/h2/1.3.168/h2-1.3.168.jar

Create a DB with 10000 runs of a pipeline:

```
./generate.rb 10000 >10000_runs.sql

java -cp h2-1.3.168.jar org.h2.tools.RunScript -url "jdbc:h2:cruise" -user sa -script 10000_runs.sql
```

Run queries against it:

```sql
$ java -cp h2-1.3.168.jar org.h2.tools.RunScript -url "jdbc:h2:./cruise" -user sa -script 01_nowhere.sql -showResults
SELECT pipelines.counter as pipelineCounter,
       stages.counter as stageCounter,
       stages.id as stageId,
       stages.rerunOfCounter,
       builds.id as buildId
  FROM stages
         JOIN pipelines ON pipelines.id = stages.pipelineId
         INNER JOIN builds ON stages.id = builds.stageId AND builds.ignored != true
 WHERE stages.id IN (
   SELECT
     id
     FROM _stages
    ORDER BY id DESC
    LIMIT 10 OFFSET 0
 )
 ORDER BY stages.id DESC;
--> 10000 1 10000 null 10000
--> 9999 1 9999 null 9999
--> 9998 1 9998 null 9998
--> 9997 1 9997 null 9997
--> 9996 1 9996 null 9996
--> 9995 1 9995 null 9995
--> 9994 1 9994 null 9994
--> 9993 1 9993 null 9993
--> 9992 1 9992 null 9992
--> 9991 1 9991 null 9991
;


$ java -cp h2-1.3.168.jar org.h2.tools.RunScript -url "jdbc:h2:./cruise" -user sa -script 02_original.sql -showResults
SELECT pipelines.counter as pipelineCounter,
       stages.counter as stageCounter,
       stages.id as stageId,
       stages.rerunOfCounter,
       builds.id as buildId
  FROM stages
         JOIN pipelines ON pipelines.id = stages.pipelineId
         INNER JOIN builds ON stages.id = builds.stageId AND builds.ignored != true
 WHERE stages.id IN (
   SELECT
     id
     FROM _stages
    WHERE name = 'ExtractCIcomponents'
      AND pipelineName = 'CreditDriver-Build'
    ORDER BY id DESC
    LIMIT 10 OFFSET 0
 )
 ORDER BY stages.id DESC
;
--> 10000 1 10000 null 10000
--> 9999 1 9999 null 9999
--> 9998 1 9998 null 9998
--> 9997 1 9997 null 9997
--> 9996 1 9996 null 9996
--> 9995 1 9995 null 9995
--> 9994 1 9994 null 9994
--> 9993 1 9993 null 9993
--> 9992 1 9992 null 9992
--> 9991 1 9991 null 9991
;


$ java -cp h2-1.3.168.jar org.h2.tools.RunScript -url "jdbc:h2:./cruise" -user sa -script 03_inner.sql -showResults
   SELECT
     id
     FROM _stages
    WHERE name = 'ExtractCIcomponents'
      AND pipelineName = 'CreditDriver-Build'
    ORDER BY id DESC
    LIMIT 10 OFFSET 0;
--> 10000
--> 9999
--> 9998
--> 9997
--> 9996
--> 9995
--> 9994
--> 9993
--> 9992
--> 9991
```

A GoCD config XML which works with this, so that you can see this pipeline:

```xml
<?xml version="1.0" encoding="utf-8"?>
<cruise xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance" xsi:noNamespaceSchemaLocation="cruise-config.xsd" schemaVersion="115">
  <server artifactsdir="artifacts" agentAutoRegisterKey="9876fc1a-c7ca-4e42-b003-68ccdb022cfd" webhookSecret="85ffa779-2e3a-4c0b-9eb8-7dc0a13c6ef1" commandRepositoryLocation="default" serverId="f35f10de-3528-44bb-bf9b-8217829a56fd" tokenGenerationKey="206ac2bd-5329-4c5f-963e-66cf52f39532">
    <backup emailOnSuccess="true" emailOnFailure="true" />
  </server>
  <pipelines group="PG1">
    <pipeline name="CreditDriver-Build">
      <materials>
        <git url="https://github.com/arvindsv/faketime.git" />
      </materials>
      <stage name="ExtractCIcomponents">
        <jobs>
          <job name="defaultJob">
            <tasks>
              <exec command="ls" />
            </tasks>
          </job>
        </jobs>
      </stage>
    </pipeline>
  </pipelines>
</cruise>
```