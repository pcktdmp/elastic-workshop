# elastic-workshop
A repository that can be used for Elastic Cloud workshops.

# Required knowledge

* You have basic knowledge how to filter data with computer systems;
* You have basic knowledge how to work with RESTful API's;
* You know how to run Docker containers.

# Workshop

## Preparation

Install Docker

https://docs.docker.com/get-docker/

## Engineer


### Part 0: Workshop Notes

You also got a student name, Elastic Cloud URL's and credentials assigned in the beginning of the workshop.
Please use the student name for `<student_name>` when being refenced by inside this workshop.

### Part 1: Start an application that sends logs to Elastic Cloud

Open up a separate terminal and run:

`docker run pcktdmp/logging-container:latest`

It should return with an error similar to the following:

```
2020/11/08 13:45:19 {"message" : "default-app;ERROR->ExecutionID:48550a38-dbea-442d-8615-8eda9101acbb@1604843119"}
2020/11/08 13:45:19 Error getting response: unsupported protocol scheme ""
```

This is expected since we need to configure the application to send its logs to our Elastic Cloud cluster, please replace the passed environment variables with received values from the instructor.

```
docker run -ti -e ES_CUSTOM_APP_NAME=<name_you_come_up_with> -e ES_CUSTOM_INDEX_NAME=<student_name> -e ES_USERNAME=user -e ES_PASSWORD=password -e ES_URL=https://example.com pcktdmp/logging-container
```

If everything is correctly configured you should see output on your terminal similar to the following:

```
2020/11/08 14:00:51 {"message" : "coolapp;UNKNOWN->ExecutionID:3b105c04-8b11-4690-8646-bb18b4d1e77a@1604844051"}
2020/11/08 14:00:52 [400 Bad Request] Error indexing document
2020/11/08 14:00:56 {"message" : "coolapp;ERROR->ExecutionID:92f4e2e3-1b17-45bf-a43d-22ec611c055a@1604844056"}
2020/11/08 14:00:56 [400 Bad Request] Error indexing document
```

This tells us that the application is able to send log data to Elastic Cloud correctly, but sadly something at the processing side seems to go wrong. This is occurs because we are trying to send to an "ingest pipeline" which has not been created yet.

### Part 2: Setup an initial ingest pipeline

Go to the Elastic Cloud deployment URL that the instructor has provided and login via "Log in with Elasticsearch" with the also provided credentials.

Click in the top left "hamburger" icon and go to "Dev Tools".

You see the "Dev Console" in front of you which we use to send API calls to the Elastic Cloud deployment. The Dev Tools are you going to use heavily in this workshop.

Going back to our outstanding error `[400 Bad Request] Error indexing document`; this is caused by the fact we don't have an "ingest pipeline" yet for our application.

Apply the following API call to the Elastic Search cluster to create a skeleton pipeline for your application by pasting this in the API call input field and click the "play" button in the corner of the field:

```
PUT _ingest/pipeline/<student_name>
{
  "description" : "This is a skeleton pipeline",
  "processors" : [
    {
      "set" : {
        "field": "foo",
        "value": "bar"
      }
    }
  ]
}
```

If everything went fine you should see in the output field something like:

```
{
  "acknowledged" : true
}
```

If everything went also here fine the error in your terminal should have disappeared:

```
2020/11/08 13:50:58 {"message" : "coolapp;WARN->ExecutionID:51fb8d31-d264-485e-bbca-24e9ce376e5e@1604843458"}
2020/11/08 13:51:03 {"message" : "coolapp;INFO->ExecutionID:8f5f3b75-3205-4bb5-a01f-f248bd7d07ba@1604843463"}
2020/11/08 13:51:08 {"message" : "coolapp;INFO->ExecutionID:b484de4b-523b-456b-995e-2bbe80052f47@1604843468"}
```

This tells us the application is sending its log data to Elastic Cloud correctly.
As you might suspect the value of `message` in this example is the data we are going to dissect in the next part of the workshop.

### Part 3: Create an index pattern and search your data

So now we have Elastic Cloud made ingest our data through a pipeline. Great.
But this is not much of use when you can't search it. We are going to solve with creating an initial "index pattern".

Go again to the "hamburger icon" in the top left and navigate to `Stack Management` and then search for `Index Patterns`.

Click on `Create Index Pattern`.

Fill in `<student_name>` for the `Index pattern name` field and click `Next step` and finally on `Create index pattern`.

Because we now have created an "index pattern" we can now search our ingested data by navigating via the "hamburger icon" to `Discover`. In the dropdown where `kibana_sample_data_logs` is selected by default select `<student_name>` as your index pattern.

If the stars are properly aligned you should now see your data being ingested into the index called `<student_name>` and you can search the data set like your favorite search engine.

Take your time to explore the possibilities that `Discover` has to bring you by switching back and forth between `kibana_sample_data_logs` and `<student_name>` as your index patterns.

As you might see during your exploration the `kibana_sample_data_logs` are nicely structured and easily searchable
by matching on key-value pairs and so on. Our data is not (yet).

### Part 4: Dissecting our own application

Because we want to be just as fancy as `kibana_sample_data_logs` we are going to dissect our (rather funky) log format also into key-value pairs by using so called "Grok" expressions.

Please take your time to study the following blog post written by Elastic Co. so get familiar with the concept:

https://www.elastic.co/blog/do-you-grok-grok

To trigger Grok's magic you need to use it as a processor in your ingest pipeline.
Apply the following basic Grok expression to your pipeline with the Dev Tools:

```
PUT _ingest/pipeline/<student_name>
{
  "description" : "this pipeline simply catches all data from the 'message' field and puts it in 'newfield'",
  "processors": [
    {
      "grok": {
        "field": "message",
        "patterns": ["%{GREEDYDATA:newfield}"]
      }
    }
  ]
}
```

When looking at `Discover` you see that the log data that is being indexed is without a timestamp. How inconvenient.

As you might have seen on `stdout` of your container behind the `@` sign there is something that looks like an epoch timestamp. It actually is.

`coolapp;ERROR->ExecutionID:298dca7a-e25a-4739-ae35-f3055e6da3d6@1604846576`

Lets use the timestamp from the application as a timestamp inside Elastic Cloud to make the data searchable based on timeranges by applying the following ingest pipeline configuration via the Dev Tools:

```
PUT _ingest/pipeline/<student_name>
{
  "description" : "with this pipeline configuration we can use the timestamp from the log message as searchable timestamp inside Elastic Cloud",
  "processors": [
   {
        "grok": {
          "field": "message",
          "patterns": ["%{DATA}@%{INT:logatime}"]
        }
      },
      {
        "date" : {
          "field" : "logatime",
          "target_field" : "@timestamp",
          "formats": ["UNIX"]
        }
      }
  ]
}
```

To activate this `@timestamp`-mechanism we need to re-create our earlier created index pattern by going via the "hamburger icon" to `Stack Management`, `Index Patterns`, click on your earlier created index pattern called `<student_name>` and delete it via the "trash can" icon.

Follow the same steps as in Part 3 but you will now be prompted for `Select a primary time field for use with the global time filter.`.
Select here the `@timestamp` field and click `Create index pattern`.
If you now go to `Discover` you will be pleased with a timeline of your log data in addition to the search functionality.

### Part 5: Grok expression development

Now you are going to develop yourself a Grok Expression within the pipeline.

Assignment: For each field in the RAW log message create a key-value pair via Grok expressions.

Use the following documentation to achieve this goal:

* https://www.elastic.co/guide/en/logstash/current/plugins-filters-grok.html
* https://github.com/logstash-plugins/logstash-patterns-core/blob/master/patterns/grok-patterns
* https://www.elastic.co/guide/en/kibana/current/xpack-grokdebugger.html
* https://www.elastic.co/guide/en/kibana/current/console-kibana.html

Expected keys with corresponding values inside Elastic Cloud:

* `appname`
* `loglevel`
* `execution_id`

Hint: you can use the built-in ingest pipeline simulator in advance before actually applying your pipeline configuration.

Example ingest pipeline simulation:

```
POST _ingest/pipeline/_simulate
{
  "pipeline": {
    "description" : "this is a simulation",
    "processors": [
      {
        "grok": {
          "field": "message",
          "patterns": ["%{DATA}@%{INT:logatime}"]
        }
      },
      {
        "date" : {
          "field" : "logatime",
          "target_field" : "@timestamp",
          "formats": ["UNIX"]
        }
      }
    ]
  },
  "docs":[
    {
      "_source": {
        "message": "coolapp;UNKNOWN->ExecutionID:7b428a0b-1db7-4911-97c5-e156f2039f25@1604836406"
      }
    }
  ]
}
```

