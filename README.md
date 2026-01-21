# coco-skill-batch-ingestion

% export PROJECT_HOME=/Users/kentontroydavis/Documents/development/cortex/projects

% cortex skill add ${PROJECT_HOME}/openflow-watermark-batch-ingestion
```
A new version (v0.26.113+005340.e3aef489) is available. Run 'cortex update' to upgrade.
Detected skills: openflow-watermark-batch-ingestion
Success: Added skill directory: /Users/kentontroydavis/Documents/development/cortex/projects/openflow-watermark-batch-ingestion
```

To see how the Skill was created using CoCo, go to [Batch Ingestion w. Watermarking](./development.md).

![Skill visible in CoCo](./images/watermarking-skill-loading.png)

For an example of using the Skill in CoCo to build an Ingestion pipeline, using the following prompt
```
Please help me create an Openflow pipeline that performs change data capture via batch ingestion.
```

You can tell CoCo to create placeholders instead of specifying every parameter value. 
See example flow w/ placeholders. [Output with Placeholders](./output/sqlserver-timestamp-cdc-flow.json)

For examples and an overivew on strategies, go to [Examples](./openflow-watermark-batch-ingestion/examples.md)


